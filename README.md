# blog

Markdown posts synced into the TmLabs database by `TmLabs.BlogApi`.

## Layout

```
Archery/     posts by theme
Coding/      posts by theme
```

Each post carries front matter (`title`, `slug`, `description`, `author`,
`publishedAt`, `theme`, `draft`) read by the API during sync.

## Publishing

Pushing markdown to `main` runs
[`.github/workflows/sync-blog.yml`](.github/workflows/sync-blog.yml) on a GitHub-hosted
runner, which:

1. gets a Keycloak token for the `tmlabs-ci` service account,
2. calls `POST https://blog-test.tiyiselani.com/api/blog/sync` with the shared
   sync token (the Ingress routes that path to the gateway),
3. calls `POST /api/revalidate` on the same host so cached pages refresh.

This repo is public, so the workflow deliberately does **not** use a self-hosted
runner: a fork could otherwise execute code inside the internal network.

You can also run it by hand: **Actions → sync-blog → Run workflow**.

### Repository configuration

Variables:

| Name | Value |
| --- | --- |
| `KEYCLOAK_CI_CLIENT_ID` | `tmlabs-ci` |
| `BLOG_REPO_OWNER` *(optional)* | defaults to `TiyiselaniIndustries` |
| `BLOG_REPO_NAME` *(optional)* | defaults to `blog` |
| `BLOG_REPO_BRANCH` *(optional)* | defaults to `main` |

Secrets:

| Name | Value |
| --- | --- |
| `KEYCLOAK_CI_CLIENT_SECRET` | client secret for `tmlabs-ci` |
| `BLOG_SYNC_TOKEN` | must equal `GitHub__SyncToken` in the cluster |
| `REVALIDATE_SECRET` | must equal `REVALIDATE_SECRET` in `web-secrets` |

### Keycloak: the `tmlabs-ci` client

Realm `non-prod`, confidential client with **Service accounts roles** enabled.

1. **Service accounts roles** → Assign role → *Filter by realm roles* → `admin`.
   `POST /api/blog/sync` is `Roles("admin")`.
2. **Client scopes** → add an *Audience* mapper with *Included Custom Audience* =
   `tmlabs-api`, added to the access token. BlogApi validates `aud = tmlabs-api`;
   without this the call returns `401`.
3. Copy the client secret into `KEYCLOAK_CI_CLIENT_SECRET`.

No browser flow is needed — the grant is `client_credentials`.

### Manual run

`workflow_dispatch` on the workflow page, or locally:

```bash
TOKEN=$(curl -sS -X POST \
  https://auth.tiyiselani.com/realms/non-prod/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=tmlabs-ci \
  -d client_secret="$KEYCLOAK_CI_CLIENT_SECRET" \
  | sed -n 's/.*"access_token":"\([^"]*\)".*/\1/p')

curl -sS -X POST https://blog-test.tiyiselani.com/api/blog/sync \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"syncToken\":\"$BLOG_SYNC_TOKEN\"}"
```

The response reports `commitSha`, `changed`, `unpublished`, `skipped`.

## Documentation

- [docs/01-authoring.md](docs/01-authoring.md) — file layout, front matter, drafts
- [docs/02-sync-pipeline.md](docs/02-sync-pipeline.md) — what a push triggers, the auth chain
- [docs/03-troubleshooting.md](docs/03-troubleshooting.md) — reading a failed run

## Themes

| Folder | Contents |
| --- | --- |
| `Platform/` | TmLabs platform notes: gateway, auth, Keycloak, Argo CD, and the deployment blueprint series |
| `Coding/` | general engineering posts |
| `Archery/` | non-technical posts |

`Platform/` was generated from `tmlabs-backend/docs/`; posts there have their title
as front matter and their source links rewritten to GitHub, because the original
files cross-reference each other.
