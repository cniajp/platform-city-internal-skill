# CLAUDE.md

このリポジトリで作業する Claude Code 向けのガイド。

## このリポジトリの目的

PEK2026 実行委員会が、目玉企画 **Platform City（プラットフォーム = The Plat）** の開発・運用に使う **内部スキル**を格納する。主な対象読者は**委員会メンバー（ドッグフーディング優先）**であり、参加者配布物ではない。

スキルの一覧と導線は [`README.md`](./README.md) を参照。

## The Plat の前提知識（要点のみ）

- AI エージェント向けに最適化された IDP。**EKS Auto Mode + ArgoCD + Gateway API（共有 ALB）+ GitLab CI/CD**。
- **2 つのデプロイ面**がある（スキルはこの境界で分ける）:
  - 参加者アプリ: GitLab CI/CD → `kubectl apply`、単一 namespace スコープ。
  - プラットフォーム基盤: ArgoCD App-of-Apps、クラスタ全体スコープ。
- **設計の真実は本体リポジトリにある**。ここでは重複させず参照する:
  - `cniajp/platform-city`（`README.md` / `AGENTS.md` / `CLAUDE.md` / `docs/adr/` / `terraform/` / `manifests/`）
  - `cniajp/pek2026-committee`
- ローカルでの参照先（環境による）: `/Users/kkusama/workspace/cnia/platform-city/`

## スキル執筆の方針

1. **1 スキル 1 ディレクトリ**。`<skill-name>/SKILL.md`。`name` はディレクトリ名と一致（kebab-case）。
2. **`description` は発火条件**。「何をする／いつ使う／トラブル時にも使う」を具体的に書く。曖昧な説明は自動選択を狂わせる。
3. **スコープを混ぜない**。アプリ作者向け（単一 namespace）と基盤運用向け（クラスタ全体）は別スキルに保つ。共通哲学（宣言的・バージョン固定・`latest` 禁止）は README で繋ぐか相互参照する。
4. **エージェント実行を前提に書く**。Claude Code が読んで手を動かせるよう:
   - ブラウザ認証や対話プロンプトを伴うコマンド（`aws sso login`, `aws configure sso` 等）は**ユーザーに `!` 付きで実行してもらう**前提で書く。可能なら設定ファイルを直接書く非対話の手順を「推奨」として併記する。
   - 破壊的コマンド（`kubectl delete` 等）は**実行前にユーザー確認**する旨を明記する。
   - 各ステップに確認コマンドを添える。
5. **本体ドキュメントを重複させない**。手順や定数は書いてよいが、設計判断（なぜそうしたか）は ADR を参照する。

## 作業上の原則（platform-city の方針を継承）

- **コスト意識**: 無駄な AI 呼び出しを避ける。ファイル作成・リネーム・移動など機械的作業はスクリプトで済ませる。レビュー依頼は本当に必要なときだけ。
- **セキュリティ意識**: 認証情報をハードコードしない。内部限定の定数（アカウント ID、SSO URL 等）はこの private リポジトリ内に限り記載可。
- **宣言的・固定**: バージョンは固定し `latest` を使わない（基盤運用スキルで特に徹底）。

## よく使う定数

EKS クラスタへのアクセス情報は [`the-plat-onboarding/SKILL.md`](./the-plat-onboarding/SKILL.md) に集約している。重複記載せず、そちらを参照・更新する。

## 困ったとき

不明点・環境の最新値は **jacopen** に確認する。
