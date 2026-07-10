# Workshop Users Component

このコンポーネントは、ワークショップ参加者用のOpenShiftユーザーとadminユーザーを一括作成し、DevSpacesとMTAへのアクセス権を付与します。

## 機能

- **adminユーザーの作成**: cluster-admin権限を持つadminユーザーを作成
- **HTPasswd Secretの作成**: OpenShift OAuth用のHTPasswd認証情報を作成
- **ワークショップユーザーの作成**: 指定した数のユーザー（user1, user2, ...）を作成
- **個別のDevSpaces namespace**: 各ユーザーに専用の`<username>-devspaces` namespaceを作成
- **権限の自動付与**:
  - adminユーザー: `cluster-admin`権限
  - 各ワークショップユーザー:
    - 自分のDevSpaces namespaceへの`admin`権限
    - `openshift-devspaces` namespaceへの`view`権限
    - `openshift-mta` namespaceへの`view`権限

## ⚠️ 重要な変更点（OAuth管理について）

**デフォルトではOAuth設定を自動管理しません。** これにより既存のIdentity Provider（kubeadmin等）が削除されるのを防ぎます。

- `manageOAuth: false`（デフォルト）: HTPasswd Secretのみ作成。OAuth設定は手動で行う必要があります
- `manageOAuth: true`: OAuth設定を自動管理（**既存のIdentity Providerを上書きするため注意**）

## 設定

### `values.yaml`

```yaml
users:
  enabled: true
  count: 20                    # 作成するユーザー数
  prefix: "user"               # ユーザー名のプレフィックス (user1, user2, ...)
  password: "openshift"        # 全ユーザー共通のパスワード
  devspacesAccess: true        # DevSpacesアクセス権の付与
  mtaAccess: true              # MTAアクセス権の付与
  manageOAuth: false           # OAuth設定の自動管理（デフォルト: false）
  admin:
    enabled: true              # adminユーザーの作成
    username: "admin"          # adminユーザー名
    password: "openshift"      # adminパスワード（htpasswdハッシュは自動生成）
```

## 作成されるリソース

### 1. HTPasswd Secret (sync-wave: 1)
- **Namespace**: `openshift-config`
- **Name**: `workshop-users-htpasswd`
- adminユーザーと全ワークショップユーザーの認証情報を含むhtpasswdファイル

### 2. OAuth Configuration (sync-wave: 2) ※`manageOAuth: true`の場合のみ
- **Name**: `cluster`
- HTPasswd Identity Providerを追加
- ⚠️ **警告**: 既存のIdentity Providerを上書きします

### 3. User & Identity (sync-wave: 3)
- adminユーザーの`User`と`Identity`リソース
- 各ワークショップユーザーごとに:
  - `User` リソース
  - `Identity` リソース
  - 専用の `<username>-devspaces` namespace

### 4. RoleBindings (sync-wave: 4)
- `admin-cluster-admin`: adminユーザーへのcluster-admin権限
- 各ワークショップユーザーごとに:
  - `<username>-admin`: 自分のDevSpaces namespaceへのadmin権限
  - `<username>-devspaces-view`: openshift-devspaces namespaceへのview権限
  - `<username>-mta-view`: openshift-mta namespaceへのview権限

## デプロイ方法

### 推奨: 手動でOAuth設定（デフォルト）

1. **Helmチャートをデプロイ**（`manageOAuth: false`のまま）
   - HTPasswd Secret、User/Identity、RoleBindingが作成されます

2. **OpenShift Consoleで手動でOAuth設定を追加**
   ```bash
   oc edit oauth cluster
   ```
   
   既存の`identityProviders`配列に以下を**追加**:
   ```yaml
   spec:
     identityProviders:
     - name: workshop-users-htpasswd
       mappingMethod: claim
       type: HTPasswd
       htpasswd:
         fileData:
           name: workshop-users-htpasswd
   ```

3. **確認**
   ```bash
   # OAuth設定の確認
   oc get oauth cluster -o yaml
   
   # ログインテスト
   oc login -u admin -p openshift
   ```

### 代替: 自動でOAuth設定（既存環境を上書き）

⚠️ **警告**: この方法は既存のIdentity Providerを削除します。kubeadminでログインできなくなる可能性があります。

```yaml
users:
  manageOAuth: true  # 自動管理を有効化
```

## デプロイ順序

ArgoCD sync-waveにより以下の順序で実行されます:

1. **Wave 1**: HTPasswd Secret作成
2. **Wave 2**: OAuth設定更新（`manageOAuth: true`の場合のみ）
3. **Wave 3**: User/Identity/Namespace作成
4. **Wave 4**: RoleBinding作成

## 使用例

### ユーザー数を10に設定

```yaml
users:
  count: 10
```

→ `user1`〜`user10`が作成されます

### パスワードの変更

デフォルトパスワード「openshift」を変更する場合:

1. 新しいパスワードのbcryptハッシュを生成:
```bash
htpasswd -nbB user1 <new-password> | cut -d: -f2
```

2. `templates/htpasswd-secret.yaml`の`$passwordHash`変数を更新

### adminユーザーを無効化

```yaml
users:
  admin:
    enabled: false
```

### MTAアクセスのみ無効化

```yaml
users:
  devspacesAccess: true
  mtaAccess: false
```

## ログイン方法

ユーザーは以下の情報でOpenShiftにログインできます:

- **adminユーザー**:
  - ユーザー名: `admin`
  - パスワード: `openshift` (デフォルト)
  - 権限: cluster-admin

- **ワークショップユーザー**:
  - ユーザー名: `user1`, `user2`, ..., `user<N>`
  - パスワード: `openshift` (デフォルト)
  - Identity Provider: `workshop-users-htpasswd`

## トラブルシューティング

### adminでログインできない

**原因**: `manageOAuth: true`にして既存のIdentity Providerが削除された可能性があります。

**対策**:
1. kubeadminでログイン（まだ有効な場合）
2. OAuth設定を手動で修正:
   ```bash
   oc edit oauth cluster
   ```
3. 次回から`manageOAuth: false`に設定し、手動でOAuth設定を管理

### ユーザーがログインできない

```bash
# HTPasswd Secretの確認
oc get secret workshop-users-htpasswd -n openshift-config
oc get secret workshop-users-htpasswd -n openshift-config -o jsonpath='{.data.htpasswd}' | base64 -d

# OAuth設定の確認
oc get oauth cluster -o yaml

# Userリソースの確認
oc get users | grep user
```

### DevSpacesにアクセスできない

```bash
# RoleBindingの確認
oc get rolebinding -n openshift-devspaces | grep user1

# Namespaceの確認
oc get namespace | grep devspaces
```

## 注意事項

- **デフォルトでは`manageOAuth: false`** - 既存のIdentity Providerを保護します
- OAuth設定を自動管理する場合（`manageOAuth: true`）、既存のIdentity Providerが削除されるため注意
- 全ユーザーが同じパスワードを使用するため、本番環境では個別パスワードの設定を推奨
- CheClusterの`defaultNamespace.template`が`<username>-devspaces`に設定されている必要があります
