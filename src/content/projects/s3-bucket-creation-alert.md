---
title: S3 バケット新規作成 監視・通知システム
description: CloudTrail証跡とCloudWatchメトリクスフィルターによるS3バケット作成の自動検知・メール通知基盤を構築
role: 個人開発。設計・構築・動作検証を単独で担当
featured: false
order: 8
techStack:
  - category: 監視
    tech: AWS CloudTrail（管理イベント・全リージョン）
  - category: ログ分析
    tech: CloudWatch Logs（カスタムメトリクスフィルター）
  - category: アラート
    tech: CloudWatch Alarms
  - category: 通知
    tech: Amazon SNS（Email）
  - category: 暗号化
    tech: AWS KMS
  - category: 対象
    tech: Amazon S3
stats:
  - value: "全"
    label: リージョン監視
  - value: "1min"
    label: 検知間隔
---

## 解決した課題

ハンズオン実施時に意図せず作成されたS3バケットの存在を把握できず、
用途不明のバケットが残留する問題が発生していた。
CloudTrailとCloudWatch Alarmを組み合わせた監視基盤を構築し、
バケット作成の即時検知・通知を実現。

## 主な機能

- **全リージョン監視**: CloudTrailコンソールで作成した証跡により、全リージョンの管理イベントを自動収集
- **CreateBucketイベント検知**: メトリクスフィルターで `eventName = "CreateBucket"` を監視
- **メール通知**: CloudWatch AlarmからSNSトピック経由でメール送信

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/s3-bucket-creation-alert-architecture.png)

## 構成上のポイント

### イベントフィルタリング

CloudTrail証跡の管理イベントを「書き込み」のみに限定。
AWS KMS・Amazon RDSイベントを除外設定し、ノイズを抑制。

### メトリクス設計

フィルターパターン `{ ($.eventName = "CreateBucket") }` でCreateBucketイベントを検出。
メトリクス値を1に設定し、合計が0を超えた場合にAlarmを発火。

### ログファイル暗号化

CloudTrail証跡ログのS3保存時にKMS暗号化を有効化。
専用KMSキーを作成し、キーポリシーでCloudTrailのアクセスを許可。
