# 2. Sync pipeline

## What a push triggers

[`../.github/workflows/sync-blog.yml`](../.github/workflows/sync-blog.yml) runs on
a push to `main` that touches `**/*.md`, or manually via **Run workflow**.

```
1. client_credentials  → Keycloak          get an access_token
2. HTTPS POST to the public host  →  /api/blog path rule  →  gateway
3. POST /api/blog/sync (Bearer + syncToken)
      gateway  → PathRemovePrefix /api/blog → blogapi /sync
      blogapi  → GitHub API: repo tree, blobs
      blogapi  → PostgreSQL: insert/update posts + revisions
4. POST /api/revalidate (x-revalidate-secret) on the public site
```

The workflow never uploads content. The API reads this repository over the GitHub
API using its own credentials — the workflow only says "look now".

## Two ways to trigger it

| Trigger | Path |
| --- | --- |
| Push to this repo | `sync-blog.yml` → `POST /api/blog/sync` |
| Signed-in admin on `/blog` | "Sync now" button → server action → same endpoint |

The in-app button forwards the user's access token and injects `BLOG_SYNC_TOKEN`
server-side, so the sync token never reaches the browser. It only renders for
sessions carrying the realm role `admin`.

## Why not a self-hosted runner

This repository is **public**. Running its workflow on the self-hosted runner
would let a fork open a pull request and execute arbitrary code inside your
network — so the job runs on `ubuntu-latest` instead.

That means it needs an internet route to the sync endpoint. The gateway Service is
still `ClusterIP`; the Ingress instead publishes exactly one path on the frontend
host:

```
blog-test.tiyiselani.com/api/blog  →  gateway-service:8080
blog-test.tiyiselani.com/          →  web-service:3000
```

Longest prefix wins, so `/api/blog/*` reaches the gateway and everything else
reaches the frontend. `/api/chat` stays unreachable from outside.

Exposing the path does not expose the operation: the endpoint still requires a
token with the realm role `admin` and a matching `syncToken`. An unauthenticated
`POST` returns `401`, which is the correct answer.

## Two credentials, both required

The endpoint checks two independent things — failing either returns `401`:

| Check | Detail |
| --- | --- |
| `Roles("admin")` | the access token must carry the realm role `admin` |
| `syncToken` in the body | must equal `GitHub__SyncToken` from the Secret |

The role requirement is why a dedicated Keycloak client exists rather than a
shared static token. Roles reach the endpoint because
`KeycloakRolesTransformer` flattens `realm_access.roles` into
`ClaimTypes.Role`.

The token also needs `aud = tmlabs-api`. A service-account token's audience
defaults to `account`, so the client needs an **Audience mapper**; without it the
call is rejected even with the correct role.

## Why the site needs revalidating

Blog pages are cached server-side (`use cache` with tag `blog`, 60s revalidate).
A sync changes the database, not the cache. Without step 4 the site keeps serving
stale HTML until the TTL expires — up to a few minutes of "my post isn't there".

`POST /api/revalidate` expires the tag immediately and revalidates `/`, `/blog`,
and the affected post. It fails closed with `401` when `REVALIDATE_SECRET` is
unset, which is why the step warns instead of failing the run.

## What the response tells you

```json
{ "commitSha": "…", "changed": 2, "unpublished": 0, "skipped": 12 }
```

| Field | Meaning |
| --- | --- |
| `commitSha` | the commit the sync read |
| `changed` | posts inserted or updated |
| `unpublished` | previously published posts whose file is no longer in the tree |
| `skipped` | files whose checksum matched the stored revision |

`changed: 0, skipped: N` on a run right after a content push means the API read a
different commit than you expected — check `BLOG_REPO_BRANCH` and that the push
actually landed on `main`.

## Configuration

| Where | Name | Purpose |
| --- | --- | --- |
| repo variable | `KEYCLOAK_CI_CLIENT_ID` | client used for `client_credentials` |
| repo secret | `KEYCLOAK_CI_CLIENT_SECRET` | its secret |
| repo secret | `BLOG_SYNC_TOKEN` | must equal `GitHub__SyncToken` in the cluster |
| repo secret | `REVALIDATE_SECRET` | must equal `REVALIDATE_SECRET` in `web-secrets` |
| repo variable | `BLOG_REPO_OWNER` / `_NAME` / `_BRANCH` | optional overrides; default to `TiyiselaniIndustries` / `blog` / `main` |

On the API side, `GitHub:Owner`, `GitHub:Repository`, and `GitHub:Branch` in
`appsettings.json` provide the same defaults. `GitHub:Token` is optional — it
raises the API rate limit and is needed if this repo is private.

## Constraint: repository size

The sync requests `git/trees/{sha}?recursive=1` in one call and fails loudly if
GitHub truncates the response (roughly 100k entries / 7 MB). If this repository
ever grows past that, the sync needs a non-recursive traversal — not a concern at
current size, but it is a hard failure rather than a partial sync.

Next: [03-troubleshooting.md](03-troubleshooting.md)
