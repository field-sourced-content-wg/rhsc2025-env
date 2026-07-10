# RHSC 2025 Environment - Field Content Demo

## プロジェクト概要

このプロジェクトは、Red Hat Summit Connect 2025のデモ環境を構築するためのHelmチャートです。ArgoCD の "App of Apps" パターンを使用して、複数のコンポーネントをGitOpsで管理します。

## 目的

OpenShiftクラスター上に以下を自動デプロイし、統合されたデモ環境を提供する：

1. **Operatorのインストール**
   - MTA (Migration Toolkit for Applications)
   - CloudNativePG (PostgreSQL)
   - DevSpaces (開発環境)
   - RHBK (Red Hat Build of Keycloak)
   - Web Terminal

2. **Showroomラボガイド**
   - インタラクティブなラボガイドとターミナル
   - Nookbag統合（LlamaStack、複数ターミナル、Zero Trust UI）

3. **OpenShift ConsoleのiFrame埋め込み許可**
   - ShowroomにConsoleを埋め込めるようにする

4. **ワークショップユーザーの一括作成**
   - HTPasswd認証プロバイダーによる複数ユーザー作成
   - 各ユーザーにDevSpacesとMTAへのアクセス権を自動付与

## プロジェクト構造

```
rhsc2025-env/
├── helm/
│   ├── Chart.yaml              # メインチャート定義
│   ├── values.yaml             # 設定値（全コンポーネント）
│   ├── templates/
│   │   └── applications.yaml   # ArgoCD Applicationリソース生成
│   └── components/             # 各コンポーネントのサブチャート
│       ├── operator/           # Operator Subscription
│       ├── secrets/            # API Keys (LiteMaaS)
│       ├── tackle/             # Tackle CR (MTA)
│       ├── checluster/         # CheCluster CR (DevSpaces)
│       ├── console-embed/      # Console iframe設定
│       ├── showroom/           # Showroomデプロイ
│       └── users/              # ワークショップユーザー作成
└── README.md
```

## アーキテクチャパターン

### App of Apps Pattern

- `helm/templates/applications.yaml`が全コンポーネントのArgoCD Applicationを生成
- 各Applicationは`helm/components/`配下のサブチャートを参照
- `values.yaml`の`enabled`フラグで各コンポーネントの有効/無効を制御

### 依存関係の順序

1. Operator → Namespace作成 + Subscription
2. Secrets → Operatorインストール後にAPIキーを配置
3. Tackle/CheCluster CR → Operator Ready後にカスタムリソース作成

## 重要な設定ファイル

### `helm/values.yaml`

全コンポーネントの設定を一元管理：

- `gitops.*`: GitリポジトリとArgoCD設定
- `operators.*`: 各Operatorの有効化とチャネル設定
- `showroom.*`: ラボガイドのコンテンツリポジトリとNookbag設定
- `consoleEmbed.enabled`: Console埋め込み許可の有効化
- `users.*`: ワークショップユーザー設定（ユーザー数、パスワード、権限）
- `litemaas.*`: LiteMaaS APIキー（RHDP環境で自動注入）
- `deployer.domain`: クラスタードメイン（RHDP環境で自動注入）

### `helm/components/console-embed/`

OpenShift ConsoleにShowroomからのiFrame埋め込みを許可するための設定：

- Routeリソースで`haproxy.router.openshift.io/hsts_header`を`none`に設定
- これにより、Showroom内にConsoleを埋め込み可能

### `helm/components/users/`

ワークショップ参加者用のOpenShiftユーザーを一括作成：

- HTPasswd Identity Providerの設定
- User/Identityリソースの作成
- 各ユーザーに専用の`<username>-devspaces` namespaceを作成
- DevSpacesとMTAへのRoleBinding設定
- 詳細は `helm/components/users/README.md` を参照

## よく使うコマンド

### Helmテンプレートの確認

```bash
helm template field-content-demo ./helm --values ./helm/values.yaml
```

### 特定のコンポーネントのみレンダリング

```bash
# MTA operator のみ
helm template field-content-demo ./helm --values ./helm/values.yaml | grep -A 50 "name: field-content-demo-mta"
```

### values.yamlの検証

```bash
helm lint ./helm
```

## デプロイフロー

1. RHDP（Red Hat Demo Platform）がHelmチャートをデプロイ
2. `deployer.domain`と`litemaas.*`の値が自動注入される
3. ArgoCD ApplicationがGitリポジトリを監視し、各コンポーネントをデプロイ
4. 依存関係に従って順次リソースが作成される

## 注意事項

### Operator設定

- `devspaces`と`webterminal`は`allNamespaces: true`が必要
- `installPlanApproval: Automatic`で自動更新を許可

### Console埋め込み

- `console-embed`コンポーネントは独立したRouteリソースを作成
- 以前の`unsupportedConfigOverrides`アプローチから変更（PR #3）

### Showroom設定

- コンテンツリポジトリは別管理（`rhsc2025-showroom`）
- Nookbagバンドルはリリース版を使用

## トラブルシューティング

### Operatorがインストールされない

```bash
oc get subscription -n <namespace>
oc get installplan -n <namespace>
```

### ArgoCD Applicationの状態確認

```bash
oc get application -n openshift-gitops
oc describe application field-content-demo-<component> -n openshift-gitops
```

### Showroomが起動しない

```bash
oc get pods -n showroom
oc logs -n showroom <showroom-pod-name>
```

## 開発ワークフロー

1. `helm/values.yaml`または各コンポーネントのテンプレートを編集
2. `helm template`でレンダリング結果を確認
3. Gitにコミット・プッシュ
4. ArgoCD自動同期または手動Sync

## 関連リポジトリ

- **環境デプロイ（このリポジトリ）**: `field-sourced-content-wg/rhsc2025-env`
- **Showroomコンテンツ**: `field-sourced-content-wg/rhsc2025-showroom`

## メンテナ

- Field Content Developer <developer@redhat.com>
