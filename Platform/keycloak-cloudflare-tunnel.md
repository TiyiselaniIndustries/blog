---
title: Exposing Keycloak (Proxmox LXC) Through Cloudflare Tunnel
description: This guide assumes cloudflared is already installed and authenticated to your Cloudflare account, and that Keycloak is running in a Proxmox LXC on you...
series: Identity and access
seriesOrder: 5
publishedAt: 2026-11-25T08:19:00Z
draft: false
---

This guide assumes `cloudflared` is already installed and authenticated to your Cloudflare account, and that Keycloak is running in a Proxmox LXC on your local network (for example `192.168.1.17:8080`, as used elsewhere in the TmLabs auth guide). It walks through pointing the tunnel at Keycloak, and — the part that actually matters for passkeys and Google/GitHub login — telling Keycloak it's sitting behind an HTTPS proxy.

If Keycloak isn't proxy-aware, it keeps generating internal `http://192.168.1.17:8080/...` URLs in its discovery document and redirects, and the WebAuthn Relying Party ID won't match the hostname the browser sees. That silently breaks passkey registration and makes Google reject the OAuth redirect. Section 2 below is what fixes that.

## 1. Point cloudflared at the Keycloak LXC

Find the LXC's internal IP and port — `192.168.1.17:8080` in this example.

**Option A: config file.** In `~/.cloudflared/config.yml` (or wherever your tunnel's config lives), add an ingress rule:

```yaml
tunnel: <your-tunnel-id>
credentials-file: /root/.cloudflared/<your-tunnel-id>.json

ingress:
  - hostname: auth.example.com
    service: http://192.168.1.17:8080
  - service: http_status:404
```

The catch-all `http_status:404` rule must be last; `cloudflared` requires it.

**Option B: Zero Trust dashboard.** Go to **Zero Trust > Networks > Tunnels > [your tunnel] > Public Hostname > Add a public hostname**:

```text
Subdomain: auth
Domain:    example.com
Service:   HTTP
URL:       192.168.1.17:8080
```

Then create the DNS record (skip this if you used the dashboard — it creates the CNAME automatically):

```bash
cloudflared tunnel route dns <tunnel-name-or-id> auth.example.com
```

Restart or reload `cloudflared` if you edited the config file directly:

```bash
sudo systemctl restart cloudflared
```

At this point `https://auth.example.com` reaches Keycloak with TLS terminated at Cloudflare's edge. Keycloak itself keeps running plain HTTP internally, which is fine — the tunnel is the trusted boundary.

## 2. Tell Keycloak it's behind a proxy

Set these on the Keycloak LXC. Depending on how the helper script installed Keycloak, this is either environment variables in a systemd unit / `.env` file, or startup flags. The setting names below are current for Keycloak's Quarkus-based distribution (`kc.sh`):

```text
KC_HOSTNAME=auth.example.com
KC_HOSTNAME_STRICT=true
KC_PROXY_HEADERS=xforwarded
KC_HTTP_ENABLED=true
```

What each one does:

- `KC_HOSTNAME` — the public hostname Keycloak should advertise in every URL it generates (discovery document, redirects, WebAuthn relying party).
- `KC_HOSTNAME_STRICT=true` — stops Keycloak from falling back to whatever `Host` header or IP the request arrived on. Without this, some internal links can still leak the LXC IP.
- `KC_PROXY_HEADERS=xforwarded` — makes Keycloak trust the `X-Forwarded-Proto` / `X-Forwarded-Host` headers cloudflared adds, so it knows the real request was `https` even though the connection it received locally is plain `http`. This is the setting that most often gets skipped.
- `KC_HTTP_ENABLED=true` — needed because Keycloak is still serving plain HTTP internally to the tunnel; only the public edge is HTTPS.

If you're running an older Keycloak version (pre-Quarkus, `standalone.sh`), the equivalent is a proxy-address-forwarding valve in `standalone.xml` plus `--hostname` server args — check your version with `kc.sh --version` or `standalone.sh --version` first, since the helper script's Keycloak build determines which config style applies.

Locate where the helper script defined these (commonly a systemd drop-in or `/etc/default/keycloak`-style env file inside the LXC), add or update the four variables, then restart Keycloak:

```bash
systemctl restart keycloak
```

## 3. Verify before changing anything else

```bash
curl https://auth.example.com/realms/non-prod/.well-known/openid-configuration
```

Every URL in the returned JSON — `issuer`, `authorization_endpoint`, `token_endpoint`, `jwks_uri` — must start with `https://auth.example.com`. If any still show the LXC IP or `http://`, the proxy headers aren't being honored yet:

- Confirm `KC_PROXY_HEADERS=xforwarded` was actually applied (check `systemctl show keycloak` or the process environment, not just the file — a stale unit can still be running).
- Confirm cloudflared isn't stripping the `X-Forwarded-*` headers. It doesn't by default, but a custom `originRequest` block in `config.yml` can override this — if you have one, make sure it doesn't set `noTLSVerify` in a way that also disables header forwarding, and that no `httpHostHeader` override is rewriting things unexpectedly.
- Restart Keycloak again after any change — hostname settings are read at boot, not hot-reloaded.

Don't move on to GitHub/Google/passkey config until this discovery document is fully correct. Everything below depends on it.

## 4. Update client and provider configuration to the new hostname

Replace every place that referenced the LXC IP or `localhost` for Keycloak:

**Your services' `Authority` setting** (gateway, Blog API, Chat Service):

```text
Authentication__Authority=https://auth.example.com/realms/non-prod
```

**`tmlabs-web` client** in Keycloak — redirect URIs and web origins move to your real frontend/Scalar origins, still `https://`:

```text
Valid redirect URIs: https://yourapp.com/*, https://yourapp.com/api/auth/callback/keycloak
Web origins: https://yourapp.com
```

Keep `http://localhost:3000/*` etc. alongside these if you still develop locally against the same realm — Keycloak allows multiple redirect URIs on one client.

**GitHub OAuth app callback:**

```text
https://auth.example.com/realms/non-prod/broker/github/endpoint
```

Update this in the GitHub OAuth app settings, not just in Keycloak — the two must match exactly.

**Google OAuth authorized redirect URI:**

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

This also fixes a problem you'd otherwise hit: Google generally rejects raw IP hosts for web OAuth clients, so moving to the DNS name is required, not just cleaner.

**WebAuthn Relying Party ID** (Authentication > Policies > WebAuthn Passwordless Policy):

```text
Relying Party ID: auth.example.com
```

The Relying Party ID must be a hostname the browser can see in its address bar during the passkey ceremony. It cannot be an IP address, and it must match `KC_HOSTNAME` above — passkeys are bound to the origin they were registered on, so if this drifts later, previously registered passkeys stop validating.

## 5. Don't put Cloudflare Access in front of Keycloak's auth paths

If this tunnel — or a helper script — also has **Cloudflare Access** (Zero Trust authentication) enabled on `auth.example.com`, do not apply it to Keycloak's login, broker, or account endpoints. That puts an authentication wall in front of your authentication server: GitHub and Google's redirect back to `/realms/non-prod/broker/.../endpoint` would hit Access's login prompt instead of reaching Keycloak, breaking every social login.

If you want Access protection on anything, scope the Access policy narrowly to the admin console only:

```text
Path: /admin/*
```

Leave `/realms/*` (login, broker callbacks, token endpoint, account console, well-known discovery) untouched by Access policies.

## 6. Test sequence

1. `curl https://auth.example.com/realms/non-prod/.well-known/openid-configuration` — confirm every URL uses the public hostname and `https`.
2. Open `https://auth.example.com/realms/non-prod/account` in a browser — confirm the Keycloak login page loads over a valid TLS connection (Cloudflare's certificate, no browser warning).
3. Start a login through your frontend or Scalar and confirm the redirect back from Keycloak lands on `https://auth.example.com/...`, never the LXC IP.
4. Test GitHub sign-in end to end; confirm no `redirect_uri_mismatch`.
5. Test Google sign-in end to end; confirm no `redirect_uri_mismatch` or `access_blocked`.
6. Register a passkey from the Account Console and confirm it completes without a WebAuthn origin error, then log out and log back in with it.
7. Decode a fresh access token and confirm `iss` reads `https://auth.example.com/realms/non-prod`.
8. If Access is enabled on the tunnel, confirm you can still reach `/realms/non-prod/broker/github/endpoint` and `/realms/non-prod/broker/google/endpoint` without an Access login prompt.

### Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Discovery document still shows the LXC IP or `http://` | `KC_PROXY_HEADERS` not set or Keycloak not restarted after the change |
| Google: `redirect_uri_mismatch` | Google credential still has the old IP-based callback, or trailing slash differs from Keycloak's displayed redirect URI |
| Passkey registration fails with an origin error | WebAuthn Relying Party ID doesn't match `KC_HOSTNAME`, or it's set to an IP |
| GitHub/Google login redirects to an Access login page instead of Keycloak | Cloudflare Access policy applied to `/realms/*` — scope it to `/admin/*` only |
| `401` from your gateway after switching hostnames | `Authentication:Authority` in the gateway/services still points at the old IP or `localhost` |
| Browser TLS warning on `auth.example.com` | DNS record not proxied through Cloudflare (grey-clouded instead of orange-clouded), or propagation still in progress |
| Login works but previously-registered passkeys no longer validate | Relying Party ID changed after passkeys were registered on the old hostname — users must re-register |
