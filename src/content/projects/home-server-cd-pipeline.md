---
title: 自宅サーバーへの CD パイプライン
description: GitHub ActionsからCloudflare Tunnel経由で自宅Ubuntu Desktopへ安全にデプロイするCDパイプラインを構築
role: 個人開発。設計・構築・運用の全工程を単独で担当
featured: false
order: 6
techStack:
  - category: CI/CD
    tech: GitHub Actions
  - category: ネットワーク
    tech: Cloudflare Tunnel / Cloudflare Access (ZTNA)
  - category: 認証
    tech: Service Token / SSH公開鍵認証 (ed25519)
  - category: デプロイ
    tech: rsync (選択的ファイル同期)
  - category: サーバー
    tech: Ubuntu 24.04 LTS / Apache Web Server
  - category: セキュリティ
    tech: UFW / Zero Trust Architecture
stats:
  - value: "6"
    label: security layers
  - value: "5"
    label: pipeline steps
---

## 解決した課題

PRマージ後の手動ファイルコピー（開発ディレクトリ → Apache公開ディレクトリ）を
CDパイプラインで自動化。同時に、自宅サーバーのパブリックIPを公開せず
外部からSSH接続する仕組みをCloudflare Tunnelで実現。

## 主な機能

- **自動デプロイ**: PRのmainマージをトリガーにrsyncでファイル同期
- **セキュア接続**: Cloudflare Tunnel経由でIP隠匿したSSH接続
- **自動認証**: Service Tokenによるプログラマティックな認証
- **選択的同期**: include/excludeパターンで転送対象を明示制御
- **依存管理**: リモートサーバーでのComposer自動インストール

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/home-server-cd-pipeline-architecture.png)

### パイプライン構成

| ステップ | 内容 |
| --- | --- |
| トリガー | mainブランチへのPRマージ |
| cloudflaredインストール | GitHub APIから最新バージョンを動的取得 |
| SSH設定 | ホスト鍵事前登録 + ed25519秘密鍵配置 |
| ファイル同期 | rsync + Cloudflare Access ProxyCommand経由 |
| 依存関係 | リモートでComposer install実行 |

## 設計方針

### Zero Trust Architecture

Cloudflare Tunnelによりルーターのポート開放を不要化し、パブリックIPを隠匿。
Cloudflare Accessで全アクセスを認証・認可。
Service Tokenによる自動認証で、人間の介入なしにセキュアなデプロイを実現。

### SSH接続のセキュリティ設計

`StrictHostKeyChecking=no`の使用を排除し、ホスト鍵の事前登録でMITM攻撃を防止。
ed25519鍵を採用し、デプロイ専用ユーザーで最小権限の原則を適用。
パスワード認証無効化、rootログイン禁止を設定。

### 6層のセキュリティレイヤー

| 層 | 実装 |
| --- | --- |
| 認証ゲート | Cloudflare Access (Service Token) |
| 通信経路 | Cloudflare Tunnel (暗号化トンネル) |
| ホスト認証 | SSH known_hosts (事前登録) |
| ユーザー認証 | SSH ed25519公開鍵認証 |
| ネットワーク | UFWファイアウォール + rate limit |
| ファイル制御 | rsync include/excludeフィルター |
