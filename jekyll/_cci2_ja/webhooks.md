---
layout: classic-docs
title: "Webhook"
short-title: "Webhook を使って CircleCI のイベントを受け取る"
description: "Webhook を使って CircleCI のイベントを受け取る"
version:
  - Cloud
---

## 概要
{: #overview}

Webhookにより、お客様が管理しているプラットフォーム（ご自身で作成した API またはサードパーティのサービス）と今後の一連の_イベント_を連携することができます。

CircleCI 上で Webhook を設定することにより、CircleCI から情報 (_イベント_ と呼ばれます) をリアルタイムで受け取ることができます。 これにより、必要な情報を得るために API をポーリングしたり、 CircleCI の Web アプリケーションを手動でチェックする必要がなくなります。

ここでは、Webhook の設定方法および Webhook の送信先にどのような形でイベントが送信されるかを詳しく説明します。

**注: ** CircleCI の Webhook 機能は、現在プレビュー版であり、ドキュメントや機能が変更または追加される場合があります。

## ユースケース
{: #use-cases}

Webhook は多くの目的にご活用いただけます。 具体的な例は以下のとおりです。

- カスタム ダッシュボードを作成して、ワークフローやジョブのイベントの可視化または分析を行う。
- インシデント管理ツール（例：Pagerduty）にデータを送信する。
- [Airtable]({{site.baseurl}}/ja/2.0/webhooks-airtable) などのツールを使ってデータを取得・可視化する。
- Slack などのコミュニケーション アプリにイベントを送信する。
- ワークフローがキャンセルされた場合に Webhook を使ってアラートを送信し、API を使ってそのワークフローを再実行する。
- ワークフローやジョブが完了したら内部通知システムをトリガーし、アラートを送信する。
- 独自の自動化ブラグインやツールを作成する。

## フックのセットアップ
{: #communication-protocol }

CircleCI では、現在以下のイベントの Webhook を利用できます。

Webhook は、HTTP POST により、Webhook 作成時に登録した URL に JSON でエンコードされた本文と共に送信されます。

CircleCI は、Webhook に応答したサーバーが 2xx のレスポンス コードを返すことを想定しています。 2xx 以外のレスポンスを受信した場合、CircleCI は、後ほど再試行します。 短時間のうちに Webhook への応答がない場合も、配信に失敗したと判断して後ほど再試行します。 タイムアウト時間は現在5秒ですが、プレビュー期間中に変更される場合があります。 再試行ポリシーの正確な詳細は現在文書化されておらず、プレビュー期間中に変更される場合があります。 タイムアウトや再試行についてフィードバックがあれば、 [サポートチームにご連絡ください](https://circleci.canny.io/webhooks)。

### プロジェクト
<sup>1</sup> こちらはテストの場合のみチェックボックスをオフのままにします。

Webhook には、以下のような多くの HTTP ヘッダーが設定されています。

| コミットのオーサー名         | 値                                                                             |
| ------------------ | ----------------------------------------------------------------------------- |
| 型                  | `application/json
`                                                           |
| User-Agent         | 送信者が CircleCI であることを示す文字列（`CircleCI-Webhook/1.0`）。 この値はプレビュー期間中に変更される場合があります。 |
| イベントタイプ            | イベントのタイプ （`workflow-completed`、`job-completed`など）                             |
| Circleci-Signature | この署名により Webhook の送信者にシークレット トークンへのアクセス権が付与されているかどうかを検証することができます。              |
{: class="table table-striped"}

## イベントの仕様
{: #setting-up-a-hook}

Webhook はプロジェクトごとにセットアップされます。 方法は以下のとおりです。

1. CircleCI 上にセットアップしたプロジェクトにアクセスします。
1. **Project Settings** をクリックします。
1. Project Settings のサイドバーで、**Webhook** をクリックします。
1. **Add Webhook** をクリックします。
1. Webhook フォームに入力します（フィールドとその説明については下の表をご覧ください）。
1. 受信用 API またはサードパーティのサービスがセットアップされている場合、**Test Ping Event** をクリックしてテストイベントをディスパッチします。

| フィールド                  | 必須？ | 説明                                                              |
| ---------------------- | --- | --------------------------------------------------------------- |
| Webhook name           | ○   | Webhook 名                                                       |
| URL                    | ○   | Webhook が Post リクエストを送信する URL                                   |
| Certificate Validation | ○   | イベント<sup>1</sup>を送信する前に受信ホストが有効な SSL 証明書を保持していることを確認します。        |
| Secret token           | ○   | 受信データが CircleCI からのデータかどうかを検証するために、ご自身の API または プラットフォームで使用します。 |
| Select an event        | ○   | Webhook をトリガーするイベントを少なくとも１つ選択しなければなりません。                        |
{: class="table table-striped"}

<sup>1</sup> こちらはテストの場合のみチェックボックスをオフのままにします。

**注: 1つのプロジェクトにつき Webhook は５つまでです。**

## 共通のトップ レベル キー
{: #payload-signature}

受信する Webhook を検証して、 送信元が CircleCI であることを確認する必要があります。 これを行うために、Webhook を作成する際に、シークレット トークンをオプションで提供することができます。 お客様のサービスへの送信HTTPリクエストごとに、 `circleci-signature` ヘッダーが含まれます。 このヘッダーは、バージョン管理された署名のリストで構成され、カンマで区切られています。

```
POST /uri HTTP/1.1
Host: your-webhook-host
circleci-signature: v1=4fcc06915b43d8a49aff193441e9e18654e6a27c2c428b02e8fcc41ccc2299f9,v2=...,v3=...

```

現在、最新の（そして唯一の）署名バージョンは v1 です。 ダウングレード攻撃を防ぐために、最新の署名タイプを*必ず*確認する必要があります。

この v1 署名は、リクエストボディのHMAC-SHA256ダイジェストであり、 設定された署名シークレットをシークレット キーとして使用しています。

プロジェクトに関するデータ

| フィールド                          | 常に表示             | 説明                                                                 |
| ------------------------------ | ---------------- | ------------------------------------------------------------------ |
| `hello World`                  | `secret`         | `734cc62f32841568f45715aeb9f4d7891324e6d948e4c6c60c0621cdac48623a` |
| `lalala`                       | `another-secret` | `daa220016c8f29a8b214fbfc3671aeec2145cfb1e6790184ffb38b6d0425fa00` |
| `an-important-request-payload` | `hunter123`      | `9be2242094a9a8c00c64306f382a7f9d691de910b4a266f67bd314ef18ac49fa` |
{: class="table table-striped"}

以下は、Pythonで署名を検証する場合の例です。

```
import hmac

def verify_signature(secret, headers, body):
    # ヘッダー`circleci-signature` から v1 署名を取得します。
    signature_from_header = {
        k: v for k, v in [
            pair.split('=') for pair in headers['circleci-signature'].split(',')
        ]
    }['v1']

    # 設定した署名シークレットを使って リクエスト ボディーで HMAC-SHA256 を実行します。
    valid_signature = hmac.new(bytes(secret, 'utf-8'), bytes(body, 'utf-8'), 'sha256').hexdigest()

    # 一定時間文字列比較を使ってタイミング攻撃を防ぎます。
    return hmac.compare_digest(valid_signature, signature_from_header)

# 以下の場合 `True` を返します。
verify_signature(
    'secret',
    {
        'circleci-signature': 'v1=773ba44693c7553d6ee20f61ea5d2757a9a4f4a44d2841ae4e95b52e4cd62db4'
    },
    'foo',
)

# 以下の場合 `False` を返します。
verify_signature(
    'secret',
    {
        'circleci-signature': 'v1=not-a-valid-signature'
    },
    'foo',
)

```

## 共通のサブエンティティ
{: #event-specifications}

CircleCI では、現在以下のイベントの Webhook を利用できます。

| イベントタイプ            | 説明                  | 状態の例                                                     | 含まれるサブエンティティ                |
| ------------------ | ------------------- | -------------------------------------------------------- | --------------------------- |
| workflow-completed | ワークフローが終了状態になっています。 | "success", "failed", "error", "canceled", "unauthorized" | プロジェクト、組織、ワークフロー、パイプライン     |
| job-completed      | ジョブが終了状態になっています。    | "success", "failed", "error", "canceled", "unauthorized" | プロジェクト、組織、ワークフロー、パイプライン、ジョブ |
{: class="table table-striped"}

## 共通のトップレベル キー
{: #common-top-level-keys}

イベントの一部として、各Webhook に共通するデータがあります。

| フィールド       | 説明                                                          | タイプ |
| ----------- | ----------------------------------------------------------- | --- |
| id          | システムからの各イベントを一意に識別するための ID (クライアントはこれを使って重複するイベントを削除できます。 ） | 文字列 |
| happened_at | イベントが発生した日時を表す ISO 8601 形式のタイムスタンプ                          | 文字列 |
| webhook     | トリガーされた Webhook を表すメタデータのマップ                                | マップ |
{: class="table table-striped"}

**注: ** イベントのペイロードはオープンなマップであり、新しいフィールドが互換性を損なう変更とみなされずにWebhook のペイロードのマップに追加される可能性があります。


## 共通のサブエンティティ
{: #common-sub-entities}

ここでは CicrcleCI の Webhook が提供する様々なイベントのペイロードについて説明します。 これらの Webhook イベントのスキーマは、多くの場合共有データを他の Webhook と共有します。 Circle CI では、このことをデータの共通マップとして「サブエンティティー」と呼びます。 例えば、`job-completed` 状態の Webhook のイベント ペイロードを受信した場合、それにはご自身の*プロジェクト、組織、ジョブ、ワークフロー、およびパイプライン* のデータマップが含まれます。

以下は、さまざまな Webhook で表示される共通のサブエンティティの例です。

### 組織
{: #project}

Webhook イベントに関連するプロジェクトに関するデータ

| フィールド | 常に表示 | 説明                                                                     |
| ----- | ---- | ---------------------------------------------------------------------- |
| id    | ○    | プロジェクトの一意の ID                                                          |
| slug  | ○    | 多くの CircleCI の API の中で特定のプロジェクト（例えば、gh/circleci/web-ui）を参照するために使用する文字列 |
| name  | ○    | プロジェクト名（例：web-ui）                                                      |
{: class="table table-striped"}

### ジョブ
{: #organization}

組織に関するデータ

| フィールド | 常に表示 | 説明               |
| ----- | ---- | ---------------- |
| id    | ○    | 組織の一意の ID        |
| name  | ○    | 組織名 (例：CircleCI) |
{: class="table table-striped"}

### ワークフロー
{: #job}

通常、CircleCI のワークロードにおけるある期間を表し（例：「ビルド」、「テスト」、または「デプロイ」）、一連のステップを含むジョブ。

| フィールド         | 常に表示 | 説明                                                                |
| ------------- | ---- | ----------------------------------------------------------------- |
| id            | ○    | ジョブの一意の ID                                                        |
| number        | ○    | ジョブの自動インクリメント番号。 CircleCI の API でプロジェクト内のジョブを識別するために使用される場合があります。 |
| name          | ○    | .circleci/config.yml で定義されているジョブ名                                 |
| status        | ○    | ジョブの現在の状態                                                         |
| started\_at | ○    | ジョブの実行が開始された時間                                                    |
| stopped\_at | ×    | ワークフローが終了状態になった時間（該当する場合）                                         |
{: class="table table-striped"}


### パイプライン
{: #workflow}

ワークフローには多くのジョブが含まれ、それらは並列で実行される、およびまたは依存関係を持っています。 １回のgit-push で、CircleCI の構成に応じて、ゼロ以上のワークフローをトリガーすることができます（通常は１つのワークフローがトリガーされます）。


| フィールド         | 常に表示 | 説明                                      |
| ------------- | ---- | --------------------------------------- |
| id            | ○    | ワークフローの一意の ID                           |
| name          | ○    | .circleci/config.yml で定義されているワークフロー名    |
| status        | ×    | ワークフローの現在の状態。 ジョブレベルの Webhook には含まれません。 |
| created\_at | ○    | ワークフローが作成された時間                          |
| stopped_at    | ×    | ワークフローが終了状態になった時間（該当する場合）               |
| url           | ○    | CircleCI の UI にあるワークフローへの URL           |
{: class="table table-striped"}

### トリガー
{: #pipeline}

パイプラインは最もハイレベルな作業単位で、ゼロ以上のワークフローが含まれます。 １回の git-push で、常に最大で１つのパイプラインをトリガーします。 パイプラインは API から手動でトリガーすることもできます。

| フィールド         | 常に表示 | 説明                                         |
| ------------- | ---- | ------------------------------------------ |
| id            | ○    | グローバルに一意なパイプラインの ID                        |
| number        | ○    | バイプラインの番号（自動インクリメントまたはプロジェクトごとに一意）         |
| created\_at | ○    | パイプラインが作成された時間                             |
| trigger       | ○    | このパイプラインが作成された原因に関するメタデータ マップ（以下を参照）       |
| vcs           | ×    | このパイプラインに関連する Git コミットに関するメタデータ マップ（以下を参照） |
{: class="table table-striped"}

### トリガー
{: #trigger}

| フィールド    | 常に表示 | 説明                                                 |
| -------- | ---- | -------------------------------------------------- |
| type     | ○    | このパイプラインがどのようにトリガーされたか（例：「Webhook」、「API」、「スケジュール」） |
| actor.id | ×    | パイプラインをトリガーしたユーザー（存在する場合）                          |
{: class="table table-striped"}


### VCS
{: #vcs}

注：将来、パイプラインが Git コミットと関連していない場合など情報が当てはまらない場合、VCS マップまたはそのコンテンツが提供されないことがあります。

| フィールド                   | 常に表示 | 説明                                                      |
| ----------------------- | ---- | ------------------------------------------------------- |
| target_repository_url | ×    | コミットをビルドするレポジトリへの URL                                   |
| origin_repository_url | ×    | コミットが作成されたレポジトリへの URL （フォークされたプルリクエストの場合のみ異なります）        |
| revision                | ×    | ビルドする Git コミット                                          |
| commit.subject          | ×    | コミットのサブジェクト（コミットメッセージの先頭行） 長いコミットサブジェクトは切り捨てられる場合があります。 |
| commit.body             | ×    | コミットの本文（コミットメッセージの後続の行） 長いコミット本文は切り捨てられる場合があります。        |
| commit.author.name      | ×    | コミットのオーサー名                                              |
| commit.author.email     | ×    | コミットのオーサーのメールアドレス                                       |
| commit.authored\_at   | ×    | コミットがオーサリングされた時のタイムスタンプ                                 |
| commit.committer.name   | ×    | コミットのコミッター名                                             |
| commit.committer.email  | ×    | コミットのコミッターのメールアドレス                                      |
| commit.committed_at     | ×    | コミットがコミットされた時のタイムスタンプ                                   |
| branch                  | ×    | ビルドされたブランチ                                              |
| tag                     | ×    | ビルドされたタグ（「ブランチ」と相互排他的）                                  |
{: class="table table-striped"}