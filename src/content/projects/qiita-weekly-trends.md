---
title: Qiita Weekly Trends
description: Qiita 週次トレンド記事を Slack に自動投稿するサーバーレスアプリケーション
role: 個人開発。要件定義 / 設計 / 実装 / インフラ / CI/CD のすべてを単独で担当
repo: https://github.com/ko-uema2/qiitaWeeklyTrends
featured: false
order: 9
techStack:
  - category: 言語
    tech: TypeScript 5.9 / Node.js 22.x
  - category: IaC
    tech: AWS CDK (TypeScript)
  - category: コンピュート
    tech: Lambda (NodejsFunction)
  - category: スケジュール
    tech: EventBridge Rule (cron)
  - category: シークレット
    tech: SSM Parameter Store SecureString
  - category: 外部連携
    tech: Slack Web API / cheerio
  - category: CI/CD
    tech: CDK Pipelines (CodePipeline)
stats:
  - value: "2"
    label: CDK Stacks
  - value: "2"
    label: Stages (dev/prod)
  - value: "1"
    label: Lambda function
---

## 解決した課題

Qiita 週次トレンドページを定常的にチェックする運用コストを削減する。
EventBridge スケジュールで JST 月・木 8:00 に Lambda を起動し、
トレンドページから記事 URL を抽出して Slack チャンネルへ自動投稿する。

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/qiita-weekly-trends-architecture.png)

EventBridge Rule から Lambda を起動し、
SSM Parameter Store の SecureString から Slack Bot Token を取得、
Qiita 週次トレンドページを HTTPS で取得して `cheerio` で記事 URL を抽出、
Slack Web API の `chat.postMessage` で順次投稿する。

## 設計

### ステージ設定の一元化

`lib/stage-config.ts` に dev / prod の差分を集約。
`buildStageConfig` ファクトリでステージ固有の設定を生成し、
スタック側はステージを意識せず設定値を参照する。

| 項目          | dev     | prod                 |
| ------------- | ------- | -------------------- |
| removalPolicy | DESTROY | RETAIN               |
| logRetention  | 1 週間  | 3 ヶ月               |
| 直デプロイ    | 許可    | パイプライン経由必須 |

### 最小権限 IAM

Lambda 実行ロールに付与する `ssm:GetParameter` は
ワイルドカードを使わず、対象パラメータの ARN を `formatArn` で
明示的に生成して指定する。

### シークレット管理

Slack Bot Token は SSM Parameter Store の SecureString に格納し、
コードにもリポジトリにも平文で残さない。
Lambda 側ではコンテナ再利用を考慮してトークンを変数にキャッシュし、
コールドスタート時のみ SSM を参照する。

## CI/CD

### パイプライン構成

![Pipeline](/diagrams/qiita-weekly-trends-pipeline.png)

dev アカウントにパイプラインをホストし、
prod アカウントへクロスアカウントで CloudFormation をデプロイする
2 アカウント構成。
`CodePipeline` の `crossAccountKeys: true` と
prod 側の `cdk bootstrap --trust` で信頼関係を構築する。

### デプロイゲート

prod への反映は `ManualApprovalStep` で
CloudFormation changeset をレビューしてから承認する運用とし、
意図しない本番変更を防ぐ。
障害対応など緊急時のみ
`cdk deploy -c stage=prod -c force=true` でパイプラインをバイパスした
直接デプロイを許可し、通常運用ではパイプライン経由を強制する。

### 自己ミューテーションパイプライン

CDK Pipelines の self-mutate を有効化し、
パイプライン定義自体の変更も main ブランチへの push のみで反映される。
