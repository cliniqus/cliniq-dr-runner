# Security Policy

## Trust model

This public repository contains workflow YAML only. Application code and data
remain in the private website repository and provider systems.

The workflow accepts only:

- GitHub manual dispatches by authorized repository operators.
- `data_fortress_job` repository dispatches created by the restricted
  Cloudflare coordinator GitHub App.

There are no pull-request, push, or schedule triggers.

## Runner protections

- Workflow permissions are explicitly `contents: read` and `id-token: write`;
  the OIDC token is exchanged for short-lived Google credentials.
- Every third-party Action reference is pinned to a full commit SHA.
- The private source token is read-only and is not persisted by checkout.
- `npm ci` uses the committed lockfile and disables install scripts.
- One concurrency group per job type prevents overlapping work of the same
  kind. The Core lease and fencing number remain the final authority.
- Jobs have a six-hour hard timeout.
- Checked-out source is explicitly removed in an `always()` step.
- No hosted-runner directory is treated as a durable local copy.

## Signed evidence

`FORTRESS_JOB_SIGNING_PRIVATE_KEY` is available only to the Core process.
The Core writes signed job evidence under:

```text
firebase/evidence/jobs/<jobId>/<sequence>.json
firebase/evidence/jobs/<jobId>/latest.json
```

A job is not successful merely because the workflow process exited. A
terminal success must be signed and stored by the Core. Consumers verify it
using the matching configured public key.

The hosted workflow does not contain the offline recovery decryption key.
Restore-drill and restore-execute request a short-lived, signed grant bound to
the exact job, plan, generation, target, and nonce. They fail closed when the
grant, target configuration, provider permissions, or exact post-restore
verification is missing. Cutover-plan produces only a signed operator
checklist; it never promotes traffic.

## Credential rules

- Use a read-only, repository-scoped source token.
- Restrict the Google workload-identity provider to this exact repository and
  workflow, and give its dedicated service account only the required
  inventory/export permissions.
- Scope each R2 credential to its required bucket and prefixes.
- Keep the signing private key separate from manifest-verification public
  keys.
- Never store an offline recovery decryption private key in GitHub.
- Rotate any credential immediately after suspected exposure.
- Never print secret values or include them in job evidence.

R2 does not provide S3 object versioning. Immutable generation protection is
provided by committed generation names, conditional writes, and R2 bucket-lock
rules.

## Reporting vulnerabilities

Use the repository Security tab and select **Report a vulnerability**. Do not
publish credentials or exploit details in a public issue.
