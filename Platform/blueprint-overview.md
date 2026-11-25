---
title: Deployment blueprint: overview
description: A step-by-step record of how TmLabs goes from source code to a running app on
publishedAt: 2026-11-25T08:11:00Z
draft: false
---

A step-by-step record of how TmLabs goes from source code to a running app on
Kubernetes, written so the same path can be reused for the next project.

Each file covers one stage. Read them in order the first time; afterwards they
work as reference.

| # | File | Answers |
| --- | --- | --- |
| 1 | [01-architecture.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/01-architecture.md) | What are the moving parts, and why so many? |
| 2 | [02-local-development.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/02-local-development.md) | How does it run on a laptop before any of this exists? |
| 3 | [03-images-and-registry.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/03-images-and-registry.md) | How are the five images built, tagged and stored? |
| 4 | [04-ci-pipeline.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/04-ci-pipeline.md) | What does GitHub Actions do, in what order? |
| 5 | [05-chart-pipeline.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/05-chart-pipeline.md) | How does the Helm chart get generated and corrected? |
| 6 | [06-gitops-and-argo.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/06-gitops-and-argo.md) | How does a commit to Git become a running pod? |
| 7 | [07-configuration-and-secrets.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/07-configuration-and-secrets.md) | What is config, what is secret, and how does each reach a pod? |
| 8 | [08-networking-and-tls.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/08-networking-and-tls.md) | How does a browser reach the app over HTTPS? |
| 9 | [09-identity-and-auth.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/09-identity-and-auth.md) | How does Keycloak sign-in work end to end? |
| 10 | [10-database-and-migrations.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/10-database-and-migrations.md) | Where does the database come from and who creates tables? |
| 11 | [11-operations.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/11-operations.md) | How do I verify, debug, roll back and rotate? |
| 12 | [12-new-app-checklist.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/12-new-app-checklist.md) | What do I change to make this a different app? |

## The golden path

```
code
 └─ Aspire AppHost            describes resources + wiring locally
     ├─ CI builds 4 images    dotnet publish -t:PublishContainer + docker build
     ├─ CI publishes a chart  aspire publish, then corrected in a Finalize step
     └─ CI commits the chart  to the GitOps repo (gated)
         └─ Argo CD syncs     Helm chart + hand-written extras
             └─ Kubernetes    Deployments → Services → Ingress → browser
```

Four repos, four responsibilities:

| Repo | Owns |
| --- | --- |
| `tmlabs-backend` | services, AppHost, CI, the generated chart |
| `tmlabs-frontend` | Next.js app, its Dockerfile, a dispatch workflow |
| `tmlabs-gitops` | Argo Applications, Ingress, ExternalSecrets, the CI-committed chart |
| OpenBao | the actual secret values |

## Reading rules

- [../argocd-deployment.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/argocd-deployment.md) is the **operational
  reference** — current variable names, secret keys, troubleshooting table. If it
  and this blueprint disagree, that file wins.
- This blueprint is the **narrative**: why each piece exists and which traps are
  waiting.
- Commands assume Linux/bash unless marked otherwise. Paths in this repo are
  relative to the repository root.

## The five decisions that shape everything

1. **Aspire generates the Kubernetes chart.** No hand-written Helm chart to
   maintain — the cost is a post-processing step, because generator output is
   not directly deployable (see [05](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/05-chart-pipeline.md)).
2. **GitOps with Argo CD.** CI writes to Git; Argo applies. CI never touches the
   cluster, so it needs no kubeconfig and no cluster credentials.
3. **Secrets live in OpenBao, not in Git.** The chart's Secret templates are
   deleted before publishing; External Secrets fills the gap
   (see [07](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/07-configuration-and-secrets.md)).
4. **The gateway is private, with one published path.** `/api/blog` is routed
   from the public host so the content repo's workflow can call sync; everything
   else (including `/api/chat`) stays cluster-internal. See
   [08](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/08-networking-and-tls.md).
5. **Runtime config over build-time config.** `API_URL` and `KEYCLOAK_*` are read
   per request so changing a URL doesn't require a rebuild.
