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
| バリデーション | Zod | 外部データを共通スキーマに正規化する境界で使う |
| API | Hono | Workers 第一級対応。`GET /articles` |
| フロント | Next.js + OpenNext（`@opennextjs/cloudflare`） | 一覧 1 ページのみ |
| パッケージ管理 | pnpm（モノレポ） | collector/api/web が型を共有 |
| デプロイ CLI | wrangler | ローカル実行は `wrangler dev`、ローカル D1 は `--local` |

**コストは完全無料**（Workers 10万req/日・D1 5GB・Cron 5本。今回の負荷に対し桁違いの余裕）。Docker / docker-compose は使わない（サーバーレス整合のため、ローカルも wrangler で完結）。

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

全層で共有する記事の正規化スキーマ。Zod スキーマ＋ TS 型として `packages/core` に定義し、collector / api / web が import する。

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

D1 テーブル定義（SQLite）:

```sql
CREATE TABLE IF NOT EXISTS articles (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  source      TEXT NOT NULL,
  title       TEXT NOT NULL,
  url         TEXT NOT NULL UNIQUE,         -- 重複排除のキー
  published_at TEXT NOT NULL,               -- ISO8601 文字列
  summary     TEXT,                         -- NULL 許容
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_articles_published_at ON articles(published_at DESC);
```

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

ソースごとに収集方式が異なる（RSS / sitemap+OG メタ）。これを **Adapter パターン**で吸収する。共通インターフェースの裏に各ソースの実装を置き、出力を `NormalizedArticle[]` に正規化する。

```ts
interface SourceAdapter {
  readonly source: 'anthropic' | 'openai' | 'deepmind';
  fetchArticles(): Promise<NormalizedArticle[]>;
}
```

MVP の最初の実装対象は **Anthropic（最難ソース）**。これを Phase 1 で先に貫通させ、Phase 2 で OpenAI / DeepMind（RSS）を「Adapter を足すだけ」で追加する。

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
- 段階 2 は記事数ぶん fetch が走るため、**レート制御**（逐次 or 小さい並列度、適度な待機）を入れる。Workers 無料枠の subrequest 上限は 50/request なので、1 回の実行で全 219 件を直列 fetch すると上限に当たる。**Phase 1 では先頭 N 件（例: 10 件）に絞って実装・検証**し、全件処理・分割は Phase 3 のレート制御で扱う
- 日付フォーマットは ISO8601 に正規化する（Anthropic の lastmod は既に ISO8601）

---

## 5. リポジトリ構成（最終形）

```
llm-source-feed/
├─ apps/
│  ├─ web/        # Next.js (OpenNext) 一覧ページ → api を fetch
│  ├─ api/        # Hono Worker. GET /articles. D1 バインディング読み取り
│  └─ collector/  # 収集 Worker. Cron で日次起動（Phase 3）。D1 へ upsert
├─ packages/
│  ├─ core/       # SourceAdapter, NormalizedArticle(Zod型), 各 Adapter, 正規化ロジック
│  └─ db/         # D1 スキーマ(migrations), クエリヘルパ
├─ docs/
│  └─ IMPLEMENTATION_SPEC.md   # 本ファイル
├─ pnpm-workspace.yaml
├─ package.json
├─ .gitignore
└─ README.md
```

データの流れは一方向: **collector が書き → api が読み → web が api を叩く**。web は D1 に直接触らず、必ず api 経由。

> 注: 一気にこの全構成を作らず、Step に従って必要な部分から作る。Phase 1 は Anthropic 単独。

---

## 6. ロードマップ（Phase 構成）

- **Phase 1**: Anthropic 単独で、収集 → D1 → API → 一覧ページ → デプロイ を **レイヤーごとに**構築（下記 Step 1〜5）
- **Phase 2**: OpenAI / DeepMind の RSS Adapter を追加（`rss-parser` をここで導入）。collector を全 Adapter を回すオーケストレーションに
- **Phase 3**: Cron Triggers で日次自動化 / 構造化ログ（`pino` 相当 or Workers のログ）/ エラー耐性（1 ソース失敗で全体が止まらない）/ リトライ / 全件・分割収集 / CI(GitHub Actions) →【MVP 完成】

---

## 7. Phase 1 — Step 単位の実装手順

Phase 1 は Anthropic 1 社のみ。各 Step は前の Step の完了条件を満たしてから着手する。一気通貫ではなく、レイヤーを 1 つずつ積み上げて各層を独立に検証する。

### Step 0 — 土台（最小限）

**やること**
- pnpm モノレポ初期化（`pnpm-workspace.yaml`、ルート `package.json`）
- TypeScript 設定（ルート `tsconfig.json`、各パッケージで extends）
- `packages/core` を作成し、`NormalizedArticle` の Zod スキーマと TS 型、`SourceAdapter` インターフェースを定義
- wrangler をインストール（collector 用の最小 `wrangler.toml` は Step 2 で D1 を足す時に整える）

**完了条件**: `pnpm install` が通り、`packages/core` から型が import できる。

### Step 1 — Anthropic 収集ロジック単体（DB なし）

**やること**
- `packages/core` に `AnthropicSitemapAdapter` を実装（`SourceAdapter` を満たす）
  1. `sitemap.xml` を fetch → 正規表現で `<loc>`（`/news/` のみ）と対応する `<lastmod>` を抽出
  2. **先頭 N 件（例: 10 件）に絞る**（Phase 1 の検証用。全件は Phase 3）
  3. 各記事 URL を fetch → HTMLRewriter で `og:title` / `og:description` を抽出
  4. `lastmod` を `publishedAt` に充当し、Zod で `NormalizedArticle` に正規化（`og:description` 欠損時は `summary: null`）
  5. 逐次 fetch（レート制御の入口。適度な待機を挟む）
- スタンドアロン実行スクリプトで、収集結果の配列を **コンソールに整形出力**（DB 保存はしない）

**完了条件**: ローカル実行で、Anthropic の記事配列（source/title/url/publishedAt/summary）がコンソールに整形表示される。

> HTMLRewriter は Workers ランタイム API。ローカル単体実行では `wrangler dev` 上の Worker として動かす、もしくは Workers 環境で実行する形にする。HTMLRewriter が使えない純 Node 実行で試す場合は、この Step だけ Worker として `wrangler dev` で叩く構成にしてよい。

### Step 2 — D1 永続化

**やること**
- `packages/db` に D1 スキーマ（§3 の `CREATE TABLE`）を migration として用意
- wrangler で D1 データベースを作成し、collector の `wrangler.toml` に D1 バインディング（例: binding 名 `DB`）を設定
- Step 1 の収集結果を、§3 の `ON CONFLICT(url) DO UPDATE` で D1 に upsert
- ローカル D1（`--local`）でマイグレーション適用 → collector 実行 → 保存を確認

**完了条件**: collector をローカル実行すると D1 に記事が入り、**2 回実行しても件数が増えない**（冪等性が確認できる）。

### Step 3 — 読み取り API（Hono Worker）

**やること**
- `apps/api` に Hono の Worker を作成
- `GET /articles` を実装: D1 バインディング経由で `SELECT ... ORDER BY published_at DESC` を返す（JSON）
- `wrangler dev` で起動し、`curl` で確認

**完了条件**: `curl http://localhost:<port>/articles` で、D1 の Anthropic 記事が公開日降順の JSON で返る。

### Step 4 — フロント一覧（Next.js + OpenNext）

**やること**
- `apps/web` に Next.js（App Router）を作成、`@opennextjs/cloudflare` でビルド/実行できるよう設定
- 一覧ページ（1 ページ）で `GET /articles` を fetch し、タイトル・出典・公開日・原文リンクを公開日降順で表示
- 原文リンクは新規タブ・`rel="noopener"` 推奨。本文は載せない（リンクのみ）

**完了条件**: ローカルブラウザで一覧が表示され、原文リンクが機能する。

### Step 5 — デプロイ（手動実行）

**やること**
- collector / api / web を Cloudflare（Workers + D1）にデプロイ（`wrangler deploy`）
- 本番 D1 にマイグレーション適用
- collector を **手動 trigger** で 1 回実行 → 本番 D1 に保存 → web 一覧に出るまで確認
- （Cron Triggers での日次自動化は Phase 3。ここではまだ手動）

**完了条件**: クラウド上で collector を手動実行すると、Anthropic 記事が公開 web 一覧に出る。

---

## 8. 実装時の確認事項（要検証・未確定）

実装中にこれらに当たったら、推測で進めず検証する。

- **sitemap の正規表現抽出**: `<loc>` と `<lastmod>` の対応付け。要素の順序・改行・空白に依存しない抽出にする。`/news/` フィルタを忘れない
- **HTMLRewriter のストリーミング特性**: テキストノードは分割されうるが、og:* は属性値（`content`）なので `getAttribute('content')` で 1 回で取れる。問題が出たら Cloudflare ドキュメントの HTMLRewriter 仕様を確認
- **subrequest 上限（無料枠 50/request）**: Phase 1 で件数を絞る理由。全件処理は Phase 3 で分割や複数回実行に分ける
- **ローカルでの HTMLRewriter 実行**: HTMLRewriter は Workers ランタイム API のため、純 Node では動かない。Step 1 の検証は `wrangler dev` 上の Worker として行う
- **wrangler / OpenNext / D1 のバージョン依存**: セットアップ時に各公式ドキュメントで最新手順を確認（特に OpenNext の Next.js 対応バージョン、compatibility date、`nodejs_compat` フラグ）

---

## 9. 検証で確認済みの一次情報（参考）

- Anthropic: RSS 無し（`rss.xml`/`feed`/`atom.xml` 等 404）。`robots.txt` は `Allow: /`＋sitemap 明示。`sitemap.xml` の `/news/` は 219 件、各 `<url>` は `<loc>`+`<lastmod>` 構造。記事ページに `og:title`/`og:description` あり、`article:published_time`・JSON-LD・`__NEXT_DATA__` は無し
- Cloudflare 無料枠: Workers 10万req/日・CPU 30 秒（cron 起動時 wall-clock 15 分）・subrequest 50/request、Cron Triggers 5 本、D1 5GB・書込 10万行/日、D1 で `ON CONFLICT` upsert 可、HTMLRewriter でメタ抽出可、Next.js は OpenNext 推奨
- OpenAI / DeepMind（Phase 2 用）: それぞれ `openai.com/blog/rss.xml`、`deepmind.google/blog/rss.xml` が RSS（HTTP 200）。pubDate あり。DeepMind は description 欠損の記事あり。公式ブログには製品リリース以外（事例・広報）も混ざるため、将来的に分類フィルタの余地がある（MVP では分類しない）

---

## 10. 実装者への指示

- **この仕様書の §1 スタックと §2 スコープは確定事項**。逸脱する場合は理由を添えて利用者に確認する
- **Step は順番に**。各 Step の「完了条件」を満たしてから次へ進む。一度に全層を作らない
- 外部ライブラリの利用方法・API は、記憶で書かず公式ドキュメントで確認する（特に wrangler / D1 / HTMLRewriter / OpenNext / Hono）
- 記事本文は決して保存・転載しない（タイトル・出典・公開日・原文リンク・OG description のみ）
- 進捗に応じて本ファイルの該当 Step に印を付けるか、別途進捗メモを残してよい
