# Security Policy

## Threat model

This repository is a **public scheduler** for scripts and Firebase
configuration that live in separate private repositories. Its public
surface consists of:

- GitHub Actions workflow YAML files.
- A README, LICENSE, and this SECURITY.md.

**It contains no secrets, no source code, no data, no credentials.**

The sensitive material (service-account keys, object-storage access
keys, authentication tokens) lives exclusively in two places:

1. **GitHub Secrets** on this repository (encrypted at rest with AES-256-GCM,
   decrypted only inside ephemeral runner VMs, never printed to logs).
2. **The private source repositories**, which are never exposed.

## Defence layers

### 1. Credential scoping (blast-radius minimisation)

| Credential | Scope | Worst-case impact if leaked |
| --- | --- | --- |
| `CLONE_TOKEN` (fine-grained PAT) | Read-only Contents access to `SRC_REPO` and `FIREBASE_APP_REPO`, 90-day TTL | Attacker can read source and Firebase config until rotation |
| `FIREBASE_SERVICE_ACCOUNT_KEY` | `roles/datastore.viewer` + `firebaseauth.viewer` only | Attacker can read Firestore; cannot write, delete, or modify IAM |
| `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` | Write-scoped to specific bucket + prefix | Attacker can upload objects; versioning prevents permanent deletion |
| `R2_BACKUP_*` | Same pattern on backup bucket | Same as above |
| `CRON_SECRET` | Bearer token for production cron endpoints | Attacker can trigger backup/mirror maintenance endpoints until rotation |
| `DISCORD_WEBHOOK_URL` (optional) | Post-only to one channel | Attacker can spam one channel |

### 2. Workflow safety

- `on:` limited to `schedule` and `workflow_dispatch`. **No `pull_request`
  trigger** — this is the single largest class of GitHub Actions
  vulnerabilities and we eliminate it by construction.
- `permissions:` explicitly set to `contents: read` (minimum) — the
  automatic `GITHUB_TOKEN` has no write capability.
- `timeout-minutes:` set on every job — no runaway executions.
- Every `uses:` entry is pinned to a full 40-character commit SHA —
  immune to tag hijacking.
- `concurrency:` group prevents overlapping runs.

### 3. Secret handling

- Secrets are only referenced via `${{ secrets.NAME }}` and assigned to
  process environment variables. Never written to files. Never echoed
  to stdout. Never concatenated into command strings.
- `npm ci --ignore-scripts` blocks npm `postinstall` scripts from
  executing on the runner.
- Runner filesystem is wiped by GitHub after each job; `if: always()`
  steps perform explicit cleanup for defence in depth.

### 4. Detection

- Every workflow posts success / failure status to an optional alerting
  channel (Discord webhook or Telegram bot).
- Post-backup verification runs `dr:check-all` — if the aggregate
  verdict degrades from the last run, the workflow fails loudly.
- Every run emits an audit record (runId, actor, SHA, timestamp) that
  can be cross-referenced against the GitHub Actions run log.

### 5. Recovery

- Cloudflare R2 bucket versioning is enabled on both primary and
  backup buckets. No object can be permanently deleted by a single
  rogue action.
- The last 30 days of backups are retained automatically; the weekly
  snapshot retention policy keeps one per week for the last 12 weeks.
- A manual restore runbook is maintained in the private repository.

## Reporting vulnerabilities

If you believe you have found a security vulnerability in the public
surface of this repository (workflow YAML, action pinning, permissions
configuration), please report it privately via:

- GitHub Security Advisories: **Security** tab → **Report a vulnerability**

Please **do not** open a public issue for security reports.

## Known non-issues

The following are **not** considered vulnerabilities:

- The existence of secret *names* in the workflow YAML. GitHub Secrets
  are designed to be referenced by name in public workflows; the values
  are never exposed.
- The cron schedule being visible. Timing information does not provide
  an attacker with any capability; the authentication surface does not
  depend on secrecy of schedules.
- The public runner running on a public schedule against private
  source repositories. This is the documented design pattern.

## Maintenance commitments

- `CLONE_TOKEN` is rotated every 90 days or immediately upon any
  suspected compromise, and its repository allow-list is kept to only
  `SRC_REPO` and `FIREBASE_APP_REPO`.
- `actions/*` SHA pins are reviewed quarterly via Dependabot PRs.
- Every secret is rotated at least annually.
- Any change to workflow YAML requires a pull request; the sole
  CODEOWNER must review and approve.
