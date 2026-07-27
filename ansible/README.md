# Ansible Post-Deploy Playbook

Helm + ArgoCD によるデプロイ完了後に、手動で行っていた設定作業を自動化する Ansible Playbook。

## 背景

FSC パターン（Helm + ArgoCD）で環境をデプロイした後、以下の作業が Helm では対応できない:

| 手動作業 | 原因 |
|---|---|
| Operator の起動完了待ち | Helm はリソース作成のみ、状態待ちは不可 |
| CR の Ready 状態待ち | 同上 |
| MTA Keycloak のパスワードリセット | Operator が作成する認証情報を外から変更できない |
| API 接続確認 | OAuth トークン取得が必要 |

この Playbook はこれらを自動化し、`ansible-playbook` 1コマンドで Post-Deploy を完了させる。

## 前提条件

- `oc` コマンドがインストール済み
- `oc login` でクラスタにログイン済み（cluster-admin 権限）
- Ansible がインストール済み（`pip install ansible`）
- Helm + ArgoCD によるデプロイが完了済み（全 Application が作成済み）

## 使い方

### 1. 変数の設定

`vars/main.yml` を環境に合わせて編集:

```yaml
# 必須: MTA admin のパスワード
mta_admin_password: <SET_YOUR_PASSWORD>
```

`cluster_domain` は空のままで OK — Playbook が自動検出する。

### 2. 実行

```bash
cd ansible
ansible-playbook post-deploy.yml
```

### 3. 実行結果

成功すると以下のサマリが表示される:

```
============================================
MTA Workshop Post-Deploy Complete
============================================
MTA UI:      https://mta-openshift-mta.<cluster-domain>
Dev Spaces:  https://devspaces.<cluster-domain>
============================================
MTA Login:   admin / <your-password>
============================================
```

## ロール構成

| ロール | 内容 | タイムアウト |
|---|---|---|
| `wait-for-operators` | 全 Operator の CSV が Succeeded になるまで待機 | 10分 |
| `wait-for-resources` | ArgoCD Application が Synced/Healthy になるまで待機 | 10分 |
| `configure-mta-keycloak` | MTA Keycloak admin パスワードリセット、requiredActions 解除 | - |
| `validate-deployment` | MTA UI / Dev Spaces / Hub API の疎通確認 | - |

## カスタマイズ

### Operator を追加する場合

`vars/main.yml` の `operators` リストに追加:

```yaml
operators:
  - name: new-operator
    namespace: new-namespace
```

### タイムアウトを変更する場合

```yaml
operator_wait_timeout: 900   # 15分に延長
resource_wait_timeout: 900
```
