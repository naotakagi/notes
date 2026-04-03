# eigoサプライチェーンリリース

# Handoff Summary — Eigo 全体リリース計画

## Context
サプライチェーン攻撃対策とバグ修正のリリース計画。フェーズ1・2が完了し、次は PR 作成とフェーズ3以降のリリース。

## Task Overview
**Main Task**: Eigo の未マージ PR リリース + サプライチェーン対応  
**Current Status**: 🟡 In Progress  
**詳細ファイル**: `tasks/eigo/in-progress/release-plan-supply-chain.md`

---

## ✅ 完了済み

### フェーズ1: 緊急バグ修正（マージ済み）
- #666 roomページ i18n翻訳キーエラー（Sentry 10K件・160ユーザー）
- #677 CheckSessionStatusJob レースコンディション
- #675 Twilio REST NoMethodError

### フェーズ2: サプライチェーン対応（コミット済み・**PR未作成**）
ブランチ: `security/supply-chain-hardening`  
worktree: `/Users/nao/ghq/github.com/prospaceinc/eigo/.worktrees/security/supply-chain-hardening`

- GitHub Actions SHA pinning（pinact で全ワークフロー）commit: `2bdc25c4`
- Dependency Review Action 追加（severity high でブロック）commit: `2bdc25c4`
- yarn → pnpm v10 移行（ゴーストパッケージ clsx/react-icons/@types/relay-runtime 明示化済み）commit: `2897a9e6`

---

## Next Steps（優先順）

### 今すぐ
1. **フェーズ2 PR 作成** — `security/supply-chain-hardening` ブランチで `/git-new-pull-request`

### その後（順番に）
2. **フェーズ3: #674** Node.js 20 → 22 LTS（pnpm移行が終わった今マージ可）
3. **フェーズ3: #667** GoodJob/pg更新・Ridgepole安全運用
4. **フェーズ4（順次）**:
   - #673 Agora Sentry過大検知抑制
   - #662 GraphQL API認可ロジック改善（staging-deploy ラベルあり・要稼働確認）
   - #669 Chrome翻訳DOM破壊防止
   - #670 Learner遅刻キャンセルページフッター重複
   - #668 CancelItemリファクタ
   - #676 未使用コンポーネント削除
   - #661 SessionNote Yjs binary encoding問題

### 要別途判断
- #635, #623 — 古い PR（2025年内）。コンフリクト確認してからリベース or クローズ
- #578 — Backlog PROSPACE_DEV-60（1ヶ月有効チケット）との関連確認要

### クローズ推奨
- #516 / #514 — 2024年6月作成（約2年前）

---

## Key Decisions
| 決定 | 理由 |
|---|---|
| SHA pinning + Dep Review + pnpm移行 を1PRにまとめる | 「サプライチェーン対応」として意味が揃う |
| pnpm移行をフェーズ3（Node.js アップグレード）の前に実施 | ゴーストパッケージの原因切り分けを容易にする |
| min-release-age を pnpm-workspace.yaml に設定 | 悪意あるパッケージは公開後1週間以内に検知される傾向 |

