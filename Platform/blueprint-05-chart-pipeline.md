---
title: 5. Chart pipeline
description: aspire publish turns the AppHost into a Helm chart. Its output is a starting
publishedAt: 2026-11-25T08:16:00Z
draft: false
---

`aspire publish` turns the AppHost into a Helm chart. Its output is a starting
point, not something Argo can apply — this file explains the three corrections in
between.

## Generate

```bash
cd src/TmLabs.AppHost
aspire publish -o "${GITHUB_WORKSPACE}/chart"
```

Output:

```
chart/Chart.yaml
chart/values.yaml
chart/templates/blogapi/{config,deployment,service,secrets}.yaml
chart/templates/chatservice/...
chart/templates/gateway/...
chart/templates/web/...
chart/templates/k8s-dashboard/...
```

## Parameters become config

The AppHost declares parameters; the workflow supplies them as environment
variables using double underscores:

| `AppHost.cs` | CI environment variable | Lands in |
| --- | --- | --- |
| `AddParameter("app-url", ...)` | `Parameters__app-url` | `config.web.app_url` → `AUTH_URL`, Keycloak redirect |
| `AddParameter("keycloak-url", ...)` | `Parameters__keycloak-url` | `config.web.keycloak_url` |
| `AddParameter("keycloak-realm", ...)` | `Parameters__keycloak-realm` | `config.web.keycloak_realm` |
| `AddParameter("keycloak-client-id", ...)` | `Parameters__keycloak-client-id` | `config.web.keycloak_client_id` |
| `AddParameter("auth-secret", secret: true)` | `Parameters__auth-secret` | `secrets.web.auth_secret` |
| `ConnectionStrings__tmlabs-db` | `ConnectionStrings__tmlabs-db` | `secrets.*.tmlabs_db` |

Secrets are passed as the literal string `placeholder` — the chart must never
carry a real value. They are then removed entirely (next section), so the
placeholder never ships.

## Finalize: three corrections

### 1. Delete the Secret templates

```bash
rm -f chart/templates/*/secrets.yaml
```

Why: the chart renders `Secret` objects from `values.yaml`, which is committed to
Git. With empty values, pods start without configuration; with real values, Git
holds the secrets. Neither is acceptable, so External Secrets owns them instead
— see [07](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/07-configuration-and-secrets.md).

This also prevents Helm and ESO fighting over the same Secret names.

### 2. Drop the dashboard

```bash
rm -rf chart/templates/k8s-dashboard
for svc in blogapi chatservice gateway web; do
  yq -i ".config.${svc}.OTEL_EXPORTER_OTLP_ENDPOINT = \"\"" chart/values.yaml
done
```

The dashboard image is a nightly build that segfaulted (exit 139), and it's
optional infrastructure. Blanking the OTLP endpoint stops the services retrying
against a collector that no longer exists.

### 3. Pin the image tags

```bash
export REG="${REGISTRY_HOST}/${REGISTRY_NS}"
for name in gateway blogapi chatservice web; do
  yq -i ".parameters.${name}.${name}_image = strenv(REG) + \"/${name}:\" + strenv(TAG)" chart/values.yaml
done
```

Without this, the chart says `gateway:latest` — which is not the image CI just
pushed, and `imagePullPolicy: IfNotPresent` may serve a stale local copy. The
Deployment template uses the value verbatim as `image:`, so it must be a full
reference including the registry host.

`strenv()` is what makes `yq` read `$REG` and `$TAG` from the environment instead
of treating them as literal strings.

### 4. Write the web config explicitly

```bash
yq -i '.config.web.app_url            = strenv(APP_URL)'            chart/values.yaml
yq -i '.config.web.keycloak_url       = strenv(KEYCLOAK_URL)'       chart/values.yaml
yq -i '.config.web.keycloak_realm     = strenv(KEYCLOAK_REALM)'     chart/values.yaml
yq -i '.config.web.keycloak_client_id = strenv(KEYCLOAK_CLIENT_ID)' chart/values.yaml
```

`aspire publish` receives the same values as `Parameters__*`, and in principle
that is enough. In practice a chart was once generated with all four blank, and
the deployed pods carried `AUTH_URL=` and `KEYCLOAK_*=`, producing
`UntrustedHost` at runtime — a symptom that points at auth, not at the chart.

Writing them in Finalize makes the chart deterministic: if the variables are set,
they are in the chart; if they are missing, the Validate step has already failed.
The `Parameters__*` env on the publish step is still useful — the AppHost derives
`NEXT_PUBLIC_KEYCLOAK_REDIRECT_URI` from `app-url` — but it is no longer the only
thing standing between you and an empty ConfigMap.

## Proof it worked

`Show generated chart` prints the file list and the whole `values.yaml`. Check:

- four `_image:` lines with `192.168.1.10:8080/tmlabs/<name>:<tag>`
- **no** `secrets.yaml` in the file list
- **no** `k8s-dashboard/` in the file list
- `config.web.app_url` and `config.web.keycloak_*` populated

Run this once with `GITOPS_PUSH_ENABLED` unset — it builds and prints without
touching the cluster, which is the cheapest possible validation.

## Trap: `src/TmLabs.AppHost/k8s-artifacts/`

That directory holds a committed copy of an older publish output. **Nothing reads
it.** Editing those files changes nothing about a deployment, which makes it a
convincing decoy. It is retained only as a reference snapshot and marked as such
by a README in the folder. Change `AppHost.cs` or the workflow instead.

## Trap: the chart is overwritten every run

`manifests/tmlabs` in the GitOps repo is replaced wholesale on each deploy. Never
hand-edit it — the next push erases the change. Everything durable belongs in:

- `AppHost.cs` (topology, env)
- the workflow's Finalize step (corrections)
- `manifests/tmlabs-extras` (resources Aspire doesn't generate)

Next: [06-gitops-and-argo.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/06-gitops-and-argo.md)
