---
title: Book Index App
description: 書籍の索引（単語とページ番号の対応）をユーザーごとに管理するフルスタックWebアプリケーション
role: 個人開発。要件定義 / 設計 / 実装 / テスト / CI/CD のすべてを単独で担当
featured: false
order: 7
techStack:
  - category: 言語
    tech: TypeScript
  - category: フロントエンド
    tech: React 18 / Vite / Mantine 7 / Tailwind CSS
  - category: API
    tech: GraphQL (Apollo Client + Apollo Server)
  - category: バックエンド
    tech: NestJS 10
  - category: ORM
    tech: Prisma 5
  - category: データベース
    tech: PostgreSQL
  - category: 認証
    tech: AWS Cognito
  - category: コンテナ
    tech: Docker / Docker Compose
  - category: CI/CD
    tech: GitHub Actions / AWS CodeBuild / ECR
stats:
  - value: "322"
    label: lines test
  - value: "8"
    label: GraphQL operations
  - value: "2"
    label: CI workflows
---

## 解決した課題

紙の書籍を読みながら索引を手動管理する手間を、
Web上でのCRUD操作に置き換えた。
AWS Cognitoによるユーザー認証と
ownerIdによるデータ分離で、
マルチユーザー対応の永続化を実現。

## 主な機能

- 書籍の登録・編集・削除
- 書籍ごとの単語索引管理（単語名 + ページ番号配列）
- AWS Cognitoによるサインアップ/サインイン
- ユーザーごとのデータ分離

## システムアーキテクチャ

### データフロー

![Architecture](/diagrams/book-index-app-architecture.png)

### GraphQL API

8つのQuery/Mutationを定義。
書籍と単語の親子関係を持つデータに対して、
クライアント側で必要なフィールドのみ取得可能な
GraphQLの特性を活用。
全操作にJWT認証を適用。

## 設計

### NestJSモジュール設計

NestJSのDIコンテナを活用し、
モジュール間の依存関係を明示的に管理。
GraphQLModule、PrismaModule、BookModule、
LoggerModuleの4モジュール構成。

### フィーチャーベースのフロントエンド構成

機能ごとに api/components/hooks/types を配置。
auth、book、word、landingの4フィーチャーに分離し、
共通ロジックはproviders/hooks層に集約。

### Cognito JWT認証フロー

フロントエンドでApollo ClientのauthLinkにより
全リクエストにJWTヘッダーを自動付与。
バックエンドではNestJSのGuardパターンで
aws-jwt-verifyによるResolver単位の認証制御を実装。

### 例外ハンドリング 2層構成

NestJSのException Filterを2層で構成。
HttpExceptionFilterとPrismaExceptionFilterにより、
エラーをGraphQLエラー拡張（extensions.code）に変換。

### 2層バリデーション

フロントエンド（Zod）とバックエンド（class-validator）で
入力検証を二重に実施。

## テスト

テストコード322行。Jest + React Testing Library +
Supertestで、ユニットテスト・E2Eテストを実装。
JWT有効性チェックフックとルートガードの
モック化による依存隔離テストを実施。

## CI/CDとインフラ

GitHub Actionsで2つのワークフローを構成。
PRへのOpenAI自動コードレビューと、
mainブランチからAWS CodeCommitへの同期。
CodeBuildでDockerビルド後、ECRにプッシュ。
Docker Composeでローカル開発環境を構築。
