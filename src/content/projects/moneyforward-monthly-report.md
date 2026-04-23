---
title: MoneyForward Monthly Report
description: MoneyForward ME の家計簿CSVを月次でカテゴリ別集計し、Notion と DynamoDB へ自動書き込みするサーバーレスパイプライン
role: 個人開発。要件定義 / 設計 / 実装 / インフラ / 運用 のすべてを単独で担当
featured: false
order: 11
techStack:
  - category: 言語
    tech: TypeScript 5.7 / Node.js 20.x
  - category: IaC
    tech: AWS CDK 2.171 (TypeScript)
  - category: コンピュート
    tech: Lambda
  - category: トリガー
    tech: S3 ObjectCreated Event
  - category: データ
    tech: DynamoDB (OnDemand)
  - category: シークレット
    tech: SSM Parameter Store (SecureString)
  - category: 可観測性
    tech: AWS X-Ray / CloudWatch Logs / Powertools
  - category: 外部連携
    tech: Notion API / papaparse / iconv-lite
  - category: テスト
    tech: Jest 29 + ts-jest
stats:
  - value: "9"
    label: Lambda modules
  - value: "4"
    label: env strategies
  - value: "6"
    label: test files
  - value: "525"
    label: lines test
---

## 解決した課題

MoneyForward ME からエクスポートした家計簿CSVを
毎月手作業でダウンロード・集計・Notionへ転記していた運用を、
S3 へのアップロードを起点にした
イベント駆動のサーバーレスパイプラインで全自動化した。

## 主な機能

- **S3トリガー**: CSVアップロードを `ObjectCreated` イベントで検知し Lambda を起動
- **エンコーディング変換**: Shift_JIS で出力される MoneyForward CSV を UTF-8 へ変換
- **カテゴリ自動分類**: 中項目のプレフィックス(`1.〜4.`)に基づきカテゴリ別に集計
- **データ整形**: 収入・振替行の除外、金額の絶対値化、日付の `yyyy-MM` 整形
- **同時書き込み**: Notion DB（月次レポート）と DynamoDB（履歴）へ並行書き込み

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/moneyforward-monthly-report-architecture.png)

S3 への CSV アップロードを `ObjectCreated` イベントで検知し、
Lambda が SSM Parameter Store から Notion 認証トークン等を取得、
Shift_JIS の CSV を UTF-8 へ変換してカテゴリ別に集計、
Notion DB（月次レポート）と DynamoDB（履歴）へ並行書き込みする。
X-Ray と CloudWatch Logs で分散トレーシングと構造化ログを出力する。

### トリガー設計

HTTP 経路やスケジュール起動は持たず、
「CSVをアップロードする」というユーザー行動に処理を紐づけ、
S3 ObjectCreated による非同期起動のみとした。
処理の起点を単一の行動に絞ることで、
運用面の責任分界点をシンプルに保つ。

## 設計・アーキテクチャ

### 処理パイプライン

![Pipeline](/diagrams/moneyforward-monthly-report-pipeline.png)

`handler.ts` は S3 イベントの受信から
SSM 取得 → 環境変数バリデーション → CSV 取得 →
UTF-8 デコード → CSV→JSON 変換 → カテゴリ別集計 →
Notion / DynamoDB 書き込みまでをオーケストレーションする。
各ステップは独立モジュールに分離し、
責務が漏れないよう 1 クラス 1 責務を徹底した。

### 責務分離（9モジュール構成）

Lambda コードは役割ごとに 9 モジュールへ分割。

```text
lambda/
├── handler.ts              # オーケストレーション
├── s3EventExtractor.ts     # S3からのCSV取得
├── encodingConverter.ts    # Shift_JIS→UTF-8変換
├── csvToJsonConverter.ts   # CSV→JSONパース
├── expenseDataExtractor.ts # 日付抽出・データ整形
├── expenseCalculator.ts    # カテゴリ別集計
├── notion.ts               # Notion API クライアント
├── dynamo.ts               # DynamoDB クライアント
├── ssm.ts                  # SSMパラメータ取得
├── error/appError.ts       # アプリ固有例外
└── utils/                  # 横断的関心事
    ├── decorator/          # エラーハンドリングデコレータ
    ├── policy/             # バリデーションポリシー
    ├── strategy/           # バリデーション戦略
    └── logger.ts           # カスタムロガー
```

### Strategy + Policy パターンによる環境変数バリデーション

SSM から取得する 4 つの環境変数
（`NOTION_AUTH` / `NOTIONDB_ID` / `CATEGORY_LIST` / `DYNAMODB_TABLE_NAME`）
のチェックを、3 層で責務分離した。

- **Strategy レイヤ**: どの変数かを切り替え（`EnvVarStrategy` インターフェース）
- **Policy レイヤ**: ルールの組合わせを定義（`EnvVarPolicy` が複数 Rule を合成）
- **Rule レイヤ**: 個別の検査ロジック
  （`NotUndefinedRule` / `NotEmptyStringRule` / `ValidNotionAuthRule` など）

新しい環境変数を追加する際は、Rule と Policy の組立てだけで
既存の Strategy に影響を与えずに拡張できる。

### Decorator パターンによる横断的エラーハンドリング

全クラスの公開メソッドに
`@asyncFuncErrorHandler` / `@syncFuncErrorHandler` を適用し、
以下を一括で実現する。

- 関数の開始・終了を自動ロギング（`logger.funcStart` / `funcEnd`）
- 例外を `AppError` に正規化して再スロー
- `Error` でない例外を汎用メッセージに変換

各メソッド内の try-catch 記述を排除し、
ドメインロジックに集中できる構造にした。

### エラー分類

`AppError` は発生した関数名を保持し、
`handler.ts` では業務例外（`AppError`）と
想定外例外（その他）を出し分けてログ出力する。
障害発生時にどの関数で何が起きたかを
ログ 1 行で特定できるようにした。

### X-Ray 分散トレーシング

`tracer.captureAWSv3Client` で S3 / SSM / DynamoDB の SDK クライアントをラップし、
Lambda の `tracing: ACTIVE` と合わせて
CSV 受信から書き込み完了までのタイムラインを
1 リクエスト単位で追跡できる。

### 機密情報管理

Notion 認証トークンは SSM Parameter Store の SecureString に格納し、
Lambda 環境変数には SSM のキー名のみを置く。
トークンの形式(`secret_[A-Za-z0-9]{43}`)は
正規表現でバリデーションし、設定ミスを起動時に検知する。

## テスト戦略

| 層                     | 方針                                             |
| ---------------------- | ------------------------------------------------ |
| 集計ロジック           | カテゴリ別合計値、収入・振替・計算対象外の除外   |
| CSV パース             | 文字列 → `ExpenseData` 配列、boolean 変換        |
| 日付抽出               | `yyyy/MM/dd` → `yyyy-MM` 形式                    |
| 環境変数バリデーション | Strategy 単位・Policy 単位で網羅                 |
| CDK スタック           | リソース合成テスト                               |

`test/fixtures/` に実データ由来の CSV（3 ヶ月分）を配置し、
エンコーディング変換を含む統合的なパスも Jest 上で検証できるようにした。

## インフラ・IaC

### CDK スタック構成

`lib/moneyforward_monthly_report-stack.ts` に 1 スタックで定義。

| リソース         | 設定                                            |
| ---------------- | ----------------------------------------------- |
| S3 Bucket        | RemovalPolicy: DESTROY（開発用）                |
| Lambda           | Node.js 20.x / timeout 30s / X-Ray Active       |
| Lambda Role      | SSM / CloudWatch / S3 / X-Ray 管理ポリシー      |
| DynamoDB Table   | OnDemand 課金 / RemovalPolicy: RETAIN           |
| S3 Notification  | ObjectCreated → Lambda Destination              |

### IAM 最小権限

Lambda ロールの DynamoDB 操作は、
対象テーブルの ARN に限定した `PolicyStatement` を
インラインで付与する。
マネージドポリシーは読み取り系のみ採用。

### 環境変数と SSM の分離

Lambda 環境変数には SSM のパラメータ名のみを格納し、
値の取得は実行時に `SSMParameterFetcher` で行う。
SecureString / String 両方を `Promise.all` で並行取得し、
コールドスタート時間を抑えつつ
シークレットの平文化を回避する。
