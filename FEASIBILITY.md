# FEASIBILITY.md — IDEで実装できるか？

> IDEに搭載されたLLM（Cursor / Windsurf / GitHub Copilot 等）が  
> このシステムを実装できるかを判断するための材料。

---

## 結論：できる

Claudeが動作確認済みの設計をもとに、SPEC.mdに仕様を明文化した。  
IDEのLLMが読めば、追加調査なしで実装を開始できる水準に整理されている。

---

## 難易度評価（モジュール別）

| モジュール | 難易度 | 理由 |
|-----------|------|------|
| `db.py` | ★☆☆ | 標準ライブラリのSQLite。スキーマはSPEC.mdに全記載 |
| `collector.py` | ★★☆ | GitHub Search APIのクエリ設計が肝。パターンはSPEC.mdに明記 |
| `scorer.py` | ★☆☆ | 数式3つ。コアロジックはSPEC.mdにそのまま書いてある |
| `filter.py` | ★☆☆ | 閾値チェックのみ |
| `reporter.py` | ★☆☆ | f文字列でMarkdown生成。フォーマットはSPEC.mdに記載 |
| `main.py` | ★☆☆ | 上記を順番に呼ぶだけ |
| `config.yaml` | ★☆☆ | SPEC.mdに構造を記載済み |

**最大の難所は `collector.py`**。  
GitHub Search APIのrate limit対策（sleep / TOKEN設定）と  
READMEの取得タイミング（全件ではなく上位N件のみ）を正しく実装できるかどうか。

---

## IDEへの推奨実装順

```
1. db.py        ← まずここ。依存なし、テストしやすい
2. collector.py ← GitHub APIと通信。最初は obsidian-plugin 1トピックで確認
3. scorer.py    ← db.pyに依存。スナップショット不足時の fallback を忘れずに
4. reporter.py  ← scorer.pyの出力を受け取るだけ
5. filter.py    ← scorer.pyの後段。閾値はconfig.yamlから読む
6. main.py      ← 全部を繋ぐ
```

---

## 最初のマイルストーン（IDEへの指示）

```
python main.py --genre pkm を実行して
reports/本日付.md が生成されれば成功。
初回はaccelerationが機能しない（スナップショット不足）が正常動作。
```

---

## 既知の制約と対処

### GITHUB_TOKENなしの場合

Search APIは無認証でも動くが、一部クエリが403になる。  
`obsidian` `obsidian-plugin` `note-taking` タグは無認証で通過確認済み。  
フル稼働にはPersonal Access Token（read:public_repo権限のみ）が必要。

取得方法:  
GitHub → Settings → Developer settings → Personal access tokens → Generate new token (classic)  
権限: `public_repo` にチェックを入れるだけ。

### rate limit

- 無認証: 10 req/min
- TOKEN有: 5,000 req/hour

`time.sleep(0.5)` を各リクエスト間に挟む実装で対処済み（Claudeが動作確認）。

### 初回〜14日間

`acceleration` はスナップショットが2週間分溜まって初めて機能する。  
それまでは `freshness` と `quality` のみでランキングされる。  
これは仕様であり、バグではない。

---

## 拡張ロードマップ（実装後）

| フェーズ | 内容 | 難易度 |
|---------|------|------|
| v0.2 | メディア生成系・パイプライン接続系を追加 | ★☆☆（config.yamlにtopic追加のみ） |
| v0.3 | Anthropic APIで「なぜ注目か」を自動要約 | ★★☆ |
| v1.0 | 週次サマリー → note.com記事の素材生成 | ★★★ |

---

## IDEへの一言

> このドキュメント群（README / SPEC / EXPERIMENTS / FEASIBILITY）を読んだ上で実装してください。  
> Claudeが動作確認した設計なので、SPEC.mdの仕様を忠実に実装すれば動きます。  
> 最初は `--genre pkm` 一本で始めてください。
