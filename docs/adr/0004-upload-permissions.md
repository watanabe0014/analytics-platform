# ADR-0004: スナップショットのアップロードは、専用の権限セット `yt-uploader` で行う

- 日付: 2026-10-01
- 状態: 承認

## 背景

- スナップショットを S3 に書き込むには、書き込み権限が要る。既存のプロファイルは次の2つ
  - `yt-readonly`（ReadOnlyAccess）: Claude Code が使う。書き込めない
  - `yt-admin`（AdministratorAccess）: `terraform apply` のときだけ使う
- 日々のアップロードを `yt-admin` で行うと、次の問題がある
  - スクリプトの誤りで、想定と異なるバケットを操作しても止まらない（tfstate を消すこともできてしまう）
  - 認証情報が漏れたとき、アカウント全体が被害を受ける
  - フェーズ3でアップロードを自動化するとき、自動実行に管理者の権限を持たせることになる
- AWS アカウントは1つで、Organizations の管理アカウントを兼ねる。Identity Center は `ap-northeast-1` で有効にしてある（`docs/setup/aws-account-setup.md`）

## 決定

### 許可する操作（最小権限）

アップロードのスクリプトは、次の順で動く。

1. clips.db を SQLite のバックアップ機能で複製する（ローカル）
2. `.db` を `raw/` に書く
3. テーブルごとに Parquet に変換する（ローカル）
4. Parquet を `curated/` に書く

許可するのは、data バケットへの次の書き込みだけ。

| 場所 | 操作 |
|---|---|
| `raw/clips_db/snapshot_date=*/clips.db` | 書く（`s3:PutObject`） |
| `curated/clips_db/*/snapshot_date=*/*.parquet` | 書く（`s3:PutObject`） |

許可しないもの:

| 操作 | 理由 |
|---|---|
| 消す | 同じ日の再実行は、同じファイル名への上書きで済む。data バケットはバージョニングを有効にしているので、上書きしても前の版は残る |
| 一覧を見る | `aws s3 cp` や boto3 の `put_object` は一覧を見ずに書ける。一覧が必要な `aws s3 sync` は使わない |
| 読む | アップロードに不要 |
| tfstate、athena-results バケット | アップロードに不要 |

### 権限の持たせ方

- IAM Identity Center の権限セット `yt-uploader` を作り、上の内容をインラインポリシーとして付ける
- 権限セット、インラインポリシー、アカウントへの割り当ては Terraform で作る（`aws_ssoadmin_permission_set` など）。既存の2つの権限セットは、初期設定としてコンソールで作ったもので、手動操作の例外のまま残す
- アップロードの前に `aws sso login --profile yt-uploader` で一時的な認証情報を得る

## 検討した選択肢

| 案 | 採否 |
|---|---|
| `yt-admin` でアップロードする | 不採用。上の「背景」の理由による |
| IAM ユーザーを作り、アクセスキーを発行する | 不採用。長期間有効な鍵が PC に残り、漏れたら止めるまで使われ続ける |
| Identity Center の権限セット `yt-uploader` | **採用**。長期間有効な鍵を持たず、既存の仕組みにそろう |

## 結果

- アップロードのスクリプトが誤っても、data バケットの決まった場所に書く以外のことはできない
- プロファイルが3つ（`yt-readonly`／`yt-uploader`／`yt-admin`）になる
- フェーズ3でアップロードを Lambda に移すときは、実行ロールに同じ内容のポリシーを付ける
- IAM ポリシーの JSON は、実装時にユーザーが書き、Claude がレビューする

## 残る課題

- **`yt-readonly` が tfstate バケットの中身を読めてしまう。** ReadOnlyAccess は全バケットの `s3:GetObject` を許可するため、Claude Code は `aws s3 cp` で state を読める。`.claude/settings.json` の `Read(**/*.tfstate*)` はローカルのファイルにしか効かない
  - 対策案: tfstate バケットのバケットポリシーで、`aws:PrincipalArn` が `arn:aws:iam::*:role/aws-reserved/sso.amazonaws.com/*/AWSReservedSSO_ReadOnlyAccess_*` に一致する場合の `s3:GetObject` を拒否する。tfstate バケットの実装時（ADR-0001）に入れるか決める
- `yt-uploader` のセッション時間（`yt-admin` と同じ1時間を想定）
