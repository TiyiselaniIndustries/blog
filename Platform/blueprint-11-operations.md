---
title: 11. Operations
description: /api/health is deliberately dependency-free: wiring readiness to the gateway
publishedAt: 2026-11-25T08:22:00Z
draft: false
---

## Health check

| Endpoint | Meaning | Safe for probes? |
| --- | --- | --- |
| `GET /api/health` | Next server is answering; no dependencies | yes |
| `GET /` | PPR shell — returns 200 even when the gateway is down | no |
| `GET /api/blog/` | full path: web → gateway → blogapi → postgres | no |

`/api/health` is deliberately dependency-free: wiring readiness to the gateway
would restart healthy web pods whenever a backend was down.

## Verify a deployment

```bash
kubectl -n tmlabs get deploy,po,svc,ingress
kubectl -n tmlabs get externalsecret
kubectl -n tmlabs get rs                     # rollback history, not leaks

curl -sS https://blog-test.tiyiselani.com/api/health
curl -sS https://blog-test.tiyiselani.com/api/blog/ | head -c 120
```

## Reading logs in Argo

Resource tree → Deployment → pod → **LOGS**. For a pod that restarts too fast to
attach, use **PREVIOUS LOGS**.

Same from a terminal:

```bash
kubectl -n tmlabs logs deploy/web-deployment --tail=50
kubectl -n tmlabs logs deploy/blogapi-deployment --previous
```

## Debug order (cheapest first)

1. `kubectl -n tmlabs get pods` — is everything `Running` and `READY 1/1`?
2. `kubectl -n tmlabs get externalsecret` — `SecretSynced` / `READY=True`?
3. `kubectl -n tmlabs get cm web-config -o yaml` — are the Keycloak values
   populated?
4. `kubectl -n tmlabs describe pod <pod>` — image pull errors, config errors,
   crash-loop reasons.
5. Pod logs.
6. `kubectl -n tmlabs get endpoints web-service gateway-service` — an empty
   `ENDPOINTS` list means the Service selects nothing.
7. Argo *APP CONDITIONS* — `InvalidSpecError` names a bad path or unreachable
   repo.

## Symptom table

| Symptom | Cause |
| --- | --- |
| `ErrImagePull` / `http: server gave HTTP response to HTTPS client` | registry is HTTP; nodes must allow it as insecure |
| `ImagePullBackOff` after that | tag missing, or registry auth required |
| `CreateContainerConfigError` | a referenced Secret/ConfigMap is missing |
| `CrashLoopBackOff` + `GitHub:SyncToken is required` | `GitHub__SyncToken` absent from `blogapi-secrets` |
| `CrashLoopBackOff` + `Format of the initialization string…` | a non-connection-string reached Npgsql |
| `CrashLoopBackOff` + `connection refused` on 5432 | database unreachable from the pod |
| exit code 139 | segfault (was the Aspire nightly dashboard) |
| `UntrustedHost` | Auth.js doesn't trust the Host header: set `trustHost: true` in `authConfig` (or `AUTH_TRUST_HOST=true`). Verify `AUTH_URL` matches the browser origin too. |
| Keycloak `Configuration` | empty `KEYCLOAK_URL` / realm / client id in the ConfigMap |
| `redirect_uri_mismatch` | callback not registered exactly as used |
| Argo *path does not contain a chart* | expected until the first CI chart push |

## Registry trust on k3s

Nodes must accept `192.168.1.10:8080` over HTTP. On a single-node k3s,
`containerd.service` and `k3s-agent.service` do not exist — containerd is
embedded in `k3s.service`, and `/etc/containerd/certs.d/` is **not** read.

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.1.10:8080":
    endpoint:
      - "http://192.168.1.10:8080"
```

```bash
sudo systemctl restart k3s
```

Verify with a tag that is *not* present locally — otherwise "Image is up to date"
tells you nothing:

```bash
sudo crictl -r unix:///run/k3s/containerd/containerd.sock \
  pull 192.168.1.10:8080/tmlabs/web:doesnotexist
```

`manifest unknown` = transport works. `server gave HTTP response to HTTPS client`
= config not applied.

## Rollback

Argo: *HISTORY AND ROLLBACK → select revision → Rollback*.

By hand:

```bash
kubectl -n tmlabs rollout undo deploy/blogapi-deployment
```

Because images are tagged per commit, rollback of the chart is exact. But note
`selfHeal` will revert a manual `kubectl rollout undo` on the next sync — use
Argo's rollback (which changes what Argo tracks) for anything lasting.

## Force a Secret refresh

ESO re-reads hourly. After changing a value in OpenBao:

```bash
kubectl -n tmlabs annotate externalsecret <name> force-sync="$(date +%s)" --overwrite
kubectl -n tmlabs rollout restart deployment/<name>-deployment   # env vars need a restart
```

Env vars are read once, at container start — refreshing the Secret alone changes
nothing in a running pod.

## Rotate a secret

1. Update the value in OpenBao.
2. `force-sync` the ExternalSecret (above).
3. Restart the consumer (`AUTH_SECRET` → `web-deployment`; rotating it signs
   everyone out).
4. For `REVALIDATE_SECRET`, also update whatever calls `POST /api/revalidate`.

## Revalidation after a content sync

```bash
curl -X POST https://blog-test.tiyiselani.com/api/revalidate \
  -H "x-revalidate-secret: $REVALIDATE_SECRET" \
  -d '{"id":"<optional post guid>"}'
```

Fails closed with `401` when `REVALIDATE_SECRET` is unset.

## Cleanup

```bash
kubectl -n tmlabs delete deployment k8s-dashboard-deployment   # one-off, pre-Finalize fix
rm -rf ~/actions-runner/_work/tmlabs-backend/tmlabs-backend/{gitops,chart}   # stale runner dirs
```

## Content pipeline

Markdown lives in the `blog` repo (public). A push there runs `sync-blog.yml` on a
GitHub-hosted runner — **not** the self-hosted one, because a fork could execute
code on your LAN. It authenticates to Keycloak as a service account and calls
`POST https://blog-test.tiyiselani.com/api/blog/sync`, which the Ingress routes to
the gateway ([08](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/08-networking-and-tls.md)), then
`POST /api/revalidate` on the same host.

Two auth requirements stack: the endpoint is `Roles("admin")` (so the token needs
the realm role `admin`) **and** it compares a `syncToken` in the body against
`GitHub:SyncToken`. Both must be configured.

The content-repo side — file layout rules, front matter, and how to read a failed
run — is documented in that repository:

- `blog/docs/01-authoring.md`
- `blog/docs/02-sync-pipeline.md`
- `blog/docs/03-troubleshooting.md`

Next: [12-new-app-checklist.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/12-new-app-checklist.md)
