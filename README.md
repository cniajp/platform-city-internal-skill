# platform-city-internal-skill

Platform Engineering Kaigi 2026（PEK2026）の目玉企画 **Platform City** の開発・運用にあたり、**PEK2026 実行委員会が自分たちの環境・開発・運用に使う Claude Code スキル**を格納するリポジトリ。

参加者へ配布するものではなく、**委員会のドッグフーディング／開発／運用**が主目的（必要に応じて参加者向けの土台にも転用する）。

## Platform City / The Plat とは

- **The Plat** = AI エージェント向けに最適化された Internal Developer Platform。EKS Auto Mode + ArgoCD（GitOps）+ Gateway API（共有 ALB）+ GitLab CI/CD で構成。
- 参加者は Claude Code / Cursor から MCP ツール経由でアプリを開発・デプロイし、仮想都市を構築する。
- 詳細・設計判断は本体リポジトリを参照（このリポジトリでは重複させない）:
  - `cniajp/platform-city` — インフラ（Terraform）/ GitOps（manifests）/ ADR
  - `cniajp/pek2026-committee` — city-hall ほか運営システム

### 2 つのデプロイ面（スキルを分ける根拠）

| | 参加者アプリ | プラットフォーム基盤 |
|---|---|---|
| 経路 | GitLab CI/CD → `kubectl apply`（自分の namespace） | ArgoCD App-of-Apps（git=source of truth） |
| リポジトリ | アプリごとのリポジトリ | `platform-city/manifests/` |
| 権限スコープ | 単一 namespace（`user-NNN`） | クラスタ全体 |

この境界に沿ってスキルを分割している。

## スキル一覧と導線

| スキル | 状態 | いつ使うか | スコープ |
|---|---|---|---|
| [`the-plat-onboarding`](./the-plat-onboarding/) | ✅ 作成済み | EKS クラスタに初めてアクセスする／kubectl が認証エラーのとき | 自分の開発環境 |
| `the-plat-app-guardrails` | 🚧 予定 | The Plat 上で**アプリを作る**とき（守るべき制約） | 単一 namespace |
| [`argocd-gitops-ops`](./argocd-gitops-ops/) | ✅ 作成済み | The Plat の**基盤 manifests を運用**するとき | クラスタ全体 |

**迷ったときの導線**
- まず環境を作りたい → `the-plat-onboarding`
- アプリを書いてデプロイする → `the-plat-app-guardrails`
- ArgoCD / 共有 Gateway / 監視など基盤をいじる → `argocd-gitops-ops`

### 今後の追加候補（優先度順）

- 中: 負荷テスト（100人同時想定）/ テンプレート整備 / 観測性（Grafana）/ インシデント対応 Runbook
- 低〜随時: MCP サーバー開発 / ADR 作成 / コスト監視 / イベント当日運営

## スキルの形式

各スキルは 1 ディレクトリ。`SKILL.md` に YAML frontmatter（`name` / `description`）を持つ。

```
<skill-name>/
└── SKILL.md
```

- `name` … ディレクトリ名と一致させる（kebab-case）。
- `description` … **「いつ発火すべきか」が伝わる説明**にする（自動選択の判断材料になるため）。
- 詳細な手順や補助ファイルが大きくなる場合は同ディレクトリに分割し、`SKILL.md` から参照する。

詳しい執筆方針は [`CLAUDE.md`](./CLAUDE.md) を参照。
