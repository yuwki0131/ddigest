# DDigest - Personal Tech Digest Generator（MVP設計ドラフト）

## 概要

本プロジェクトは **自分専用の技術情報ダイジェスト生成CLIツール** である。

RSS / Atom などの技術情報ソースを定期的に取得し、以下を行う。

- 新規記事・更新記事のみを検出
- 自分の技術スタックに対する「影響度」でスコアリング
- READ NOW / LATER / IGNORE に分類
- **READ NOW のみ** LLM（ChatGPT 等）で要約

目的は「技術ニュースを読むこと」ではなく、

> **技術の変化から、今は無視してよいものを判断すること**

にある。

---

## 想定利用シーン

### 1. 毎日のクイックチェック（約1分）

```bash
ddigest fetch
ddigest score
ddigest today
```

- READ NOW が 0 件なら、何も読まない
- 数件のみなら即読む or later に送る

### 2. 環境更新前の事前確認

以下を実行する前に READ NOW を確認する。

- `nix flake update`
- `cargo update`
- Docker image 更新

破壊的変更・セキュリティ影響を事前に把握することが目的。

---

## MVPのゴール

- RSS / Atom を取得しローカルDBに保存
- 新規・更新記事の検出（差分管理）
- 重要度スコアリングと分類
- READ NOW のみ LLM 要約
- CLIで完結（Web UIなし）

---

## CLI設計（MVP）

```bash
ddigest fetch          # RSS取得・更新検出
ddigest score          # スコアリングと分類
ddigest today          # 今日のダイジェスト表示
ddigest show <id>      # 要約と判定理由を表示
ddigest open <id>      # ブラウザで開く
ddigest mark <id>      # ignore | later | done
ddigest queue          # later一覧
```

オプション：

```bash
ddigest today --read-now
ddigest today --json
```

---

## 出力イメージ

```
2025-12-31 | sources: 18 | new items: 42 | actionable: 4

🔴 READ NOW
1) nixpkgs: OpenSSL ABI change
   - Why you: flake.lock が nixpkgs を使用
   - Action: 更新延期 or openssl を pin

🟡 LATER
2) Rust Cargo resolver note

⚪ IGNORE
- 38 items
```

---

## 全体構成

```
RSS / Atom
  ↓
Fetcher
  ↓
Normalizer
  ↓
SQLite Store
  ↓
Scorer（ルールベース）
  ↓
Summarizer（LLM, read_now のみ）
  ↓
CLI Output
```

---

## 設定ファイル（ddigest.yaml）

```yaml
db_path: "~/.local/share/ddigest/ddigest.sqlite"

llm:
  provider: "openai"
  model: "gpt-4.1-mini"
  api_key_env: "OPENAI_API_KEY"
  max_items_per_run: 6
  summarize_only_read_now: true

fetch:
  user_agent: "ddigest/0.1"
  timeout_seconds: 15
  fetch_article_body: true
  article_body_max_chars: 20000

sources:
  - name: "NixOS Discourse"
    url: "https://discourse.nixos.org/latest.rss"
    tags: ["nix"]

scoring:
  read_now_threshold: 80
  later_threshold: 45

  boost_keywords:
    - pattern: "(CVE-|security|vulnerability|RCE)"
      score: 90
    - pattern: "(breaking change|deprecat(ed|ion)|ABI)"
      score: 70

  penalize_keywords:
    - pattern: "(React|Next\.js|frontend)"
      score: -30
```

---

## DB設計（SQLite）

### sources
- id
- name
- url
- tags (json)
- last_fetched_at

### items
- id
- source_id
- url（unique）
- title
- author
- published_at
- fetched_at
- content_text
- content_hash
- status（new / read_now / later / ignore / done）
- score
- reasons（json）
- summary
- why_you

### item_events（任意）
- item_id
- event_type
- at
- payload

---

## 処理フロー

### digest fetch
- RSS取得
- URL単位で既存チェック
- 新規 or 更新を検出
- 本文取得・抽出
- status = new

### digest score
- ルールベースでスコア算出
- 閾値で分類
- reasons を保存

### summarize
- read_now のみ
- 最大N件
- summary / why_you / recommended_action を生成

---

## LLM出力形式（JSON）

```json
{
  "summary": "数行の要約",
  "why_you": [
    "自分の環境に影響",
    "破壊的変更の可能性"
  ],
  "recommended_action": "wait | update now | investigate | ignore",
  "confidence": 0.0
}
```

---

## 実装方針

- スコアリングはルール主導、LLMは補助
- 状態（既読・無視）を必ず永続化
- 差分検出を最重要視
- LLM呼び出しは最小限

---

## MVPでやらないこと

- Web UI
- 全件LLM要約
- 学習ベースの最適化

---

## 将来拡張

- lockfile等からの asset scan
- TUI（ratatui 等）
- systemd timer / cron
- 通知連携

---

## 設計思想（再確認）

このツールは「技術ニュースを読む」ためのものではない。
**技術の変化から、今は無視してよいものを決めるためのツール**である。
