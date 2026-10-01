# Google Cloud セットアップ手順（分析用の OAuth クライアントを作るまで）

作成日: 2026-10-01 ／ 方針: [ADR-0006](../adr/0006-google-cloud-oauth.md)

> この手順は CLAUDE.md の「手動操作の例外」に含まれる。ここに書いたもの以外を Google Cloud のコンソールで作らない。
> 画面の文言や配置はよく変わる。食い違ったときは、各 Step の公式ドキュメントを優先する。

## ゴール

test-app と別の Google Cloud プロジェクトに、読み取り専用のスコープだけを持つ OAuth クライアントを作る。同意画面は本番にする。
所要時間は15〜20分ほど。

完了時の構成:

```
Google Cloud プロジェクト（分析用。test-app とは別）
├─ 有効にする API
│    ├─ YouTube Analytics API
│    ├─ YouTube Reporting API
│    └─ YouTube Data API v3
├─ Google Auth Platform（同意画面）…… 外部、本番、未審査
│    └─ スコープ: yt-analytics.readonly、youtube.readonly
└─ OAuth クライアント …… 種類はデスクトップアプリ
```

## 注意

- **test-app のプロジェクトを選んだまま作業しない。** 画面上部のプロジェクト名を、Step ごとに確かめる
- クライアントシークレットの JSON は、リポジトリの `secrets/` に置く。`.gitignore` の `secrets/` と `client_secret*.json` で除外されることを、`git status` で確かめる
- プロジェクトID、クライアントID、シークレットを、ドキュメントやコミットに書かない

---

## Step 1. プロジェクトを作る

1. Google Cloud コンソールで、プロジェクトの選択から「新しいプロジェクト」を選ぶ
2. 名前は用途がわかるものにする（例: `yt-analytics`）。組織は「組織なし」でよい
3. 作成後、画面上部でそのプロジェクトが選ばれていることを確かめる

公式: https://cloud.google.com/resource-manager/docs/creating-managing-projects

## Step 2. API を有効にする

「API とサービス」→「ライブラリ」で、次の3つを検索して有効にする。

| API | 使い道 |
|---|---|
| YouTube Analytics API | 日次の指標、動画内の位置ごとの維持率 |
| YouTube Reporting API | インプレッション数、クリック率 |
| YouTube Data API v3 | 動画の一覧、メタデータ |

課金アカウントの紐付けは不要（どれも無料のクォータの範囲で使う）。

## Step 3. 同意画面を設定する

「Google Auth Platform」（旧称「OAuth 同意画面」）で設定する。

1. **ブランディング:** アプリ名（例: `yt-analytics`）、サポートメール、デベロッパーの連絡先を入れる。ロゴは設定しない（ロゴを入れると審査の対象になる）
   - **ホームページとプライバシーポリシーの URL も入れる。** 空欄のままだと、Step 3-4 の「アプリを公開」ボタンが無効になる（2026-10 時点）
     - ホームページ: このリポジトリの GitHub の URL
     - プライバシーポリシー: [`docs/privacy-policy.md`](../privacy-policy.md) の GitHub 上の URL
     - 承認済みドメインを求められたら `github.com`
   - URL が実在するように、先にリポジトリを GitHub に push しておく
2. **対象（Audience）:** ユーザーの種類は「外部」
3. **データアクセス:** 次の2つのスコープだけを追加する
   - `https://www.googleapis.com/auth/yt-analytics.readonly`
   - `https://www.googleapis.com/auth/youtube.readonly`
   - 収益（`yt-analytics-monetary.readonly`）、アップロード、管理のスコープは追加しない
4. **公開ステータス:** 「アプリを公開」で「本番環境」に切り替える。審査は申請しない

公式: https://developers.google.com/workspace/guides/configure-oauth-consent

## Step 4. OAuth クライアントを作る

1. 「Google Auth Platform」→「クライアント」→「クライアントを作成」
2. アプリケーションの種類は「デスクトップアプリ」。名前は例: `yt-analytics-local`
3. 作成後に JSON をダウンロードし、`secrets/client_secret.json` として保存する
4. リポジトリのルートで `git status` を実行し、`secrets/` が表示されないことを確かめる

公式: https://developers.google.com/identity/protocols/oauth2/native-app

## Step 5. 確認

| 確認すること | 期待する状態 |
|---|---|
| プロジェクト | test-app と別のプロジェクトになっている |
| 公開ステータス | 本番環境 |
| スコープ | 読み取り専用の2つだけ |
| シークレット | `secrets/client_secret.json` にあり、`git status` に出ない |

トークンの取得（認可）は、このあと作るスクリプトで行う。初回の認可では「Google でこのアプリは確認されていません」と表示されるので、「詳細」→「（アプリ名）に移動」で進める。
