# ADR-0001: Terraform の state を S3 に置き、state 用バケットは bootstrap 用の Terraform で作る

- 日付: 2026-10-01
- 状態: 承認

## 背景

Terraform は、作ったリソースの記録（state）と、コードに書いたあるべき姿を比べて、差分を反映する。state を失うと、Terraform は AWS 上のリソースを管理できなくなる。

state をローカルの PC にだけ置くと、次の問題がある。

- PC が壊れると state が失われ、既存のリソースを `terraform import` で1つずつ取り込み直すことになる
- GitHub Actions（フェーズ5で予定）から state を参照できず、CI で plan を実行できない
- 複数の場所から同時に実行したとき、state の書き込みが衝突する

state には、リソースの ID や設定値が平文で入る。このリポジトリは公開するので、state を Git に入れることはできない。

そこで、state を S3 に置く（リモートバックエンド）。ただし、その S3 バケットを誰がどう作るかという「鶏と卵」の問題が残る。

## 検討した選択肢

| 案 | 内容 | 採否 |
|---|---|---|
| A. 手で作る | AWS CLI かコンソールで state 用バケットを作る | 不採用。「コンソールで手動操作しない」の例外が増え、バケットの設定（バージョニング、暗号化）がコードに残らない |
| B. bootstrap 用の Terraform | state 用バケットだけを作る小さな Terraform を、ローカルの state で apply する。そのあと、その state 自体を作ったバケットに移す | **採用** |
| C. CloudFormation で作る | state 用バケットだけを CloudFormation のスタックで作る | 不採用。IaC のツールが2種類になる |

## 決定

- state は S3 バケットに置く
- state 用バケットは、`infra/bootstrap/` の Terraform で作る
  1. `infra/bootstrap/` を、ローカルの state で apply する
  2. できたバケットをバックエンドに設定し、`terraform init -migrate-state` で bootstrap 自身の state もバケットに移す
- 本体のインフラ（フェーズ1以降）は、別のディレクトリで同じバケットを使う（state のキーを分ける）
- ロックは S3 のネイティブロック（`use_lockfile = true`）を使う。DynamoDB は使わない。そのため Terraform 1.10 以上を前提にする
- state 用バケットには、次の設定を入れる
  - バージョニング（state が壊れたときに前の版へ戻すため）
  - サーバー側暗号化（SSE-S3。追加費用なし）
  - パブリックアクセスのブロック
  - `prevent_destroy`（誤って消さないため）

## 結果

- 手動操作の例外を増やさずに、すべてのリソースをコードで管理できる
- state 用バケットの設定を、コードとしてレビューできる
- 最初の apply のときだけ、state がローカルにある。移し終えたら、ローカルの state ファイルは削除する
- 想定月額は $0.01 未満（state は KB 単位）

## 残る課題

- **バケット名にアカウントIDを入れない。** S3 のバケット名は全世界で一意である必要があり、アカウントIDを付けるのが一般的な方法だが、公開リポジトリには書けない。ランダムな接尾辞を付けるか、バックエンドの設定を別ファイル（`-backend-config=backend.hcl`）に分けて Git から除外するかを、実装時に決める。後者を選ぶ場合は `.gitignore` に追記する
- KMS のカスタマー管理キー（月 $1）による暗号化は、今は使わない。必要になったら見直す
