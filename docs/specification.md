# Vibe Query Web Application 仕様書（Specification / Updated）

## 1. プロジェクト概要

- 名称: Vibe Query Web App（PoC）
- 概要:  
  Supabase 上の PostgreSQL データベースに対して、ユーザーが自然文（日本語）で質問を入力すると、Claude が SQL を自動生成し、結果をテーブル形式で返す Web アプリケーション。  
- 目的:
  - 「自然文 → SQL → DB 実行 → 結果表示」の一連の Vibe Query 体験を、最小構成で動作確認する。
  - 将来的な拡張（RLS、マルチデータソース、MCP 対応、エージェント的ワークフローなど）を見据えたアーキテクチャにする。
  - LLM の structured outputs（JSON outputs）を活用し、LLM 応答の構造的な信頼性を確保した上で、アプリ側で SQL 安全性を担保する。

---

## 2. 用語定義

- Vibe Query:  
  ユーザーの自然言語による質問を、LLM により構造化クエリ（ここでは SQL）に変換し、データベースに実行して結果を返す一連の処理。
- LLM:  
  Claude（Anthropic API）を想定。Structured outputs（JSON outputs）機能を利用する。
- Supabase:  
  マネージド PostgreSQL を提供する BaaS。本システムでは主データストアとして利用。
- PoC:  
  Proof of Concept。技術検証のための最小実装。
- Structured outputs（JSON outputs）:  
  Claude の出力を JSON Schema によって制約し、常にスキーマ準拠の JSON を返す機能。本システムでは LLM からの `sql` / `explanation` 取得に利用する。

---

## 3. スコープ

### 3.1 スコープ内

1. Web UI 上でのチャット風インターフェース（単一画面）。
2. ユーザーの自然言語の質問を受け取り、Claude で SQL を生成。
   - Structured outputs（JSON outputs）により、`{ sql, explanation }` をスキーマ準拠の JSON として受領。
3. 生成された SQL の静的検査（バリデーション）。
4. バリデーションを通過した SQL のみ、Supabase/PostgreSQL に対して実行。
5. クエリ結果（テーブル）および LLM による説明テキストを画面に表示。
6. クエリ実行ログの保存（DB テーブルへの記録）。
7. PoC の観点で、LLM レイテンシ／DB レイテンシなどを計測し、粗いボトルネックを把握可能にする。

### 3.2 スコープ外（Non-goals）

1. 本番運用レベルの認証・認可（PoC では無認証または単一ユーザー想定）。
2. マルチテナント対応。
3. Supabase 以外のデータソース（SaaS・DWH 等）の接続。
4. 更新系クエリ（INSERT/UPDATE/DELETE/DDL）の実行。
5. パフォーマンスチューニング（インデックス設計、クエリ最適化の自動化など）の高度化。
6. MCP（Model Context Protocol）対応は将来拡張とし、現時点では含めない。
7. strict tool use を用いた複数ツール呼び分け型のエージェント実装（将来の検討対象とする）。

---

## 4. 想定ユーザー

- 情報システム部門／データ担当エンジニア
- ビジネス部門のパワーユーザー（社内 PoC の見学者）  
※PoC 時点ではインターネット公開ではなく、社内限定アクセスを想定。

---

## 5. ユースケース

### UC-01: 自然文でデータを問い合わせる

1. ユーザーがブラウザでアプリにアクセスする。
2. 画面下部の入力欄に自然文（日本語）で質問を入力する。
3. 「送信」ボタンを押下する。
4. サーバー側で Claude（structured outputs JSON outputs 利用）に問い合わせ、`sql` / `explanation` を含む JSON を生成する。
5. サーバー側で SQL を検査し、問題なければ Supabase に送信する。
6. Supabase から返却された結果をテーブル形式に整形する。
7. 画面に以下を表示する：
   - ユーザーの質問
   - Claude が生成した説明テキスト（「このクエリは〜を取得しています」等）
   - 実行された SQL（オプション）
   - 結果テーブル（最大 N 行）
   - 行数・実行時間（LLM/DB）
8. 同時にクエリログを DB に保存する。

### UC-02: クエリ失敗時のフィードバック

1. UC-01 の 4〜5 の過程で、以下のいずれかのエラーが発生する：
   - SQL バリデーションに失敗（禁止キーワード含む、許可されていないテーブル参照、LIMIT なし等）
   - Claude の API エラー（ネットワーク・認証・スキーマエラー等）
   - DB 実行時エラー（構文エラー、タイムアウトなど）
2. サーバー側でエラー種別を判定し、ユーザー向けメッセージを生成する。
3. 画面に以下を表示する：
   - ユーザーの質問
   - エラー種別を示すメッセージ（例：「SQL が安全ではありません」「サーバー側エラーが発生しました」）
4. エラーログを DB に保存する。

---

## 6. 機能要件（Functional Requirements）

### FR-001: チャット UI

- ユーザーはテキスト入力欄に質問を入力できること。
- 「送信」ボタンまたは Enter キーで送信できること。
- 過去の質問と回答（メッセージ履歴）が同一画面に表示されること。
- 直近セッション内でのみ履歴をブラウザメモリ上で保持すればよい（サーバー側永続化は必須ではない）。

### FR-002: LLM による SQL 生成（Structured Outputs 利用）

- サーバー側は、ユーザーの質問とスキーマメタデータを Claude に渡し、SQL 生成用プロンプトを用いて SQL を生成すること。
- Claude への呼び出しは、Anthropic の structured outputs（JSON outputs）機能を利用し、`output_format: { type: "json_schema", ... }` を指定すること。
- TypeScript 側では Zod 等のスキーマライブラリを用い、`LlmSqlResponse` 型としてパース・検証すること。
- Claude からのレスポンス JSON は、少なくとも以下の項目を含むこと：
  - `sql`: 実行候補の SQL 文（文字列）
  - `explanation`: クエリの説明（自然言語）
- LLM レスポンスは structured outputs により JSON スキーマ準拠であることが前提だが、以下の場合は `LLM_ERROR` とすること：
  - Claude API からの 4xx/5xx エラー
  - Structured outputs のスキーマ定義不備によるエラー

### FR-003: SQL バリデーション

- 実行前に、生成された SQL に対して以下のチェックを必須とする：

  1. ステートメント種別の制約
     - 最外レベルのステートメントが `SELECT` または `WITH ... SELECT` のみで構成されていること。
     - `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `DROP`, `ALTER`, `TRUNCATE`, `CREATE`, `GRANT`, `REVOKE` などの更新系・DDL・権限操作ステートメントが一切含まれていないこと。

  2. 複数ステートメント禁止
     - `;` による複数ステートメントになっていないこと。
     - AST レベルで複数のステートメントが検出された場合はエラーとすること。

  3. テーブルアクセス制御
     - 参照するテーブル（`FROM` / `JOIN` / サブクエリ内を含む）が「許可されたテーブル一覧（`ALLOWED_TABLES`）」に含まれていること。
     - PoC 時点ではテーブル単位の制御を行う。将来的にカラム単位の制御（NG カラム）を追加可能な設計とする。

  4. LIMIT 句の必須化
     - 最外側の SELECT に `LIMIT` 句が含まれていることを必須とする。
     - `LIMIT` が存在しない SQL は、サーバー側で LIMIT を自動付与せず、`VALIDATION_ERROR` として扱うこと。
     - `LIMIT` の値はアプリ設定値 `QUERY_MAX_ROWS` 以下であること（大きい場合は `VALIDATION_ERROR`）。

  5. SELECT * の扱い
     - PoC では `SELECT *` を許可する。
     - 将来的に NG カラム設定を導入する際には、`*` 展開時に NG カラムを除外する仕組みを追加可能な設計とする。

- 実装方針：
  - 原則として PostgreSQL 対応の SQL パーサライブラリを用い、AST ベースで上記チェックを行うこと。
  - 正規表現ベースの単純な文字列検索のみで完結させることは避ける（CTE・サブクエリ・JOIN などのケースで不完全になるため）。
- バリデーション結果は以下の情報を含むこと：
  - `isValid: boolean`
  - `errors: string[]`
  - `safeSql?: string`（※PoC では SQL の自動書き換えは行わず、LLM が返した SQL をそのまま `safeSql` とする）

### FR-004: SQL 実行

- バリデーションを通過した SQL のみ、Supabase/PostgreSQL に対して実行すること。
- 実行結果はテーブル形式（列名＋行データ）として取得すること。
- DB 実行時間（ms）と取得行数を計測すること。
- LLM 呼び出し時間（ms）も計測し、クエリログに保存すること。

### FR-005: 結果表示

- 画面には以下を表示すること：
  - ユーザーの質問テキスト
  - Claude の説明テキスト
  - 結果テーブル（列名・行）
  - 取得行数・DB 実行時間（ms）
  - LLM 応答時間（ms）
  - オプションで、実行した SQL を折りたたみ表示（開閉可能）
- 行数が 0 件の場合は、空テーブル表示に加え、「結果が 0 件である」旨の簡易メッセージを表示すること（UX 向上目的）。

### FR-006: クエリログ保存

- 以下の情報を、クエリログテーブルに保存すること：
  - ID
  - 質問テキスト
  - 生成された SQL
  - 実行結果ステータス（成功/失敗）
  - 取得行数（成功時）
  - DB 実行時間（ms）
  - LLM 応答時間（ms）
  - 実行日時
  - エラー種別（`VALIDATION_ERROR` / `LLM_ERROR` / `DB_ERROR` / `UNKNOWN`）
  - エラー内容（失敗時）
- 将来的なユーザー紐づけを見据え、`user_id` を追加可能な設計とする（PoC では null 固定でもよい）。

### FR-007: エラーハンドリング

- LLM エラー、SQL バリデーション失敗、DB エラーを区別してログに保存すること。
- ユーザーには、内部情報を出しすぎない形で簡潔なエラーメッセージを表示すること。
- LLM エラー例：
  - Anthropic API からの 4xx/5xx エラー
  - structured outputs に対するスキーマ定義エラー
- SQL バリデーションエラー例：
  - 禁止ステートメント検出
  - LIMIT 欠如・オーバーな LIMIT 値
  - 未許可テーブル参照
- DB エラー例：
  - 構文エラー（LLM 生成 SQL が PostgreSQL の文法に合致しない）
  - タイムアウト
  - 接続エラー

---

## 7. 非機能要件（Non-Functional Requirements）

### NFR-001: パフォーマンス

- 単一ユーザー利用を前提とし、1 リクエストあたりの総応答時間は 10 秒以内を目標とする。
  - 内訳の目安：
    - LLM レスポンス: 数秒程度
    - DB 実行: 1 秒未満（通常）
- クエリ結果は最大 `QUERY_MAX_ROWS` 行の取得を前提とする（デフォルト例: 100 行）。
- structured outputs 利用により、初回リクエスト時は grammar コンパイルによるレイテンシ増加が発生する可能性がある。  
  同一スキーマ（`LlmSqlResponse`）を継続利用することでキャッシュを活用し、2 回目以降のレイテンシを抑える。

### NFR-002: セキュリティ

- 更新系クエリは一切実行しないこと（FR-003 のバリデーションで担保）。
- DB 接続情報および LLM API キーは、環境変数で管理すること。
- ブラウザから DB への直接通信は行わず、必ずサーバー側経由で実行すること。
- PoC では閲覧対象データはセンシティブ度の低いサンプル／マスク済みデータを用いることが望ましい。  
  本番データを用いる場合は、事前にアクセス範囲の承認を得ること。
- LLM に渡すスキーマ情報（テーブル・カラム情報）は、開示してよい範囲のものに限定すること。

### NFR-003: 可用性・運用

- PoC のため、単一インスタンス構成でよい。
- ログ保存により、問題発生時に原因調査ができること。
  - 特に、LLM レイテンシと DB レイテンシの区別ができること。
- 簡易な Rate Limiting（例: IP/クライアント単位で数秒あたり数リクエスト程度）を導入し、誤操作による連打や負荷集中を避ける。

### NFR-004: メンテナンス性

- TypeScript を使用し、型安全に実装すること。
- LLM プロンプト、スキーマメタデータ、SQL バリデータロジックは、モジュールとして分離すること。
- 許可テーブル一覧はコード上に明示的に定義し、カラム一覧は可能であれば DB からの introspection によって取得すること。
- 将来的なデータソース追加や権限強化に備え、依存を局所化すること。
- LLM structured outputs のスキーマ (`LlmSqlResponse`) は 1 箇所（型定義ファイル）で定義し、フロント／バック双方で再利用できるようにすること。

---

## 8. 画面仕様（UI 概要）

### 8.1 画面構成

- 画面名: Query Console
- URL: `/`
- 構成要素:
  - ヘッダー: アプリ名、簡単な説明テキスト（例: 「自然文から安全な SQL を自動生成してデータを検索します」）
  - メッセージ履歴エリア:
    - ユーザー質問、システム回答（説明＋結果テーブル）を時系列で表示
    - 各メッセージブロックに、LLM 応答時間・DB 実行時間・取得行数を表示
  - 入力エリア:
    - テキストエリア（質問入力用）
    - 送信ボタン
    - 実行中インジケータ（スピナー）
  - （任意）サンプル質問ショートカット
    - 代表的な問い合わせ例をボタンとして用意し、クリックで入力欄に挿入する。

### 8.2 表示項目（回答メッセージ）

- 質問テキスト
- 説明テキスト（Claude 生成）
- 実行した SQL（折りたたみ表示）
- 結果テーブル
- 取得行数
- DB 実行時間（ms）
- LLM 応答時間（ms）
- 0 件時の補助メッセージ

---

## 9. API 仕様

### 9.1 `POST /api/query`

- 概要:  
  ユーザー質問を受け取り、LLM で SQL を生成し、DB 実行した結果を返す。

- リクエストボディ（JSON）:

  - `question`: string  
    ユーザーの自然文質問。

- レスポンスボディ（成功時, 200）:

  - `question`: string  
  - `explanation`: string  
  - `sql`: string  
  - `rows`: array of object  
    - 例: `[{ "column1": "value1", "column2": 123 }, ...]`
  - `rowCount`: number  
  - `executionTimeMs`: number  （DB 実行時間）
  - `llmTimeMs`: number         （LLM 応答時間）

- レスポンスボディ（エラー時, 4xx/5xx）:

  - `errorType`: string  
    - `"VALIDATION_ERROR" | "LLM_ERROR" | "DB_ERROR" | "UNKNOWN"`
  - `message`: string  

- 認証:  
  PoC のため不要（将来の拡張でトークン認証などを追加可能な設計とする）。

---

## 10. エラー仕様

- SQL バリデーションエラー:
  - HTTP ステータス: 400
  - `errorType`: `"VALIDATION_ERROR"`
  - 例: 未許可テーブル参照、更新系キーワード検出、LIMIT 欠如、LIMIT 上限超え
- LLM API / Structured outputs エラー:
  - HTTP ステータス: 502（上流サービスエラー扱い）
  - `errorType`: `"LLM_ERROR"`
  - 例: Anthropic API 4xx/5xx、structured outputs のスキーマ不正
- DB 実行エラー:
  - HTTP ステータス: 500
  - `errorType`: `"DB_ERROR"`
  - 例: PostgreSQL 構文エラー、タイムアウト、接続エラー
- その他予期せぬエラー:
  - HTTP ステータス: 500
  - `errorType`: `"UNKNOWN"`

---

## 11. 環境変数

- `DATABASE_URL`: Supabase PostgreSQL 接続文字列
- `ANTHROPIC_API_KEY`: Claude API キー
- `LLM_MODEL_NAME`: 使用するモデル名（例: `claude-3-5-sonnet-20241022` など）
- `QUERY_MAX_ROWS`: 取得する最大行数（例: 100）
- `ANTHROPIC_BETA_HEADERS`: `"structured-outputs-2025-11-13"` など（構造化出力の Beta ヘッダ）

---

# Vibe Query Web Application 設計書（Design / Updated）

## 1. アーキテクチャ概要

- フロントエンド: Next.js（App Router） / TypeScript
- バックエンド: Next.js Route Handler（Node.js / TypeScript）
- データベース: Supabase（PostgreSQL）
- LLM: Claude（Anthropic API, structured outputs JSON outputs 利用）

全体構成:

- ブラウザ → Next.js App（UI）
- Next.js Route Handler `/api/query` → LLM クライアント（structured outputs） → SQL バリデータ → DB クライアント（Supabase/pg）
- DB → Query Logs テーブルに履歴を保存

---

## 2. システム構成図（テキスト）

- Browser
  - Next.js UI（`src/app/page.tsx`）
    - 呼び出し: `POST /api/query`
- Next.js API（`src/app/api/query/route.ts`）
  - `lib/llm/claudeClient`（structured outputs）
  - `lib/query/sqlValidator`
  - `lib/query/queryExecutor`
  - `lib/query/schemaMeta`（許可テーブル・カラムメタ）
  - `lib/db/supabaseClient` または `pg` クライアント
  - `lib/logging/queryLogger`
- Supabase PostgreSQL
  - 業務データテーブル（例: `orders`, `customers`）
  - `query_logs` テーブル

---

## 3. ディレクトリ構成（案）

- `src/app/page.tsx`  
  メイン画面（チャット UI）

- `src/app/api/query/route.ts`  
  API エンドポイント実装（Next.js Route Handler）

- `src/lib/db/supabaseClient.ts`  
  DB クライアント（`pg` もしくは Supabase JS）

- `src/lib/llm/claudeClient.ts`  
  Claude API 呼び出しラッパ（structured outputs 利用）

- `src/lib/llm/types.ts`  
  LLM 出力スキーマ定義（Zod 等、`LlmSqlResponse`）

- `src/lib/llm/prompts/sqlPrompt.ts`  
  SQL 生成用プロンプト定義

- `src/lib/query/schemaMeta.ts`  
  許可テーブル・カラムのメタデータ定義（および DB introspection ロジック）

- `src/lib/query/sqlValidator.ts`  
  SQL バリデータ（AST ベース検証）

- `src/lib/query/queryExecutor.ts`  
  SQL 実行と結果整形処理

- `src/lib/logging/queryLogger.ts`  
  クエリログの保存処理

- `src/lib/errors/appError.ts`  
  共通エラー型 `AppError` 定義

---

## 4. モジュール設計

### 4.1 `claudeClient`

**責務:**

- Claude API との通信を一元管理する。
- Structured outputs（JSON outputs）により、`LlmSqlResponse` スキーマ準拠のレスポンスを取得する。

**主要関数:**

- `generateSql(question: string, schemaMeta: SchemaMeta): Promise<LlmSqlResponse>`

`LlmSqlResponse`:

- `sql: string`
- `explanation: string`

**処理概要:**

1. プロンプトテンプレートに `question` と `schemaMeta` を埋め込む。
2. Anthropic SDK の `beta.messages.parse()` を用い、`output_format` に Zod スキーマ（`LlmSqlResponseSchema`）を指定して呼び出す。
3. Structured outputs によりスキーマ準拠が保証された `parsed_output` を `LlmSqlResponse` 型で返却する。
4. LLM 呼び出し前後で時間を計測し、`llmTimeMs` を後続処理（`route.ts` → `queryLogger`）に渡せるようにする。

### 4.2 `schemaMeta`

**責務:**

- LLM に渡すスキーマメタデータ（テーブル・カラム一覧）を提供する。
- SQL バリデータが参照する許可テーブル一覧・カラムメタを提供する。

**設計方針:**

- `ALLOWED_TABLES`（許可テーブル名の配列）はコード上に明示的に定義する。
- 各テーブルのカラム一覧は、可能であれば起動時／初回アクセス時に DB の `information_schema` / `pg_catalog` から introspection して生成する。
- LLM に渡すスキーマ情報は、説明文付きで構造化された文字列としてプロンプトに埋め込む。

### 4.3 `sqlValidator`

**責務:**

- 生成された SQL が安全かどうかを AST ベースで判定する。

**主要関数:**

- `validateGeneratedSql(sql: string, allowedTables: string[], maxRows: number): ValidationResult`

`ValidationResult`:

- `isValid: boolean`
- `errors: string[]`
- `safeSql?: string`

**チェック内容:**

- 禁止ステートメントの存在チェック（`SELECT` / `WITH ... SELECT` 以外禁止）。
- テーブル名が `allowedTables` のみで構成されているかチェック。
- 複数ステートメントになっていないかチェック。
- 最外側の `LIMIT` 句の有無および値の上限チェック。
- `SELECT *` は PoC では許可（将来拡張で NG カラム制御を追加可能）。

### 4.4 `queryExecutor`

**責務:**

- 検証済み SQL を DB に対して実行し、結果を標準化した形で返す。

**主要関数:**

- `executeQuery(sql: string): Promise<ExecutedQueryResult>`

`ExecutedQueryResult`:

- `rows: Array<Record<string, any>>`
- `rowCount: number`
- `executionTimeMs: number`

**処理概要:**

1. 実行前に現在時刻を取得。
2. DB クライアントで SQL を実行。
3. 実行後に時間を計測し、`executionTimeMs` を算出。
4. 結果を `ExecutedQueryResult` 型で返す。

### 4.5 `queryLogger`

**責務:**

- クエリ実行やエラーを DB に保存する。

**主要関数:**

- `logQuery(log: QueryLogInput): Promise<void>`

`QueryLogInput`:

- `question: string`
- `sql: string`
- `status: "SUCCESS" | "FAILURE"`
- `rowCount?: number`
- `executionTimeMs?: number` （DB）
- `llmTimeMs?: number`
- `errorType?: "VALIDATION_ERROR" | "LLM_ERROR" | "DB_ERROR" | "UNKNOWN"`
- `errorMessage?: string`
- `createdAt: Date`

---

## 5. データベース設計

### 5.1 ログテーブル `query_logs`

- `id` (bigserial, PK)
- `question` (text, not null)
- `generated_sql` (text, not null)
- `status` (varchar, not null)  
  - `"SUCCESS"` or `"FAILURE"`
- `row_count` (integer, nullable)
- `execution_time_ms` (integer, nullable)  — DB 実行時間
- `llm_time_ms` (integer, nullable)        — LLM 応答時間
- `error_type` (varchar, nullable)         — `VALIDATION_ERROR` / `LLM_ERROR` / `DB_ERROR` / `UNKNOWN`
- `error_message` (text, nullable)
- `created_at` (timestamp with time zone, default now())
- `user_id` (text, nullable)               — 将来の認証連携用（PoC では null）

インデックス:

- `idx_query_logs_created_at` on `created_at`

### 5.2 データテーブル

- PoC では既存の業務テーブル（例: `orders`, `customers`）またはサンプルデータテーブルを想定。
- 許可テーブルの一覧は `schemaMeta.ts` に明示的に定義する。
  - 例: `ALLOWED_TABLES = ["orders", "customers"]`
- センシティブカラムを含むテーブルを許可する場合は、PoC の範囲・閲覧者を限定する。

---

## 6. LLM プロンプト設計

### 6.1 システムプロンプト概要

内容（要約）:

- 役割:  
  あなたはデータベースエンジニアであり、与えられたスキーマとユーザーの質問から、安全な SQL（SELECT のみ）を生成する。
- 制約:
  - SELECT 文のみを生成すること（必要に応じて WITH 句を用いてもよい）。
  - 更新系・DDL 文を含めないこと。
  - 複数ステートメントを生成しないこと。
  - 指定されたテーブル以外を参照しないこと。
  - 結果行数を限定するために、最外側の SELECT に必ず `LIMIT` を付けること（例: 100 行以内）。
- 出力形式:
  - Structured outputs（JSON outputs）を用いて、スキーマにしたがった JSON のみを返すこと（`sql`, `explanation` の 2 つの文字列フィールド）。

### 6.2 ユーザープロンプト概要

内容:

- ユーザーの質問
- 利用可能なテーブルとカラムの一覧（説明付き）
- 期待されるフィルタ条件や集計方法の例（任意）
- 生成してほしいクエリの粒度感（例: 売上サマリ、最新レコード、ユーザー単位集計など）

---

## 7. シーケンス設計（UC-01）

1. `page.tsx`:
   - ユーザー入力を `POST /api/query` に送信。
2. `route.ts`:
   - リクエストボディから `question` を取得。
   - `schemaMeta` を読み込み。
   - `claudeClient.generateSql(question, schemaMeta)` を呼び出し（structured outputs 利用）。
3. `claudeClient`:
   - プロンプトを構築し、Anthropic SDK の `beta.messages.parse()` を用いて Claude API を呼び出す。
   - structured outputs（JSON outputs）によりスキーマ準拠のレスポンスを取得。
   - `LlmSqlResponse`（`sql`, `explanation`）と `llmTimeMs` を返却。
4. `route.ts`:
   - `sqlValidator.validateGeneratedSql` を呼び出し。
   - `isValid` でなければ、エラーレスポンス（`VALIDATION_ERROR`）とともに `queryLogger.logQuery` に `FAILURE` で記録。
   - 問題なければ `queryExecutor.executeQuery` を呼び出し。
5. `queryExecutor`:
   - DB で SQL を実行し、`ExecutedQueryResult`（`rows`, `rowCount`, `executionTimeMs`）を返す。
6. `route.ts`:
   - 成功レスポンス JSON を組み立てる。
   - `queryLogger.logQuery` に `SUCCESS` で記録。
   - フロントにレスポンスを返却。
7. `page.tsx`:
   - レスポンスを受け取り、メッセージ履歴に追加。
   - 結果テーブルと説明・SQL・各種メトリクス（行数・時間）を表示。

---

## 8. エラーハンドリング設計

- 各層でのエラーをキャッチし、`AppError` 型で統一的に扱う：

`AppError`（例）:

- `type`: `"VALIDATION_ERROR" | "LLM_ERROR" | "DB_ERROR" | "UNKNOWN"`
- `message`: string
- `cause?`: unknown

`route.ts` では:

- `AppError` を捕捉し、`errorType` を HTTP ステータスとともにレスポンスとして返却。
- 同時に `queryLogger` にエラー内容を保存。
- 予期せぬ例外については、`UNKNOWN` としてラップして扱う。

---

## 9. 拡張方針

- 認証の追加:
  - NextAuth/Supabase Auth を導入し、`query_logs` に `user_id` 列を追加。
  - ユーザーごとのクエリ履歴画面への拡張を可能とする。
- データソース追加:
  - `queryExecutor` の実装を抽象化し、データソースごとの実装（RDB, SaaS API 等）を差し替え可能な構造にする。
- MCP 対応:
  - LLM から直接 MCP サーバーを呼び出せるようにし、`claudeClient` の役割を MCP クライアントに再マッピングする。
- strict tool use の活用:
  - 将来的に、複数データソースへのルーティングやメタ操作（スキーマ取得・可視化）などをツールとして切り出し、strict tool use による型安全なツール呼び出しを検討する。

---
