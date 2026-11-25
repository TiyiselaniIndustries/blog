---
title: 8. Networking and TLS
description: No ingressClassName — the clusters default (traefik on k3s) serves it. Add
publishedAt: 2026-11-25T08:19:00Z
draft: false
---

One public hostname, two paths, one private service mesh.

## The Ingress

`manifests/tmlabs-extras/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tmlabs-web
  namespace: tmlabs
spec:
  rules:
    - host: blog-test.tiyiselani.com
      http:
        paths:
          - path: /api/blog
            pathType: Prefix
            backend:
              service: { name: gateway-service, port: { number: 8080 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: web-service, port: { number: 3000 } }
```

No `ingressClassName` — the cluster's default (`traefik` on k3s) serves it. Add
one explicitly if your cluster has no default.

Longest prefix wins, so `/api/blog/*` reaches the gateway and everything else
reaches the frontend.

### Why `/api/blog` is public

The blog content repository is **public**, so its workflow cannot run on the
self-hosted runner — a fork could execute arbitrary code on your LAN. It runs on
GitHub-hosted infrastructure instead, which means it needs an internet route to
`POST /api/blog/sync`.

Exposing that path does not expose the operation: the endpoint requires a Keycloak
token with the realm role `admin` (audience `tmlabs-api`) **and** a matching
`syncToken` in the body. The GET endpoints under the same prefix were already
public content.

What remains private: `/api/chat`, and the gateway itself as a Service. The Next
server still reaches it in-cluster via `API_URL=http://gateway-service:8080`.

## Verify

```bash
kubectl -n tmlabs get ingress tmlabs-web        # ADDRESS should be filled in
kubectl get ingressclass                        # traefik (default)
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Host: blog-test.tiyiselani.com' http://192.168.1.9/
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Host: blog-test.tiyiselani.com' http://192.168.1.9/api/blog/health
```

`200` on the first proves Ingress → Service → web pod. `200` on the second proves
the `/api/blog` path reaches the gateway before any DNS exists. An unauthenticated
`POST /api/blog/sync` should return `401` — that is the correct answer.


## TLS: two options

### Cloudflare Tunnel (used here)

TLS terminates at Cloudflare's edge; the origin stays plain HTTP. No
cert-manager, no certificate objects.

Zero Trust → Networks → Tunnels → your tunnel → **Public Hostname → Add**:

| Field | Value |
| --- | --- |
| Subdomain | `blog-test` |
| Domain | `tiyiselani.com` |
| Type | `HTTP` |
| URL | `192.168.1.9:80` |
| HTTP Host Header *(under Additional application settings)* | `blog-test.tiyiselani.com` |

YAML equivalent:

```yaml
ingress:
  - hostname: blog-test.tiyiselani.com
    service: http://192.168.1.9:80
    originRequest:
      httpHostHeader: blog-test.tiyiselani.com
  - service: http_status:404
```

`httpHostHeader` is mandatory. Without it Cloudflare sends `Host: localhost`,
Traefik matches no rule, and you get `404 page not found`. The
`http_status:404` catch-all must stay last or `cloudflared` won't start.

**Do not put Cloudflare Access in front of this hostname.** It would intercept
`/api/auth/callback/keycloak` and break sign-in — the same reason the Keycloak
guide says not to put Access on `/realms/*`.

### cert-manager

If you'd rather terminate at the cluster, add the `tls:` block to the Ingress and
a `cert-manager.io/cluster-issuer` annotation, then install cert-manager. More
moving parts for the same result; the tunnel avoids the certificate lifecycle
entirely.

## Trap: HTTPS matters to Auth.js

The Ingress serves port 80 only. `AUTH_URL` is `https://blog-test.tiyiselani.com`,
so:

- Through Cloudflare — fine, the browser sees HTTPS.
- Over plain `http://192.168.1.9` — the homepage works, but sign-in redirects to
  `https://…` which nothing serves locally.

`AUTH_URL` must match the origin in the browser bar exactly. A mismatch surfaces
as `UntrustedHost` or `redirect_uri_mismatch`, which look like Keycloak problems
but are not.

## Trap: `selfHeal` reverts manual routing fixes

Editing the Service or Ingress with `kubectl edit` is undone at the next sync.
Change the manifest in `manifests/tmlabs-extras` and push.

Next: [09-identity-and-auth.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/09-identity-and-auth.md)
