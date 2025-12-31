# Architecture Decision Records

このドキュメントは、DDigest プロジェクトにおける主要なアーキテクチャ判断を記録します。

---

## ADR-001: 実装言語として Rust を採用

**Status**: Accepted

**Context**:

DDigest は個人用の CLI ツールであり、以下の特性が求められる：
- 毎日使用するため、起動速度とパフォーマンスが重要
- シングルバイナリで配布できることが望ましい
- RSS パース、HTTP リクエスト、DB 操作、LLM API 呼び出しなど多様な処理が必要
- 型安全性によるバグの早期発見

**Decision**:

実装言語として **Rust** を採用する。

**Consequences**:

**良い点**:
- 高速な実行速度とメモリ効率（起動時間の最小化）
- メモリ安全性と型安全性（バグの早期発見）
- シングルバイナリでの配布が容易（依存関係の管理不要）
- 豊富なエコシステム（clap, tokio, sqlx, reqwest など）
- クロスコンパイルによる複数プラットフォーム対応

**悪い点**:
- 学習曲線が急（所有権システム、ライフタイム）
- コンパイル時間が長い（開発イテレーション速度への影響）
- 開発初期は Python などと比較して実装速度が遅い

**Alternatives**:

- **Python**: 開発速度が速く、LLM 統合ライブラリが豊富。ただし配布が複雑（venv, pyinstaller）、起動速度が遅い
- **Go**: シンプルな言語仕様、並行処理が得意。ただしエコシステムが Rust より小さい

---

## ADR-002: LLM プロバイダーは OpenAI のみサポート

**Status**: Accepted

**Context**:

LLM 統合は DDigest のコア機能の一つであり、要約生成に使用される。複数のプロバイダーをサポートすることは柔軟性を提供するが、MVP では以下を考慮する必要がある：
- 実装の複雑性
- テストの容易さ
- API の安定性
- コスト

**Decision**:

MVP では **OpenAI API のみ**をサポートする（GPT-4, GPT-3.5, GPT-4o-mini など）。

**Consequences**:

**良い点**:
- 実装がシンプル（単一の API クライアント）
- API が安定しており、ドキュメントが充実
- 広く使用されているため、トラブルシューティングが容易
- MVP のスコープを絞ることで開発速度向上

**悪い点**:
- ベンダーロックイン（OpenAI の価格変更、API 変更に依存）
- コストが固定（ローカル LLM と比較して高コスト）
- プライバシー懸念（コンテンツを外部 API に送信）

**Alternatives**:

- **Anthropic Claude**: 高品質な出力、長いコンテキスト。ただし API の成熟度で OpenAI に劣る
- **ローカル LLM（Ollama など）**: コスト削減、プライバシー保護。ただし品質が低く、セットアップが複雑
- **複数プロバイダー対応**: 柔軟性は高いが、MVP では実装コストが高すぎる

**Future Considerations**:

MVP 後に抽象化レイヤーを導入し、他のプロバイダーをサポートすることを検討する。

---

## ADR-003: RSS フィードからの本文取得戦略

**Status**: Accepted

**Context**:

RSS フィードの `description` フィールドには、記事の全文が含まれていないことが多い（要約のみ、または冒頭部分のみ）。スコアリングの精度を高めるためには、記事の全文を取得することが望ましい。ただし、全文取得には以下のトレードオフがある：
- フェッチ時間の増加
- サーバーへの負荷
- パース失敗のリスク

**Decision**:

設定ファイルで `fetch_article_body: true` が設定されている場合、以下の戦略で本文を取得する：
1. RSS フィードの `link` から HTTP リクエストで HTML を取得
2. **readability 系ライブラリ**（Mozilla Readability の Rust 実装）でメインコンテンツを抽出
3. 抽出した本文を `content_text` として保存

**Consequences**:

**良い点**:
- スコアリング精度の向上（キーワードマッチの精度向上）
- LLM 要約の質向上（全文コンテキストによる要約）
- 柔軟性（設定で ON/OFF 切り替え可能）

**悪い点**:
- フェッチ時間の増加（HTTP リクエスト + HTML パース）
- サーバー負荷（RSS + 本文の 2 回リクエスト）
- パース失敗のリスク（readability が対応していないサイト）

**Alternatives**:

- **description のみ使用**: 高速だがスコアリング精度が低下
- **常に本文取得**: 設定不要だが柔軟性が低下
- **スクレイピング**: サイトごとのカスタムパース。精度は高いがメンテナンスコストが高い

**Implementation Notes**:

- タイムアウトを設定（`fetch.timeout_seconds`）
- 本文取得失敗時は `description` にフォールバック
- 最大文字数制限（`fetch.article_body_max_chars`）

---

## ADR-004: スコアリングアルゴリズムの設計

**Status**: Accepted

**Context**:

スコアリングは DDigest の中核機能であり、以下の要件を満たす必要がある：
- 判断の一貫性（日によってブレない）
- 透明性（なぜそのスコアになったかを説明可能）
- 調整可能性（ユーザーが閾値やキーワードを調整可能）
- 下限の保証（セキュリティ、破壊的変更は必ず検出）

**Decision**:

ルールベースのスコアリングアルゴリズムを採用する：

1. **ベーススコア**: 全アイテム 50（中立）から開始
2. **キーワードマッチ**: 正規表現パターンで加減算
   - 複数マッチ時は**全て加算**
   - 例: CVE (+90) + breaking change (+70) = +160
3. **分類閾値**:
   - READ NOW: スコア ≥ 80
   - LATER: 45 ≤ スコア < 80
   - IGNORE: スコア < 45
4. **判定理由**: マッチしたキーワードとスコア調整を `reasons` として記録

**Consequences**:

**良い点**:
- 透明性（ルールが明示的）
- 調整可能性（YAML でキーワードとスコアを変更可能）
- 説明可能性（`reasons` でなぜそのスコアになったかを説明）
- 一貫性（同じ入力なら同じ出力）

**悪い点**:
- 手動チューニング必要（最適なキーワードとスコアを見つける必要）
- 自然言語理解の限界（文脈を理解できない）
- メンテナンスコスト（新しいパターンを追加し続ける必要）

**Alternatives**:

- **LLM のみスコアリング**: 柔軟だが一貫性が低く、コストが高い
- **機械学習モデル**: 高精度だがブラックボックス化、学習データ必要
- **ソースごとにベーススコア設定**: 柔軟性は高いが複雑性が増す

**Implementation Notes**:

設定例（`ddigest.yaml`）：

```yaml
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

## ADR-005: エラーハンドリング戦略

**Status**: Accepted

**Context**:

DDigest は外部システム（RSS フィード、HTTP サーバー、LLM API）に依存しており、以下のエラーが発生する可能性がある：
- ネットワークエラー（タイムアウト、接続失敗）
- パースエラー（無効な RSS、HTML）
- LLM API エラー（レート制限、認証失敗、タイムアウト）

エラー処理の方針として、以下を考慮する必要がある：
- データ損失の防止
- 処理の継続性（一部失敗でも他の処理を継続）
- ユーザーへの通知

**Decision**:

以下のエラーハンドリング戦略を採用する：

1. **RSS フェッチエラー**:
   - ログ記録（WARN レベル）
   - 該当ソースをスキップし、他のソースを継続
   - 最終的にエラーサマリーを表示

2. **LLM API エラー**:
   - 要約なしでアイテムを保存（`summary = NULL`）
   - ログ記録（ERROR レベル）
   - 後で手動再試行可能（`ddigest summarize --retry` など）

3. **パースエラー**:
   - ログ記録（WARN レベル）
   - 該当アイテムをスキップ

**Consequences**:

**良い点**:
- データ損失なし（アイテムは必ず保存）
- 部分的失敗でも処理継続（他のソースは正常処理）
- 後で再試行可能（LLM エラー時）

**悪い点**:
- エラー時の通知が弱い（ログのみ）
- ユーザーがエラーに気づかない可能性
- 再試行ロジックの実装が必要

**Alternatives**:

- **エラー時は全体を停止**: データ一貫性は高いが、一部失敗で全体が停止
- **自動リトライ**: 透明性は高いが、リトライ戦略が複雑
- **デスクトップ通知**: エラー時に通知。ただし MVP では実装コストが高い

**Implementation Notes**:

- `env_logger` でログ出力（`RUST_LOG` 環境変数で制御）
- エラーは `anyhow::Result` で統一
- 最終的にエラーカウントを表示（`Fetched: 42 items (3 errors)`）

---

## ADR-006: データベース設計とマイグレーション

**Status**: Accepted

**Context**:

DDigest はローカル DB で状態管理を行う必要がある：
- フィード定義（sources）
- 記事（items）
- 監査ログ（item_events、任意）

以下を考慮する必要がある：
- スキーマ変更の管理
- 型安全性
- パフォーマンス
- ポータビリティ

**Decision**:

以下の技術スタックを採用する：

1. **DB**: SQLite（シングルファイル、ポータブル）
2. **ORM**: sqlx（コンパイル時検証、型安全）
3. **マイグレーション**: sqlx migrate（`migrations/` ディレクトリで管理）

**Consequences**:

**良い点**:
- 型安全性（コンパイル時に SQL 検証）
- マイグレーション履歴管理（`migrations/` ディレクトリ）
- ポータビリティ（SQLite はシングルファイル）
- パフォーマンス（ローカル DB、インデックス最適化可能）

**悪い点**:
- コンパイル時に DB 接続必要（`DATABASE_URL` 環境変数）
- sqlx の学習曲線（クエリマクロ、オフラインモード）

**Alternatives**:

- **rusqlite + 手動マイグレーション**: 軽量だがマイグレーション管理が手動
- **diesel**: ORM として成熟しているが、sqlx より重い
- **PostgreSQL**: 高機能だがシングルユーザーツールではオーバースペック

**Implementation Notes**:

DB スキーマ（DRAFT.md に基づく）：

```sql
-- migrations/001_initial.sql

CREATE TABLE sources (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    url TEXT NOT NULL UNIQUE,
    tags TEXT, -- JSON array
    last_fetched_at TEXT
);

CREATE TABLE items (
    id INTEGER PRIMARY KEY,
    source_id INTEGER NOT NULL,
    url TEXT NOT NULL UNIQUE,
    title TEXT,
    author TEXT,
    published_at TEXT,
    fetched_at TEXT NOT NULL,
    content_text TEXT,
    content_hash TEXT NOT NULL,
    status TEXT NOT NULL, -- new | read_now | later | ignore | done
    score INTEGER,
    reasons TEXT, -- JSON array
    summary TEXT,
    why_you TEXT, -- JSON array
    FOREIGN KEY (source_id) REFERENCES sources(id)
);

CREATE INDEX idx_items_status ON items(status);
CREATE INDEX idx_items_fetched_at ON items(fetched_at);
CREATE INDEX idx_items_content_hash ON items(content_hash);
```

---

## ADR-007: CLI UX とコマンド自動実行

**Status**: Accepted

**Context**:

DDigest は毎日使用するツールであり、UX が重要である。特に以下を考慮する必要がある：
- コマンド数の最小化（手間を減らす）
- 視認性（重要な情報が一目で分かる）
- 進捗表示（フェッチ中、スコアリング中の状態が分かる）

**Decision**:

以下の UX 設計を採用する：

1. **自動実行**: `ddigest fetch` 実行時に自動的に `score` も実行
   - ユーザーは `fetch` のみ実行すれば良い
   - スキップオプション不要（MVP では）

2. **カラー出力**:
   - READ NOW: 赤（緊急性を強調）
   - LATER: 黄（注意喚起）
   - IGNORE: グレー（低優先度）

3. **進捗表示**:
   - フェッチ中: `Fetching 18 sources...`
   - スコアリング中: `Scoring 42 items...`

**Consequences**:

**良い点**:
- コマンド数削減（`fetch` のみ実行すれば良い）
- 視認性向上（カラー出力で重要度が一目で分かる）
- 認知負荷低減（毎日のルーチンを簡素化）

**悪い点**:
- fetch のみ実行したい場合の柔軟性低下
- カラー出力が不要な環境での対応必要（CI など）

**Alternatives**:

- **完全分離**: `fetch` と `score` を独立。柔軟性は高いが手間が増える
- **モノクロ出力**: シンプルだが視認性が低下
- **オプションで制御**: `fetch --no-score` など。柔軟性は高いが複雑性が増す

**Implementation Notes**:

- カラー出力は `colored` クレート使用
- `NO_COLOR` 環境変数でカラー無効化対応
- 出力フォーマット例（DRAFT.md に基づく）：

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

## ADR-008: テスト戦略

**Status**: Accepted

**Context**:

DDigest は信頼性が重要なツールであり（誤判定は機会損失やリスク見逃しにつながる）、以下のテストが必要：
- スコアリングロジックの正確性
- RSS パース、DB 操作の正確性
- エラーハンドリングの適切性

**Decision**:

以下のテスト戦略を採用する：

1. **ユニットテスト**:
   - スコアリングロジック（キーワードマッチ、スコア計算）
   - コンテンツ正規化（`content_hash` 生成）
   - ルール評価（正規表現マッチ）

2. **統合テスト**:
   - RSS フェッチ → DB 保存 → スコアリングの全体フロー
   - LLM API 呼び出し（モック使用）
   - CLI コマンド実行（`ddigest fetch`, `ddigest today` など）

3. **モック**:
   - RSS フィード: `mockito` で HTTP モック
   - LLM API: `mockito` でレスポンスモック

**Consequences**:

**良い点**:
- リグレッション防止（変更時の既存機能の保証）
- リファクタリング容易（テストがあれば安心して変更可能）
- ドキュメントとしての価値（テストが仕様を示す）

**悪い点**:
- テストコード量増加（メンテナンスコスト）
- テスト実行時間の増加

**Alternatives**:

- **E2E のみ**: シンプルだがカバレッジが低く、失敗時の原因特定が困難
- **スナップショットテスト**: CLI 出力の検証に有効。`insta` クレート使用を検討

**Implementation Notes**:

テストディレクトリ構成：

```
tests/
  unit/
    scorer_test.rs
    normalizer_test.rs
  integration/
    fetch_test.rs
    cli_test.rs
```

モック例（`mockito`）：

```rust
#[tokio::test]
async fn test_fetch_rss() {
    let mock = mockito::mock("GET", "/feed.rss")
        .with_status(200)
        .with_body(SAMPLE_RSS)
        .create();

    // テストロジック
}
```

---

## ADR-009: デプロイメントとバイナリ配布

**Status**: Accepted

**Context**:

DDigest は個人用ツールであり、インストールの容易さが重要である。以下を考慮する必要がある：
- 幅広いプラットフォーム対応（Linux, macOS, Windows）
- インストールの簡単さ（依存関係不要）
- アップデートの容易さ

**Decision**:

**GitHub Releases** でプリビルドバイナリを配布する：

1. **CI/CD**: GitHub Actions でビルド
2. **プラットフォーム**: Linux (x86_64), macOS (x86_64, arm64), Windows (x86_64)
3. **配布**: GitHub Releases にアップロード
4. **インストール**: バイナリをダウンロードして PATH に配置

**Consequences**:

**良い点**:
- インストールが容易（バイナリをダウンロードするだけ）
- 広範囲なユーザー対応（Rust 環境不要）
- アップデートが明示的（リリースノートで変更確認）

**悪い点**:
- クロスコンパイル設定が必要（GitHub Actions ワークフロー）
- パッケージマネージャー未対応（cargo install, Homebrew など）

**Alternatives**:

- **cargo install**: Rust ユーザー向けには便利だが、一般ユーザーには敷居が高い
- **Nix flake**: NixOS ユーザー向けには理想的だが、対象ユーザーが限定的
- **Homebrew**: macOS ユーザー向けに便利だが、メンテナンスコストが高い

**Implementation Notes**:

GitHub Actions ワークフロー例（`.github/workflows/release.yml`）：

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v2
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      - run: cargo build --release
      - uses: actions/upload-artifact@v2
        with:
          name: ddigest-${{ matrix.os }}
          path: target/release/ddigest
```

**Future Considerations**:

MVP 後に以下を検討：
- cargo install 対応（crates.io に公開）
- Homebrew tap（macOS ユーザー向け）
- Nix flake（NixOS ユーザー向け）

---

## ADR-010: コンテンツ重複検出の実装

**Status**: Accepted

**Context**:

RSS フィードから取得した記事の重複を検出し、更新を検出する必要がある：
- 同一記事の重複保存を防ぐ
- 記事の更新を検出する（同じ URL だが内容が変わった）
- 差分管理を確実に行う

**Decision**:

以下の戦略でコンテンツ重複検出を実装する：

1. **ハッシュ対象**: `title + content_text` を正規化して SHA256 ハッシュ
2. **正規化ルール**:
   - 空白文字を正規化（連続する空白を単一スペースに）
   - 先頭・末尾の空白を削除
   - 小文字化は**しない**（タイトル変更を検出するため）
3. **挿入ロジック**:
   - URL で既存チェック
   - 存在しない → 新規挿入
   - 存在する → `content_hash` 比較
     - 同一 → スキップ
     - 異なる → 更新として扱う

**Consequences**:

**良い点**:
- 確実な重複検出（URL + content_hash で判定）
- 更新検出可能（同一 URL でも内容変更を検出）
- シンプルな実装（SHA256 ハッシュのみ）

**悪い点**:
- 微小な変更でも更新扱い（誤字修正など）
- ハッシュ計算コスト（ただし SHA256 は高速）

**Alternatives**:

- **URL のみ**: シンプルだが更新検出不可
- **content_text のみ**: タイトル変更を無視するが、実質的な内容変更のみ検出
- **Fuzzy hash**: 類似度検出可能だが複雑性が高い

**Implementation Notes**:

正規化関数例：

```rust
use sha2::{Sha256, Digest};

fn compute_content_hash(title: &str, content: &str) -> String {
    let normalized = format!(
        "{} {}",
        normalize_whitespace(title),
        normalize_whitespace(content)
    );
    let mut hasher = Sha256::new();
    hasher.update(normalized.as_bytes());
    format!("{:x}", hasher.finalize())
}

fn normalize_whitespace(s: &str) -> String {
    s.split_whitespace().collect::<Vec<_>>().join(" ").trim().to_string()
}
```

挿入ロジック例：

```rust
async fn insert_or_update_item(pool: &SqlitePool, item: &Item) -> Result<()> {
    let existing = sqlx::query_as::<_, Item>(
        "SELECT * FROM items WHERE url = ?"
    )
    .bind(&item.url)
    .fetch_optional(pool)
    .await?;

    match existing {
        None => {
            // 新規挿入
            sqlx::query("INSERT INTO items (...) VALUES (...)")
                .execute(pool)
                .await?;
        }
        Some(existing_item) => {
            if existing_item.content_hash != item.content_hash {
                // 更新
                sqlx::query("UPDATE items SET content_hash = ?, ... WHERE url = ?")
                    .execute(pool)
                    .await?;
            }
            // else: スキップ
        }
    }
    Ok(())
}
```

---

## まとめ

これらの ADR は、DDigest の MVP 実装における主要なアーキテクチャ判断を記録しています。各判断は、DRAFT.md と CONCEPT.md の設計思想に基づき、実装の詳細レベルで具体化されています。

### 次のステップ

1. ADR をレビューし、必要に応じて修正
2. Cargo プロジェクトの初期化（`cargo init`）
3. 依存クレートの追加
4. DB スキーマと migrations/ ディレクトリの作成
5. Phase 1 の実装開始（DB セットアップ、RSS フェッチャー）

### 参考

- DRAFT.md: MVP 仕様
- CONCEPT.md: 設計思想
- プランファイル: `~/.claude/plans/radiant-churning-melody.md`
