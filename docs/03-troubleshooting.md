# 3. Troubleshooting

Start with the workflow run log: **Actions → sync-blog → the failed run**. Each
step fails for a different reason, and the step name tells you which layer broke.

## Reading the failure by step

| Failing step | Layer | Look at |
| --- | --- | --- |
| *Get Keycloak token* | identity | client id/secret, realm, client enabled |
| *Sync blog content* | routing / auth / payload | the `HTTP <code>` line printed before the error |
| *Revalidate site cache* | cache only | non-fatal; site refreshes on TTL |

## `401` from `/api/blog/sync`

Two different causes, same status. The workflow log distinguishes them only by
which check ran first — the token check happens before the body check.

**Token rejected** (role or audience wrong):

- realm role `admin` not assigned to the service account
- missing Audience mapper for `tmlabs-api`
- token from the wrong realm

Decode the token to see what it actually carries:

```bash
curl -sS -X POST https://auth.tiyiselani.com/realms/non-prod/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=tmlabs-ci \
  -d client_secret="$KEYCLOAK_CI_CLIENT_SECRET" \
  | sed -n 's/.*"access_token":"\([^"]*\)".*/\1/p' \
  | cut -d. -f2 \
  | tr '_-' '/+' \
  | base64 -d 2>/dev/null | python3 -m json.tool
```

Look for `"aud"` containing `tmlabs-api` and `"realm_access": {"roles": [ ...
"admin" ...]}`.

**Sync token rejected** — the body token does not match `GitHub__SyncToken`:

```bash
kubectl -n tmlabs get secret blogapi-secrets \
  -o jsonpath='{.data.GitHub__SyncToken}' | base64 -d | cut -c1-8; echo
```

Compare with the `BLOG_SYNC_TOKEN` secret in this repo. They must match exactly.

## `403`

A valid token without the `admin` role. The endpoint is `Roles("admin")`; the
role mapping comes from `realm_access.roles`.

## `502` from `/api/blog/sync`

The API reached GitHub and failed. `detail` in the response holds the message.

| Message | Cause |
| --- | --- |
| `GitHub truncated the repository tree` | repo too large for one recursive call |
| `401` from GitHub | `GitHub__Token` missing on a private repo, or expired |
| `404` | wrong owner/repository/branch |

## `404` on the curl

`/api/blog/sync` is not routed. Either the Ingress path rule is missing, or the
pods behind it are gone. Check from a machine with `kubectl`:

```bash
kubectl -n tmlabs get ingress tmlabs-web -o yaml | grep -A3 "/api/blog"
kubectl -n tmlabs exec deploy/gateway-deployment -- \
  curl -sS http://127.0.0.1:8080/api/blog/health
kubectl -n tmlabs get pods
```

If the gateway answers internally but the public call 404s, the Ingress path rule
from `tmlabs-gitops/manifests/tmlabs-extras/ingress.yaml` hasn't been synced.

## `502` / timeouts before the API answers

Cloudflare reaches the tunnel but the origin doesn't respond — the pods are down
or Traefik has no endpoints:

```bash
kubectl -n tmlabs get pods
kubectl -n tmlabs get endpoints gateway-service
```

An empty `ENDPOINTS` list means the Service selects nothing.

## Content pushed but the site still shows the old version

In order:

1. Did the run finish? A failed *Revalidate* step still leaves the DB updated.
2. Revalidate by hand:
   ```bash
   curl -X POST https://blog-test.tiyiselani.com/api/revalidate \
     -H "x-revalidate-secret: $REVALIDATE_SECRET" -d '{}'
   ```
3. Confirm the database has it — the public API is the fastest check:
   ```bash
   curl -s https://blog-test.tiyiselani.com/api/blog/ | head -c 200
   ```
4. Browser cache: hard reload.

## A post does not appear at all

Check the path rule first — it is the most common cause:

```
Theme/file.md   ✅      README.md ❌      Theme/sub/file.md ❌
```

Then `draft: true`, then whether the file is on `main` (the sync follows
`BLOG_REPO_BRANCH`, default `main`).

## Nothing runs on push

The workflow filters on `**/*.md`. A push touching only images or other files
will not trigger it. Use **Run workflow** to force a sync.
