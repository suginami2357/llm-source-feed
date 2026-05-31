# 実装仕様書 — llm-source-feed

このファイルは、別セッションの実装者（LLM）が前提を再導出せずに **Step 単位で実装に着手できる** ことを目的とした仕様書である。上から順に読めば、何を・なぜ・どう作るかが分かる。各 Step には「完了条件」があり、それを満たしたら次の Step へ進む。

---

## 0. このプロジェクトは何か

LLM 提供元（OpenAI / Google DeepMind / Anthropic）の公式ブログ・リリース情報を毎日自動収集し、出典リンクを主役にした時系列一覧として公開するアグリゲータ。

**提供価値**: AI 生成の粗悪な二次記事を排し、「一次ソースの新着を、出典・公開日つきで網羅的かつ鮮度高く一覧する」。

**重要な制約（恒久）**: 記事本文は転載しない。保持するのは **タイトル・出典・公開日・原文リンク・概要（OG description）のみ**。常に原文へ誘導する。

---

## 1. 確定済みの技術スタック（変更しないこと）

この構成は議論の末に確定済み。実装者は勝手に別技術へ置き換えない。変更が必要と判断した場合は、理由を添えて利用者に確認すること。

| 層 | 採用技術 | 補足 |
| --- | --- | --- |
| 言語 | TypeScript | 全層で Article 型を共有 |
| 実行基盤 | Cloudflare Workers | 完全無料枠。常駐しないサーバーレス |
| スケジュール | Cloudflare Cron Triggers | 日次起動。**Phase 3 で導入**（Phase 1-2 は手動実行） |
| DB | Cloudflare D1（SQLite） | `INSERT ... ON CONFLICT(url) DO UPDATE` で冪等 upsert |
| HTTP 取得 | Workers 標準 `fetch` | undici 等は使わない |
| HTML メタ抽出 | **HTMLRewriter**（Cloudflare 標準） | cheerio は Workers で動作保証されないため使わない |
| sitemap パース | **正規表現で `<loc>`/`<lastmod>` 抽出** | XML パーサライブラリは持ち込まない（最低限主義） |
| RSS パース | `rss-parser`（**Phase 2 で導入**） | OpenAI / DeepMind 用。Phase 1 では未使用 |
| バリデーション | Zod | 外部データを共通スキーマに正規化する境界（collector）で使う。api/web は型として import |
| API | Hono | Workers 第一級対応。`GET /articles` |
| フロント | Next.js + OpenNext（`@opennextjs/cloudflare`） | 一覧 1 ページのみ |
| パッケージ管理 | pnpm（モノレポ） | collector/api/web が型を共有 |
| デプロイ CLI | wrangler | ローカル実行は `wrangler dev`、ローカル D1 は `--local` |

Docker / docker-compose は使わない（サーバーレス整合のため、ローカルも wrangler で完結）。

### Cloudflare 無料枠の正確な制約（設計の根拠）

数値を誤認すると無料枠の前提が崩れるため、経路ごとに分けて記す（2026 年時点・要再確認は §9）。

- **Workers リクエスト数**: 100,000 req/日（無料）。本プロジェクトの負荷では桁違いに余裕。
- **CPU 時間（経路で異なる・重要）**:
  - **HTTP リクエスト経路**（api / web へのアクセス）: 無料枠は **CPU 10 ms/リクエスト（固定・引き上げ不可）**。30 秒は Paid プランの既定値であり無料では使えない。ネットワーク待ち（fetch・D1 の I/O）は CPU 時間に算入されないため通常は問題ないが、Zod 検証や整形を重くすると 10 ms に当たりうる。api/web は処理を軽く保つ。
  - **Cron Triggers 経路**（collector の日次起動）: **Duration（実時間）は最大 15 分**まで許容される（HTTP 経路に実時間上限はないが、Cron は 15 分で打ち切られる）。ただし **CPU 時間は無料枠では HTTP と同じく 10 ms** で、Cron でも緩和されない（緩和は Paid のみ）。fetch・D1 の I/O 待ちは CPU に算入されないため、待ち時間が大半の収集は 15 分の Duration 枠内で動かせるが、**正味の CPU 計算量（正規表現抽出・HTMLRewriter・Zod 正規化の総和）は 10 ms に収める**必要がある。
- **subrequest（外部 fetch）**: 無料枠 **50/リクエスト**。段階 2 で記事ごとに fetch するため、1 回の実行で全 219 件は処理できない（→ Phase 1 は件数を絞る）。
- **D1**: アカウント合計ストレージ **5 GB**、**1 データベースあたり 500 MB**、書込 **100,000 行/日**、読取 500 万行/日、**50 queries/Worker invocation**。最後の「50 queries/invocation」は subrequest 50 とは別の制約で、段階 2 で記事ごとに upsert する設計では件数次第で当たりうる（→ Phase 1 の件数絞りで回避、Phase 3 でバッチ化を検討）。
- **Cron Triggers**: アカウントあたり **5 本**（無料）。

**結論**: 収集（collector）は Cron 経路（Duration 15 分枠）で動かすため fetch 待ちは余裕だが、**正味の CPU は 10 ms 上限**なので計算量を抑える。api/web も HTTP 経路で CPU 10 ms なので処理を軽く保つ。Phase 1 で収集件数を絞るのは、subrequest 50・D1 queries 50/invocation・CPU 10 ms の各制約を避けるため。

---

## 2. スコープ（MVP に入れる / 入れない）

### MVP に入れる（In Scope）

- 3 社（OpenAI / DeepMind / Anthropic）の公式ブログを毎日 1 回自動収集
- 収集器は **ソース別 Adapter で抽象化**し、共通スキーマに正規化
- **重複排除**（url を unique キーに upsert。再実行で重複しない冪等な取り込み）
- 収集結果を D1 に永続化
- 分類は **ソース別のみ**（タグ・カテゴリなし）
- 公開 UI は **時系列一覧の 1 ページのみ**（タイトル・出典・公開日・原文リンク、公開日降順）
- クラウドで毎日自動収集・公開が回る状態（= MVP 完成）

### MVP に入れない（Out of Scope = 拡張送り）

AI 要約 / 記事のタグ・カテゴリ分類 / ソース絞り込み UI / 全文検索 / ユーザー登録・ログイン・お気に入り / 通知 / 管理画面 / arXiv 等の学術ソース / 本文の全文保存・転載。

---

## 3. 共通データモデル

全層で共有する記事の正規化スキーマ。Zod スキーマ＋ TS 型として `packages/core` に定義する。**collector は実行時検証に使い、api / web は型として import するのみ**（実行時検証は collector の境界に限定）。

```ts
// NormalizedArticle（収集器の出力 = DB の 1 行 = API のレスポンス要素）
{
  source: 'anthropic' | 'openai' | 'deepmind'  // 出典
  title: string                                 // 記事タイトル（必須）
  url: string                                   // 原文 URL（必須・ユニークキー）
  publishedAt: string                           // ISO8601。Anthropic は sitemap の lastmod を充当
  summary: string | null                        // OG description 等。欠損可
}
```

**フィールド名の対応（camelCase ⇔ snake_case）**: TS の `NormalizedArticle` は camelCase、D1 テーブルは SQLite 慣習で snake_case。境界で変換する。

| NormalizedArticle (TS) | articles テーブル (D1) |
| --- | --- |
| `source` | `source` |
| `title` | `title` |
| `url` | `url` |
| `publishedAt` | `published_at` |
| `summary` | `summary` |

§0 の「保持するフィールド」との対応: 出典=`source`、タイトル=`title`、原文リンク=`url`、公開日=`publishedAt`、概要=`summary`。過不足なし。

D1 テーブル定義（SQLite）:

```sql
CREATE TABLE IF NOT EXISTS articles (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  source      TEXT NOT NULL,
  title       TEXT NOT NULL,
  url         TEXT NOT NULL UNIQUE,         -- 重複排除のキー
  published_at TEXT NOT NULL,               -- ISO8601 文字列
  summary     TEXT,                         -- NULL 許容
  created_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))  -- ISO8601 で統一
);
CREATE INDEX IF NOT EXISTS idx_articles_published_at ON articles(published_at DESC);
```

> `created_at` は `published_at`（ISO8601・`...Z`）とフォーマットを揃えるため `strftime` で生成する。

冪等 upsert（再実行で重複せず、内容が変われば更新）:

```sql
INSERT INTO articles (source, title, url, published_at, summary)
VALUES (?, ?, ?, ?, ?)
ON CONFLICT(url) DO UPDATE SET
  title = excluded.title,
  published_at = excluded.published_at,
  summary = excluded.summary;
```

---

## 4. 収集器の設計（この題材の核）

「収集器」＝ `apps/collector`（収集 Worker）と、その中核ロジックである `packages/core` の各 Adapter を指す。本書では「収集器」「collector」を同一物として扱う。

ソースごとに収集方式が異なる（RSS / sitemap+OG メタ）。これを **Adapter パターン**で吸収する。共通インターフェースの裏に各ソースの実装を置き、出力を `NormalizedArticle[]` に正規化する。

```ts
interface SourceAdapter {
  readonly source: 'anthropic' | 'openai' | 'deepmind';
  fetchArticles(): Promise<NormalizedArticle[]>;
}
```

MVP の最初の実装対象は **Anthropic（最難ソース）**。これを Phase 1 で先に貫通させ、Phase 2 で OpenAI / DeepMind（RSS）を「Adapter を足すだけ」で追加する。

> **設計判断（重要）**: `AnthropicSitemapAdapter` は内部で **HTMLRewriter（Workers ランタイム API）** を使う。そのため `packages/core` は「Workers ランタイム前提のコードを含むライブラリ」になる（純 Node では動かない）。検証・実行は常に Worker（`wrangler dev`）経由で行う（§8 Step 1 参照）。

### Anthropic の収集方式（実データで検証済み）

Anthropic には RSS が存在しない（`rss.xml` 等は全パス 404 を確認済み）。`robots.txt` は `Allow: /` で全面許可、`sitemap.xml` を明示している。収集は 2 段階:

**段階 1: sitemap から記事 URL 一覧を得る**
- URL: `https://www.anthropic.com/sitemap.xml`（HTTP 200・`application/xml`）
- `/news/` を含む `<loc>` が記事。検証時点で **219 件**
- `<url>` 要素の構造（実データ）:
  ```xml
  <url>
    <loc>https://www.anthropic.com/news/100k-context-windows</loc>
    <lastmod>2025-05-02T00:49:29.000Z</lastmod>
  </url>
  ```
- **正規表現で `<loc>` と直後の `<lastmod>` を抽出**する（XML パーサは使わない）。`lastmod` を `publishedAt` に充当する（記事ページに公開日メタが無いため）

**段階 2: 各記事 URL を fetch し、OG メタを HTMLRewriter で抽出**
- 各 `/news/...` URL を `fetch` し、レスポンス HTML を HTMLRewriter に通す
- 抽出対象（実データで確認済みの形）:
  ```html
  <meta property="og:title" content="Introducing Claude Opus 4.8"/>
  <meta property="og:description" content="Our latest model, Claude Opus 4.8, is an upgrade ..."/>
  ```
- `og:title` → `title`、`og:description` → `summary`
- HTMLRewriter のハンドラ例: `.on('meta[property="og:title"]', ...)` で `element.getAttribute('content')` を読む
- **注意**: `article:published_time` メタは存在しない。JSON-LD も `__NEXT_DATA__` も無い。公開日は段階 1 の `lastmod` のみが頼り
- **注意**: `og:description` が欠損する記事がありうる。その場合 `summary` は `null` とする（フォールバックで本文を取りに行かない）

### 収集時の運用上の注意

- HTTP は HTTPS 必須（`http://` はリダイレクトされ本文が空になる）。`fetch` はリダイレクト追従する
- 段階 2 は記事数ぶん fetch が走る。**subrequest 上限 50/request と D1 queries 50/invocation の両方**に当たるため、**Phase 1 では先頭 N 件（既定 10 件）に絞って実装・検証**する。N は **環境変数または定数 1 箇所に集約**し、Phase 3 で全件化する際にそこだけ変えればよいようにする
- 逐次 fetch（小さい並列度・適度な待機）でレート制御の入口を作る
- 日付フォーマットは ISO8601 に正規化する（Anthropic の lastmod は既に ISO8601）

---

## 5. リポジトリ構成（最終形）

各ディレクトリを **どの Step で作るか** を注記する。一気に全部作らず Step に従う。

```
llm-source-feed/
├─ apps/
│  ├─ web/        # (Step 4) Next.js (OpenNext) 一覧ページ → api を fetch
│  ├─ api/        # (Step 3) Hono Worker. GET /articles. D1 バインディング読み取り
│  └─ collector/  # (Step 1 で雛形作成) 収集 Worker. wrangler dev で起動し core の Adapter を呼ぶ。
│                 #          Step 2 で D1 upsert を追加。Cron 化は Phase 3
├─ packages/
│  ├─ core/       # (Step 0) SourceAdapter, NormalizedArticle(Zod型), 各 Adapter, 正規化ロジック
│  └─ db/         # (Step 2) D1 スキーマ(migrations), クエリヘルパ
├─ docs/
│  └─ IMPLEMENTATION_SPEC.md   # 本ファイル
├─ pnpm-workspace.yaml          # (Step 0)
├─ package.json                 # (Step 0)
├─ .gitignore
└─ README.md
```

データの流れは一方向: **collector が書き → api が読み → web が api を叩く**。web は D1 に直接触らず、必ず api 経由。

---

## 6. ロードマップ（Phase 構成）

- **Phase 1**: Anthropic 単独で、収集 → D1 → API → 一覧ページ → デプロイ を **レイヤーごとに**構築（下記 **Step 0〜5**。Step 0 の土台を含む）
- **Phase 2**: OpenAI / DeepMind の RSS Adapter を追加（`rss-parser` をここで導入）。collector を全 Adapter を回すオーケストレーションに
- **Phase 3**: Cron Triggers で日次自動化 / 構造化ログ（Workers のログ or 構造化出力）/ エラー耐性（1 ソース失敗で全体が止まらない）/ リトライ / 全件・分割収集（subrequest 50・D1 queries 50/invocation を踏まえバッチ分割）/ CI(GitHub Actions) →【MVP 完成】

---

## 7. 前提（Prerequisites）— Step 0 着手前に整える

実装に入る前に、以下が揃っていることを確認する。揃っていなければここで用意する。

- **Node.js 20 以上** と **pnpm**（`npm i -g pnpm` 等）。バージョンは各 Step 着手時に wrangler / OpenNext の要件を満たすか確認
- **Cloudflare アカウント**（無料）。D1 作成・デプロイに必要
- **wrangler の認証**: `pnpm dlx wrangler login`（ブラウザ OAuth）。CI 等では `CLOUDFLARE_API_TOKEN` 環境変数でも可。
  - ローカルのみで完結する作業（`wrangler dev`、`--local` の D1 操作）は認証不要。
  - **リモートに触れる操作**（`wrangler d1 create`、`wrangler deploy`）は認証必須。Step 2 と Step 5 で必要になる。
- このリポジトリは作成済み（`~/Projects/llm-source-feed`、GitHub 連携済み）

各 Step の冒頭に「ローカルのみ / リモート認証が必要」を明記する。

---

## 8. Phase 1 — Step 単位の実装手順

Phase 1 は Anthropic 1 社のみ。各 Step は前の Step の完了条件を満たしてから着手する。一気通貫ではなく、レイヤーを 1 つずつ積み上げて各層を独立に検証する。

### Step 0 — 土台（最小限）  ［ローカルのみ］

**作るもの**: `pnpm-workspace.yaml` / ルート `package.json` / ルート `tsconfig.json` / `packages/core`

**やること**
- pnpm モノレポ初期化（`pnpm-workspace.yaml` に `apps/*`・`packages/*`、ルート `package.json`）
- TypeScript 設定（ルート `tsconfig.json`、各パッケージで extends）
- `packages/core` を作成し、`NormalizedArticle` の Zod スキーマと TS 型、`SourceAdapter` インターフェースを定義
- wrangler を devDependency に追加（Worker 用の設定は Step 1 で）

**完了条件**: `pnpm install` が通り、かつ `pnpm -F core exec tsc --noEmit`（または同等の型チェック）が通って `packages/core` の型が利用できる。

### Step 1 — Anthropic 収集ロジック単体（DB なし）  ［ローカルのみ］

**作るもの**: `packages/core` の `AnthropicSitemapAdapter` ＋ `apps/collector`（Worker 雛形）

**構成の確定（重要・ここで迷わないこと）**:
- HTMLRewriter は Workers ランタイム API のため **純 Node では動かない**。よってこの Step で `apps/collector` を Worker として作り、`wrangler dev` で起動して検証する。純 Node のスタンドアロンスクリプトでは検証しない。
- `apps/collector` に最小の `wrangler.toml` と Worker エントリ（`fetch` ハンドラ）を置く。`fetch` ハンドラ内で `packages/core` の `AnthropicSitemapAdapter.fetchArticles()` を呼び、結果の配列を **JSON でレスポンス返却**（兼ログ出力）する。
- HTMLRewriter は `packages/core` の Adapter 内で直接使ってよい（§4 の設計判断のとおり core は Workers 前提ライブラリになる）。

**やること**
- `AnthropicSitemapAdapter` を実装（`SourceAdapter` を満たす）
  1. `sitemap.xml` を fetch → 正規表現で `<loc>`（`/news/` のみ）と対応する `<lastmod>` を抽出
  2. **先頭 N 件（既定 10 件・定数/env 1 箇所に集約）に絞る**（Phase 1 の検証用。全件は Phase 3）
  3. 各記事 URL を fetch → HTMLRewriter で `og:title` / `og:description` を抽出
  4. `lastmod` を `publishedAt` に充当し、Zod で `NormalizedArticle` に正規化（`og:description` 欠損時は `summary: null`）
  5. 逐次 fetch（レート制御の入口。適度な待機を挟む）
- `apps/collector` の Worker `fetch` ハンドラから上記を呼び、記事配列を JSON で返す

**完了条件**: `pnpm -F collector exec wrangler dev` で起動し、`curl http://localhost:8787/`（ポートは起動ログに従う、既定 8787）で Anthropic の記事配列（source/title/url/publishedAt/summary）が JSON で返る。

### Step 2 — D1 永続化  ［D1 作成・マイグレーションでリモート認証が必要］

**作るもの**: `packages/db`（スキーマ/マイグレーション）＋ `apps/collector` に D1 バインディングと upsert を追加

**やること**
- `packages/db` に D1 スキーマ（§3 の `CREATE TABLE` / `CREATE INDEX`）を migration ファイルとして用意
- `wrangler d1 create <db-name>` で D1 を作成（**リモート認証必要**）し、出力された `database_id` を `apps/collector/wrangler.toml` の `[[d1_databases]]`（binding 名 `DB`）に記載
- ローカル D1 にマイグレーション適用: `wrangler d1 migrations apply <db-name> --local`
- Step 1 の収集結果を、§3 の `ON CONFLICT(url) DO UPDATE` で D1 に upsert する処理を collector に追加（収集 → 正規化 → upsert を Worker 内で続けて実行）。D1 アクセスは `env.DB.prepare(sql).bind(...).run()`
- ローカル（`--local`）で collector 実行 → 保存を確認。件数確認は `wrangler d1 execute <db-name> --local --command "SELECT COUNT(*) FROM articles"`

**完了条件**: collector をローカル実行すると D1 に記事が入り、**2 回実行しても件数が増えない**（`SELECT COUNT(*)` が同じ）。冪等性が確認できる。

### Step 3 — 読み取り API（Hono Worker）  ［ローカルのみ］

**作るもの**: `apps/api`（Hono Worker）

**やること**
- `apps/api` に Hono の Worker を作成し、同じ D1 を `wrangler.toml` にバインド（binding 名 `DB`）
- `GET /articles` を実装: `env.DB.prepare("SELECT source,title,url,published_at,summary FROM articles ORDER BY published_at DESC").all()` を実行する。`.all()` の戻り値は `{ success, meta, results }` 構造で、記事行は **`results` 配列**に入る。戻り値全体をそのまま返すと meta が混入するため、**`results` を取り出し、snake_case → camelCase に変換した配列**を JSON で返す
- web からローカルで叩けるよう **CORS を許可**（Hono の cors ミドルウェア等。MVP は緩く許可で可）
- `wrangler dev` で起動し `curl` で確認

**完了条件**: `curl http://localhost:<port>/articles`（ポートは起動ログに従う）で、D1 の Anthropic 記事が公開日降順の JSON（camelCase）で返る。

### Step 4 — フロント一覧（Next.js + OpenNext）  ［ローカルのみ］

**作るもの**: `apps/web`（Next.js + OpenNext）

**やること**
- `apps/web` に Next.js（App Router）を作成、`@opennextjs/cloudflare` でビルド/実行できるよう設定（手順は OpenNext 公式で最新確認）
- 一覧ページ（1 ページ）で `GET /articles` を fetch し、タイトル・出典・公開日・原文リンクを公開日降順で表示
- API の接続先 URL は **環境変数で切替**（ローカルは Step 3 の `wrangler dev` ポート、本番は api Worker の URL）
- 原文リンクは新規タブ・`rel="noopener"`。本文は載せない（リンクのみ）
- ローカル起動手段（`next dev` か OpenNext のプレビュー）は OpenNext 公式の現行手順に従う

**完了条件**: ローカルブラウザで一覧が表示され、原文リンクが機能する。

### Step 5 — デプロイ（手動実行）  ［リモート認証が必要］

**やること**
- 本番 D1 にマイグレーション適用: `wrangler d1 migrations apply <db-name> --remote`
- collector / api / web を Cloudflare（Workers + D1）にデプロイ（各 `wrangler deploy`、web は OpenNext のデプロイ手順）
- collector を **手動 trigger** で 1 回実行（Step 1 で `fetch` ハンドラにしてあるので、デプロイ後の collector Worker URL を `curl` で叩く）→ 本番 D1 に保存 → web 一覧に出るまで確認
- （Cron Triggers での日次自動化は Phase 3。ここではまだ手動）

**完了条件**: クラウド上で collector の URL を叩くと、Anthropic 記事が本番 D1 に保存され、公開 web 一覧に出る。

---

## 9. 実装時の確認事項（要検証・未確定）

実装中にこれらに当たったら、推測で進めず検証する。

- **sitemap の正規表現抽出**: `<loc>` と `<lastmod>` の対応付け。要素の順序・改行・空白に依存しない抽出にする。`/news/` フィルタを忘れない
- **HTMLRewriter のストリーミング特性**: テキストノードは分割されうるが、og:* は属性値（`content`）なので `getAttribute('content')` で 1 回で取れる。問題が出たら Cloudflare ドキュメントの HTMLRewriter 仕様を確認
- **subrequest 50/request と D1 queries 50/invocation**: Phase 1 で件数を絞る理由。全件処理は Phase 3 でバッチ分割や複数回実行に分ける
- **CPU 時間（無料枠は経路を問わず 10 ms）**: api/web（HTTP 経路）も collector（Cron 経路）も CPU は 10 ms 上限。Cron で緩むのは Duration（実時間 15 分）であって CPU ではない。fetch/D1 の I/O 待ちは CPU 非算入なので収集は回るが、計算処理は軽く保つ。§1 の数値が最新か、着手時に公式で再確認
- **wrangler CLI の最新フラグ**: 近年の wrangler はローカル実行が既定で `--local`/`--remote` の扱いが変わりうる。`wrangler d1 create` / `migrations apply` / `dev` / `deploy` のフラグは着手時に wrangler 公式で確認
- **OpenNext / D1 のバージョン依存**: OpenNext の Next.js 対応バージョン、compatibility date、`nodejs_compat` フラグを公式で確認

---

## 10. 検証で確認済みの一次情報（参考）

- Anthropic: RSS 無し（`rss.xml`/`feed`/`atom.xml` 等 404）。`robots.txt` は `Allow: /`＋sitemap 明示。`sitemap.xml` の `/news/` は 219 件、各 `<url>` は `<loc>`+`<lastmod>` 構造。記事ページに `og:title`/`og:description` あり、`article:published_time`・JSON-LD・`__NEXT_DATA__` は無し
- Cloudflare 無料枠（developers.cloudflare.com で確認）: Workers 10 万 req/日、**CPU は HTTP 経路・Cron 経路とも 10 ms 固定（30 秒は Paid 既定）。Cron は Duration（実時間）15 分まで動けるが CPU は 10 ms のまま**、subrequest 50/request、Cron Triggers 5 本、D1 5 GB（1 DB 500 MB・書込 10 万行/日・**50 queries/invocation**）、D1 で `ON CONFLICT` upsert 可、HTMLRewriter で `getAttribute('content')` によるメタ抽出可、Next.js は OpenNext 推奨
- OpenAI / DeepMind（Phase 2 用）: それぞれ `openai.com/blog/rss.xml`、`deepmind.google/blog/rss.xml` が RSS（HTTP 200）。pubDate あり。DeepMind は description 欠損の記事あり。公式ブログには製品リリース以外（事例・広報）も混ざるため、将来的に分類フィルタの余地がある（MVP では分類しない）

---

## 11. 実装者への指示

- **この仕様書の §1 スタックと §2 スコープは確定事項**。逸脱する場合は理由を添えて利用者に確認する
- **着手前に「前提（Prerequisites）」節を満たす**（Cloudflare アカウント・wrangler 認証・Node/pnpm）
- **Step は順番に**。各 Step の「完了条件」を満たしてから次へ進む。一度に全層を作らない
- 外部ライブラリの利用方法・API は、記憶で書かず公式ドキュメントで確認する（特に wrangler / D1 / HTMLRewriter / OpenNext / Hono）。§1 の Cloudflare 無料枠数値も着手時に再確認する
- 記事本文は決して保存・転載しない（タイトル・出典・公開日・原文リンク・OG description のみ）
- 進捗に応じて本ファイルの該当 Step に印を付けるか、別途進捗メモを残してよい
