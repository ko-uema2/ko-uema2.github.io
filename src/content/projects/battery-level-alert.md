---
title: Battery Level Alert
description: Macのバッテリー残量をAWS IoT Core経由で監視し、閾値以下で自動メール通知するIoTサーバーレスシステム
role: 個人開発。デバイス側（Python）からクラウドインフラ（AWS CDK）まで全工程を単独で担当
featured: false
order: 4
techStack:
  - category: 言語
    tech: TypeScript 5.6 / Python
  - category: IaC
    tech: AWS CDK v2 (TypeScript)
  - category: IoT
    tech: IoT Core / IoT Events / IoT Shadow
  - category: コンピュート
    tech: Lambda (Node.js 22.x, ARM64)
  - category: 通知
    tech: SES
  - category: セキュリティ
    tech: cdk-nag (AwsSolutions)
  - category: テスト
    tech: pytest / Jest
stats:
  - value: "1,900"
    label: lines total
  - value: "3"
    label: CDK Constructs
  - value: "2"
    label: environments
---

## 解決した課題

バッテリー残量の手動確認に依存せず、
閾値ベースの自動通知パイプラインをデバイス〜クラウド間で構築。
IoT Events状態機械によるアラート重複防止と、
IoT Shadowによる監視間隔の動的リモート制御を実装。

## 主な機能

- **バッテリー監視**: Macの`ioreg`コマンドでバッテリー残量を定期取得し、MQTT経由でAWS IoT Coreへ送信
- **状態機械判定**: IoT Events Detector Modelで閾値判定。Normal/Alertの2状態で重複通知を防止
- **メール通知**: Lambda + SESでHTML形式のアラートメールを送信
- **動的制御**: IoT Device Shadowで監視間隔（`wait_time_sec`）をリモート変更可能
- **環境分離**: develop（閾値100%）/ production（閾値30%）をCDKコンテキストで切り替え

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/battery-level-alert-architecture.png)

## 設計

### イベント駆動3層分離

CDKスタックを受信・判定・通知の3つのConstructに分離。

```text
BatteryLevelAlertStack
├── BatteryLevelReceiverResources   # IoT Core / Topic Rules
├── BatteryLevelAlertJudgeResources # IoT Events Detector Model
└── BatteryLevelAlertActionResources # Lambda / SES
```

### 状態機械によるアラート制御

IoT Events Detector ModelでNormal/Alertの2状態を管理。
閾値を下回った最初の1回のみ通知し、回復するまで再通知しない。

### セキュリティ

- cdk-nag（AwsSolutionsChecks）適用。抑制項目は理由を明記
- IAMポリシーはリソースARN限定、ワイルドカード排除
- IoTデバイス証明書はAwsCustomResourceで自動生成しSecrets Managerに保管
- 機密情報（メールアドレス）はSSM Parameter Storeで管理

## テスト

- **Python**: pytest（496行）。MagicMockでAWS IoT SDKをモック化し、正常系・異常系をカバー
- **TypeScript**: Jest設定済み
