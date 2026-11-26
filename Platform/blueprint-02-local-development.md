---
title: 2. Local development
description: Everything that deploys exists first as something that runs on a laptop. Get this
series: Deployment blueprint
seriesOrder: 3
publishedAt: 2026-11-25T08:02:00Z
draft: false
---

Everything that deploys exists first as something that runs on a laptop. Get this
working before touching CI or Kubernetes — every later problem is easier to
diagnose from a working local baseline.

## Prerequisites

| Tool | Version | Notes |
| --- | --- | --- |
| .NET SDK | 10.0 | projects target `net10.0` |
| Aspire CLI | 17.0 | `dotnet tool install -g Aspire.Cli` |
| Docker | any recent | used by Aspire for containers |
| PostgreSQL | 16 | or run it in Docker |
| Node.js | 22 | frontend only |
| `dotnet-ef` | 10.0.4 | pinned in `dotnet-tools.json` |

Restore the pinned EF tool from the repo:

```bash
dotnet tool restore
```

## Database

```bash
docker run --name tmlabs-postgres \
  -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=tmlabs \
  -p 5432:5432 -d postgres:16
```

Point the AppHost at it through user secrets (never a committed file):

```bash
dotnet user-secrets set "ConnectionStrings:tmlabs-db" \
  "Host=localhost;Port=5432;Database=tmlabs;Username=postgres;Password=postgres" \
  --project src/TmLabs.AppHost
```

The other parameters have defaults in `AppHost.cs` except the two secrets:

```bash
dotnet user-secrets set "Parameters:auth-secret" "local-dev-only" --project src/TmLabs.AppHost
dotnet user-secrets set "Parameters:revalidate-secret" "local-dev-only" --project src/TmLabs.AppHost
```

## Run

```bash
dotnet run --project src/TmLabs.AppHost
```

You get the Aspire dashboard, plus the web app on `http://localhost:3000`.
The AppHost sets `AUTH_URL=http://localhost:3000`, which is how Auth.js knows its
own origin.

The frontend needs its own `.env.local` when run outside Aspire (from the
`tmlabs-frontend` repo):

```
NEXT_PUBLIC_API_BASE_URL=http://localhost:5020
NEXT_PUBLIC_KEYCLOAK_URL=https://auth.tiyiselani.com
NEXT_PUBLIC_KEYCLOAK_REALM=non-prod
NEXT_PUBLIC_KEYCLOAK_CLIENT_ID=tmlabs-frontend
NEXT_PUBLIC_KEYCLOAK_REDIRECT_URI=http://localhost:3000/api/auth/callback/keycloak
```

`localhost:3000/api/auth/callback/keycloak` must be registered on the Keycloak
client, otherwise local sign-in fails with `redirect_uri_mismatch`.

## Verify before moving on

| Check | Expected |
| --- | --- |
| `dotnet build` | succeeds, no warnings you didn't cause |
| dashboard | all resources green |
| `http://localhost:3000/api/health` | `{"status":"ok"}` |
| `http://localhost:3000/blog` | post list, or an empty-state message |
| `http://localhost:3000/login` | Keycloak sign-in completes |

## Traps

- **`localhost` inside a container is not your host.** If Next runs in Docker and
  the gateway on the host, `http://localhost:5020` resolves inside the container.
  Use `host.docker.internal` (or the host's LAN IP).
- **AppHost hostnames don't resolve outside Aspire.** Running the gateway binary
  directly means `blogapi` and `chatservice` don't exist; set
  `Gateway__BlogApiUrl` and `Gateway__ChatApiUrl` to real URLs.
- **The AppHost is the only place topology is described.** If a service needs a
  new environment variable in production, add it here too, or the generated
  ConfigMap won't have it.

Next: [03-images-and-registry.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/03-images-and-registry.md)
