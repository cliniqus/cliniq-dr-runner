# cliniq-dr-runner

This public repository contains one thin GitHub-hosted runner Adapter. It has
no backup rules and no scheduler.

Cloudflare sends a `data_fortress_job` repository dispatch. The workflow
validates the payload, checks out the private website repository, installs its
locked dependencies, and runs:

```text
npx tsx scripts/fortress/run-job.ts --job-id <id> --job-type <type> --nonce <nonce>
```

The website `DataFortressCore` owns backup, verification, signing, and evidence
behavior. The runner also contains the isolated-target restore Adapter. Restore
execution remains fail-closed until the registered target configuration,
short-lived offline key grant, provider permissions, and live restore gates are
present and verified.

## Triggers

- `repository_dispatch` event type `data_fortress_job`
- Manual `workflow_dispatch`

There is deliberately no GitHub `schedule:` trigger. All automatic scheduling
is owned by the Cloudflare coordinator.

Supported runner jobs:

- `backup`
- `auth-continuity`
- `media-parity`
- `deep-verify`
- `restore-drill`
- `restore-plan`
- `restore-execute`
- `cutover-plan`
- `local-copy-verify`

`restore-drill` and `restore-execute` request a short-lived generation-key
grant from an offline operator, then restore only to a registered non-primary
target. `cutover-plan` writes an operator checklist; it never changes DNS,
Vercel traffic, OAuth, payments, notifications, or mobile configuration. No
hosted runner contains the offline recovery decryption key.

Coordinator-created jobs carry an expiring, Ed25519-signed request in
secondary R2. The runner verifies that request, conditionally claims its
request-state object, and only then consumes the dispatch nonce. A coordinator
dispatch failure revokes the pending state. Recovery jobs also acquire the
secondary-R2 fenced restore lease. Three missed heartbeats abort its control
signal and prevent a successful commit; the missing restore Adapter must check
that signal before every mutating batch.

`local-copy-verify` accepts only a pending receipt key and its SHA-256. The
Core validates the receipt before publishing verified local-copy evidence.

## Repository configuration

Variables used by the installed jobs:

- `SRC_REPO`
- `FIREBASE_PROJECT_ID`
- `FIREBASE_STORAGE_BUCKET`
- `R2_BUCKET_NAME`
- `R2_BACKUP_BUCKET_NAME`
- `GCS_STAGING_BUCKET`
- `GCP_WORKLOAD_IDENTITY_PROVIDER`
- `GCP_FORTRESS_SERVICE_ACCOUNT`
- `GCP_FIREBASE_ADMIN_KEY_SECRET`
- `FORTRESS_RECOVERY_TARGETS_JSON`
- `FORTRESS_RECOVERY_TARGET_CONFIG_JSON`
- `FORTRESS_SIGNING_KEY_ID`
- `FORTRESS_ENCRYPTION_KEY_ID`

Secrets used by the installed jobs:

- `SOURCE_REPOSITORY_TOKEN`, a read-only token scoped to `SRC_REPO`
- `FIREBASE_ISOLATED_SERVICE_ACCOUNT_KEY`
- `FIREBASE_RECOVERY_CANARY_API_KEY`
- `FIREBASE_RECOVERY_CANARY_EMAIL`
- `FIREBASE_RECOVERY_CANARY_PASSWORD`
- `FIREBASE_AUTH_HASH_CONFIG_JSON`
- Primary and secondary R2 credentials
- `FORTRESS_JOB_SIGNING_PRIVATE_KEY`
- `FORTRESS_SIGNING_PUBLIC_KEYS_JSON`
- `FORTRESS_COORDINATOR_SIGNING_PUBLIC_KEYS_JSON`
- `FORTRESS_APPROVAL_SIGNING_PUBLIC_KEYS_JSON`
- `FORTRESS_RECOVERY_OPERATOR_PUBLIC_KEYS_JSON`
- `R2_RECOVERY_ACCESS_KEY_ID`
- `R2_RECOVERY_SECRET_ACCESS_KEY`
- `FORTRESS_ENCRYPTION_PUBLIC_KEY`
- `FORTRESS_RECOVERY_CAPSULE_JSON` (encrypted by the Core into each
  generation before upload)
- `FIREBASE_MIRROR_REGISTRY_JSON`
- `FORTRESS_COST_MODEL_JSON`
- `FIRESTORE_RULES_SNAPSHOT`
- `FIREBASE_STORAGE_RULES_SNAPSHOT`
- Optional alert-delivery credentials

`FORTRESS_RECOVERY_TARGET_CONFIG_JSON` maps each registered target to its
Firebase Storage bucket, target GCS staging bucket, target R2 endpoint and
bucket, target R2 credential environment names, target service-account
environment name, Firebase API-key environment name, and recovery-canary
credential environment names. Its target entries must not point at primary
storage or staging, and the service-account
`project_id` must exactly equal the registered recovery project. Each target
also requires an HTTPS application smoke URL whose JSON identifies the same
Firebase project and R2 bucket; a generic HTTP 200 is not accepted.
`FORTRESS_RECOVERY_OPERATOR_PUBLIC_KEYS_JSON` trusts only public keys used to
sign the one-time offline generation-key grant. The corresponding private keys
and the offline recovery private key stay outside hosted systems.

No local-copy path is configured on the hosted runner. Its temporary
filesystem is never presented as a local backup.

Google Cloud API authentication is provided by the pinned
`google-github-actions/auth` Action with `id-token: write`. Firebase Admin SDK
does not accept external-account credentials, so the workflow reads the
Firebase Admin key from Secret Manager using the short-lived Google token,
marks it as a masked runtime environment value, and never stores it as a
GitHub repository secret.

See `SECURITY.md` for the trust and secret-handling rules.
