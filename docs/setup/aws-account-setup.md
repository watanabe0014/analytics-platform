# AWS学習環境セットアップ手順（アカウント作成〜`aws sso login`）

作成日: 2026-09-30 ／ 対象: 分析基盤ハンズオン用の個人AWSアカウント（方針A: Organizations + IAM Identity Center）

> この手順書は分析基盤用の別リポジトリができたら、そちらの `docs/` へ移す想定です。
> AWSの画面の文言や配置はよく変わります。記載は2026年9月時点の公式ドキュメントに基づきます。
> 食い違ったときは各Stepに付けた公式ドキュメントを優先してください。

## ゴール

個人のAWSアカウントで `aws sso login` が通り、ルートユーザーを普段使わない状態にする。
所要時間は合計1.5〜2時間ほどです（電話認証やメール確認の待ち時間を含む）。

完了時の構成:

```
AWSアカウント（1つ。AWS Organizations の管理アカウントを兼ねる）
├─ ルートユーザー …… MFAあり。請求・アカウント設定のときだけ使う
├─ AWS Budgets …… 月額予算のアラート
└─ IAM Identity Center（東京リージョン）
     └─ ユーザー: 自分
          └─ グループ: Admins
               ├─ 権限セット AdministratorAccess … 自分が構築・apply するとき
               └─ 権限セット ReadOnlyAccess ……… Claude Code に調査させるとき
```

## 準備するもの

| 項目 | 注意点 |
|---|---|
| メールアドレス | ルートユーザー用です。仕事のアドレスは使わないこと。Gmail なら `名前+aws@gmail.com` のようなエイリアスにすると、AWSからのメールを振り分けやすくなります |
| クレジットカード（またはデビットカード） | 本人確認のために少額の仮売上がかかることがあります |
| 電話番号 | SMS か音声通話で本人確認をします |
| 認証アプリ | Google Authenticator、1Password など。パスキーやハードウェアキーがあればそちらの方が安全です |
| パスワードマネージャー | ルートのパスワードとMFAの復旧手段を保管します |

---

## ⚠️ 先に知っておくこと：無料プランとクレジット

2025年7月以降に作るアカウントでは、登録時に「無料プラン」か「有料プラン」かを選びます。

- どちらのプランでも、新規登録で100ドル分のクレジットがもらえます。所定のアクティビティ（Budgetsの作成など）をこなすと、最大100ドルが追加されます。
- **無料プランのまま AWS Organizations を作成すると、自動で有料プランに切り替わります。このとき無料利用枠のクレジットは即時失効し、追加のクレジットも獲得できなくなります**（[AWS Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/)、[IAM Identity Center の有効化](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-identity-center.html)）。
- 無料プランは6か月たつか、クレジットを使い切ると、アカウントが自動で閉鎖されます（[プランの選択](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)）。

方針Aでは Step 4 で Organizations を作るので、**クレジットは最大200ドル失われる前提で進めます**。
このプロジェクトは S3・Athena・Lambda が中心で、想定費用は月に数百円なので、クレジットがなくても困りません。
なお、有料プランを最初から選んだ場合に Organizations を作ってもクレジットが残るかどうかは、公式ドキュメントに明記がありません。気になる場合は、Step 4 の前後に「Billing and Cost Management → クレジット」で残高を確認してください。

**推奨: 登録時に「有料プラン」を選ぶ。** どうせ Step 4 で有料プランに切り替わるうえ、無料プランのまま6か月たってアカウントが閉鎖される事故も防げます。

---

## Step 1: AWSアカウントを作る（20分）

1. https://aws.amazon.com/ を開き、「無料で始める」または「AWSアカウントを作成」を押す
2. **ルートユーザーのメールアドレス**と**アカウント名**を入力する（例: `twata-data-platform`。アカウント名は後から変更できます）
3. メールに届いた確認コードを入力し、ルートのパスワードを設定する。パスワードマネージャーで生成した長いものにします
4. 連絡先情報を入力する。用途は「個人（Personal）」を選ぶ
5. プランを選ぶ → **有料プラン**（理由は上記）
6. 支払い情報を入力する
7. 電話番号で本人確認をする
8. サポートプランは **ベーシック（無料）** を選ぶ
9. 完了メールが届いたら、コンソールにルートユーザーでサインインできることを確認する

> アカウントが有効になるまで数分〜最大24時間かかることがあります。サービスが使えないと表示されたら、しばらく待ってください。

控えておくもの:
- 12桁のアカウントID（コンソール右上のアカウント名 → 表示されます）

---

## Step 2: ルートユーザーを守る（10分）

ルートユーザーは何でもできるので、乗っ取られるとアカウントがまるごと失われます。ここは省略しないでください。

1. コンソール右上のアカウント名 → **セキュリティ認証情報**
2. **多要素認証（MFA）→ MFAデバイスの割り当て**
   - パスキーまたはハードウェアキー（推奨）か、認証アプリを選ぶ
   - 可能なら**2つ目のMFAデバイスも登録**しておく（スマホを紛失したときの復旧用。ルートには最大8個登録できます）
3. 同じ画面で**アクセスキーが「なし」であることを確認**する。ルートのアクセスキーは作りません
4. アカウント名 → **アカウント** → 「代替連絡先」に、請求・セキュリティの連絡先として自分のメールアドレスを入れる（任意ですが、セキュリティ通知の取りこぼしを防げます）

参考: [ルートユーザーのベストプラクティス](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)

---

## Step 3: 課金の安全装置を付ける（15分）

**ほかのどの作業よりも先に行ってください。** 学習用アカウントの事故で一番多いのは、消し忘れによる高額請求です。

### 3-1. IAMユーザーやロールに請求情報を見せる設定

これを有効にしないと、Step 5 で作る SSO ユーザーでは管理者権限があっても請求画面を見られません。

1. ルートでサインインしたまま、アカウント名 → **アカウント**
2. 「IAM ユーザーおよびロールによる請求情報へのアクセス」→ **編集** → 「IAM アクセスをアクティブ化」にチェックを入れて更新する

参考: [請求情報へのアクセスのアクティブ化](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/control-access-billing.html)

### 3-2. 予算アラート（AWS Budgets）

1. **Billing and Cost Management → 予算（Budgets）→ 予算を作成**
2. テンプレートを使って、次の2つを作る

| 予算 | 設定 | 目的 |
|---|---|---|
| ゼロ支出予算（Zero spend budget） | 通知先: 自分のメールアドレス | 1セントでも課金が発生したら気づけるようにする |
| 月次コスト予算（Monthly cost budget） | 金額: 例 **10 USD**。通知先: 自分のメールアドレス | 実績が85%・100%に達したとき、予測が100%を超えたときに通知される |

> 予算の金額は、Step 0 で決めた月額予算に合わせてください。Budgets の作成は、無料プランで追加クレジットをもらえるアクティビティの1つです。有料プランを選んだ場合も、この時点ではクレジットが残っているかもしれません。

### 3-3. Cost Explorer を有効にする

**Billing and Cost Management → Cost Explorer** を一度開くだけで有効になります。データが表示されるまでに最大24時間かかります。

---

## Step 4 + 5: Organizations と IAM Identity Center を有効にする（20分）

スタンドアロンのアカウントで IAM Identity Center を有効にすると、Organizations の作成も同時に行われます。Organizations を個別に作る必要はありません。

### 4-1. リージョンを決める

**Identity Center を有効にするリージョンは、後から変更できません**（変えるには設定を削除して作り直す必要があります）。
このプロジェクトでは **アジアパシフィック（東京）`ap-northeast-1`** に固定します。コンソール右上のリージョン表示が「東京」になっていることを確認してから進めてください。

### 4-2. 有効にする

1. ルートでサインインしたまま、[IAM Identity Center コンソール](https://console.aws.amazon.com/singlesignon)を開く
2. **有効にする**を押す
3. インスタンスの構成を聞かれたら **「単一リージョンインスタンス（Single-Region instance）」** を選ぶ
   - マルチリージョンを選ぶと、カスタマー管理のKMSキーが作られて料金がかかります。学習用には不要です
4. 「AWS Organizations で IAM Identity Center を有効にする」画面が出たら、内容を確認して**有効にする**を押す
   - ここで Organizations が作られ、このアカウントが管理アカウントになります（**無料プランの場合はここで有料プランに切り替わります**）
   - **この画面が出ずに、そのままダッシュボードに進むことがあります**（2026-09 に実際に確認）。その場合は次の3点を見て、すべて当てはまれば正常です
     - Identity Center のダッシュボードで、インスタンスの種類が「組織インスタンス（Organization instance）」になっている
     - 左メニューに「マルチアカウント許可」→「AWS アカウント」がある
     - [AWS Organizations](https://console.aws.amazon.com/organizations) に組織ができていて、自分のアカウントが管理アカウントになっている
   - 「アカウントインスタンス」になっていたら、方針Aでは使えません。「設定」→「管理」→「IAM Identity Center の設定を削除」で削除し、組織インスタンスで有効にし直します。「AWS アカウント」のメニューだけがない場合は、「設定」→「管理」でマルチアカウント許可を有効にします
5. 登録したメールアドレスに Organizations の確認メールが届いたら、リンクを開いて確認を済ませる
   - アカウント作成時にメールアドレスを確認済みなら、**確認メールは届きません**（2026-09 に実際に確認）。Organizations のコンソールに「メールアドレスを確認してください」という警告が出ていなければ、何もしなくてかまいません。確認が必要になるのは、他のアカウントを組織に招待するときだけです

参考: [IAM Identity Center の有効化](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-identity-center.html)

### 5-1. MFAを必須にする

1. Identity Center → **設定** → **認証** タブ → 多要素認証の **設定** を開く
2. 次のように設定する
   - MFAを求めるタイミング: **サインインのたび（Every time they sign in）**。「サインインのコンテキストが変わったとき」でも実用上は問題ありません
   - 使えるMFAの種類: 認証アプリとセキュリティキー（パスキー）
   - MFAデバイスを持っていないユーザーには: **サインイン時にMFAデバイスの登録を求める**

### 5-2. アクセスポータルのURLを控える

**ダッシュボード** の「設定の概要」に表示される **AWS アクセスポータルの URL**（`https://d-xxxxxxxxxx.awsapps.com/start` の形式）を控えます。
右の「カスタマイズ」から、`https://<好きな名前>.awsapps.com/start` のように覚えやすい名前に変更できます（任意）。

### 5-3. 自分のユーザーを作る

1. **ユーザー → ユーザーを追加**
2. ユーザー名（例: `twata`）、メールアドレス、姓・名を入力する
   - 「パスワードの設定方法」は **パスワード設定の手順をメールで送信** を選ぶ
3. グループはこの時点では選ばずに進め、ユーザーを作る
4. 招待メールの **Accept invitation** を開き、パスワードを設定してMFAを登録する

### 5-4. グループを作る

1. **グループ → グループを作成**
2. グループ名 `Admins` とし、メンバーに 5-3 のユーザーを追加する

> 権限はユーザーに直接付けずにグループへ付けます。実務と同じやり方で、後から人や用途が増えても整理しやすくなります。

### 5-5. 権限セットを作る

**マルチアカウント許可 → 許可セット → 許可セットを作成** で、次の2つを作ります。どちらも種類は **事前定義された許可セット** です。

| 許可セット | ポリシー | セッション時間 | 用途 |
|---|---|---|---|
| `AdministratorAccess` | AdministratorAccess | **1時間**（既定値） | 自分が構築するとき（`terraform apply` など） |
| `ReadOnlyAccess` | ReadOnlyAccess | 4時間 | Claude Code に状況を調べさせるとき。書き込みはできません |

> AWS の CLI ガイドは、日常の開発用に `PowerUserAccess`（IAM 以外は何でもできる）を勧めています。
> ただし、このプロジェクトでは Terraform で Lambda 用の IAM ロールも作るので、PowerUserAccess だと apply が途中で失敗します。
> そのため「作るときは Admin」「見るだけなら ReadOnly」と、用途で分けます。

### 5-6. アカウントに割り当てる

1. **マルチアカウント許可 → AWS アカウント** で、管理アカウント（自分のアカウント）にチェックを入れる
2. **ユーザーまたはグループを割り当て** → グループ `Admins` を選ぶ
3. 許可セット `AdministratorAccess` と `ReadOnlyAccess` の両方を選んで送信する

### 5-7. ポータルからサインインできるか確認する

1. ブラウザのシークレットウィンドウで 5-2 のアクセスポータルURLを開き、5-3 のユーザーでサインインする（MFAを求められます）
2. アカウントの下に `AdministratorAccess` と `ReadOnlyAccess` が表示されることを確認する
3. `AdministratorAccess` からコンソールを開き、Billing（請求）画面が見られることを確認する（3-1 が効いているかの確認です）

**ここからはルートでサインインしません。** ルートからサインアウトしてください。

---

## Step 6: AWS CLI を設定する（WSL2、15分）

### 6-1. AWS CLI v2 をインストールする

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install
aws --version   # aws-cli/2.x.x と表示されればOK
rm -rf aws awscliv2.zip
```

参考: [AWS CLI のインストール](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### 6-2. SSO のプロファイルを作る（管理者用）

WSL2 ではブラウザが自動で開かないことがあります。その場合は `--use-device-code` を付けると、表示されたURLをWindows側のブラウザに貼り、コードを入力する方式でサインインできます。

```bash
aws configure sso --use-device-code
```

聞かれたら次のように答えます。

```
SSO session name (Recommended): yt-sso
SSO start URL [None]: https://<5-2で控えたURL>.awsapps.com/start
SSO region [None]: ap-northeast-1
SSO registration scopes [None]: sso:account:access
  （ブラウザでサインイン → 許可）
  （ロールの選択 → AdministratorAccess）
Default client Region [None]: ap-northeast-1
CLI default output format (json if not specified) [None]: json
Profile name [...]: yt-admin
```

### 6-3. 読み取り専用のプロファイルを追加する

`aws configure sso --use-device-code` をもう一度実行し、同じ SSO セッション名 `yt-sso` を指定します。ロールは `ReadOnlyAccess` を選び、プロファイル名は `yt-readonly` にします。

できあがる `~/.aws/config` はおおむね次の形です（アカウントIDは実際の値）。

```ini
[profile yt-admin]
sso_session = yt-sso
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
region = ap-northeast-1
output = json

[profile yt-readonly]
sso_session = yt-sso
sso_account_id = 123456789012
sso_role_name = ReadOnlyAccess
region = ap-northeast-1
output = json

[sso-session yt-sso]
sso_region = ap-northeast-1
sso_start_url = https://<あなたのURL>.awsapps.com/start
sso_registration_scopes = sso:account:access
```

### 6-4. 動作を確認する

```bash
aws sso login --sso-session yt-sso --use-device-code
aws sts get-caller-identity --profile yt-admin
aws sts get-caller-identity --profile yt-readonly
```

`Arn` がそれぞれ `...:assumed-role/AWSReservedSSO_AdministratorAccess_.../twata` と `...AWSReservedSSO_ReadOnlyAccess_...` になっていれば完了です。

読み取り専用が効いているかも確かめておきます。

```bash
aws s3 mb s3://readonly-test-$(date +%s) --profile yt-readonly
# → AccessDenied になれば正しい
```

作業が終わったら `aws sso logout` で認証情報のキャッシュを消せます。

参考: [AWS CLI での IAM Identity Center 認証の設定](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)

---

## Step 7: 仕上げ（10分）

### 7-1. 操作履歴を確認できるようにする（CloudTrail）

CloudTrail の **イベント履歴**は、設定しなくても直近90日分の管理操作を無料で見られます。学習の初期はこれで十分です。
「誰がいつ何を作ったか」を長期間残す証跡（Trail）は、分析基盤を作るフェーズでS3と合わせて Terraform で作ります。その方が、IaC の題材にもなります。

### 7-2. Claude Code に渡す権限のルール

| 場面 | 使うプロファイル | 渡し方 |
|---|---|---|
| Claude Code に調査・設計をさせる | `yt-readonly` | `AWS_PROFILE=yt-readonly claude` で起動する |
| `terraform apply`、リソースの削除 | `yt-admin` | **自分の端末で自分が実行する**。Claude Code には実行させない |

新しいリポジトリの `.claude/settings.json` では、次のような deny ルールで二重に縛ります（書式は Claude Code のバージョンで確認してください）。

```json
{
  "permissions": {
    "deny": [
      "Bash(terraform apply:*)",
      "Bash(terraform destroy:*)",
      "Bash(aws --profile yt-admin:*)",
      "Read(~/.aws/sso/cache/**)"
    ]
  }
}
```

### 7-3. 毎日の使い方

```bash
aws sso login --sso-session yt-sso --use-device-code   # 1日の最初に1回
export AWS_PROFILE=yt-readonly                         # 普段は読み取り専用
```

---

## 完了チェックリスト

- [ ] ルートユーザーにMFAを設定した（できれば2つ）
- [ ] ルートのアクセスキーが存在しない
- [ ] IAMユーザー・ロールの請求情報へのアクセスを有効にした
- [ ] ゼロ支出予算と月次コスト予算を作り、通知メールの宛先を確認した
- [ ] Identity Center を東京リージョンで有効にした（Organizations も作成済み）
- [ ] Identity Center のユーザーでMFA付きでサインインできた
- [ ] `yt-admin` と `yt-readonly` の両方で `aws sts get-caller-identity` が通った
- [ ] `yt-readonly` で書き込みが拒否されることを確認した
- [ ] ルートからサインアウトし、ルートの認証情報をパスワードマネージャーに保管した

## よくあるつまずき

| 症状 | 原因と対処 |
|---|---|
| 「サービスにサインアップしていません」と表示される | アカウントの有効化待ちです。最大24時間待ってください |
| SSO ユーザーで請求画面が「アクセス拒否」になる | 3-1 を行っていません。ルートでサインインして有効にします |
| `aws configure sso` でブラウザが開かず、止まったように見える | WSL2 では自動で開かないことがあります。`--use-device-code` を付けて、表示されたURLをWindows側のブラウザで開きます |
| `aws sso login` 後に `Token has expired` | セッションが切れています。もう一度 `aws sso login` を実行します |
| ポータルにアカウントが表示されない | 5-6 の割り当てが済んでいません。割り当て直後は、一度サインアウトして入り直してください |
| Identity Center のリージョンを間違えた | 変更はできません。設定を削除して作り直します（下記の切り戻し手順を参照） |

## 切り戻し手順

途中で失敗したときや、やり直したいときに使います。上から順に戻せます。

| 戻したいもの | 手順 | 注意 |
|---|---|---|
| CLI の設定 | `aws sso logout` を実行し、`~/.aws/config` から該当のプロファイルと `sso-session` を削除する | ローカルだけの操作です |
| 権限セットの割り当て | Identity Center → AWS アカウント → 割り当てを削除する | |
| IAM Identity Center | ルートで Identity Center → 設定 → 管理 → **IAM Identity Center の設定を削除** | ユーザー・グループ・権限セットもすべて消えます。リージョンを間違えたときはこれで作り直します |
| AWS Organizations | ルートで Organizations → 設定 → **組織を削除** | メンバーアカウントがあると削除できません。削除しても無料プランには戻りません |
| AWS アカウント自体 | ルートでアカウント → **アカウントを閉鎖** | 閉鎖後90日間は再開できます。90日を過ぎると完全に削除されます |

## 参考資料（2026-09-30 に確認）

- [プランの選択（無料プランと有料プラン）](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)
- [AWS Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/)
- [IAM Identity Center の有効化](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-identity-center.html)
- [AWS CLI での IAM Identity Center 認証の設定](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
- [AWS CLI のインストール](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
