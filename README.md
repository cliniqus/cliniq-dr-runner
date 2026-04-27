# cliniq-dr-runner

Scheduled disaster-recovery automation for a private project.

This repository contains **only GitHub Actions workflow definitions** —
no application code, no business logic, no data. Its sole purpose is to
run scheduled backup and verification jobs on GitHub-hosted runners,
which clone separate private source repositories at execution time.

## What this repo does

On scheduled intervals (UTC), a GitHub-hosted runner:

1. Clones the private website and Firebase app repositories using a read-only token.
2. Loads runtime credentials from GitHub Secrets (never committed).
3. Executes backup / verification scripts against the project's
   production data plane.
4. Writes evidence artifacts to object storage.
5. Destroys the runner VM.

All credentials, source code, customer data, and business logic live
outside this repository. This repo is intentionally minimal so its
public surface is just infrastructure-as-code YAML.

## Schedule

| Workflow | Cadence (UTC) | Purpose |
| --- | --- | --- |
| `backup.yml` | Daily 02:00 | Full backup + snapshot to off-site storage |
| `mirror-heartbeat.yml` | Every 10 minutes | Probe real-time Firebase mirror lag without Vercel Cron |
| `vault-sync.yml` | Daily 03:00 | Sync verification vault to off-site storage |
| `mirror-auth-sync.yml` | Daily 03:15 | Sync recoverable Firebase Auth export into mirrors |
| `dr-checks.yml` | Daily 06:00 | Run 6 read-only integrity checks, update evidence |

## Why public?

GitHub-hosted Actions minutes are **unlimited on public repositories**
for standard runners. The actual source code and data remain in a
private repositories, cloned ephemerally at runtime. This gives us
production-grade scheduled automation at zero cost without exposing
any sensitive material.

See [`SECURITY.md`](./SECURITY.md) for the full threat model and
hardening design.

## Maintenance

This repo is designed to be low-touch:

- **No dependencies to update** — the YAML workflows call a separate
  repository whose own dependencies are managed there.
- **Pinned actions** — every `uses:` entry is pinned to a specific
  commit SHA; Dependabot proposes updates via PR.
- **Annual rotation** — the `CLONE_TOKEN` fine-grained PAT has at
  most a 1-year lifetime, must be rotated before expiry, and must have
  read-only Contents access to both `SRC_REPO` and `FIREBASE_APP_REPO`.

## Reporting security issues

See [`SECURITY.md`](./SECURITY.md).

## Licence

MIT — see [`LICENSE`](./LICENSE).
