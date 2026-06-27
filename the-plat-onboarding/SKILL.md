---
name: the-plat-onboarding
description: >-
  Platform City（The Plat）の EKS クラスタへ初めてアクセスするためのオンボーディング手順。AWS
  SSO プロファイル作成、EKS 用 IAM ロールの AssumeRole、kubeconfig 設定、専用
  namespace の作成、動作確認用リソースのデプロイ、後片付けまでを案内する。PEK2026
  実行委員会メンバーが自分の開発環境をセットアップするとき、または「EKS につながらない /
  kubectl が認証エラー」のトラブル時に使う。Platform City / The Plat / pek2026 の EKS や
  kubeconfig に触れる作業では、明示的に「オンボーディング」と言われなくても本スキルを参照すること。
---

# The Plat オンボーディング

Platform City の EKS クラスタにアクセスし、Kubernetes リソースを作成・削除できるようになるまでの手順。**PEK2026 実行委員会メンバーが自分の開発環境をセットアップする**ためのもの（参加者向けの GitLab/MCP フローとは別）。

## このスキルの使い方（エージェント向け）

- ブラウザ認証を伴うコマンド（`aws sso login` など）は**ユーザー自身に実行してもらう**。プロンプトで `! aws sso login --profile pek2026` のように `!` 付きで打つよう案内するか、ユーザーがブラウザでログインを完了するのを待つ。
- `kubectl delete namespace` などの破壊的コマンドは**実行前に必ずユーザーに確認**する。
- 各ステップ後に確認コマンドの出力を見て、次に進めるか判断する。

## 前提条件

- AWS Identity Center（AWS access portal）から **pek2026 アカウントの AdministratorAccess** が選択できること。
- 必要な CLI がインストール済みであること（下記「必要な CLI」を参照）。

## 必要な CLI

以下の CLI を使う。**手順を始める前に、まず全てが入っているか確認し、不足していればユーザーに案内してセットアップする**（下記「事前チェック」参照）。

> **対応 OS**: macOS / Linux に対応。**Windows の場合は WSL2（Ubuntu などの Linux 環境）上で作業し、以下の Linux 手順に従う**（PowerShell ネイティブは想定しない）。以降のコマンドはすべて macOS または Linux/WSL2 のシェルで実行する。

| CLI | 用途 | 確認 |
|---|---|---|
| `aws`（**v2.9 以降**） | AWS SSO ログイン・kubeconfig 更新 | `aws --version` |
| `kubectl` | Kubernetes 操作 | `kubectl version --client` |
| `gh` | GitHub ユーザー名取得（namespace 名に使用） | `gh auth status` |
| `kubectx` / `kubens` | コンテキスト / namespace 切り替え | `kubens --version` |

> ⚠️ **aws CLI は v2.9 以降が必須。** 手順 1 の `[sso-session]` 形式は v2.9（2022年11月）以降でしか解釈されない。古い v2 だと `aws sso login` が `Missing the following required SSO configuration values: sso_start_url, sso_region` で失敗する。`aws --version` が v2.9 未満なら下記「インストール」の aws CLI 更新手順（`--update` 付き）で最新化する。

### 事前チェック（エージェント向け）

手順 1 に進む前に、各 CLI の有無をまとめて確認する（OS 共通）。

```bash
for c in aws kubectl gh kubens; do
  if command -v "$c" >/dev/null 2>&1; then
    echo "OK   $c"
  else
    echo "MISSING $c"
  fi
done
```

- `MISSING` があれば、**勝手に入れず**にユーザーへ伝え、下記「インストール」から OS に合った手順の実行を促す。インストールはユーザーに `! <install command>` の形で実行してもらう。
- `gh` は入っていても未ログインだと `gh api user` が失敗するため、`gh auth status` でログイン状態も確認し、未ログインなら `! gh auth login` を促す。
- `aws --version` も確認し、**v2.9 未満なら更新を促す**（下記「インストール」の aws CLI 更新手順）。古いままだと手順 1 の SSO ログインが失敗する。
- 全て揃っていることを確認してから手順 1 に進む。

### インストール

**macOS（Homebrew）**

```bash
brew install awscli kubectl gh kubectx   # kubectx は kubens も同梱
```

**Linux / WSL2**

Homebrew on Linux が使えるなら macOS と同じ `brew install` でよい。ネイティブに入れる場合は以下（公式手順ベース。ディストリのパッケージは古いことがあるため公式インストーラを優先）。

```bash
# aws CLI v2（公式インストーラ / x86_64。arm64 は awscli-exe-linux-aarch64.zip）
# 新規インストール:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
# 既存が v2.9 未満で更新する場合は --update を付ける:
#   unzip -o awscliv2.zip && sudo ./aws/install --update

# kubectl（公式）
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# gh（Debian/Ubuntu = WSL2 既定。他ディストリは https://github.com/cli/cli/blob/trunk/docs/install_linux.md）
sudo apt update && sudo apt install -y gh   # 古い/未提供なら上記公式リポを参照

# kubectx / kubens（Debian/Ubuntu。無ければ https://github.com/ahmetb/kubectx#installation 参照）
sudo apt install -y kubectx
```

## 環境の定数

| 項目 | 値 |
|---|---|
| SSO start URL | `https://d-95675816a0.awsapps.com/start` |
| SSO / デフォルトリージョン | `ap-northeast-1` |
| AWS アカウント ID | `396510133136` |
| EKS クラスタ名 | `platform-city` |
| ベース SSO プロファイル | `pek2026`（AdministratorAccess） |

### EKS 用 IAM ロール

| ロール | 用途 |
|---|---|
| `platform-city-eks-cluster-admin` | クラスター管理者（全リソース操作可）。**namespace の作成/削除など クラスタスコープ操作はこれが必要** |
| `platform-city-eks-editor` | 編集者（namespace 内の Deployment/Service 等の作成・削除可）。**日常作業はこれ** |
| `platform-city-eks-viewer` | 閲覧者（参照のみ） |

ARN は `arn:aws:iam::396510133136:role/<ロール名>`。

> ⚠️ **editor ロールは namespace 内のリソース操作専用。** `namespace` や `nodes` などの**クラスタスコープのリソースは editor では作成・一覧できず `Forbidden` になる**（RBAC 制限であり接続エラーではない）。namespace の作成・削除は **cluster-admin** コンテキストで行い、それ以外の日常作業は editor に戻す、という使い分けをする（手順 2 で両方のコンテキストを用意する）。

## 1. AWS SSO プロファイルの設定

### 推奨（エージェント実行向け）: 設定を直接書き込む

`~/.aws/config` に以下を追記する。対話プロンプトを回避できる。

```ini
[sso-session pek2026]
sso_start_url = https://d-95675816a0.awsapps.com/start
sso_region = ap-northeast-1
sso_registration_scopes = sso:account:access

[profile pek2026]
sso_session = pek2026
sso_account_id = 396510133136
sso_role_name = AdministratorAccess
region = ap-northeast-1
output = json

[profile platform-city-eks-editor]
role_arn = arn:aws:iam::396510133136:role/platform-city-eks-editor
source_profile = pek2026
region = ap-northeast-1

[profile platform-city-eks-cluster-admin]
role_arn = arn:aws:iam::396510133136:role/platform-city-eks-cluster-admin
source_profile = pek2026
region = ap-northeast-1
```

> editor だけでなく **cluster-admin も用意しておく**（namespace の作成/削除に必要）。viewer を使う場合は `role_arn` のロール名を差し替えた profile ブロックを同様に追加する。

書き込んだらログインする（ブラウザが開くので **ユーザーがログインを完了**）。

```bash
aws sso login --profile pek2026
```

### 代替（対話）: `aws configure sso`

```bash
aws configure sso --profile pek2026
```

プロンプトには上表の値（SSO start URL / リージョン / 出力形式 json）を入力する。ブラウザで Auth0 のログイン後、pek2026 アカウントの AdministratorAccess を選択する。その後、AssumeRole 用の `[profile platform-city-eks-editor]` ブロックを上記の通り `~/.aws/config` に追記する。

## 2. kubeconfig の更新

editor（日常用）と cluster-admin（namespace 作成/削除用）の **両方の kubectl コンテキストを追加**する。

```bash
# editor（日常作業用）
aws eks update-kubeconfig \
  --name platform-city \
  --region ap-northeast-1 \
  --profile platform-city-eks-editor \
  --alias platform-city-editor

# cluster-admin（namespace の作成/削除など クラスタスコープ操作用）
aws eks update-kubeconfig \
  --name platform-city \
  --region ap-northeast-1 \
  --profile platform-city-eks-cluster-admin \
  --alias platform-city-admin
```

`~/.kube/config` に `platform-city-editor` と `platform-city-admin` のコンテキストが追加される。

## 3. 接続確認

クラスタへ到達できているかを確認する。`kubectl get nodes` は**クラスタスコープなので admin コンテキストで**実行する（editor では `Forbidden` になる — これは認証成功の証でもある）。

```bash
kubectl --context platform-city-admin get nodes   # → ノード一覧が表示される
```

> 補足: editor コンテキストで `Forbidden` が返るのは「API サーバーに認証が通ったうえで RBAC により拒否された」状態であり、**接続自体は成功している**。逆に SSO セッション切れなどでは `Unauthorized` や接続エラーになる。

## 4. 専用 namespace の作成

GitHub ユーザー名を小文字にしたものを namespace 名に使う（分かりやすさのため）。

namespace の作成は**クラスタスコープなので admin コンテキストで行う**（editor では `Forbidden`）。作成後は editor コンテキストに戻し、その namespace を既定にする。

```bash
export NAMESPACE=$(gh api user --jq .login | tr '[:upper:]' '[:lower:]')

# 作成は admin コンテキストで
kubectl --context platform-city-admin create namespace "$NAMESPACE"
kubectl --context platform-city-admin get namespace "$NAMESPACE"

# 日常作業は editor に戻す
kubectl config use-context platform-city-editor
kubens "$NAMESPACE"   # デフォルト namespace を切り替え
```

## 5. 動作確認用リソースの作成

nginx の Deployment と Service を作成する。

```bash
cat <<EOF | kubectl apply -n "$NAMESPACE" -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: public.ecr.aws/docker/library/nginx:latest
          ports:
            - containerPort: 80
EOF

cat <<EOF | kubectl apply -n "$NAMESPACE" -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
EOF
```

確認:

```bash
kubectl get deployment -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
kubectl get service -n "$NAMESPACE"
```

ローカル動作確認（port-forward）:

```bash
kubectl port-forward -n "$NAMESPACE" service/nginx 8080:80
# 別ターミナルで
curl http://localhost:8080
```

## 6. 後片付け（必ず実施）

> ⚠️ **作業が終わったら作成したリソースを必ず削除する。** コストとセキュリティの観点から、不要なリソースを残さない。削除を忘れると Pod や Service が動作し続けコストが発生する。

namespace ごと削除すれば配下のリソースもまとめて消える。namespace の削除も**クラスタスコープなので admin コンテキストで**行う（editor では `Forbidden`）。**破壊的コマンドなので実行前にユーザーへ確認**する。

```bash
kubectl --context platform-city-admin delete namespace "$NAMESPACE"

kubectl --context platform-city-admin get namespace "$NAMESPACE"
# → Error from server (NotFound): namespaces "xxx" not found ならOK
```

## トラブルシューティング

- **`aws sso login` が `Missing the following required SSO configuration values: sso_start_url, sso_region` で失敗** → aws CLI が古く（v2.9 未満）`[sso-session]` 形式を解釈できていない。aws CLI を更新する（「インストール」の `--update` 手順）。更新後は同じ config のまま再ログインできる。
- **`kubectl get nodes` / `create namespace` で `Forbidden`** → editor コンテキストで実行している。クラスタスコープ操作なので admin コンテキストで実行する:
  ```bash
  kubectl --context platform-city-admin get nodes
  ```
- **`Unauthorized` や接続エラー（認証情報が読めない）** → AWS SSO セッション切れの可能性。再ログイン:
  ```bash
  aws sso login --profile pek2026
  ```
- それでも解決しない場合は **jacopen** に連絡する。
