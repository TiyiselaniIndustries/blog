---
title: 7. Configuration and secrets
description: The hardest part to get right, because three different mechanisms look similar and
series: Deployment blueprint
seriesOrder: 8
publishedAt: 2026-11-25T08:07:00Z
draft: false
---

The hardest part to get right, because three different mechanisms look similar and
one silent failure mode looks like another.

## The rule

| Kind | Mechanism | Where it lives |
| --- | --- | --- |
| Non-secret config | generated ConfigMap, `envFrom` | `values.yaml` → committed chart |
| Secret | ExternalSecret → k8s Secret, `envFrom` | OpenBao, never Git |
| Build-time constant | Docker `ARG`/`ENV` | the image |

If a value would be embarrassing in a public repo, it is a secret. If it changes
per environment, it is config — and it should be read **at runtime**, not baked in.

## Config: what the chart carries

```
config.web.app_url            → AUTH_URL, NEXT_PUBLIC_KEYCLOAK_REDIRECT_URI
config.web.keycloak_url       → KEYCLOAK_URL, NEXT_PUBLIC_KEYCLOAK_URL
config.web.keycloak_realm     → KEYCLOAK_REALM
config.web.keycloak_client_id → KEYCLOAK_CLIENT_ID
config.gateway.Gateway__BlogApiUrl → http://blogapi-service:8080
```

`config.web.app_url` being empty is a real failure: `AUTH_URL` unset plus an
empty Keycloak base produces the issuer `/realms/`, and every sign-in fails with
`Configuration`. That is why the workflow validates the variables up front.

## Secrets: the three Secret objects

| Secret | Keys | Consumer |
| --- | --- | --- |
| `web-secrets` | `AUTH_SECRET`, `REVALIDATE_SECRET`, `BLOG_SYNC_TOKEN` | `web-deployment` |
| `web-secrets` | `S3_ACCESS_KEY`, `S3_SECRET_KEY` | `web-deployment` (avatar upload) |
| `blogapi-secrets` | `ConnectionStrings__tmlabs_db`, `GitHub__SyncToken` | `blogapi-deployment` |
| `chatservice-secrets` | `ConnectionStrings__tmlabs_db` | `chatservice-deployment` |

Names must match exactly — the Deployments reference them by name in `envFrom`,
and `AppHost.cs` determines them.

`GitHub__SyncToken` is **mandatory**: `BlogApi` throws at startup when
`GitHub:SyncToken` is empty, so the pod crash-loops without it. It is a shared
secret for `POST /api/blog/sync`, not a GitHub token.

`AUTH_SECRET` rotating signs everyone out. `REVALIDATE_SECRET` rotating only
requires updating the caller.

## OpenBao layout

KV v2 under the `secret` mount:

```
secret/tmlabs/web          auth_secret, revalidate_secret
secret/tmlabs/blogapi      connection_string, github_sync_token
secret/tmlabs/chatservice  connection_string
```

Reading it requires:

1. **Policy** `tmlabs-read`:
   ```hcl
   path "secret/data/tmlabs/*" { capabilities = ["read"] }
   ```
   Note `data/` — KV v2 paths differ between the UI and the API.
2. **Kubernetes auth role** `external-secrets` bound to ESO's service account,
   with `tmlabs-read` in its token policies.

Because OpenBao runs outside the cluster here, the auth method needs:

- Kubernetes host = the API server address (e.g. `https://192.168.1.20:6443`)
- Kubernetes CA certificate = the cluster CA, pasted as text
- **"Disable use of local CA and service account JWT" checked** — otherwise it
  looks for `/var/run/secrets/...` files that don't exist
- Token reviewer JWT from a `system:auth-delegator` service account

If OpenBao later moves **into** the cluster, uncheck that box and use
`https://kubernetes.default.svc` instead.

## Why the chart's Secrets are deleted

The generated `secrets.yaml` templates render real `Secret` objects from
`values.yaml`. Committing the chart would commit either empty values (pods start
broken) or real values (leak). Deleting the templates makes ESO the single
writer, and Helm stops fighting it on every sync.

## Trap: env var names cannot contain hyphens

Kubernetes env var names must be C identifiers. The connection string is called
`tmlabs-db`, so `ConnectionStrings__tmlabs-db` **cannot** be injected — the key is
silently skipped by `envFrom`.

Aspire knows this and generates `ConnectionStrings__tmlabs_db`. If the app reads
only `ConnectionStrings:tmlabs-db`, it never sees the value. Both services accept
both names:

```csharp
var conn = builder.Configuration.GetConnectionString("tmlabs-db")
    ?? builder.Configuration.GetConnectionString("tmlabs_db")
    ?? Environment.GetEnvironmentVariable("TMLABS_DB")
    ?? builder.Configuration["tmlabs-db"];
```

## Trap: a sentinel value shadows the real one

`appsettings.json` used to contain:

```json
"ConnectionStrings": { "tmlabs-db": "<<set in secrets.json>>" }
```

`GetConnectionString("tmlabs-db")` returned that non-null string, so the `??`
chain stopped there and the real Secret was never reached. Result:

```
System.ArgumentException: Format of the initialization string does not conform
to specification starting at index 0.
```

The value in OpenBao was correct the whole time. The block has been removed, so a
missing Secret now yields a clear *not found* error instead of a parse trace.

**Lesson for the next app:** never put a placeholder where a real value is
expected. Absent is diagnosable; plausible-looking garbage is not.

## Trap: `NEXT_PUBLIC_` vs runtime

Names beginning `NEXT_PUBLIC_` are inlined into the browser bundle at build time.
After the API calls moved server-side, that was misleading — a deployment could
change the gateway URL in the ConfigMap and nothing would happen, because the
image already held the build-time value.

The frontend reads runtime names first, with the public ones as fallbacks:

```ts
const baseUrl = process.env.API_URL || process.env.NEXT_PUBLIC_API_BASE_URL;
```

`API_URL`, `KEYCLOAK_URL`, `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_ID` come from the
ConfigMap. Keep `NEXT_PUBLIC_` only for values a **client** component genuinely
needs.

Next: [08-networking-and-tls.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/08-networking-and-tls.md)
