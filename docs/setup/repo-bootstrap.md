# リポジトリ初期整備の手順（最初のコミットの前に行う）

作成日: 2026-09-30

最初のコミットの前に、このリポジトリで直す4点をまとめた作業指示書です。
Claude Code に渡す場合は「`docs/setup/repo-bootstrap.md` の手順1〜4を実施して」と指示してください。

- 対象はこのリポジトリ（`~/projects/analytics-platform`）だけです。ほかのリポジトリには触れません。
- **既存ファイルの記述は消さずに追記します。** 特に CLAUDE.md の既存の節（プロジェクト概要、私について、あなたの役割、制約）は、文言を変えないでください。
- この手順では**コミットしません**。最後の「完了確認」まで終えたら、ユーザーに差分を見せて判断を仰いでください。

| # | 作業 | 理由 |
|---|---|---|
| 1 | 空のディレクトリに `.gitkeep` を置く | Git は空のディレクトリを管理しないので、GitHub に構成が表示されない |
| 2 | `.gitignore` を作る | 公開リポジトリなので、Terraform の state や認証情報、手元のデータが誤ってコミットされるのを防ぐ |
| 3 | CLAUDE.md に不足している前提を追記する | データソース、AWS の前提、手動操作の例外、公開時の注意が書かれていない |
| 4 | `.claude/settings.json` の deny ルールを補強する | 既存のルールでは `aws s3 rm` などが止まらず、認証情報のファイルも読めてしまう |

---

## 手順1: 空のディレクトリに `.gitkeep` を置く

```bash
cd ~/projects/analytics-platform
for d in docs/adr docs/learning-log infra ingestion transform; do
  mkdir -p "$d"
  [ -z "$(ls -A "$d")" ] && touch "$d/.gitkeep"
done
```

中身のファイルができたら、そのディレクトリの `.gitkeep` は消してかまいません。

---

## 手順2: `.gitignore` を作る

リポジトリ直下に `.gitignore` を作り、次の内容にします（ファイルがすでにある場合は、足りない行だけ追記します）。

```gitignore
# Terraform: state には機密情報が平文で入ることがある
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!example.tfvars
crash.log
*.tfplan

# 認証情報
.env
.env.*
client_secret*.json
*token*.json

# ローカルに置くデータ（clips.db のスナップショットなど）
data/
*.db
*.sqlite*
*.parquet

# dbt
transform/target/
transform/dbt_packages/
transform/logs/

# Python
.venv/
__pycache__/
*.pyc

# OS / エディタ
.DS_Store
```

> `.terraform.lock.hcl` は無視しません。プロバイダのバージョンを固定するファイルなので、コミットするのが正しい扱いです。

---

## 手順3: CLAUDE.md に追記する

既存の `# 制約` 節の**後ろ**に、次の4つの節を追加します。

```markdown
# データソース
- 運用DB（SQLite）: `~/projects/test-app/data/clips.db`
  - 稼働中のアプリ（切り抜き動画の自動作成システム）が書き込んでいるため、**直接読まない**。
    必ず読み取り専用のスナップショットを作り、それを取り込み元にする
  - 主なテーブル: videos, clips, longform_projects, longform_candidates, speakers, compilations
  - `metrics` テーブルは空（再生数などを未収集）。これを埋めることが分析基盤の最初の目的
- YouTube Data API / YouTube Analytics API
  - OAuth トークンはこのリポジトリ用に新しく取得する（スコープ `yt-analytics.readonly` が必要）
  - test-app のトークン（`data/youtube_token.json`）は流用しない。投稿用の本番トークンを壊さないため
  - OAuth 同意画面が「テスト」のままだと、トークンは7日で失効する。定期実行を組む前に対処する

# AWS の前提
- リージョンは `ap-northeast-1`（東京）に固定する
- 認証は IAM Identity Center（SSO）。プロファイルは2つ
  - `yt-readonly`: Claude Code が使う。Claude Code は `AWS_PROFILE=yt-readonly claude` で起動される前提
  - `yt-admin`: 私が `terraform apply` などを実行するときだけ使う。**あなたはこのプロファイルを使わない**
- 書き込みが必要な操作は、コマンドを提示して私に実行を依頼すること

# 手動操作の例外
- 「コンソールで手動操作しない」の例外は、`docs/setup/aws-account-setup.md` に書いた初期設定だけ
  （アカウント作成、ルートの MFA、Budgets、IAM Identity Center、権限セット）
- それ以外のリソースはすべて Terraform で作る。コンソールで作ったものを見つけたら、Terraform に取り込むか削除するかを提案すること

# 公開リポジトリとしての注意
- このリポジトリは GitHub で公開する
- 次のものはコミットしない: AWS アカウントID、アクセスポータルの URL、OAuth トークン・クライアントシークレット、
  Terraform の state、チャンネルの収益額、視聴者の個人情報
- 例やドキュメントでアカウントIDが必要なときは `123456789012` を使う
- コミット前に `git diff --cached` を確認し、上のどれかが含まれていたら私に知らせること
```

> 手順書 `docs/setup/aws-account-setup.md` は、test-app リポジトリの `docs/aws-account-setup.md` から移す予定です。まだ移していなければ、ユーザーに確認してください。

---

## 手順4: `.claude/settings.json` の deny ルールを補強する

既存の4つのルールは残し、次のように足します。

```json
{
  "permissions": {
    "deny": [
      "Bash(terraform apply:*)",
      "Bash(terraform destroy:*)",
      "Bash(aws * delete*)",
      "Read(.env)",

      "Bash(terraform import:*)",
      "Bash(terraform state rm:*)",
      "Bash(terraform state mv:*)",
      "Bash(terraform force-unlock:*)",
      "Bash(aws s3 rm:*)",
      "Bash(aws s3 rb:*)",
      "Bash(aws s3 mv:*)",
      "Bash(aws * terminate*)",
      "Bash(aws * --profile yt-admin*)",
      "Read(**/.env*)",
      "Read(~/.aws/**)",
      "Read(**/*.tfstate*)",
      "Read(**/client_secret*.json)",
      "Read(**/*token*.json)"
    ]
  }
}
```

上の例の空行は、既存のルールと追加分の区切りです。空行は JSON として問題ありませんが、JSON にはコメントを書けないので、区切りの説明をファイルに書き込まないでください。

### 重要：deny ルールは補助、本当の防壁は IAM

deny ルールはコマンドの**文字列**で判定されるので、書き方を変えれば抜けられます。例えば `AWS_PROFILE=yt-admin aws s3api delete-bucket ...` のように環境変数を前に付けると、ルールと一致しない場合があります。

本当の防壁は、**Claude Code を `yt-readonly` プロファイルで起動する**ことです。読み取り専用のロールなら、どんなコマンドを打ってもAWS側で拒否されます。
つまり、防御は次の二重構えになります。

1. **IAM（主）**: `yt-readonly` の権限セット（ReadOnlyAccess）では、書き込み系のAPIがすべて拒否される
2. **settings.json（補助）**: うっかり打ったコマンドを実行前に止める

この考え方は、面接で「AIエージェントに権限をどう渡したか」を聞かれたときの回答にもなります。

---

## 完了確認

```bash
cd ~/projects/analytics-platform

# 1. 空のディレクトリが Git に認識されている
git status --short --untracked-files=all | grep .gitkeep

# 2. 無視すべきファイルが無視される（すべて .gitignore の行が表示されればOK）
git check-ignore -v terraform.tfstate .env data/x.db transform/target/x client_secret_x.json

# 3. CLAUDE.md に4つの節が追加され、既存の節が残っている
grep -n '^# ' CLAUDE.md

# 4. settings.json が JSON として正しい
python3 -m json.tool .claude/settings.json > /dev/null && echo OK
```

- [ ] `.gitkeep` が5つ表示された
- [ ] `git check-ignore` で5つのファイルがすべて無視された
- [ ] CLAUDE.md の見出しが「プロジェクト概要／私について／あなたの役割：先生兼ペアプロ相手／制約／データソース／AWS の前提／手動操作の例外／公開リポジトリとしての注意」の8つになっている
- [ ] settings.json の検証で `OK` が表示された
- [ ] `git status` に、コミットしてはいけないファイル（state、トークン、`.db`）が出ていない

ここまで終わったら、ユーザーに差分を見せ、最初のコミットをするかどうか確認してください。
