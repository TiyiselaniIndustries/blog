---
title: 1. Architecture
description: The browser never talks to the gateway. All API traffic is server-side
series: Deployment blueprint
seriesOrder: 2
publishedAt: 2026-11-25T08:01:00Z
draft: false
---

## What runs

| Resource | Kind | Port | Reachable from |
| --- | --- | --- | --- |
| `web` | Next.js 16 (standalone) | 3000 | internet, via Ingress |
| `gateway` | YARP reverse proxy | 8080 | cluster only |
| `blogapi` | FastEndpoints + EF Core | 8080 | cluster only |
| `chatservice` | FastEndpoints + EF Core | 8080 | cluster only |
| PostgreSQL | external | 5432 | cluster only |

## Request path

```
browser ──HTTPS──▶ Cloudflare ──HTTP──▶ Traefik ──▶ web-service:3000
                                                      │  Next server (RSC + route handlers)
                                                      │  fetches http://gateway-service:8080
                                                      ▼
                                                 gateway-service:8080
                                                   ├─ /api/blog/* → blogapi-service:8080
                                                   └─ /api/chat/* → chatservice-service:8080
                                                        └─ PostgreSQL
```

Two consequences worth internalising:

- **The browser never talks to the gateway.** All API traffic is server-side,
  which is why the gateway can stay private and why no `NEXT_PUBLIC_` variable
  needs to hold its address.
- **Keycloak is the one public backend.** The browser must reach
  `https://auth.tiyiselani.com` directly to sign in; it is not proxied through
  this app.

## Why a gateway at all

The frontend needs one stable base URL. Without it, the Next server would need to
know every service's address and port, and CORS/auth policy would be duplicated
per service. The gateway owns routing (`/api/blog/*`, `/api/chat/*`) and JWT
validation; downstream services trust it.

See [../gateway.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/gateway.md) for the original rationale.

## Why Aspire

`TmLabs.AppHost` is executable documentation of the topology:

```csharp
var blogApi = builder.AddProject<Projects.TmLabs_BlogApi>("blogapi").WithReference(db);
var gateway = builder.AddProject<Projects.TmLabs_Gateway>("gateway")
    .WithReference(blogApi)
    .WithReference(chatService)
    .WithEnvironment("Gateway__BlogApiUrl", blogApi.GetEndpoint("http"));
builder.AddKubernetesEnvironment("k8s");
```

One description produces both a local dev environment and, via
`aspire publish`, a Helm chart with matching service names and environment
wiring. That is the trade: a generator you must post-process, instead of two
descriptions that drift apart.

## Names you can rely on

These are generated, not chosen by hand — anything referencing them must match:

| Thing | Value |
| --- | --- |
| Service names | `web-service`, `gateway-service`, `blogapi-service`, `chatservice-service` |
| Deployment names | `<component>-deployment` |
| ConfigMap names | `<component>-config` |
| Secret names | `<component>-secrets` |
| Internal gateway URL | `http://gateway-service:8080` |
| Release name | `tmlabs` (set by Argo) |
| Chart name | `tmlabs-apphost` |

## Where each concern lives

| Concern | Location |
| --- | --- |
| Topology, env wiring, parameters | `src/TmLabs.AppHost/AppHost.cs` |
| Image build + publish | `.github/workflows/deploy.yml` |
| Chart corrections | the `Finalize chart` step in the same workflow |
| Cluster resources Argo needs | `tmlabs-gitops/apps/`, `tmlabs-gitops/manifests/tmlabs-extras/` |
| Secret values | OpenBao, `secret/tmlabs/*` |

Nothing in `src/TmLabs.AppHost/k8s-artifacts/` is deployed — see
[05](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/05-chart-pipeline.md).
