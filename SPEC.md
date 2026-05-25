# SPEC.md — 要求仕様書

> IDEに搭載されたLLMへの実装依頼書。  
> このファイルを読んで、実装を開始してください。

---

## 1. システムの目的

GitHubから、**「生まれて日が浅く、かつ先週より今週のほうがスターが伸びている」** リポジトリを自動収集し、Markdownレポートとして出力する。

キーワード：**加速度・若さ・内容密度**

---

## 2. 実行環境

- Python 3.11+
- ローカルスクリプト（cronまたはscheduleライブラリで定期実行）
- 外部ライブラリ: `requests` `python-dotenv` `pyyaml` のみ
- DB: SQLite（標準ライブラリ `sqlite3`）
- APIキー: 環境変数 `GITHUB_TOKEN`（`.env`ファイルから読む）

---

## 3. ファイル構成（実装対象）

```
tool-tide/
├── .env                  # GITHUB_TOKEN=xxx
├── config.yaml           # フィルタ設定・ジャンルタグ
├── main.py               # エントリポイント
├── collector.py          # GitHub Search API → データ収集
├── db.py                 # SQLite スナップショット管理
├── scorer.py             # スコアリング（3軸）
├── filter.py             # 閾値フィルタ
├── reporter.py           # Markdown出力
└── reports/              # 出力先ディレクトリ
    └── YYYY-MM-DD.md
```

---

## 4. データ構造

```python
@dataclass
class RawItem:
    id:          str        # "github:{owner}/{repo}"
    genre:       str        # "media" | "pkm" | "pipeline"
    name:        str        # "{owner}/{repo}"
    url:         str
    description: str
    created_at:  datetime
    fetched_at:  datetime
    stars:       int
    language:    str
    topics:      list[str]
    readme:      str        # 先頭3000文字のみ取得

@dataclass
class ScoredItem:
    raw:          RawItem
    acceleration: float     # 今週増加 ÷ 先週増加（倍率）
    freshness:    float     # 0.0〜1.0（若いほど高い）
    quality:      float     # 0.0〜1.0（README密度）
    total_score:  float
    signals:      list[str] # 根拠メモ（例: "🌱 生後5日", "🚀 先週比4.2倍"）
```

---

## 5. スコアリング仕様

### acceleration（重み 45%）

```python
stars_this_week = snap_now   - snap_7d_ago
stars_last_week = snap_7d_ago - snap_14d_ago
acceleration    = stars_this_week / max(stars_last_week, 1)

# スナップショット不足時（初回）は acceleration = 1.0 として扱う
```

### freshness（重み 30%）

```python
freshness = math.exp(-age_days / 30)
# 0日=1.00 / 7日=0.79 / 30日=0.37 / 90日=0.05
```

### quality — READMEの密度（重み 25%）

以下5項目の充足率（0.0〜1.0）：

- コードブロック（` ``` `）の存在
- インストール手順のキーワード（install / pip / npm / brew / cargo）
- 使用例のキーワード（usage / example / quickstart / demo）
- バッジ（`![`）の存在
- ライセンス記載

### 総合スコア

```python
total_score = (
    min(acceleration / 5.0, 1.0) * 0.45 +
    freshness                     * 0.30 +
    quality                       * 0.25
)
```

---

## 6. GitHub収集クエリ仕様

```python
# ジャンルごとのtopicタグは config.yaml で管理
# 以下の2パターンを各topicに対して実行

f"topic:{topic} stars:5..500  created:>{7日前の日付}"   # 新星
f"topic:{topic} stars:30..3000 created:>{30日前の日付}" # 加速中
# sort=stars&order=desc
```

---

## 7. フィルタ仕様

| 条件 | 値 |
|------|-----|
| 最大スター数 | 10,000（有名株除外） |
| 最大日齢 | 180日 |
| 最低スコア | 0.40（初回は緩め） |
| 除外トピック | awesome-list, tutorial, course |

---

## 8. SQLite スキーマ

```sql
CREATE TABLE IF NOT EXISTS snapshots (
    id         TEXT,
    genre      TEXT,
    name       TEXT,
    fetched_at TEXT,   -- ISO8601
    stars      INTEGER,
    PRIMARY KEY (id, fetched_at)
);
```

差分取得：

```sql
SELECT stars FROM snapshots
WHERE id = ? AND fetched_at <= ?
ORDER BY fetched_at DESC LIMIT 1;
```

---

## 9. 出力形式（Markdownレポート）

```
# tool-tide — YYYY-MM-DD

## 🧠 知識管理系（PKM）

### 1. [owner/repo](url)  ★ {stars}  [生後{N}日]
> {description}

- {signals をスペース区切りで並べる}
- 言語: {language} | score: {total_score:.2f}
- acceleration: {:.2f}x | freshness: {:.2f} | quality: {:.2f}

---
```

---

## 10. config.yaml 構造

```yaml
filter:
  min_total_score: 0.40
  max_total_stars: 10000
  max_age_days: 180

output:
  top_n: 15
  report_dir: "./reports"

genres:
  pkm:
    topics:
      - obsidian-plugin
      - obsidian
      - note-taking
      - pkm
      - personal-knowledge-management
      - second-brain
      - zettelkasten
      - knowledge-graph
      - local-first
      - logseq
      - markdown-editor
  media:
    topics:
      - video-generation
      - remotion
      - ffmpeg
      - manim
      - animation
      - text-to-video
      - audio-generation
      - media-processing
  pipeline:
    topics:
      - cli
      - automation
      - workflow
      - file-converter
      - format-converter
      - data-pipeline
      - etl
      - template-engine
      - pdf-generator
```

---

## 11. 実装優先順位

1. `db.py` — SQLite wrapper（init / save / get_stars_ago）
2. `collector.py` — GitHub Search API → `RawItem[]`（READMEはスコア上位のみ取得）
3. `scorer.py` — 3軸スコアリング → `ScoredItem[]`
4. `reporter.py` — Markdown生成
5. `main.py` — 上記を繋ぐ。`python main.py --genre pkm` から始める

**最初のマイルストーン**: `python main.py --genre pkm` を実行し、`reports/本日付.md` が生成されること。

---

## 12. 注意事項

- エラーは例外を投げず、ログ出力してスキップする（1件の失敗で全体を止めない）
- READMEの取得はrate limit節約のため、スコア上位15件のみに限定する
- 型ヒントを全関数につける
- 初回実行時はスナップショット不足のため `acceleration = 1.0`（中立値）として扱い、翌日以降に差分計算が機能し始める
