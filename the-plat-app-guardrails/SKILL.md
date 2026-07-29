---
name: the-plat-app-guardrails
description: >-
  Platform City（The Plat）の EKS クラスタ上でアプリを作る・デプロイする・外部公開するときに
  守るべき制約（ガードレール）。共有 ALB への HTTPRoute 追加、TargetGroupConfiguration、
  ドメイン（core.paas.jp / city.paas.jp）、namespace とテナント権限、PVC の StorageClass、
  Secret の扱いを扱う。アプリのマニフェストを書く / Service を公開する / 「HTTPRoute を作ったのに
  502 になる」「PVC が Pending のまま」「namespace が作れない」といったトラブル時に使う。
  単一 namespace スコープのアプリ作者向け（基盤 manifests の運用は argocd-gitops-ops）。
  The Plat 上のアプリの K8s マニフェストに触れる作業では、明示的に指示されなくても本スキルを参照すること。
---

# The Plat アプリ・ガードレール

Platform City のクラスタ上で**アプリを動かす側**が守る制約。**単一 namespace スコープ**の作業向け
（ArgoCD / 共有 Gateway 本体など基盤の運用は [`argocd-gitops-ops`](../argocd-gitops-ops/SKILL.md)）。

## 真実の源（必ず参照）

本スキルはチェックリストと定型 YAML の早見表。**規約の一次情報は本体リポジトリ**にあり、ここでは重複させない:

- `platform-city/docs/conventions.md` — **お作法の総まとめ（まずこれ）**
- `platform-city/manifests/README.md` — 共有 Gateway・基盤コンポーネントの詳細
- `platform-city/docs/adr/` — なぜその決定になったか

ローカル参照先（環境による）: `/Users/kkusama/workspace/cnia/platform-city/`

## エージェント実行時の注意

- 作業前に**どの namespace で作業しているか**を確定させる（`kubens` / `kubectl config current-context`）。
- **他人の namespace や基盤 namespace（`argocd` / `gateway` / `kube-system` / `monitoring` / `crossplane-system`）のリソースを変更しない。** 読むのは可。
- `kubectl delete` などの破壊的コマンドは**実行前にユーザーへ確認**する。
- 「Forbidden が出た」ときは**まず権限設計どおりの挙動でないかを疑う**（下記トラブルシュート）。闇雲に権限を足さない。

---

## 1. 外部公開 — 共有 ALB に相乗りする

### 最初に頭に入れること

**ALB は Gateway 1 つにつき 1 台払い出される = そのまま課金される。**
だからクラスタ全体で Gateway は `gateway/shared` の **1 つだけ**。アプリはここへ HTTPRoute で attach する。

❌ **Gateway を新設しない。GatewayClass も増やさない。**
❌ 証明書を自分で用意しない（TLS は ALB で終端済み）。
❌ Route53 を直接触らない（ExternalDNS が HTTPRoute から自動作成する）。

### 書くものは 2 つだけ

**(1) TargetGroupConfiguration — 忘れると 502 になる**

ALB の既定は instance ターゲットで、これは NodePort / LoadBalancer Service 前提。**ClusterIP には繋がらない。**

```yaml
apiVersion: gateway.k8s.aws/v1beta1
kind: TargetGroupConfiguration
metadata:
  name: <service-name>
  namespace: <your-namespace>
spec:
  targetReference:
    kind: Service
    name: <service-name>
  defaultConfiguration:
    targetType: ip
```

**(2) HTTPRoute を 2 本（HTTPS 本体 + HTTP→HTTPS リダイレクト）**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>-redirect
  namespace: <your-namespace>
spec:
  parentRefs:
    - name: shared
      namespace: gateway
      sectionName: http          # リダイレクト用
  hostnames:
    - <xtenant>.city.paas.jp
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>
  namespace: <your-namespace>
spec:
  parentRefs:
    - name: shared
      namespace: gateway
      sectionName: https         # 本体
  hostnames:
    - <xtenant>.city.paas.jp
  rules:
    - backendRefs:
        - name: <service-name>
          port: 80
```

- `parentRefs` はクロス namespace だが、Gateway 側が `allowedRoutes.namespaces.from: All` なので **attach に ReferenceGrant は不要**。
  `backendRefs` を別 namespace に向ける場合のみ ReferenceGrant が要る（通常は同 namespace に置く）。
- **バックエンドは平文で受ける**（TLS は ALB で終端済み）。アプリ側で TLS を張らない。

### ドメイン — 参加者は `<xtenant>.city.paas.jp` 固定

| ゾーン | ホスト名の形 | 用途 |
|---|---|---|
| `city.paas.jp` | **`<xtenant>.city.paas.jp`** | 参加者アプリ。XTenant 名 = namespace 名 = サブドメイン |
| `core.paas.jp` | `<component>.core.paas.jp` | プラットフォーム基盤・運営系（`argocd.` / `grafana.`） |

- **1 テナント 1 ホスト名。** アプリを複数動かすときは**パスで分ける**（同じホスト名に HTTPRoute を複数 attach してよい）。
- ❌ **サブドメインを 2 段にしない**（`api.<xtenant>.city.paas.jp` は証明書 `*.city.paas.jp` の対象外で TLS エラーになる）。
- ❌ 参加者アプリに `core.paas.jp` を使わない。

### 確認（read-only）

```bash
kubectl -n <ns> get httproute                      # Accepted / attach 状況
kubectl -n <ns> describe httproute <name>          # parentRef の条件を見る
kubectl -n gateway get gateway shared              # Programmed=True / ALB DNS 名
kubectl -n <ns> get targetgroupconfiguration
curl -I https://<app>.core.paas.jp
```

---

## 2. namespace と権限

- **namespace 名 = XTenant 名。** 同じ名前が公開ホスト名（`<xtenant>.city.paas.jp`）と GitLab のサブグループ／プロジェクト path にもなる。
- **自分の namespace 内だけで作業する。** テナントには組み込み ClusterRole `edit` が RoleBinding で付いており、
  ClusterRoleBinding は無い。クラスタスコープのリソース（namespace / node / CRD）は参照すらできない。
- `admin` ではなく `edit` なので、**namespace 内で RBAC（Role / RoleBinding）を作ることもできない**。
- **HTTPRoute と TargetGroupConfiguration は作れる**（集約 ClusterRole で `edit` に追加済み）。
  一方 **Gateway / GatewayClass / LoadBalancerConfiguration は権限が無い** — 共有 ALB への集約を RBAC で強制しているため。
  「Gateway を作る」提案をしない。
- **ResourceQuota がある**（既定 `requests.cpu: 4` / `requests.memory: 8Gi` / `pods: 20`）。
  Pod に `resources.requests` を書かないと quota 違反で作成が弾かれることがある。**requests は必ず書く。**
- 委員会メンバーが検証用 namespace を作るときは cluster-admin コンテキストが必要
  （→ [`the-plat-onboarding`](../the-plat-onboarding/SKILL.md)）。

---

## 3. ストレージ

**`storageClassName` は省略する。** 既定の `gp3`（`ebs.csi.eks.amazonaws.com`）が使われる。

❌ **`gp2` を書かない。** in-tree provisioner は EKS Auto Mode では実際にプロビジョニングできず、PVC が Pending で止まる。

`volumeBindingMode: WaitForFirstConsumer` なので、**PVC は Pod がスケジュールされるまで Pending のまま**。これは正常。

---

## 4. シークレット

- **git にコミットしない。** `secret.example.yaml` のような雛形を置き、実体はクラスタへ直接投入する。
- 投入は**アプリを同期/デプロイする前**に行う。無いと Pod が `CreateContainerConfigError` で起動しない。
- ローテーション時は Secret 更新後に **Pod を削除して再作成**する（`envFrom` は起動時にしか読まれない）。

---

## 5. イメージとバージョン

- **`latest` や floating tag に依存しない。** バージョンは固定する。
- 運営アプリのイメージ更新は Argo CD Image Updater が git write-back する経路がある（基盤側の設定）。
  アプリ側でタグを手書きして直接 `kubectl set image` するような運用はしない。

---

## トラブルシュート

| 症状 | 原因 | 対処 |
|---|---|---|
| HTTPRoute は Accepted なのに **502 / 504** | `TargetGroupConfiguration` が無い（instance ターゲットで ClusterIP に繋がらない） | `targetType: ip` の TGC を追加 |
| ホスト名が **名前解決できない** | ExternalDNS がまだ反映していない / `hostnames` が domainFilter 外 | 数分待つ。`core.paas.jp` / `city.paas.jp` 以外は自動作成されない |
| **PVC が Pending** | `WaitForFirstConsumer` で Pod 待ち（正常）、または `gp2` を指定している | Pod を作る / `storageClassName` を消す |
| **Pod が起動しない**（`CreateContainerConfigError`） | 参照している Secret / ConfigMap が無い | 先にクラスタへ投入する |
| `create namespace` が **Forbidden** | 権限設計どおり（editor / テナントはクラスタスコープ不可） | cluster-admin コンテキストで実行 |
| `kubectl get nodes` が **Forbidden** | 同上。**認証は成功している** | 気にしない。`Unauthorized` なら SSO セッション切れ |
| Pod 作成が **quota で弾かれる** | ResourceQuota に対して `requests` 未記載 or 超過 | `resources.requests` を書く / `kubectl -n <ns> describe quota` で確認 |
| **Gateway** を作れない（Forbidden） | 権限設計どおり。共有 ALB へ集約するため意図的に与えていない | 共有 Gateway に HTTPRoute で attach する |
| ホスト名が他テナントに取られている | Gateway API は同一ホスト名+パスで**先に作られた HTTPRoute が勝つ** | `<xtenant>.city.paas.jp` は自分専用。他人のホスト名を書かない |

解決しない場合は **jacopen** に確認する。
