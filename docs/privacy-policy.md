# プライバシーポリシー

最終更新: 2026-10-01

このリポジトリの取り込みスクリプト（以下、本アプリ）が、Google の API から取得するデータの扱いを定める。

## 利用者

本アプリは、リポジトリの作成者が自分の YouTube チャンネルを分析するためだけに使う。ほかの人に提供しない。

## 取得するデータ

OAuth で、次の読み取り専用のスコープだけを要求する。

| スコープ | 取得するデータ |
|---|---|
| `https://www.googleapis.com/auth/yt-analytics.readonly` | 自分のチャンネルの動画ごとの集計指標（再生数、視聴時間、視聴者維持率、インプレッション数など） |
| `https://www.googleapis.com/auth/youtube.readonly` | 自分のチャンネルの動画の一覧とメタデータ（タイトル、公開日時など） |

視聴者個人を識別できるデータは取得しない。動画の投稿、編集、削除は行わない。

## 利用目的

動画の作り方や投稿のタイミングを改善するための分析だけに使う。

## 保存と共有

- 取得したデータは、作成者が管理する AWS アカウントと、作成者の PC に保存する
- 第三者に提供、販売、共有しない
- 取得したデータそのものは、このリポジトリに含めない

## アクセスの取り消し

Google アカウントの「サードパーティ製のアプリとサービス」（https://myaccount.google.com/connections）から、本アプリのアクセスをいつでも取り消せる。

## Google API サービスのユーザーデータに関するポリシー

本アプリによる Google API から取得した情報の利用は、[Google API Services User Data Policy](https://developers.google.com/terms/api-services-user-data-policy)（Limited Use の要件を含む）に従う。

## 問い合わせ

このリポジトリの Issue で受け付ける。
