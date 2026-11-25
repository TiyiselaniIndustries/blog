---
title: Deploying TmLabs to Kubernetes with Argo CD
description: End-to-end reference for the test environment: what CI produces, what Argo CD
publishedAt: 2026-11-25T08:10:00Z
draft: false
---

End-to-end reference for the test environment: what CI produces, what Argo CD
applies, and what has to exist in the cluster and in Keycloak before the first
sync succeeds.

## How a deploy flows

```
push to tmlabs-backend main
  └─ deploy-test workflow (.github/workflows/deploy.yml)
       ├─ dotnet publish -t:PublishContainer  → Zot: gateway / blogapi / chatservice
       ├─ docker build                        → Zot: web
       ├─ aspire publish -o chart             → Helm chart
       ├─ finalize chart                      → strip Secrets, pin image tags
       └─ commit chart → tmlabs-gitops/manifests/tmlabs   (gated by GITOPS_PUSH_ENABLED)
                                                             │
                                      Argo CD syncs ─────────┘
                                        ├─ tmlabs        → manifests/tmlabs (Helm)
                                        └─ tmlabs-extras → manifests/tmlabs-extras
```

The frontend image is built **by the backend workflow** from a checkout of
`tmlabs-frontend`. A push to the frontend repo triggers
`tmlabs-frontend/.github/workflows/notify-backend.yml`, which sends the
`frontend-changed` repository dispatch.

## Images

| Project | Image |
| --- | --- |
| `TmLabs.Gateway` | `192.168.1.10:8080/tmlabs/gateway:<TAG>` |
| `TmLabs.BlogApi` | `192.168.1.10:8080/tmlabs/blogapi:<TAG>` |
| `TmLabs.ChatService` | `192.168.1.10:8080/tmlabs/chatservice:<TAG>` |
| `tmlabs-frontend` | `192.168.1.10:8080/tmlabs/web:<TAG>` |

`<TAG>` is `<short-sha>-<run-number>`. The workflow writes the four values into
`chart/values.yaml` (`parameters.<name>.<name>_image`) before pushing, and the
Deployment templates use those values verbatim as `image:`.

## GitHub configuration

Repository **variables** (`Settings → Secrets and variables → Actions → Variables`)
on `TiyiselaniIndustries/tmlabs-backend`:

| Variable | Value | Notes |
| --- | --- | --- |
| `APP_URL` | `https://blog-test.tiyiselani.com` | Becomes `AUTH_URL` and the Keycloak redirect URI |
| `API_BASE_URL` | `http://gateway-service:8080` | In-cluster gateway. Baked into the image as a fallback only |
| `KEYCLOAK_URL` | `https://auth.tiyiselani.com` | No trailing slash |
| `KEYCLOAK_REALM` | `non-prod` | |
| `KEYCLOAK_CLIENT_ID` | `tmlabs-frontend` | Public client |
| `GITOPS_PUSH_ENABLED` | `true` | Only after the checklist below |

`API_BASE_URL` is **not** a public URL. The gateway Service stays `ClusterIP`; the
Next server calls `http://gateway-service:8080`, and nothing in the browser bundle
reads it. The only public gateway path is `/api/blog`, routed from the frontend
host so the content repo's workflow can call `POST /api/blog/sync` — the endpoint
requires an `admin` token and a sync token, so publishing the path does not
publish the operation.

Repository **secrets**:

| Secret | Where | Purpose |
| --- | --- | --- |
| `CROSS_REPO_TOKEN` | tmlabs-backend | Check out `tmlabs-frontend` (contents: read) |
| `GITOPS_TOKEN` | tmlabs-backend | Push the chart to `tmlabs-gitops` (contents: read and write) |
| `BACKEND_DISPATCH_TOKEN` | tmlabs-frontend | Send `frontend-changed` (contents: read and write) |

`BACKEND_DISPATCH_TOKEN` is optional. Without it the dispatch step logs a warning
and skips, so frontend changes only deploy via a backend push or a manual run.

### Setting them

UI: repo → **Settings → Secrets and variables → Actions**, then the **Variables**
tab (for `APP_URL` etc.) or the **Secrets** tab (for tokens).

With the GitHub CLI (`gh auth login` first):

```bash
# tmlabs-backend — variables
gh variable set APP_URL            -b "https://blog-test.tiyiselani.com" -R TiyiselaniIndustries/tmlabs-backend
gh variable set API_BASE_URL       -b "http://gateway-service:8080"      -R TiyiselaniIndustries/tmlabs-backend
gh variable set KEYCLOAK_URL       -b "https://auth.tiyiselani.com"      -R TiyiselaniIndustries/tmlabs-backend
gh variable set KEYCLOAK_REALM     -b "non-prod"                         -R TiyiselaniIndustries/tmlabs-backend
gh variable set KEYCLOAK_CLIENT_ID -b "tmlabs-frontend"                  -R TiyiselaniIndustries/tmlabs-backend
# only once the gated run has been verified:
# gh variable set GITOPS_PUSH_ENABLED -b "true" -R TiyiselaniIndustries/tmlabs-backend

# tmlabs-backend — secrets
gh secret set CROSS_REPO_TOKEN -R TiyiselaniIndustries/tmlabs-backend
gh secret set GITOPS_TOKEN     -R TiyiselaniIndustries/tmlabs-backend

# tmlabs-frontend — secret
gh secret set BACKEND_DISPATCH_TOKEN -R TiyiselaniIndustries/tmlabs-frontend
```

Verify:

```bash
gh variable list -R TiyiselaniIndustries/tmlabs-backend
gh secret list   -R TiyiselaniIndustries/tmlabs-backend
```

## Cluster prerequisites

1. **Argo CD** installed, Applications applied from `tmlabs-gitops/apps/`.
2. **Registry access** — `192.168.1.10:8080` is plain HTTP, so containerd/kubelet
   must allow it as an insecure registry on every node:

   ```toml
   # /etc/rancher/k3s/registries.yaml (k3s) or /etc/containerd/config.toml
   mirrors:
     "192.168.1.10:8080":
       endpoint:
         - "http://192.168.1.10:8080"
   ```

   The chart sets no `imagePullSecrets`. Either keep the registry anonymous or add
   a pull secret and reference it in the Deployments.
3. **External Secrets Operator** with a `ClusterSecretStore` (the manifests assume
   `openbao`). Alternatively pre-create the three Secrets by hand — see below.
4. **Ingress controller** and, if you want HTTPS, cert-manager or an external TLS
   terminator.
5. **PostgreSQL** reachable from the cluster. The chart does not provision a
   database.

### Secrets the chart no longer creates

`Finalize chart` deletes `chart/templates/*/secrets.yaml`, so Helm never writes
Secret objects. The ExternalSecrets in
`tmlabs-gitops/manifests/tmlabs-extras/externalsecrets.yaml` own them instead.
Names and keys must match exactly or the pods start without configuration:

| Secret | Keys | Required because |
| --- | --- | --- |
| `web-secrets` | `AUTH_SECRET`, `REVALIDATE_SECRET` | Session cookie encryption; the revalidate endpoint fails closed without the second |
| `blogapi-secrets` | `ConnectionStrings__tmlabs_db`, `GitHub__SyncToken` | **BlogApi throws at startup** without `GitHub:SyncToken` |
| `chatservice-secrets` | `ConnectionStrings__tmlabs_db` | ChatService throws at startup without a connection string |

Optional: `GitHub__Token` in `blogapi-secrets` raises the GitHub API rate limit
and allows syncing a private repo. It has a commented-out entry in the manifest.

No ESO? Create them manually instead:

```bash
kubectl -n tmlabs create secret generic web-secrets \
  --from-literal=AUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=REVALIDATE_SECRET="$(openssl rand -base64 32)"

kubectl -n tmlabs create secret generic blogapi-secrets \
  --from-literal=GitHub__SyncToken="$(openssl rand -base64 32)" \
  --from-literal=ConnectionStrings__tmlabs_db='Host=...;Database=tmlabs;Username=...;Password=...'

kubectl -n tmlabs create secret generic chatservice-secrets \
  --from-literal=ConnectionStrings__tmlabs_db='Host=...;Database=tmlabs;Username=...;Password=...'
```

### OpenBao layout

Create these KV v2 entries under the `secret` mount (values are yours; generate
the random ones):

```bash
# web
bao kv put secret/tmlabs/web \
  auth_secret="$(openssl rand -base64 32)" \
  revalidate_secret="$(openssl rand -base64 32)"

# blog api — github_sync_token is mandatory
bao kv put secret/tmlabs/blogapi \
  connection_string='Host=<host>;Port=5432;Database=tmlabs;Username=<user>;Password=<pass>' \
  github_sync_token="$(openssl rand -base64 32)"

# chat service
bao kv put secret/tmlabs/chatservice \
  connection_string='Host=<host>;Port=5432;Database=tmlabs;Username=<user>;Password=<pass>'
```

Optional, only if the blog repo is private or you hit rate limits:

```bash
bao kv patch secret/tmlabs/blogapi github_token="<github-pat>"
```

Then uncomment the `GitHub__Token` entry in `externalsecrets.yaml`.

The ExternalSecrets assume a `ClusterSecretStore` named `openbao`. If you do not
have one:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: openbao
spec:
  provider:
    vault:
      server: https://bao.tiyiselani.com   # your OpenBao address
      path: secret                          # KV v2 mount
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

Adjust the auth block to whatever your OpenBao uses (AppRole, or a static token
in a `Secret`, also work). Rename the store or update `secretStoreRef.name` if you
already have one.

## Configuration the chart carries

`aspire publish` writes the non-secret values from the workflow's
`Parameters__*` environment variables, so these land in the committed chart:

- `config.web.app_url` → `AUTH_URL` and `NEXT_PUBLIC_KEYCLOAK_REDIRECT_URI`
- `config.web.keycloak_url` / `keycloak_realm` / `keycloak_client_id`
- `config.web.API_URL` and `NEXT_PUBLIC_API_BASE_URL` → `http://gateway-service:8080`
- `config.gateway.Gateway__BlogApiUrl` → `http://blogapi-service:8080`
- `config.gateway.Gateway__ChatApiUrl` → `http://chatservice-service:8080`

Because the workflow rewrites the chart each run, change the GitHub variables and
re-run; do not hand-edit `manifests/tmlabs`.

### Build-time vs runtime variables

The frontend reads `API_URL`, `KEYCLOAK_URL`, `KEYCLOAK_REALM` and
`KEYCLOAK_CLIENT_ID` at request time (the chart supplies them). The
`NEXT_PUBLIC_*` copies are inlined into the image at build time and only act as
fallbacks, so a realm or URL change takes effect without rebuilding — but a
rebuild is still required if you ever read a `NEXT_PUBLIC_*` value from a client
component.

## Keycloak

Register on the `tmlabs-frontend` client in realm `non-prod`:

- Valid redirect URI: `https://blog-test.tiyiselani.com/api/auth/callback/keycloak`
- Web origin: `https://blog-test.tiyiselani.com`

Do not put Cloudflare Access in front of `/realms/*` or the callback path — it
breaks both social login and the Auth.js callback.

## Ingress and DNS

`manifests/tmlabs-extras/ingress.yaml` routes `blog-test.tiyiselani.com` to
`web-service:3000`.

It sets no `ingressClassName`, so the cluster's default IngressClass serves it. If
your cluster has no default, add `spec.ingressClassName: traefik` (k3s) or
`nginx` (ingress-nginx). Uncomment the `tls:` block once an issuer is configured.

DNS: point `blog-test.tiyiselani.com` at the ingress controller or tunnel.

Health endpoints: `/api/health` (dependency-free, safe for probes), `/` (PPR
shell — returns 200 even when the gateway is down, so don't use it for readiness).

## Database migrations

Services do not create tables at startup. Run EF migrations once the database
exists and before the API pods serve traffic:

```bash
dotnet ef database update --project src/TmLabs.BlogApi    --startup-project src/TmLabs.BlogApi
dotnet ef database update --project src/TmLabs.ChatService --startup-project src/TmLabs.ChatService
```

## Argo CD bootstrap

Run these from a machine with `kubectl` pointed at the cluster (not from the
runner). `argocd` CLI is optional.

### 1. Check Argo is installed

```bash
kubectl get namespace argocd
kubectl -n argocd get pods
```

If the namespace is missing, install it first:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Give Argo read access to tmlabs-gitops

Only needed while the repo is private. Create a fine-grained PAT with
**Contents: read** on `tmlabs-gitops`, then:

```bash
kubectl -n argocd create secret generic tmlabs-gitops-repo \
  --from-literal=type=git \
  --from-literal=url=https://github.com/TiyiselaniIndustries/tmlabs-gitops.git \
  --from-literal=username=x-access-token \
  --from-literal=password=<THE_PAT>

kubectl -n argocd label secret tmlabs-gitops-repo \
  argocd.argoproj.io/secret-type=repository
```

### 3. Apply the Applications

```bash
cd ../tmlabs-gitops
kubectl apply -f apps/
kubectl -n argocd get applications
```

### 4. Sync extras first, then the app

```bash
# argocd CLI alternative to the UI buttons:
argocd app sync tmlabs-extras
argocd app sync tmlabs
```

Until the first CI push populates `manifests/tmlabs`, the `tmlabs` app reports
*path does not contain a chart* — expected, and it clears on the next sync after
the chart is committed.

### 5. Turn on auto-sync for future commits

Both Applications already carry `syncPolicy.automated` with `prune` and
`selfHeal`, so once applied, later commits to `manifests/tmlabs` roll out without
further `kubectl` commands.

## First-deploy checklist

1. `tmlabs-gitops` has a `main` branch (the repo was empty when this was written;
   `apps/`, `manifests/tmlabs/.gitkeep`, and `manifests/tmlabs-extras/` are the
   first commit).
2. Argo CD can read `tmlabs-gitops` (repository credential configured).
3. `ClusterSecretStore` (or manual Secrets) ready in namespace `tmlabs`.
4. Registry reachable and allowed as insecure from the nodes.
5. GitHub variables and secrets set per the tables above.
6. `kubectl apply -f tmlabs-gitops/apps/`, then sync `tmlabs-extras` first.
7. Run the backend workflow with `GITOPS_PUSH_ENABLED` unset, and inspect the
   `Finalize chart` output — the four `_image:` lines must show
   `192.168.1.10:8080/tmlabs/<name>:<tag>`, and the `Show generated chart` list
   must contain no `secrets.yaml`.
8. Set `GITOPS_PUSH_ENABLED=true` and re-run; confirm the commit appears in
   `tmlabs-gitops`.
9. Watch `kubectl -n tmlabs get pods` and the Argo CD UI.

## Verifying a deploy

```bash
kubectl -n tmlabs get deploy,po,svc,ingress
kubectl -n tmlabs get externalsecret
kubectl -n tmlabs logs deploy/web-deployment --tail=50

curl -sS https://blog-test.tiyiselani.com/api/health
curl -sS -o /dev/null -w '%{http_code}\n' https://blog-test.tiyiselani.com/
```

Revalidation after a content sync:

```bash
curl -X POST https://blog-test.tiyiselani.com/api/revalidate \
  -H "x-revalidate-secret: $REVALIDATE_SECRET" \
  -d '{"id":"<optional post guid>"}'
```

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `ASPIRE009: bundle could not be resolved` | Aspire CLI bundle not set up on the runner. The workflow runs `aspire setup` before publishing. |
| `/usr/local/bin/aspire: Permission denied` | System-wide install not executable by the runner account. The workflow installs into `$RUNNER_TEMP` instead. |
| `empty-generate-static-params` during `npm run build` | Blog routes fetch at build time. They now defer to request time; don't reintroduce a build-time `getBlogList()` in `generateStaticParams`. |
| Pods `CreateContainerConfigError` | `web-secrets` / `blogapi-secrets` / `chatservice-secrets` missing or wrong keys. |
| `blogapi` in `CrashLoopBackOff` with `GitHub:SyncToken is required` | The `GitHub__SyncToken` key is missing from `blogapi-secrets`. |
| `Format of the initialization string does not conform to specification starting at index 0` | A non-connection-string value reached Npgsql. Check `kubectl -n tmlabs get secret <svc>-secrets -o jsonpath='{.data.ConnectionStrings__tmlabs_db}' \| base64 -d \| cut -c1-30` — it must start with `Host=` (not a `postgres://` URL). The sentinel `<<set in secrets.json>>` was removed from `appsettings.json` so this now fails with a clear "not found" error instead. |
| `ErrImagePull` / `ImagePullBackOff` | Registry not allowed as insecure, or pull auth missing. |
| `UntrustedHost` in web logs | `AUTH_URL` doesn't match the public origin. Check `APP_URL` and the Ingress host. |
| Keycloak `Configuration` error on sign-in | Empty `KEYCLOAK_URL` / realm / client id in the chart. Check the variables and re-run. |
| `redirect_uri_mismatch` | Callback not registered, or `APP_URL` differs from the browser origin. |
| Argo `path does not contain a chart` | Expected until the first CI push populates `manifests/tmlabs`. |
| Workflow push step "succeeds" without a commit | Fixed: the step now exits cleanly only when there is genuinely no diff. |

## Known limitations

- The chart deploys the Aspire dashboard from
  `mcr.microsoft.com/aspire/nightly/dashboard:17.0` — a nightly image. Decide
  whether it belongs in the cluster or should be removed from the AppHost.
- `src/TmLabs.AppHost/k8s-artifacts/` is a stale reference snapshot; see the
  README in that folder.
- Single replica per workload, `imagePullPolicy: IfNotPresent`, no resource
  requests/limits, no HPA — adequate for the test environment only.
