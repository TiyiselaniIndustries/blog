---
title: 4. CI pipeline
description: concurrency: deploy-test with cancel-in-progress: false — a second push
series: Deployment blueprint
seriesOrder: 5
publishedAt: 2026-11-25T08:04:00Z
draft: false
---

One workflow: [../../.github/workflows/deploy.yml](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/.github/workflows/deploy.yml).
It runs on a **self-hosted Linux runner** with the `linux` label.

## Triggers

| Event | Source |
| --- | --- |
| `push` to `main` | this repo |
| `repository_dispatch: frontend-changed` | `tmlabs-frontend`'s `notify-backend.yml` |
| `workflow_dispatch` | manual |

`concurrency: deploy-test` with `cancel-in-progress: false` — a second push
queues behind the first rather than cancelling a half-finished deploy.

## Steps, in order

| # | Step | Purpose |
| --- | --- | --- |
| 1 | Checkout backend | this repo, into `tmlabs-backend/` |
| 2 | Checkout frontend | `tmlabs-frontend` via `CROSS_REPO_TOKEN` |
| 3 | Set image tag | `TAG=<short-sha>-<run>` → `$GITHUB_ENV` |
| 4 | Validate deployment variables | fails fast, lists every missing variable |
| 5 | Build and push .NET images | 3 × `dotnet publish -t:PublishContainer` |
| 6 | Build and push web image | `docker build` + `docker push` |
| 7 | Install Aspire CLI | into `$RUNNER_TEMP`, not system-wide |
| 8 | Install yq | same pattern |
| 9 | Set up Aspire CLI bundle | `aspire setup` |
| 10 | Generate Helm chart | `aspire publish` |
| 11 | Finalize chart | strip Secrets, drop dashboard, pin tags |
| 12 | Show generated chart | prints the file list and `values.yaml` |
| 13 | Commit chart to GitOps repo | gated by `GITOPS_PUSH_ENABLED` |

Steps 7–13 are covered in [05](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/05-chart-pipeline.md); 13 in [06](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/06-gitops-and-argo.md).

## Variables vs secrets

Repository **variables** — non-sensitive, safe in logs:

`APP_URL`, `API_BASE_URL`, `KEYCLOAK_URL`, `KEYCLOAK_REALM`,
`KEYCLOAK_CLIENT_ID`, `GITOPS_PUSH_ENABLED`

Repository **secrets** — tokens:

`CROSS_REPO_TOKEN` (read on frontend), `GITOPS_TOKEN` (write on gitops)

Step 4 exists because the old workflow happily built an image with
`Parameters__app-url` empty, and the resulting pods failed at runtime with
`Configuration` errors from Auth.js. Failing at minute one is cheaper.

## Runner prerequisites

Installed **once** on the runner, not per job:

| Tool | Why |
| --- | --- |
| .NET SDK 10 | `dotnet publish` |
| Docker (with access to build/push) | frontend image |
| `git`, `curl` | checkout, downloads |
| `unzip` | Aspire CLI install |

Node.js is **not** needed on the host — the frontend builds inside Docker.

Aspire CLI and yq are installed by the workflow itself; anything installed by
hand must be executable **by the runner account**, not just root.

## Traps

**`/usr/local/bin/aspire: Permission denied`**
A root-owned install the runner can't execute (or a `noexec` mount). Fixed by
installing with `dotnet tool install --tool-path "$RUNNER_TEMP/dotnet-tools"` and
appending that to `$GITHUB_PATH`.

**Self-hosted runners reuse their workspace.**
Unlike GitHub-hosted runners, `actions-runner/_work/` persists between jobs.
Two consequences:

- `git clone` into a fixed path fails with *destination path already exists*.
  Clone into `$RUNNER_TEMP/gitops-${GITHUB_RUN_ID}-${GITHUB_RUN_ATTEMPT}`.
- `chart/` sits **beside** the checkout (`$GITHUB_WORKSPACE/chart`), so
  `actions/checkout`'s cleanup never touches it and stale templates merge into
  the new chart. `rm -rf "${GITHUB_WORKSPACE}/chart"` before publishing.

**`repository_dispatch` needs write access.**
A fine-grained PAT scoped to *Contents: read* is rejected. `BACKEND_DISPATCH_TOKEN`
needs *Contents: read and write* on `tmlabs-backend`.

**Aspire bundle setup times out (120s).**
Seen as `ASPIRE009: Automatic Aspire CLI bundle setup failed: timed out`.
An explicit `aspire setup` step gives it its own time budget and a clear failure
point instead of a confusing MSBuild error.

## What CI does *not* do

- No `kubectl`, no `helm`, no cluster credentials. CI writes to Git; Argo applies.
- No database migrations (the services migrate themselves — [10](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/10-database-and-migrations.md)).
- No smoke tests against the deployed app.

Next: [05-chart-pipeline.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/05-chart-pipeline.md)
