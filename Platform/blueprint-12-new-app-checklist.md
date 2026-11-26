---
title: 12. New app checklist
description: Use this to spin up a second app from the same pattern. Ordered by dependency
series: Deployment blueprint
seriesOrder: 13
publishedAt: 2026-11-25T08:12:00Z
draft: false
---

Use this to spin up a second app from the same pattern. Ordered by dependency:
each section assumes the previous one is done.

## 1. Choose names once

Write these down before starting; they appear in five places each.

| Thing | This app | Yours |
| --- | --- | --- |
| App name / Argo Application | `tmlabs` | |
| Namespace | `tmlabs` | |
| Public hostname | `blog-test.tiyiselani.com` | |
| GitOps repo | `tmlabs-gitops` | |
| Registry namespace | `192.168.1.10:8080/tmlabs` | |
| OpenBao path prefix | `secret/tmlabs` | |

## 2. Backend

- [ ] Copy the solution; update `AppHost.cs` resources, parameters, and the
      frontend path in `AddNextJsApp`.
- [ ] Keep `AddKubernetesEnvironment("k8s")`.
- [ ] Add every env var a service needs to the AppHost — that is the only place
      the ConfigMap is derived from.
- [ ] If a config key contains a hyphen, plan for the env-var form with `_`
      ([07](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/07-configuration-and-secrets.md)).
- [ ] Remove any `appsettings.json` placeholder standing in for a real value.
- [ ] Decide config vs secret for each value; secrets never go in `values.yaml`.

## 3. Frontend

- [ ] Dockerfile with `deps`/`builder`/`runner`, `output: "standalone"`.
- [ ] Any build-time data fetch must be deferred to request time
      ([03](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/03-images-and-registry.md)).
- [ ] `generateStaticParams` must return at least one entry.
- [ ] Add `app/api/health/route.ts` — dependency-free.
- [ ] Read runtime env names (`API_URL`, `KEYCLOAK_*`) with `NEXT_PUBLIC_` only
      as fallback.
- [ ] Add `.github/workflows/notify-backend.yml` if the backend repo builds this
      image.

## 4. CI workflow

- [ ] Copy `deploy.yml`; update `REGISTRY_NS`, repo names, image list.
- [ ] Keep the **Validate deployment variables** step and extend its list.
- [ ] Keep the `Finalize chart` step, adjusting the image-name loop.
- [ ] Keep the per-run `$RUNNER_TEMP` clone path and `rm -rf chart`.
- [ ] Set variables: `APP_URL`, `API_BASE_URL`, `KEYCLOAK_*`, `GITOPS_PUSH_ENABLED`.
- [ ] Set secrets: `CROSS_REPO_TOKEN`, `GITOPS_TOKEN`.

## 5. GitOps repo

- [ ] `apps/<app>.yaml` — Helm source, path `manifests/<app>`.
- [ ] `apps/<app>-extras.yaml` — Directory source, path `manifests/<app>-extras`.
- [ ] `manifests/<app>/.gitkeep` so the path exists before the first push.
- [ ] `manifests/<app>-extras/ingress.yaml`.
- [ ] `manifests/<app>-extras/externalsecrets.yaml`.
- [ ] Register the repo credential in Argo.
- [ ] Apply `apps/`; sync **extras first**.

## 6. OpenBao + ESO

- [ ] KV v2 entry per service under `secret/<app>/`.
- [ ] Policy allowing `read` on `secret/data/<app>/*`.
- [ ] Kubernetes auth role bound to ESO's service account.
- [ ] `ClusterSecretStore` pointing at the right address/role.
- [ ] Confirm `kubectl get externalsecret` shows `SecretSynced` before deploying.

## 7. Cluster prerequisites

- [ ] Registry reachable and trusted as insecure from every node.
- [ ] Ingress controller present; `IngressClass` default or explicit.
- [ ] TLS: Cloudflare tunnel hostname (with `httpHostHeader`) or cert-manager.
- [ ] PostgreSQL reachable; user has DDL rights.

## 8. Identity

- [ ] Keycloak client with exact redirect URI and web origin.
- [ ] No Cloudflare Access on the callback path.
- [ ] `AUTH_SECRET` generated and stored in OpenBao.

## 9. First deploy, in this order

- [ ] Set the five variables; leave `GITOPS_PUSH_ENABLED` unset.
- [ ] Push; read **Finalize chart** (four pinned `_image` lines) and
      **Show generated chart** (no `secrets.yaml`, no `k8s-dashboard`).
- [ ] Set `GITOPS_PUSH_ENABLED=true`; push again.
- [ ] Argo **REFRESH** → sync `tmlabs` → watch pods.
- [ ] `curl https://<host>/api/health` → `{"status":"ok"}`.
- [ ] `curl https://<host>/api/blog/` → JSON, proving the whole chain.
- [ ] Sign in; confirm the session survives a page reload.

## 10. Carry these lessons forward

1. Never commit a placeholder where a real value is expected — absent is
   diagnosable, plausible garbage is not.
2. Names that cross a boundary (env vars, Secret keys, service DNS) must be
   verified against the consumer, not assumed.
3. On self-hosted runners, assume the workspace is dirty: unique paths, `rm -rf`
   before generating.
4. Anything generated will be overwritten — put durable changes in the generator
   input or the Finalize step.
5. `selfHeal: true` means live edits don't stick; fix the manifest.
6. Validate the cheapest step first: an ungated CI run that only prints the chart
   costs minutes and catches most mistakes before the cluster is involved.
7. Give every service a dependency-free `/api/health` so probes describe the
   process, not its dependencies.
