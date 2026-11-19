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

### 1. GitHub Actions による事前品質検証

- **トリガー**: アプリ開発ブランチから `main` への PR 作成・更新時、または `main` ブランチへの push 時
- **ランナー挙動**: AWS VPC 内の SonarQube と通信可能な CodeBuild エピメラルランナーがオンデマンドで起動
- **検証フロー**:
  1. Node.js 20 環境のセットアップ
  2. SonarQube サーバーへの疎通・認証チェック
  3. Karma + ChromeHeadless による単体テストおよびカバレッジ（`coverage/lcov.info`）出力
  4. ESLint による静的解析（`eslint-report.json` 出力）
  5. SonarQube への解析送信と Quality Gate 判定（判定結果が出るまで待機し、NG の場合はジョブ失敗＝PR マージ不可）

### 2. AWS CodePipeline (本番 CI/CD) との連携

- **アプリコード更新（PR マージ後）**:
  - `main` ブランチへのマージを検知して CodePipeline CI（App-Build → Image-Build）が起動
  - Image-Build でビルドされたコンテナイメージが ECR へ push され、イメージタグが CD パイプラインへ通知される
  - CD パイプラインが本リポジトリ（`ecs-config-angular-repo`）の `ecspresso` 設定と `deploy-config.json` を参照し、新しいイメージで ECS サービスをローリングアップデート
- **ECS 設定更新時（本リポジトリへの push）**:
  - `ecspresso/**` や `deploy-config.json` の変更を検知して **CD パイプライン単体** が起動
  - イメージタグには現在稼働中のタグ（`__CURRENT__`）が自動適用され、コンテナイメージを再ビルドすることなく CPU / メモリ / タスク数 / 環境変数の変更が即座に ECS へ反映

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
