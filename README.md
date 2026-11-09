# fav-like-ed-basic — X のいいね・リポスト・引用を EmDash に保存

X の「いいね」「リポスト（RT）」「引用」を取得し、**1ポスト＝1記事**として EmDash に保存する Cloudflare Workers プロジェクトです。画像・動画は EmDash の R2 に保存し、記事は EmDash の D1 を使って表示します。GitHub への Markdown 保存にも対応します。

EmDash サイト本体と同期 Worker は別々にデプロイします。WordPress は使用しません。

## 1. 必要な GitHub Actions Secrets

リポジトリの **Settings → Secrets and variables → Actions** に登録します。

### Cloudflare

| Secret | 内容 |
|---|---|
| `CLOUDFLARE_API_TOKEN` | Workers 編集、D1 編集、R2 編集権限の Cloudflare API Token |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare Account ID |

### X API

| Secret | 内容 |
|---|---|
| `X_USERNAME` | Xユーザー名（`@`なし） |
| `X_USER_ID` | 任意。設定するとユーザー名検索を省略 |
| `X_BEARER_TOKEN` | X API Bearer Token（RT・引用等の公開API用） |
| `X_API_KEY` | OAuth 1.0a のコンシューマーキー |
| `X_API_SECRET` | OAuth 1.0a のコンシューマーキーシークレット |
| `X_ACCESS_TOKEN` | OAuth 1.0a のアクセストークン |
| `X_ACCESS_SECRET` | OAuth 1.0a のアクセストークンシークレット |

「いいね」の取得には同一X Appから発行した OAuth 1.0a の4項目が必要です。OAuth 2.0 の Client ID / Client Secret はこの4項目の代わりにはなりません。

### EmDash と任意設定

| Secret | 内容 |
|---|---|
| `TRIGGER_SECRET` | 任意の十分に長いランダムな文字列。同期Workerの `/trigger` とEmDash APIを保護するため、**両方のActionで同じ値**を使用 |
| `GH_SYNC_TOKEN` | 任意。MarkdownをGitHubへ保存する場合のPersonal Access Token |
| `GH_SYNC_REPO` | 任意。Markdown保存先（例: `owner/repository`） |

## 2. EmDash サイトを作成

1. Actions から **1. Setup EmDash & R2/D1 (初回サイト構築)** を実行します。
2. 実行結果の Summary にEmDashサイトURLが表示されます。URLのSecret登録は不要です。同期WorkerのデプロイActionがCloudflareの `workers.dev` サブドメインからURLを自動生成します。

このActionは EmDash サイトをデプロイし、D1データベースとR2バケットを作成します。サイトのAPIを保護する場合は、実行前に `TRIGGER_SECRET` を登録してください。

## 3. favo-like Worker をデプロイ

Actions から **Deploy fav-like-ed to Cloudflare Workers** を実行します。`EmDash site name` はセットアップActionの `site_name` と同じ値にしてください（両方の初期値は `fl-ed`）。初回のデプロイとSecrets変更後は、このActionを再実行してください。

`workflow_dispatch` の入力で Cron 間隔（分）、取得対象（Likes / RT / Quotes）、GitHub Markdown出力対象を設定できます。Cronの初期値は15分間隔、X APIからの取得件数は有効な対象ごとに最新5件です。画像が多い場合はWorkerのサブリクエスト数が増えるため、まずは初期値で動作を確認してください。定期取得を使わず、手動で実行する場合はデプロイ後に次のように呼び出します。

```sh
curl -X POST \
  -H "Authorization: Bearer YOUR_TRIGGER_SECRET" \
  https://YOUR_WORKER.workers.dev/trigger
```

`TRIGGER_SECRET` を設定していない場合は `Authorization` ヘッダーを省略できます。秘密の値を公開リポジトリ、チャット、ログに貼らないでください。

## 保存形式

- 1ポストごとに独立した記事を作成します。
- EmDash上のSlugは `YYYY-MM-DD-x-favo-{like|rt|quote}-{post-id}` 形式です。日付が先頭にあるためEmDashサイトの日別・月別一覧に表示されます。
- 記事の再同期では既存Slugを検索し、重複作成を防止します。
- 画像・動画は EmDash の `/_emdash/api/media` API 経由で R2 に保存します。

## ローカルテスト

```sh
npm test
```
