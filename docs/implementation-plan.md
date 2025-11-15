# Vibe Query Web App 実装計画書

本ドキュメントは `docs/specification.md` を踏まえた実装計画とToDoの整理です。MVP完了までの道筋、優先度、受け入れ条件を明確化します。

## 1. ゴール（MVP）
- 自然文→LLM（structured outputs）→SQL生成→AST検証→PostgreSQL実行→結果返却の一連を `/api/query` で成立させる。
- UIで質問→説明文・SQL（折りたたみ）・結果テーブル・行数・LLM/DB時間を表示。
- クエリログをDBに保存（成功/失敗、メトリクス、エラー種別）。

## 2. マイルストーン
- M1: APIスケルトンと型・エラー基盤（AppError, 型定義, ルート雛形）
- M2: LLMクライアント（structured outputs, Zod schema, プロンプト）
- M3: SQLバリデータ（ASTベース検証: SELECT限定/許可テーブル/単一ステートメント/LIMIT上限）
- M4: DB実行（`pg`またはSupabase JS）、タイミング計測と整形
- M5: クエリログ保存（テーブルDDL/ロガー）
- M6: UI（チャット風、結果表示、折りたたみSQL、メトリクス表示、簡易サンプル質問）
- M7: エラーハンドリング統一・Rate Limit・最終調整

## 3. 作業ブレークダウン（ToDo）

### 3.1 基盤・設定
- [ ] プロジェクト初期化（Next.js App Router / TypeScript）
- [ ] 環境変数の読み込みと型定義（`DATABASE_URL`, `ANTHROPIC_API_KEY`, `LLM_MODEL_NAME`, `QUERY_MAX_ROWS`, `ANTHROPIC_BETA_HEADERS`）
- [ ] 共通エラー `AppError` とエラー種別の列挙
- [ ] 依存ライブラリ選定（下記参照）

### 3.2 LLM（structured outputs）
- [ ] `src/lib/llm/types.ts` に `LlmSqlResponse`（Zod）定義（`sql`, `explanation`）
- [ ] `src/lib/llm/prompts/sqlPrompt.ts` にシステム/ユーザープロンプト雛形
- [ ] `src/lib/llm/claudeClient.ts` 実装（Anthropic SDKのstructured outputs呼び出し + 応答時間計測）

### 3.3 スキーマメタ/許可テーブル
- [ ] `src/lib/query/schemaMeta.ts` に `ALLOWED_TABLES` を定義
- [ ] （任意）DB introspection でのカラム取得（PoCでは後回し可）

### 3.4 SQLバリデータ（AST）
- [ ] `src/lib/query/sqlValidator.ts` 実装
  - [ ] 最外レベル `SELECT` / `WITH ... SELECT` 以外禁止
  - [ ] 複数ステートメント禁止（`;` 不可/AST上単一）
  - [ ] 許可テーブルのみ参照
  - [ ] LIMIT 必須かつ `QUERY_MAX_ROWS` 以下
  - [ ] 検証結果 `ValidationResult`（`isValid`, `errors`, `safeSql`）

### 3.5 DB実行
- [ ] `src/lib/query/queryExecutor.ts`（SQL実行, 行配列/件数/実行時間ms）
- [ ] DBクライアント（`pg` もしくは Supabase JS）の接続設定

### 3.6 ログ保存
- [ ] DDL: `query_logs` テーブル作成
- [ ] `src/lib/logging/queryLogger.ts` 実装（成功/失敗、メトリクス、エラー種別）

### 3.7 API ルート
- [ ] `src/app/api/query/route.ts`
  - [ ] `question` 受領→LLM→バリデーション→DB実行
  - [ ] 成功/失敗レスポンスの整形
  - [ ] 例外→`AppError` で分類、ログ保存
  - [ ] 簡易Rate Limit

### 3.8 UI
- [ ] `src/app/page.tsx`（チャット風UI, 入力/送信）
- [ ] 結果表示（説明文/折りたたみSQL/結果テーブル/行数/LLM・DB時間）
- [ ] 0件時メッセージ/実行中インジケータ
- [ ] サンプル質問ショートカット

### 3.9 ドキュメント/運用
- [ ] `.env.example` 追加
- [ ] ローカル起動手順/環境変数の説明追記（README）
- [ ] リリース手順（最小）

## 4. 受け入れ条件（MVP DoD）
- API: `POST /api/query` が仕様通りの成功/エラー応答を返す。
- SQL検証: 禁止ステートメントや未許可テーブル、LIMIT欠如/上限超過が弾かれる。
- UI: 質問→説明・SQL・結果・メトリクスが1画面で確認できる。
- ログ: `query_logs` に成功/失敗が保存され、行数・時間・エラー種別が取得できる。
- Config: `.env` で必要な鍵/URLが設定できる。

## 5. 選定ライブラリ（候補）
- DB: `pg`（軽量・標準的） or `@supabase/supabase-js`
- LLM: `@anthropic-ai/sdk`（structured outputs / beta `messages.parse`）
- 型/検証: `zod`
- SQL AST: `pgsql-ast-parser` or `node-sql-parser`（PostgreSQL対応の成熟度を比較検討）
- Rate Limit: シンプルなメモリ実装（PoC）

## 6. スキーマ/DDL（PoC）
```sql
create table if not exists query_logs (
  id bigserial primary key,
  question text not null,
  generated_sql text not null,
  status varchar(16) not null, -- SUCCESS | FAILURE
  row_count integer,
  execution_time_ms integer,
  llm_time_ms integer,
  error_type varchar(32),      -- VALIDATION_ERROR | LLM_ERROR | DB_ERROR | UNKNOWN
  error_message text,
  created_at timestamptz default now(),
  user_id text
);
create index if not exists idx_query_logs_created_at on query_logs(created_at);
```

## 7. テスト方針
- 単体: `sqlValidator`（代表ケース: 正常/未許可テーブル/複数ステートメント/LIMIT欠如/上限超過）
- 疎通: `/api/query` に対してモックLLM→DB実行までの正常系/異常系
- UI: 入力→表示のスモーク（結果テーブル/メトリクス/折りたたみSQL）

## 8. リスクと対応
- Structured outputs Beta変更: バージョン固定ヘッダをenvで管理、SDK更新に追随
- SQL ASTの網羅性: ライブラリ選定段階で実データで検証、NG時は補助ルール導入
- DBスキーマ差異: `ALLOWED_TABLES` 明示とPoC対象の限定、introspectionは任意化
- LLM遅延: 初回のみ増大のためキャッシュ/再利用、UIで実行中表示とキャンセル設計（任意）

## 9. 作業チケット案（GitHub Issues）
- [ ] chore: Next.js/TS 初期化
- [ ] feat(api): `/api/query` スケルトン
- [ ] feat(llm): structured outputs クライアント + Zod
- [ ] feat(sql): ASTバリデータ実装
- [ ] feat(db): pg クライアント + 実行
- [ ] feat(log): `query_logs` DDL + ロガー
- [ ] feat(ui): チャットUI + 結果表示
- [ ] feat(rate): 簡易Rate Limit
- [ ] docs: `.env.example` & README更新
- [ ] ci: 最小の型チェック/ビルド

---
本計画はMVP着地を最優先とし、将来拡張（認証、MCP、複数ソース）に備えた分離を維持します。

