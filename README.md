# ECS Config — Angular (ecs-config-angular-repo)

Multi-pipeline ECS Platform における Angular アプリケーション（`ecs-angular-repo`）向けの ECS (Fargate) デプロイ設定・ecspresso 定義を管理するリポジトリです。

## ディレクトリ構成

```
.
├── buildspec_scripts/
│   ├── buildspec_deploy.yml           # CD パイプライン用デプロイスクリプト (ecspresso)
│   └── buildspec_hotfix_deploy.yml    # ホットフィックス用デプロイスクリプト
├── ecspresso/
│   ├── ecspresso-config.yml           # ecspresso 基本設定 (クラスター名・サービス名など)
│   ├── service-def.json               # ECS サービス定義テンプレート
│   └── task-def.json                  # ECS タスク定義テンプレート (コンテナポート 8080)
├── deploy-config.json                 # CPU / メモリ / タスク数 (desiredCount) 設定
└── README.md
```

## 各設定ファイルの役割

| ファイル / ディレクトリ | 説明 |
| --- | --- |
| `deploy-config.json` | タスクに割り当てる CPU (`256`)、メモリ (`512` MiB)、タスク数 (`desiredCount`) を定義します。 |
| `ecspresso/task-def.json` | Fargate タスク定義のテンプレートです。ECR イメージ URI、コンテナポート（`8080`）、IAM ロール、ロググループなどを定義します。 |
| `ecspresso/service-def.json` | ECS サービス定義のテンプレートです。デプロイ設定やネットワーク設定などを定義します。 |
| `buildspec_scripts/buildspec_deploy.yml` | CodeBuild 内で ecspresso をインストールし、開発アカウントへ AssumeRole して SSM パラメータと `deploy-config.json` を取得・マージした上で `ecspresso deploy` を実行します。 |

## CI/CD および GitHub Actions との連携

全体のデプロイフローおよび検証の仕組みは以下のとおりです。

```mermaid
graph TD
    subgraph GitHub
        PR[Pull Request / Push] --> GHA["GitHub Actions (CodeBuild ランナー)<br/>sonar-analysis.yml"]
        GHA -->|検証OK & マージ| Main[main ブランチ]
    end

    subgraph AWS CI/CD (senjo2)
        Main -->|アプリコード変更| CI["CI パイプライン<br/>(App-Build → Image-Build)"]
        CI -->|IMAGE_TAG 通知| CD["CD パイプライン<br/>(Deploy: ecspresso)"]
        ConfigPush["ecs-config push<br/>(設定変更)"] -->|IMAGE_TAG=__CURRENT__| CD
    end

    subgraph AWS ECS (senjo)
        CD -->|ecspresso deploy| Service["ECS Fargate サービス<br/>(Angular App)"]
    end
```

### 1. GitHub Actions による事前検証

- アプリリポジトリ (`app/angular`) の `.github/workflows/sonar-analysis.yml` により、PR 作成・更新時および `main` push 時に単体テスト（Karma + ChromeHeadless）、静的解析（ESLint）、SonarQube 解析（品質ゲート待機）が実行されます。
- GitHub Actions の実行基盤には AWS CodeBuild マネージドランナー (`runs-on: codebuild-...`) が使用されます。

### 2. CD パイプラインのトリガー

- **アプリコード更新時**: CI パイプライン（App-Build → Image-Build）完了後、生成された ECR イメージタグを受け取って CD パイプラインが自動起動し、本リポジトリの設定を使って ECS へデプロイします。
- **ECS 設定更新時**: 本リポジトリ（`ecs-config-angular-repo`）の `ecspresso/**` や `deploy-config.json` を変更して push すると、CD パイプライン単体が起動します。イメージタグは稼働中のタグ（`__CURRENT__`）が維持され、タスク定義やサービス定義の設定のみが更新されます。

## 設定変更とデプロイ手順

### タスク数 (`desiredCount`) の変更

初期状態では `deploy-config.json` の `desiredCount` は `0` に設定されています。
タスクを起動・変更する場合は、値を編集して push します。

```json
{
    "memoryLimitMiB": 512,
    "cpu": 256,
    "desiredCount": 1
}
```

コミット & push すると CD パイプラインが自動起動し、指定したタスク数で ECS サービスが更新されます。
