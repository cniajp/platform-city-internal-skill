---
name: argocd-gitops-ops
description: >-
  Platform City（The Plat）の基盤 manifests を ArgoCD / GitOps で運用するためのスキル。
  `platform-city/manifests/` のコンポーネント（共有 Gateway・監視スタック・LBC・GatewayClass 等）
  を追加・更新・トラブルシュートするとき、新しいアプリを共有 ALB で公開するとき、ArgoCD の
  同期状態を確認・修復するとき、クラスタを初回ブートストラップするときに使う。クラスタ全体スコープの
  作業向け（アプリ作者の単一 namespace 作業ではない）。
---

# ArgoCD / GitOps 運用（基盤 manifests）

Platform City の**プラットフォーム基盤**を ArgoCD で宣言的に管理するためのスキル。対象は `platform-city/manifests/` 配下。**クラスタ全体スコープ**の運用作業向け（参加者アプリの単一 namespace 作業は対象外）。

## 真実の源（必ず参照）

このスキルは要点と手順の早見表。詳細・背景・最新値は本体リポジトリの以下を**一次情報として参照**する（ここでは重複させない）:

- `platform-city/manifests/README.md` — 構成、共有 Gateway、監視、ブートストラップ、検証
- `platform-city/CLAUDE.md` の「manifests の運用方針（GitOps / ArgoCD）」節

ローカル参照先（環境による）: `/Users/kkusama/workspace/cnia/platform-city/`

## 黄金律（破ってはいけない）

1. **すべて宣言的に。git が単一の真実。** クラスタ変更は git 経由（push → ArgoCD 自動同期）。`kubectl apply` の直叩き・`kubectl edit` での手当ては**しない**（selfHeal で巻き戻り、ドリフトの元）。
2. **バージョンは必ず固定**（chart / CRD / コントローラ）。`latest` や floating tag は使わない。更新も git の変更（PR）として扱う。
3. **main へ直 push しない。** 変更は PR 経由。マージ後に ArgoCD が同期する。
4. **シークレットは git に置かない。** ArgoCD のリポ認証や Grafana admin などはクラスタへ直接投入する。
5. **新しいアプリのために Gateway を新設しない。** 共有 Gateway（`gateway/shared` = 単一 ALB）を `parentRef` で指す（ALB は Gateway 1 つにつき 1 台＝コスト増のため）。

## エージェント実行時の注意

- 変更はファイル編集 → PR が基本。**`kubectl apply`/`edit` を実行しない**（黄金律1）。状態確認の read-only な `kubectl get` はよい。
- ブートストラップや手動同期など破壊的・特権操作は**実行前にユーザーへ確認**し、`platform-city-eks-cluster-admin` ロールが必要な点を伝える（→ [`the-plat-onboarding`](../the-plat-onboarding/SKILL.md)）。
- バージョンを上げる提案をするときは固定値で書く。`latest` を書かない。

## 構成（App-of-Apps）

```
manifests/
├── argocd/
│   ├── project.yaml   # AppProject "platform"（許可リポ/デプロイ先を制限）
│   ├── root.yaml      # App-of-Apps ルート（apps/ を再帰監視）
│   ├── values/        # Helm values（Application から $values で参照）
│   └── apps/          # 子 Application（1 コンポーネント = 1 ファイル）
└── platform/          # 素の K8s リソース（GatewayClass / StorageClass / shared-gateway 等）
```

- 子 Application は `syncPolicy.automated`（prune + selfHeal）が基本。
- 導入順序は `argocd.argoproj.io/sync-wave` で制御:
  `storageclass`(0) / `gateway-api-crds`(0) / `argocd`(0) → `aws-load-balancer-controller`(1) → `gatewayclass`(2) / `kube-prometheus-stack`(2) → `grafana-gateway`(3)。
- 大きい CRD は `syncOptions: ServerSideApply=true`（last-applied-configuration の annotation 上限回避）。

## ワークフロー

### A. Helm アプリを追加する

1. `manifests/argocd/values/<app>.yaml` に values を作成（インライン散在させない）。
2. `manifests/argocd/apps/<app>.yaml` に Application を追加。**multi-source** で chart 本体 + `$values` を参照、`targetRevision` で**バージョン固定**、sync-wave を設定。

   ```yaml
   spec:
     sources:
       - repoURL: https://example.github.io/charts   # chart 本体
         chart: <chart-name>
         targetRevision: <pinned-version>
         helm:
           valueFiles:
             - $values/manifests/argocd/values/<app>.yaml
       - repoURL: https://github.com/cniajp/platform-city  # values 参照用
         targetRevision: main
         ref: values
   ```
3. commit / push（PR）→ root が検知し ArgoCD が自動同期。

### B. 素の manifest（非 Helm）を追加する

1. `manifests/platform/<name>/` に YAML（必要なら kustomization）を置く。
2. `manifests/argocd/apps/<name>.yaml` に Application を追加（`source.path` で上記ディレクトリを指す）。
3. commit / push（PR）。

### C. 新しいアプリを共有 ALB で公開する

**Gateway は新設しない。** 共有 Gateway に attach する。

1. ClusterIP Service 用に `TargetGroupConfiguration(targetType: ip)` を置く（instance ターゲットは NodePort/LB 前提のため不可）。
2. `HTTPRoute` を作成: `parentRefs: [{name: shared, namespace: gateway, sectionName: https}]`、`hostnames: [<app>.core.paas.jp]`（HTTP→HTTPS リダイレクト用に http listener 向けも）。
3. Application を追加（A/B の手順）。

- TLS は共有 LoadBalancerConfiguration の HTTPS:443 で ACM 証明書を SNI 束ねして終端。
- ホスト名 → ALB の Route53 レコードは ExternalDNS が HTTPRoute から自動作成。

## 検証（read-only）

```bash
kubectl -n argocd get applications                 # 各 App の Synced/Healthy
kubectl get gatewayclass aws-alb                   # Accepted=True
kubectl -n gateway get gateway shared              # Programmed=True / ALB DNS 名（1 台）
kubectl -n argocd get httproute                    # 共有 Gateway に attach
kubectl -n monitoring get pods                      # prometheus/alertmanager/grafana/exporters
kubectl get storageclass gp3                        # default であること
curl -I https://argocd.core.paas.jp                # 200（Grafana も同様）
```

## ブートストラップ / 修復（特権操作・要確認）

- **初回ブートストラップ手順**は `manifests/README.md` の「ブートストラップ手順」を参照（`platform-city-eks-cluster-admin` ロールで kubeconfig 取得済みが前提）。Grafana admin Secret は ArgoCD 同期前に作成しておく（無いと Grafana Pod が起動しない）。
- ドリフトや同期失敗時は、まず `kubectl -n argocd get applications` と各 App の `status` を確認し、**git 側を直して push**で直すのが基本。手動同期が必要な場合のみユーザー確認のうえ ArgoCD UI / `argocd app sync` を使う。

## 環境固有の前提

- クラスタは **EKS Auto Mode**。内蔵ロードバランシングは Ingress/Service(LB) のみで Gateway API 非対応のため、標準 AWS Load Balancer Controller を別途導入（Auto Mode=`eks.amazonaws.com/alb` / 本コントローラ Gateway=`gateway.k8s.aws/alb` で共存）。
- Auto Mode ノードは IMDS 制限のため、LBC values に `vpcId` を明示する（自動検出は失敗）。
- LBC の IAM は **EKS Pod Identity**（`terraform/lbc.tf`）。manifests 適用前に `terraform apply` 完了が前提（apply はマージ後に CI が実行）。
