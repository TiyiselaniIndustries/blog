---
title: 9. Identity and auth
description: Auth.js (NextAuth v5) on the frontend, Keycloak as the OIDC provider, JWT
series: Deployment blueprint
seriesOrder: 10
publishedAt: 2026-11-25T08:09:00Z
draft: false
---

Auth.js (NextAuth v5) on the frontend, Keycloak as the OIDC provider, JWT
validation in the gateway.

## Flow

```
1. browser   GET  /login
2. web       redirect to Keycloak  /realms/non-prod/protocol/openid-connect/auth
                                     ?redirect_uri=https://blog-test…/api/auth/callback/keycloak
3. Keycloak  user signs in
4. browser   GET  /api/auth/callback/keycloak?code=…
5. web       exchanges code for tokens (server-side)
             stores them in an encrypted HttpOnly cookie (AUTH_SECRET)
6. browser   requests pages; the Next server decrypts the cookie
7. web       calls gateway with Authorization: Bearer <access_token>
8. gateway   validates the JWT against Keycloak's JWKS, applies policies
```

Tokens never reach browser JavaScript — they live inside the encrypted cookie and
are read server-side only (`lib/auth/tokens.ts`).

## Keycloak client

Realm `non-prod`, client `tmlabs-frontend`, **public** client with PKCE.

| Setting | Value |
| --- | --- |
| Valid redirect URIs | `https://blog-test.tiyiselani.com/api/auth/callback/keycloak` |
| | `http://localhost:3000/api/auth/callback/keycloak` (local dev) |
| Web origins | `https://blog-test.tiyiselani.com`, `http://localhost:3000` |

`KEYCLOAK_CLIENT_SECRET` is only needed for a confidential client; leave it empty
for public + PKCE.

The gateway validates the same issuer with audience `tmlabs-api`.

## Required config

| Variable | Source | Failure if missing |
| --- | --- | --- |
| `AUTH_SECRET` | `web-secrets` | `MissingSecret` on every request |
| `AUTH_URL` | `config.web.app_url` | `UntrustedHost`, wrong callback |
| `KEYCLOAK_URL` | `config.web.keycloak_url` | issuer becomes `/realms/…` |
| `KEYCLOAK_REALM` | `config.web.keycloak_realm` | same |
| `KEYCLOAK_CLIENT_ID` | `config.web.keycloak_client_id` | `Configuration` |
| `AUTH_TRUST_HOST` | chart, `"true"` | `UntrustedHost` behind a proxy |

## Trap: an empty URL produces a valid-looking but wrong issuer

```ts
const issuer = `${base}/realms/${realm}`;
```

With `base` empty you get `/realms/non-prod`, a relative URL Auth.js cannot
resolve. The error is `Configuration`, which reads like "Keycloak is down" but
means "we never told it where Keycloak is".

Check what the pod actually has:

```bash
kubectl -n tmlabs get cm web-config -o yaml | grep -i keycloak
```

Empty values here mean the GitHub variables weren't set when the chart was
generated.

## Trap: trailing slash

`NEXT_PUBLIC_KEYCLOAK_URL=https://auth.tiyiselani.com/` yields
`https://auth.tiyiselani.com//realms/…`, which Keycloak rejects with HTTP 400,
surfaced by Auth.js as:

```
OperationProcessingError: "response" is not a conform Authorization Server Metadata response
```

The code strips trailing slashes defensively, but store the value without one.

## Trap: `redirect_uri_mismatch`

Keycloak compares the callback character for character. Common mismatches:

- `http` vs `https`
- a trailing slash on the redirect URI
- `localhost` vs `127.0.0.1`
- the value in `AUTH_URL` not matching the hostname in the browser bar

## Trap: `UntrustedHost` behind a proxy

```
[auth][error] UntrustedHost: Host must be trusted.
URL was: http://blog-test.tiyiselani.com/api/auth/session
URL was: http://0.0.0.0:3000/api/auth/signin/keycloak
```

Two different hosts, one cause. Auth.js refuses to derive its own origin from an
untrusted `Host` header — and behind Traefik + Cloudflare the header is rewritten,
while Next's internal self-requests use `0.0.0.0:3000`.

Fix it in code, not just env:

```ts
export const authConfig = {
  trustHost: true,   // equivalent to AUTH_TRUST_HOST=true
  ...
}
```

`AUTH_TRUST_HOST=true` from the ConfigMap is the documented alternative, but it
only works if the value actually reaches the container. Declaring it in
`authConfig` removes that dependency — and it is safe here because the Ingress
matches a single hostname.

Related detail: the request arrives at the pod as `http://`, because Cloudflare
terminates TLS. `AUTH_URL` (https) still governs generated redirect URIs, so this
is only a concern if you key logic off the request protocol.

## Trap: Cloudflare Access in front of the callback

If `blog-test.tiyiselani.com` sits behind a Zero Trust application, the callback
arrives at an Access login prompt instead of the Next route handler. Leave Access
off for this hostname, or exclude `/api/auth/*`.

Next: [10-database-and-migrations.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/10-database-and-migrations.md)
