# cliniq-dr-runner

Scheduled disaster-recovery automation for a private project.

This repository contains **only GitHub Actions workflow definitions** —
no application code, no business logic, no data. Its sole purpose is to
run scheduled backup and verification jobs on GitHub-hosted runners,
which clone a separate private repository at execution time.

## What this repo does

Every day at fixed times (UTC), a GitHub-hosted runner:

1. Clones a private source repository using a read-only token.
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
| `vault-sync.yml` | Daily 03:00 | Sync verification vault to off-site storage |
| `dr-checks.yml` | Daily 06:00 | Run 6 read-only integrity checks, update evidence |

## Why public?

GitHub-hosted Actions minutes are **unlimited on public repositories**
for standard runners. The actual source code and data remain in a
private repository, cloned ephemerally at runtime. This gives us
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
  most a 1-year lifetime and must be rotated before expiry.

## Reporting security issues

See [`SECURITY.md`](./SECURITY.md).

## Licence

MIT — see [`LICENSE`](./LICENSE).
