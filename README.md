# llm-source-feed

LLM 提供元（OpenAI / Google DeepMind / Anthropic）の公式ブログ・リリース情報を毎日自動収集し、出典リンクを主役にした時系列一覧として公開するアグリゲータ。

AI 生成の粗悪な二次記事を排し、「一次ソースの新着を、出典・公開日つきで網羅的かつ鮮度高く一覧する」ことを目的とする。記事本文は転載せず、タイトル・出典・公開日・原文リンクのみを扱う。

## 技術構成（Cloudflare サーバーレス・完全無料枠）

| 層 | 採用 |
| --- | --- |
| 収集ワーカー | Cloudflare Workers（日次 Cron Triggers で起動） |
| HTML メタ抽出 | HTMLRewriter（Cloudflare 標準） |
| DB | Cloudflare D1（SQLite、`ON CONFLICT` で冪等 upsert） |
| API | Cloudflare Workers + Hono（`GET /articles`） |
| フロント | Next.js（OpenNext で Workers にデプロイ、一覧 1 ページ） |
| 共通 | TypeScript / Zod / pnpm モノレポ / wrangler |

## 収集ソースと方式

| ソース | 方式 |
| --- | --- |
| Anthropic | sitemap.xml で `/news/` 記事一覧を取得 → 各記事の OG メタ（og:title / og:description）を抽出、公開日は lastmod を充当 |
| OpenAI | RSS（拡張フェーズで追加） |
| Google DeepMind | RSS（拡張フェーズで追加） |

## ロードマップ（MVP）

- **Phase 1**: Anthropic 単独で 収集 → D1 → API → 一覧ページ → デプロイ をレイヤーごとに構築
- **Phase 2**: OpenAI / DeepMind の RSS Adapter を追加（収集器の抽象化の真価）
- **Phase 3**: Cron Triggers での日次自動化・構造化ログ・エラー耐性 →【MVP 完成 = クラウドで毎日自動更新】

### スコープ外（拡張フェーズ送り）

AI 要約 / タグ分類 / 絞り込み UI / 全文検索 / ユーザー登録 / 通知。
