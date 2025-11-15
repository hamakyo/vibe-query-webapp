# Vibe Query Web App (PoC)

自然文（日本語）から安全な SQL を自動生成し、Supabase 上の PostgreSQL に対して実行し結果を返す最小構成の Web アプリケーション。Claude の structured outputs（JSON outputs）を活用し、アプリ側で SQL の安全性を検証してから実行します。

## 概要
- 入力: ユーザーの自然文の質問
- 生成: Claude が `sql` と `explanation` を JSON で返却（structured outputs）
- 検証: SQL を AST ベースでバリデーション（SELECT/単一ステートメント/許可テーブル/LIMIT など）
- 実行: 検証通過 SQL を Supabase/PostgreSQL に実行
- 出力: 結果テーブル＋説明文＋行数・実行時間（LLM/DB）
- 保存: クエリログ（成功/失敗、各種メトリクス、エラー種別）

## 主な機能
- チャット風 UI（単一画面）
- Claude structured outputs による型安全な SQL 生成
- SQL バリデーション（SELECT 限定、複数ステートメント禁止、許可テーブル制御、LIMIT 必須）
- 実行結果のテーブル表示とメトリクス表示
- クエリログ保存とエラーハンドリング（VALIDATION/LLM/DB）

## API（PoC）
- `POST /api/query`
  - Request: `{ "question": string }`
  - Response (200): `{ question, explanation, sql, rows, rowCount, executionTimeMs, llmTimeMs }`
  - Error: `{ errorType: "VALIDATION_ERROR" | "LLM_ERROR" | "DB_ERROR" | "UNKNOWN", message }`

## 必要な環境変数
- `DATABASE_URL`（Supabase PostgreSQL 接続文字列）
- `ANTHROPIC_API_KEY`（Claude API キー）
- `LLM_MODEL_NAME`（例: `claude-3-5-sonnet-20241022`）
- `QUERY_MAX_ROWS`（最大取得行数の上限 例: `100`）
- `ANTHROPIC_BETA_HEADERS`（例: `structured-outputs-2025-11-13`）

## 技術スタック（想定）
- フロント/バック: Next.js（App Router）/ TypeScript
- DB: Supabase（PostgreSQL）
- LLM: Claude（Anthropic API, structured outputs）
- 型/検証: Zod、SQL パーサ（AST バリデーション）

## ディレクトリ構成（予定）
- `src/app/page.tsx`（UI）
- `src/app/api/query/route.ts`（API）
- `src/lib/llm/*`（Claude クライアント、型、プロンプト）
- `src/lib/query/*`（スキーマメタ、SQL バリデータ、実行）
- `src/lib/logging/*`（クエリログ保存）
- `docs/specification.md`（仕様書）

## 現状と今後
- 現状: 仕様書のみを含む PoC リポジトリ（実装 WIP）
- 次ステップ: ルート/API/LLM クライアント/SQL バリデータ/DB 実行/ログ保存の順に最小実装を追加

## セキュリティ/制約（抜粋）
- 更新系/DDL/権限操作は不可。`SELECT`（および `WITH ... SELECT`）のみ
- 複数ステートメント禁止（`;` 分割不可）
- 許可テーブルのみ参照可、`LIMIT` 必須かつ上限あり
- DB/LLM の秘匿情報は環境変数で管理

詳細は `docs/specification.md` を参照してください。

