---
title: MoneyForward2Sheet
description: Google DriveへのCSVアップロードをトリガーに、MoneyForwardの取引データをGoogle Sheetsへ自動集計するサーバーレスアプリケーション
role: 個人開発。要件定義 / 設計 / 実装 / インフラ / CI/CD のすべてを単独で担当
repo: https://github.com/ko-uema2/MoneyForward2Sheet
featured: true
order: 1
techStack:
  - category: 言語
    tech: TypeScript 5.6 / Node.js 22.x
  - category: IaC
    tech: AWS CDK (TypeScript)
  - category: コンピュート
    tech: Lambda (6 functions)
  - category: API
    tech: API Gateway HTTP API
  - category: データ
    tech: DynamoDB (single-table)
  - category: 外部連携
    tech: Google Drive API / Sheets API
  - category: CI/CD
    tech: CDK Pipelines (CodePipeline)
  - category: テスト
    tech: Jest + ts-jest
stats:
  - value: "46,000"
    label: lines test
  - value: "21"
    label: ADRs
  - value: "6"
    label: Lambda functions
  - value: "40+"
    label: Value Objects
---

## 解決した課題

外部サービス（MoneyForward）のCSVデータを
Google Sheetsに自動集計する仕組みが求められていた。
手作業でのCSV転記は煩雑でミスが起きやすいため、
Google Drive Push Notificationをトリガーにした
イベント駆動の自動化パイプラインを構築した。

## 主な機能

- **ファイル検知**: Google Drive Push Notificationによるリアルタイム検知
- **CSV解析・集計**: MoneyForwardのCSVを解析し、中項目×月で金額を集計
- **Sheet自動更新**: Google Sheets APIで該当セルに集計値を書き込み
- **エラー通知**: 処理失敗時にAmazon SESでHTML形式のメール通知
- **購読管理API**: Drive監視の開始・停止・状態確認・設定変更をAPIで操作
- **自動更新**: EventBridgeによる12時間ごとの購読自動更新

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/moneyforward2sheet-architecture.png)

### API設計

| エンドポイント | メソッド | 認証 | 用途 |
| --- | --- | --- | --- |
| `/webhook` | POST | Lambda内トークン検証 | Drive Push通知受信 |
| `/status` | GET | IAM (SigV4) | 購読状態の確認 |
| `/subscribe` | POST | IAM (SigV4) | Drive監視の開始 |
| `/unsubscribe` | POST | IAM (SigV4) | Drive監視の停止 |
| `/config` | POST | IAM (SigV4) | 設定の保存 |

Webhookエンドポイントのみ認証方式が異なるのは、
Drive Push Notificationの制約による設計判断。
購読登録時にsync通知が送信されるが、
この時点では購読が未完了でトークンが確定していない。
API Gatewayのルート別認証により、
Webhookは認証なしでアクセス可能としつつ、
Lambda内でトークン検証を行うことでこの問題を解決した（ADR-014）。

## 設計

### Clean Architecture + DDD

4層構造を採用し、DDDの戦術的パターンで実装。

```text
lambda/src/
├── Domain/           ← ビジネスルール（外部依存なし）
├── Application/      ← ユースケース
├── Port/             ← インターフェース定義（依存性逆転）
├── Infrastructure/   ← 外部サービス実装
├── Interface/        ← Lambdaハンドラー
└── Shared/           ← 共有ユーティリティ
```

40以上の値オブジェクト・エンティティを定義し、プリミティブ型への依存を排除。
Port層にインターフェースを定義し、Infrastructure層が実装を提供。
ユースケースはPort層のインターフェースにのみ依存する。

### エラーハンドリング 3層分離

エラーを3つの層で分類・翻訳するアーキテクチャを設計（ADR-006）。

```text
Infrastructure層 → Application層 → Interface層
（外部エラー）    （ドメインエラー）  （HTTPレスポンス）
```

- **ドメインエラー**: ビジネスルール違反（カテゴリ不一致、月列未発見など）
- **インフラエラー**: 外部サービス障害（Google API, AWS SDKエラー）
- **制御不可能エラー**: 予期せぬ例外

Chain of Responsibilityパターンで複数のエラートランスレーターを連結し、
各インフラストラクチャのエラーを適切なドメインエラーに変換。

### Decorator パターン

ロギングとキャッシングをDecoratorパターンで実装し、
ビジネスロジックから分離（ADR-004）。

```text
CachingApplicationConfigDecorator    ← キャッシュ責務
  └── LoggingConfigRepositoryDecorator ← ロギング責務
        └── DynamoDbConfigRepository     ← データアクセス
```

各層が単一の責務のみを持ち、テスタビリティと拡張性を確保。

### DIコンテナ

手動DIコンテナを2層構成で実装し、
Lambda関数ごとに必要最小限の依存のみを注入。

- **BaseContainer**: SSM不要なハンドラー向け（configHandler等）
- **FullContainer**: BaseContainerを継承し、全UseCase・認証ポリシーを構築

### べき等性設計

同一CSVの重複通知に対して、スプレッドシートの値は
上書き方式で更新されるため、何度処理されても結果は同一（ADR-017）。
リトライやデデュプリケーションの複雑さを排除しつつ、
データの整合性を保証。

### セキュリティ設計

| 観点 | 実装 |
| --- | --- |
| 認証情報管理 | Secrets Managerでサービスアカウントキー管理 |
| API認証 | 管理API: IAM認証 / Webhook: トークン検証 |
| IAM最小権限 | 各Lambdaに必要最小限のポリシーのみ付与 |
| Webhook検証 | 購読トークンとチャネルIDの二重検証 |

## テスト戦略

- **テストコード**: 約46,000行
- **フレームワーク**: Jest + ts-jest
- **モック**: aws-sdk-client-mock（AWS SDK v3用）

| 層 | テスト対象 | 方針 |
| --- | --- | --- |
| Domain | 値オブジェクト、エンティティ | 純粋な単体テスト |
| Application | ユースケース、認証ポリシー | Port層をモックして検証 |
| Infrastructure | リポジトリ、デコレーター | AWS SDK Mockで検証 |
| Interface | Lambdaハンドラー | イベント→レスポンス検証 |

- 値オブジェクトのバリデーション、等価性、不変性を網羅的に検証
- 正常系だけでなく、各層のエラー変換・伝播を詳細にテスト
- キャッシュ・ロギングデコレータの個別テスト＋統合テスト

## 技術的意思決定（ADR）

21件のArchitecture Decision Recordを記録し、
設計判断の根拠をドキュメント化。代表的なもの：

- **ADR-006**: エラー処理アーキテクチャ（3層分離 + エラートランスレーター）
- **ADR-014**: API Gateway vs Lambda Function URL（ルート別認証の必要性からAPI Gatewayを採用）
- **ADR-015**: DynamoDBシングルテーブル設計（高頻度アクセス・トランザクション・TTL対応）
- **ADR-021**: マルチアカウント・パイプライン（本番環境の安全性を物理的に担保）

| ADR | テーマ |
| --- | --- |
| ADR-001 | Repository vs Service 命名判断 |
| ADR-002 | 月次トランザクションのMapキー設計 |
| ADR-003 | CSVパーサーのインフラ層配置 |
| ADR-004 | キャッシング用Decoratorパターン |
| ADR-007 | Money値オブジェクトの型設計 |
| ADR-008 | A1記号法パースのドメイン層配置 |
| ADR-009 | Sheet構造マッピングのFactoryパターン |
| ADR-010 | UseCase責務（検証と通知） |
| ADR-011 | ドメインサービス責務分離 |
| ADR-012 | Chain of Responsibilityループスタイル |
| ADR-013 | エラーコンテキストレイヤー分離 |
| ADR-016 | DriveWatch Aggregate設計 |
| ADR-017 | べき等性設計（上書き方式） |
| ADR-018 | PageTokenベース変更検出 |
| ADR-019 | PageTokenと購読の分離 |
| ADR-020 | 変更検出でページネーション非実施 |

## CI/CDとインフラ

### パイプライン構成

AWS CDK Pipelines（CodePipeline + CodeBuild）を採用。

![Pipeline](/diagrams/moneyforward2sheet-pipeline.png)

### マルチアカウント構成

| アカウント | 用途 | デプロイ方法 |
| --- | --- | --- |
| dev | パイプライン + 開発 | 手動 `cdk deploy` |
| prod | 本番環境 | パイプライン経由のみ |

### IaC

すべてのAWSリソースをCDK TypeScriptで定義。
手動構築のリソースは存在しない。

- 各Lambdaの権限は必要最小限のみ付与
- CDK Feature Flags（95+）を有効化し、最新のベストプラクティスを適用
- リソース命名にステージプレフィックスを付与し、環境間の分離を保証
