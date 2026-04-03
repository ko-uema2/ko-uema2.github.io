---
title: オンプレミスデバイスの CloudWatch 監視
description: Raspberry Pi NASの死活監視を目的に、CloudWatch Agentによるオンプレミスメトリクス収集基盤を構築
role: 個人開発。IAM設計からエージェント導入、監視設定まで単独で担当
featured: false
order: 3
techStack:
  - category: 監視
    tech: Amazon CloudWatch Agent (darwin/arm64)
  - category: クラウド
    tech: CloudWatch / IAM / CloudWatch Alarm
  - category: 対象OS
    tech: macOS Sonoma 14.5 / Raspberry Pi OS
  - category: リージョン
    tech: ap-northeast-1
stats:
  - value: "6"
    label: metrics
  - value: "60s"
    label: interval
---

## 解決した課題

NASとして稼働するRaspberry Piが無通知で停止する問題が発生していた。
CloudWatch Agentによるメトリクス収集とAlarmによる異常検知基盤を構築し、
停止の即時検知を実現。

## 主な機能

- **メトリクス収集**: CPU、メモリ、ディスク、ディスクIO、ネットワーク、スワップの6種を60秒間隔で収集
- **CloudWatch送信**: CWAgentカスタム名前空間へのメトリクス自動送信
- **異常検知**: CloudWatch Alarmによる閾値ベースの通知（Raspberry Pi環境で設定）

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/cloudwatch-onpremise-monitoring-architecture.png)

### 収集メトリクス

| メトリクス | 項目 | 間隔 |
| --- | --- | --- |
| CPU | cpu_usage_idle（コア別 + 合計） | 60s |
| メモリ | mem_used_percent | 60s |
| ディスク | used_percent | 60s |
| ディスクIO | write_bytes, read_bytes, writes, reads | 60s |
| ネットワーク | bytes_sent, bytes_recv, packets_sent, packets_recv | 60s |
| スワップ | swap_used_percent | 60s |

## 構成上のポイント

### IAM設計

CloudWatch Agent専用のIAMユーザーを作成し、
`CloudWatchAgentServerPolicy`のみをアタッチ。
`AmazonCloudWatchAgent`プロファイルで認証情報を分離。

### オンプレミス固有の設定

EC2メタデータが使用できないため、
`common-config.toml`で認証情報ファイルパスを明示指定。
`shared_credential_profile`と`shared_credential_file`の
両方の設定が必須（macOS環境ではデフォルト設定でエラーが発生）。

### エージェント構成

config wizardでメトリクス構成を生成。
StatsD、CollectDは無効化し不要な依存を排除。
X-Rayトレース、ログ収集は対象外とし、メトリクス監視に特化。
