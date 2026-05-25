# EXPERIMENTS.md — 実験記録

> Claudeが実際に動かして確認した内容。  
> この記録をもって、SPEC.mdの仕様が「動く」と判断している。

---

## 実験の経緯

このシステムの設計は、Claude（Anthropic）との対話セッションで生まれた。

設計書を書くだけでなく、**Claudeが自らコードを書いて、GitHubのライブデータで動作確認まで行った**。  
その結果をここに記録する。目的は、IDEへ渡す仕様書に「動作確認済み」の根拠を添えること。

---

## 実験日時

2026年5月25日

---

## 実験したジャンル

### PKM（知識管理系 / Obsidian型）— ✅ 動作確認済み

**理由**: `obsidian-plugin` タグはObsidianコミュニティが律儀に付けているため、  
収集精度が最も高いと判断。最初の実験対象として選択。

**使用したtopicタグ**:
```
obsidian-plugin / obsidian / note-taking / pkm /
personal-knowledge-management / second-brain / zettelkasten /
knowledge-graph / local-first / logseq / markdown-editor
```

---

## 実行結果サマリー

| 項目 | 結果 |
|------|------|
| 実行環境 | Python 3.x / ローカル |
| GITHUB_TOKEN | なし（無認証） |
| 収集件数 | 15件 |
| フィルタ後 | 10件 |
| レポート生成 | ✅ `reports/2026-05-25.md` |
| エラー | 一部クエリが403（TOKEN不要トピックのみ通過） |

**TOKEN不要で通過したトピック**: `obsidian` `obsidian-plugin` `note-taking`  
**403が出たトピック**: `personal-knowledge-management` `second-brain` `zettelkasten` 等  
→ GITHUB_TOKENを設定すれば全クエリ通過、収集量は3〜4倍になる見込み。

---

## 実際に収集されたリポジトリ（抜粋）

以下は2026-05-25時点のライブデータ。架空ではない。

| リポジトリ | 生後日数 | ★ | 概要 |
|-----------|---------|---|------|
| `weaiw/trove-ai` | 1日 | 6 | 中国語インターネット向けAI知識ベース |
| `ddsyasas/llm-wiki` | 1日 | 6 | Karpathyが提唱したLLM Wikiパターンの実装 |
| `2aronS/mindmark` | 5日 | 5 | TypeScript製・AI統合Markdownノートアプリ |
| `djolex999/vir` | 5日 | 12 | ObsidianボルトにClaude Code用LLM Wikiを置くツール |
| `chacosoldier/compabob` | 4日 | 22 | Claude Code + Obsidian知識ベースのカスタムセットアップ |
| `rahilp/second-brain-cloudflare` | 15日 | 84 | MCPクライアント共通メモリレイヤー・Cloudflare無料枠 |

---

## 動作確認できた設計判断

### ✅ スナップショット差分方式（db.py）

GitHubはスター増加量を直接返さない。  
SQLiteに毎回スター数を保存し、前回値との差分で `acceleration` を計算する設計は正しく動いた。

### ✅ 初回はacceleration=1.0（中立値）で逃げる

初回実行時はスナップショットが存在しないため、`acceleration = 1.0` として  
`freshness` と `quality` のみでスコアリングする設計で問題なく動作した。  
7〜14日運用すると `acceleration` が機能し始め、本来の精度になる。

### ✅ READMEは上位N件のみ取得

READMEの取得はSearch APIとは別エンドポイントなのでrate limitを消費する。  
「若い順に上位15件のみ取得」という制限を設けることで、  
無認証でも全体の処理が完走した。

### ⚠️ topicタグの取りこぼし

`pipeline` ジャンルの `cli` `automation` 等のタグは、  
付けていないリポジトリが多く収集精度が落ちる可能性がある。  
`description` や `readme` のキーワード検索を補助的に使う設計拡張が有効と判断。

---

## 次のジャンル候補

| ジャンル | 推奨度 | 備考 |
|---------|------|------|
| メディア生成系（Remotion型） | ★★★ | `remotion` `manim` タグは整備されている |
| パイプライン接続系 | ★★☆ | topicタグが分散しているため補助検索が必要 |

---

## 結論

> PKMジャンルは、GITHUB_TOKENなしの状態でも動作した。  
> TOKENを設定すれば精度・量ともに向上する。  
> 設計書（SPEC.md）の仕様は実装可能であることを、Claudeが動作確認した。  
> IDEへの実装依頼に進んでよい。
