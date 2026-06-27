---
name: the-plat-onboarding
description: >-
  Platform City（The Plat）の EKS クラスタへ初めてアクセスするためのオンボーディング手順。AWS
  SSO プロファイル作成、EKS 用 IAM ロールの AssumeRole、kubeconfig 設定、専用
  namespace の作成、動作確認用リソースのデプロイ、後片付けまでを案内する。PEK2026
  実行委員会メンバーが自分の開発環境をセットアップするとき、または「EKS につながらない /
  kubectl が認証エラー」のトラブル時に使う。
---

# The Plat オンボーディング

Platform City の EKS クラスタにアクセスし、Kubernetes リソースを作成・削除できるようになるまでの手順。**PEK2026 実行委員会メンバーが自分の開発環境をセットアップする**ためのもの（参加者向けの GitLab/MCP フローとは別）。

## このスキルの使い方（エージェント向け）

- ブラウザ認証を伴うコマンド（`aws sso login` など）は**ユーザー自身に実行してもらう**。プロンプトで `! aws sso login --profile pek2026` のように `!` 付きで打つよう案内するか、ユーザーがブラウザでログインを完了するのを待つ。
- `kubectl delete namespace` などの破壊的コマンドは**実行前に必ずユーザーに確認**する。
- 各ステップ後に確認コマンドの出力を見て、次に進めるか判断する。

## 前提条件

- AWS Identity Center（AWS access portal）から **pek2026 アカウントの AdministratorAccess** が選択できること。
- 以下の CLI がインストール済みであること: `aws`（v2）, `kubectl`, `gh`, `kubectx`/`kubens`。

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
| `platform-city-eks-cluster-admin` | クラスター管理者（全リソース操作可）。**必要な場合のみ** |
| `platform-city-eks-editor` | 編集者（Deployment/Service 等の作成・削除可）。**通常はこれ** |
| `platform-city-eks-viewer` | 閲覧者（参照のみ） |

ARN は `arn:aws:iam::396510133136:role/<ロール名>`。

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
```

> cluster-admin / viewer を使う場合は、`role_arn` のロール名を差し替えた profile ブロックを同様に追加する。

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

editor ロールで kubectl コンテキストを追加する。

```bash
aws eks update-kubeconfig \
  --name platform-city \
  --region ap-northeast-1 \
  --profile platform-city-eks-editor \
  --alias platform-city-editor
```

`~/.kube/config` に `platform-city-editor` コンテキストが追加される。

## 3. 接続確認

```bash
kubectl config current-context   # → platform-city-editor
kubectl get nodes                # → ノード一覧が表示される
```

## 4. 専用 namespace の作成

GitHub ユーザー名を小文字にしたものを namespace 名に使う（分かりやすさのため）。

```bash
export NAMESPACE=$(gh api user --jq .login | tr '[:upper:]' '[:lower:]')

kubectl create namespace "$NAMESPACE"
kubectl get namespace "$NAMESPACE"

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

namespace ごと削除すれば配下のリソースもまとめて消える（**破壊的コマンドなので実行前にユーザーへ確認**）。

```bash
kubectl delete namespace "$NAMESPACE"

kubectl get namespace "$NAMESPACE"
# → Error from server (NotFound): namespaces "xxx" not found ならOK
```

## トラブルシューティング

- **`kubectl get nodes` で認証エラー** → AWS SSO セッション切れの可能性。再ログイン:
  ```bash
  aws sso login --profile pek2026
  ```
- それでも解決しない場合は **jacopen** に連絡する。
