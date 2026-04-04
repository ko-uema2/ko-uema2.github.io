---
title: countAndNotifyNotionPagesToSlack
description: Notionデータベースの先週完了タスク数を自動集計し、Slackへ定期通知するサーバーレスアプリケーション
role: 個人開発。設計 / 実装 / インフラ構築の全工程を単独で担当
featured: false
order: 5
techStack:
  - category: 言語
    tech: TypeScript 5.7 / Node.js 20.x
  - category: IaC
    tech: AWS CDK (TypeScript)
  - category: コンピュート
    tech: Lambda (ARM64)
  - category: スケジューリング
    tech: EventBridge
  - category: 外部連携
    tech: Notion API / Slack Web API
  - category: 可観測性
    tech: Lambda Powertools / X-Ray
stats:
  - value: "790"
    label: lines code
  - value: "4"
    label: validation policies
---

## 解決した課題

Notionに蓄積されたタスク完了実績の週次集計を
手作業で行っていたプロセスを、
EventBridge + Lambdaによる定期実行パイプラインで自動化。

## 主な機能

- **タスク集計**: Notion APIで先週完了タスクを抽出・カウント
- **Slack通知**: 集計結果をSlackチャンネルへ自動投稿
- **シークレット管理**: SSM Parameter Storeで認証情報を分離管理
- **定期実行**: EventBridgeで毎週月曜AM 8:00にトリガー

## システムアーキテクチャ

![Architecture](/diagrams/count-and-notify-notion-pages-to-slack-architecture.png)

EventBridgeが毎週月曜にLambdaをトリガーし、
SSM Parameter Storeから認証情報を取得後、
Notion APIで完了タスクを集計し、Slack Web APIで通知する。

## 設計

### Strategy + Policy パターンによるバリデーション

環境変数ごとに異なるバリデーションルールを構造化。
4種の環境変数に対し、それぞれ専用のPolicyクラスと
バリデーションルールを定義。
不正な変数をまとめてエラー報告する仕組みを実装。

### デコレータによる横断的関心事の分離

`asyncFuncErrorHandler`デコレータで
ロギング（関数開始・終了）とエラーハンドリング
（`AppError`への変換・再スロー）を
ビジネスロジックから分離。

### 可観測性

Lambda Powertools Logger/Tracerを導入し、
構造化ログとX-Rayトレーシングを実現。
Lambdaのログ出力はJSON形式に設定。

## インフラ（CDK）

| リソース | 設定 |
| --- | --- |
| Lambda | Node.js 20.x, ARM64, 30秒タイムアウト |
| EventBridge | 毎週日曜23:00 UTC（月曜08:00 JST） |
| IAM Role | SSM読取 / CloudWatch Logs / X-Ray書込 |

`NodejsFunction`でTypeScriptを直接バンドル。
最小権限のIAMロールを個別定義。
