# Eigo session-check-in bot / Ando onboarding notes

#eigo #prospace #slackbot #infra #onboarding

# Eigo session-check-in bot / Ando onboarding notes

Date: 2026-05-17

## Context

Izumi Ando shared updates about the session-check-in bot and asked for repository/admin access, possible GCP access, production code protection, and an explanation of the development flow: ticket creation -> development -> staging verification -> deploy -> report.

The bot repository has been transferred to prospaceinc/session-check-in-bot. The transfer notification itself requires no action.

## Current bot shape

Repository: https://github.com/prospaceinc/session-check-in-bot

Current implementation is a small Node.js Slack Events bot using Slack Bolt.

Files observed:
- index.js: main bot logic
- package.json / package-lock.json: Node dependencies and scripts
- dp_map.csv: Discussion Partner name -> Slack user ID mapping
- slack-prospaceinc-members.csv: Slack member export/reference data
- README.md: local setup and Slack App configuration
- DEPLOYMENT.md: Railway deployment guide
- TEST_PLAN.md: manual test plan

Runtime/spec:
- Node.js
- @slack/bolt
- Starts with npm start -> node index.js
- Slack endpoint is Bolt default /slack/events
- No DB
- No persistent state
- Config is env vars + dp_map.csv
- Current docs assume Railway deployment

Required env vars:
- SLACK_BOT_TOKEN
- SLACK_SIGNING_SECRET
- MY_USER_ID
- STAFF_USER_ID_1
- STAFF_USER_ID_2
- CHANNEL_ID
- PORT

Required Slack scopes documented:
- channels:history
- channels:read
- chat:write
- im:write
- mpim:write
- users:read

## Current processing flow

1. Eigo app posts a no-show notification to Slack #need_operation.
2. Slack Events API sends message.channels event to the bot.
3. Bot checks the channel equals CHANNEL_ID.
4. Bot checks message text contains required Japanese markers.
5. Bot parses the line after 受講生 as learner name.
6. Bot parses the line after ディスカッションパートナー as DP name.
7. Bot looks up DP name in dp_map.csv.
8. If found, bot opens group DM with DP + MY_USER_ID + STAFF_USER_ID_1 + STAFF_USER_ID_2 using conversations.open.
9. Bot posts a follow-up message to that group DM.
10. If not found, bot posts an alert to #need_operation and DMs MY_USER_ID.

## Important design observation

The current shape is indirect:

Eigo detects no-show -> Slack notification -> Slack bot parses Slack message -> group DM

But Eigo already has the session, learner, and DP information before posting the Slack notification. The cleaner long-term design is:

Eigo detects no-show -> Slack notification + group DM follow-up directly

This would remove Slack message parsing, Slack Events webhook handling, separate bot deployment, cold start concerns, duplicate event handling, and one extra secret/deploy surface.

## Deployment options

### Option A: Keep Railway for now

Recommended short-term option.

Pros:
- Already documented and likely already running.
- Ando can iterate easily.
- Separate repo/deploy is convenient for a small automation.
- Avoids GCP/Terraform work before the long-term design is clear.

Requirements:
- Move Railway project/admin ownership under Prospace or make Prospace owner visible.
- Keep repo under prospaceinc/session-check-in-bot.
- Let Ando work on bot code and verification.
- Keep production secrets and owner-level operations controlled by Satoshi/Nao.

### Option B: Move external bot to GCP Cloud Run

Possible, but not the best first move.

Pros:
- Prospace-controlled infrastructure.
- IAM, Secret Manager, logging, billing are cleaner.
- Can be Terraform-managed.

Cons:
- Still keeps the indirect architecture: Eigo -> Slack -> bot -> Slack DM.
- Adds Cloud Run, Secret Manager, Artifact Registry, deploy pipeline, Slack URL switch, and monitoring work.
- If Cloud Run min instances is 0, Slack Events may be affected by cold starts. Slack expects fast 2xx responses. If using Cloud Run, either set min instances 1 or change implementation to immediate ack + async processing.

### Option C: Integrate into Eigo

Recommended long-term option if this workflow remains valuable.

Pros:
- No extra bot deployment required.
- No Slack notification parsing.
- Eigo has canonical learner/DP/session data.
- Easier to avoid name mismatch and duplicate handling.

Cons:
- Requires eigo production code changes.
- Needs Nao review and controlled deploy.
- Not as easy for Ando to iterate independently.

## Terraform / infra management

There is an old eigo PR #514, Add infra codes, adding GCP Terraform structure. It shows there was a real intent to manage Prospace/Eigo infrastructure as code.

Important point: having Terraform in eigo repo and app code in a separate bot repo is normal. Terraform can manage deploy targets/resources for an app whose source lives elsewhere.

Possible structure:
- prospaceinc/eigo: infra/terraform/... for Prospace/Eigo GCP infra management
- prospaceinc/session-check-in-bot: bot application code

This is not inherently a problem, but it must be documented so people know where the infra for the bot lives.

Terraform makes Ando involvement safer because infra changes can be reviewed as code:

branch -> Terraform code change -> PR -> terraform plan review -> approved apply

Console-only Railway/GCP changes are harder to review because they do not leave a complete Git trail.

## Permissions / Ando involvement

Do not start by granting broad production infrastructure permissions.

Reason: infrastructure changes often have wider impact than the visible task. The issue is not ability; it is auditability, reversibility, and blast radius.

Reasonable to allow:
- bot repo code changes
- branch creation and PRs
- bot behavior changes
- Railway development/test deploys if scoped to the bot project
- log checking for the bot project
- staging verification
- dp_map.csv updates

Be careful with:
- repo/org Admin as a standing permission
- eigo production deploy
- eigo production DB/secrets
- GCP IAM
- Secret Manager values
- Terraform apply
- Slack production token/signing secret
- Railway production env var changes

Good initial boundary:
- Ando can own bot app iteration and staging/test confirmation.
- Nao/Satoshi manage production secrets, deploy ownership, GCP IAM, and production deploy.
- If infra is Terraform-managed, Ando can propose PRs and Nao/Satoshi review plan/apply.

## GitHub / production protection

Write permission alone is not enough to protect production code. If a Write user can create PR and merge to main/master, they can change production code.

Protection should separate ability to propose changes from ability to land/deploy them:
- no direct push to main/master
- PR required
- merge requires trusted review for eigo body changes
- production deploy is controlled by Nao/Owner
- repo settings/secrets/rulesets are Admin/Owner only

Caveat: repo-wide required reviews mean Nao also needs someone else to approve Nao PRs. This may be too heavy for one-person operation. For eigo, use rules carefully or start with operational rule + branch protection, then tighten later.

## Staging access / IAP notes

The staging docs describe Google Cloud IAP, not Basic Auth.

There are two layers:
1. Access to staging site itself via Google login + IAP.
2. Application login inside Prospace admin/corp/learner/partner.

@prospace.co.jp accounts may already pass staging IAP for browser access.

SSH/GCP console/service management is separate and requires IAM/OS Login/GKE/Compute permissions. IAP-secured Web App User is not enough for SSH or service management.

A screenshot of eigo-staging IAM suggests current access may be direct IAM user assignment rather than the old developer/debug Google Groups documented earlier. For Ando, start with debug/staging-viewer level, not Editor/Project IAM Admin.

## Development flow onboarding

Ando asked to learn ticket creation -> development -> staging verification -> deploy -> report.

Suggested approach:
- First share existing documentation.
- Explain full flow at a high level.
- Because staging is a single shared environment, emphasize coordination before staging checks.
- Production deploy remains Nao/Owner-controlled at first.

Flow to explain:
1. Ticket: purpose, background, expected behavior, repro/impact if bug.
2. Development: branch, small changes, tests where meaningful, no production secrets.
3. PR: summary, verification steps, impact scope, reviewer request.
4. Staging: coordinate timing because staging is shared; beware real email/SMS notifications even in staging; record results.
5. Deploy: production deploy only after agreement; Nao/Owner executes initially.
6. Report: what changed, what was verified, remaining issues.

Need to ask Ando before deciding permissions:
- Is this one-off bot improvement or ongoing automation work?
- Will she work on eigo body code too?
- How often does she expect to make changes?
- Does she need staging verification only, or production deploy involvement?
- What does she expect Nao/Satoshi to do?

## Suggested near-term stance

Short term:
- Keep session-check-in-bot separate and on Railway.
- Bring Railway ownership/visibility under Prospace.
- Let Ando iterate on bot code and test flow.
- Keep production secrets/deploy/admin controlled by Nao/Satoshi.
- Share development/staging docs and coordinate staging use.

Medium term:
- If the workflow continues to matter, consider implementing the follow-up DM directly in eigo where no-show is detected.

Long term:
- Continue or revive Terraform/IaC direction for Prospace/Eigo infra.
- Let infra changes be proposed by PR and reviewed with plan before apply.

