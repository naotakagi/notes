# Eigo session-check-in bot / Ando さんオンボーディングメモ

#eigo #prospace #slackbot #infra #onboarding

Date: 2026-05-17

## 背景

Izumi Ando さんから、session-check-in bot について共有と相談があった。相談内容は、リポジトリや Admin 権限、必要に応じた GCP アクセス、本番コードの保護、そして「チケット作成 -> 開発 -> ステージング確認 -> デプロイ -> 報告」という開発フローの説明について。

bot のリポジトリは `prospaceinc/session-check-in-bot` に移管済み。移管通知そのものには追加対応は不要。

## 現在の bot の形

Repository: https://github.com/prospaceinc/session-check-in-bot

現状は Slack Bolt を使った小さな Node.js の Slack Events bot。

確認したファイル:

- `index.js`: bot 本体のロジック
- `package.json` / `package-lock.json`: Node 依存関係と scripts
- `dp_map.csv`: Discussion Partner 名から Slack user ID への対応表
- `slack-prospaceinc-members.csv`: Slack メンバーのエクスポート/参照データ
- `README.md`: ローカルセットアップと Slack App 設定
- `DEPLOYMENT.md`: Railway デプロイ手順
- `TEST_PLAN.md`: 手動テスト計画

実行環境/仕様:

- Node.js
- `@slack/bolt`
- `npm start` -> `node index.js` で起動
- Slack endpoint は Bolt デフォルトの `/slack/events`
- DB なし
- 永続 state なし
- 設定は環境変数 + `dp_map.csv`
- 現在のドキュメントは Railway デプロイ前提

必要な環境変数:

- `SLACK_BOT_TOKEN`
- `SLACK_SIGNING_SECRET`
- `MY_USER_ID`
- `STAFF_USER_ID_1`
- `STAFF_USER_ID_2`
- `CHANNEL_ID`
- `PORT`

ドキュメント上必要とされている Slack scopes:

- `channels:history`
- `channels:read`
- `chat:write`
- `im:write`
- `mpim:write`
- `users:read`

## 現在の処理フロー

1. Eigo app が no-show 通知を Slack の `#need_operation` に投稿する。
2. Slack Events API が `message.channels` event を bot に送る。
3. bot が channel が `CHANNEL_ID` と一致するか確認する。
4. bot が message text に必要な日本語マーカーが含まれるか確認する。
5. bot が `受講生` の次の行から learner 名を parse する。
6. bot が `ディスカッションパートナー` の次の行から DP 名を parse する。
7. bot が DP 名を `dp_map.csv` で引く。
8. 見つかった場合、`conversations.open` で DP + `MY_USER_ID` + `STAFF_USER_ID_1` + `STAFF_USER_ID_2` の group DM を開く。
9. bot がその group DM に follow-up message を投稿する。
10. 見つからなかった場合、`#need_operation` に alert を投稿し、`MY_USER_ID` に DM する。

## 重要な設計上の観察

現在の形は少し遠回りになっている。

```text
Eigo が no-show を検知 -> Slack 通知 -> Slack bot が Slack message を parse -> group DM
```

ただし Eigo は Slack 通知を投稿する前の時点で、session / learner / DP の情報をすでに持っている。長期的によりきれいな設計は次の形。

```text
Eigo が no-show を検知 -> Slack 通知 + group DM follow-up を直接実行
```

これにすると、Slack message parsing、Slack Events webhook、別 bot のデプロイ、cold start、重複 event handling、追加の secret/deploy surface を減らせる。

## デプロイ選択肢

### Option A: 当面 Railway のままにする

短期的には推奨。

Pros:

- すでにドキュメント化されていて、おそらく稼働済み。
- Ando さんが iteraton しやすい。
- 小さな自動化としては repo/deploy が分かれているのは便利。
- 長期設計が固まる前に GCP/Terraform 作業を増やさずに済む。

Requirements:

- Railway project/admin ownership を Prospace 配下に移す、または Prospace owner が見える状態にする。
- repo は `prospaceinc/session-check-in-bot` 配下に置く。
- Ando さんには bot code と検証を進めてもらう。
- production secrets と owner-level operation は Satoshi/Nao が管理する。

### Option B: 外部 bot のまま GCP Cloud Run に移す

可能ではあるが、最初にやることとしては最適ではない。

Pros:

- Prospace 管理の infrastructure になる。
- IAM、Secret Manager、logging、billing は整理しやすい。
- Terraform 管理も可能。

Cons:

- 間接的な architecture は残る: Eigo -> Slack -> bot -> Slack DM
- Cloud Run、Secret Manager、Artifact Registry、deploy pipeline、Slack URL 切り替え、monitoring が追加で必要になる。
- Cloud Run の min instances が 0 の場合、Slack Events が cold start の影響を受ける可能性がある。Slack は速い 2xx 応答を期待するため、Cloud Run にするなら min instances 1 にするか、即 ack + async processing に変える必要がある。

### Option C: Eigo に統合する

この workflow が今後も重要なら長期的には推奨。

Pros:

- 追加の bot deployment が不要。
- Slack notification parsing が不要。
- Eigo が learner / DP / session の canonical data を持っている。
- 名前ズレや重複処理を避けやすい。

Cons:

- eigo production code の変更が必要。
- Nao review と controlled deploy が必要。
- Ando さんが独立して iterate するには外部 bot よりやりにくい。

## Terraform / infra 管理

古い eigo PR #514 `Add infra codes` で GCP Terraform 構成を追加しようとしていた形跡がある。Prospace/Eigo の infrastructure を code 管理しようという意図は実際にあった。

重要なのは、Terraform が eigo repo にあり、app code が別 bot repo にあること自体は普通にあり得るという点。Terraform は、source が別 repo にある app の deploy target/resource を管理できる。

あり得る構成:

- `prospaceinc/eigo`: `infra/terraform/...` で Prospace/Eigo GCP infra を管理
- `prospaceinc/session-check-in-bot`: bot application code

これはそれ自体は問題ではない。ただし bot の infra がどこにあるかをドキュメント化しておく必要がある。

Terraform 化すると、infra change を code review できるので Ando さんの関与も安全にしやすい。

```text
branch -> Terraform code change -> PR -> terraform plan review -> approved apply
```

Railway/GCP console だけで変更すると、完全な Git trail が残らないため review しづらい。

## 権限 / Ando さんの関与

最初から広い production infrastructure 権限を渡すべきではない。

理由は、infra change は見えている task より影響範囲が広くなりやすいため。能力の問題ではなく、auditability、reversibility、blast radius の問題。

許可してよさそうな範囲:

- bot repo の code change
- branch 作成と PR
- bot behavior の変更
- bot project に scope された Railway development/test deploy
- bot project の log 確認
- staging verification
- `dp_map.csv` 更新

注意が必要な範囲:

- repo/org Admin の常時権限
- eigo production deploy
- eigo production DB/secrets
- GCP IAM
- Secret Manager values
- Terraform apply
- Slack production token/signing secret
- Railway production env var changes

最初の境界線としては次がよい。

- Ando さんは bot app の iteration と staging/test confirmation を担当できる。
- Nao/Satoshi は production secrets、deploy ownership、GCP IAM、production deploy を管理する。
- infra が Terraform 管理なら、Ando さんは PR を出し、Nao/Satoshi が plan/apply を review する。

## GitHub / production protection

Write permission だけでは production code protection として不十分。Write user が PR を作って main/master に merge できるなら、production code を変更できる。

提案する能力と、land/deploy する能力を分けるべき。

- main/master への direct push 禁止
- PR 必須
- eigo 本体への変更は trusted review が必要
- production deploy は Nao/Owner が control
- repo settings/secrets/rulesets は Admin/Owner のみ

注意点: repo-wide required reviews にすると、Nao の PR も誰か別の approve が必要になる。一人運用では重すぎる可能性がある。eigo では rules を慎重に使うか、まず operational rule + branch protection から始め、必要に応じて締めるのがよい。

## Staging access / IAP notes

staging docs が説明しているのは Basic Auth ではなく Google Cloud IAP。

layer は 2 つある。

1. Google login + IAP による staging site 自体への access
2. Prospace admin/corp/learner/partner 内での application login

`@prospace.co.jp` account は、browser access 用の staging IAP をすでに通れる可能性がある。

SSH/GCP console/service management は別で、IAM/OS Login/GKE/Compute permissions が必要。IAP-secured Web App User だけでは SSH や service management はできない。

eigo-staging IAM の screenshot を見る限り、現在の access は以前ドキュメントにあった developer/debug Google Groups ではなく、直接 IAM user assignment になっている可能性がある。Ando さんには Editor/Project IAM Admin ではなく、debug/staging-viewer level から始めるのがよい。

## Development flow onboarding

Ando さんは「チケット作成 -> 開発 -> ステージング確認 -> デプロイ -> 報告」を知りたいと言っている。

進め方の案:

- まず既存ドキュメントを共有する。
- 全体フローを high level で説明する。
- staging は単一の shared environment なので、staging check 前の coordination を強調する。
- production deploy は最初は Nao/Owner controlled のままにする。

説明するフロー:

1. Ticket: 目的、背景、期待する挙動、bug なら再現手順/影響範囲。
2. Development: branch、small changes、意味のある範囲で tests、production secrets は触らない。
3. PR: summary、verification steps、impact scope、reviewer request。
4. Staging: staging は共有環境なので timing を調整する。staging でも実メール/SMS 通知に注意する。結果を記録する。
5. Deploy: production deploy は合意後のみ。最初は Nao/Owner が実行する。
6. Report: 何を変えたか、何を確認したか、残課題。

権限を決める前に Ando さんに聞くこと:

- これは一回限りの bot 改善か、継続的な automation work か。
- eigo 本体の code も触る予定があるか。
- どれくらいの頻度で変更する想定か。
- staging verification だけ必要なのか、production deploy involvement も必要なのか。
- Nao/Satoshi に何を期待しているか。

## 当面の推奨スタンス

短期:

- `session-check-in-bot` は別 repo のまま Railway で維持する。
- Railway ownership/visibility を Prospace 配下に寄せる。
- Ando さんには bot code と test flow を iterate してもらう。
- production secrets/deploy/admin は Nao/Satoshi が管理する。
- development/staging docs を共有し、staging 利用は調整する。

中期:

- この workflow が今後も重要なら、no-show 検知箇所で Eigo から直接 follow-up DM を送る実装を検討する。

長期:

- Prospace/Eigo infra は Terraform/IaC 方針を継続または復活させる。
- infra change は PR で提案し、plan を review してから apply する。
