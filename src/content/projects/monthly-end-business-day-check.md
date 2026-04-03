---
title: 月末営業日チェックシステム
description: 月末最終営業日にLINE通知とGoogleカレンダー予定作成を自動実行するサーバーレスシステム
role: 個人開発。要件定義 / 設計 / 実装 / テスト / インフラ構築を単独で担当
featured: false
order: 2
techStack:
  - category: 言語
    tech: TypeScript 5.6 / Node.js 22
  - category: IaC
    tech: AWS CDK 2.x (TypeScript)
  - category: コンピュート
    tech: Lambda (ARM64, 2 functions)
  - category: スケジューラ
    tech: Amazon EventBridge
  - category: 外部API
    tech: LINE Messaging API / Google Calendar API
  - category: 認証
    tech: JWT (RSA256) / サービスアカウント
  - category: テスト
    tech: Jest + ts-jest
stats:
  - value: "635"
    label: lines test
  - value: "3"
    label: ADRs
  - value: "1.1"
    label: test/src ratio
---

## 解決した課題

月末最終営業日に必要な定型作業のリマインドを手動管理していた。
土日祝日を考慮した営業日判定と通知を自動化し、
人的な確認漏れを排除。

## 主な機能

- **営業日自動判定**: Google Calendar APIから祝日を取得し、土日祝を除外した月末最終営業日を判定
- **LINE通知**: JWT認証によるチャネルアクセストークンv2.1の動的発行でプッシュ通知を送信
- **カレンダー予定作成**: リマインド付きの予定をGoogleカレンダーに自動作成
- **自動リトライ**: 失敗時にSQS DLQへ転送し、別Lambdaが自動再実行（無限ループ防止付き）

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/monthly-end-business-day-check-architecture.png)

## 設計

### Adapterパターン + 依存性注入

外部API（LINE、Google Calendar、Secrets Manager）ごとに
専用アダプタークラスを実装。
`BusinessDayChecker`はコンストラクタで
`GoogleCalendarHolidayFetcher`を受け取るDIを採用し、
単体テストでのモック差し替えを容易にした。

### 単一責任原則によるクラス分割

Google Calendar APIクラスを責務別に3分割。

- `GoogleCalendarBase`: 共通認証処理（抽象基底クラス）
- `GoogleCalendarHolidayFetcher`: 祝日取得（読み取り専用）
- `GoogleCalendarEventCreator`: 予定作成（書き込み専用）

### エラーハンドリングとリトライ戦略

Lambda失敗時にSQS DLQへイベントを転送し、
DLQをイベントソースとする別Lambdaが自動再実行。
リトライLambdaにはDLQを設定せず無限ループを防止。
AWS Lambda Powertools for TypeScriptによる構造化JSONログを採用。

## テスト戦略

| 層 | テスト対象 | 方針 |
| --- | --- | --- |
| 単体 | Lambdaハンドラ、営業日判定 | 外部依存をモック化 |
| 統合 | Google Calendar API | 実APIを呼び出して検証 |
| CDK | CloudFormationテンプレート | アサーションで検証 |

## 技術的意思決定（ADR）

3件のADRを記録。

- **ADR-001**: LINE APIクライアントの遅延インスタンス化（トークン有効期限対応）
- **ADR-002**: Google Calendarクラスの責務分離（SRP遵守）
- **ADR-003**: BusinessDayCheckerへの依存性注入（テスタビリティ確保）

## インフラ

AWS CDKで全リソースを定義。
CDK Context（`envType=dev/prod`）で環境を切り替え。
