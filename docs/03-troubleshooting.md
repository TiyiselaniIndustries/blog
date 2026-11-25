# 3. Troubleshooting

Start with the workflow run log: **Actions → sync-blog → the failed run**. Each
step fails for a different reason, and the step name tells you which layer broke.

## Reading the failure by step

| Failing step | Layer | Look at |
| --- | --- | --- |
| *Get Keycloak token* | identity | client id/secret, realm, client enabled |
| *Sync blog content* (before "Calling") | network / cluster | port-forward log printed on failure |
| *Sync blog content* (at the curl) | auth or payload | HTTP status in the log |
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

## `404 page not found` on the curl

The gateway did not match `/api/blog/sync`. Confirm the route exists:

```bash
kubectl -n tmlabs exec deploy/gateway-deployment -- \
  curl -sS http://127.0.0.1:8080/api/blog/health
```

If that works, the request never reached the gateway — check the port-forward
port in the log.

## Port-forward fails

| Message | Cause |
| --- | --- |
| `unable to forward ... address already in use` | a previous run left 18080 bound; reboot the runner or `kill` it |
| `error: unable to upgrade connection` | the service or pods are gone |
| `must be logged in to the server` | the runner's kubeconfig expired |

Check what is actually running in the cluster:

```bash
kubectl -n tmlabs get pods
kubectl -n tmlabs get svc gateway-service
```

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
