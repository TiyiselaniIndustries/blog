---
title: 10. Database and migrations
description: PostgreSQL is not provisioned by this chart. Both services connect to an
publishedAt: 2026-11-25T08:21:00Z
draft: false
---

PostgreSQL is **not** provisioned by this chart. Both services connect to an
external instance.

## Connection string

Npgsql key/value form, semicolon-separated — not a URL:

```
Host=192.168.1.30;Port=5432;Database=tmlabs;Username=tmlabs;Password=<pass>
```

Stored in OpenBao (`secret/tmlabs/{blogapi,chatservice}`) and surfaced as
`ConnectionStrings__tmlabs_db` in each service's Secret.

A `postgres://` URL fails with:

```
Format of the initialization string does not conform to specification starting at index 0
```

## Migrations run on startup

`Program.cs` in both services:

```csharp
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<BlogDbContext>();
    dbContext.Database.Migrate();
}
```

Consequences:

- **No manual migration step in CI or Argo.** A rollout applies pending
  migrations automatically.
- **The database user needs DDL rights** (`CREATE TABLE`, `ALTER TABLE`). A
  read/write-only user will start, fail to migrate, and crash-loop.
- **Concurrency:** with `replicas: 1` this is safe. Scaling to several replicas
  means several `Migrate()` calls racing; only scale after moving migrations to
  an init container or a Job.
- **Startup is gated on the database.** If Postgres is unreachable the pod never
  becomes ready — check `kubectl -n tmlabs logs deploy/blogapi-deployment` for
  connection errors rather than assuming the image is broken.

> `README.md` still describes running `dotnet ef database update` by hand. That
> contradicts the code; the code wins. Reconcile the README when convenient.

## Separate schemas, separate histories

Each service owns its own `DbContext` and migration set, in the same database.
That keeps them independently deployable but means both need DDL rights.

| Service | DbContext | Migrations |
| --- | --- | --- |
| `TmLabs.BlogApi` | `BlogDbContext` | `src/TmLabs.BlogApi/Migrations/` |
| `TmLabs.ChatService` | `ChatDbContext` | `src/TmLabs.ChatService/Migrations/` |

## Creating a migration

```bash
dotnet tool restore

dotnet ef migrations add <Name> \
  --project src/TmLabs.BlogApi --startup-project src/TmLabs.BlogApi
```

Commit the generated files — they are the schema history and must ship with the
image, since `Migrate()` reads them from the assembly.

## Trap: local vs cluster connection strings diverge

Locally you point at `localhost`; in the cluster at `192.168.1.30`. Verify the
cluster value without printing the password:

```bash
kubectl -n tmlabs get secret blogapi-secrets \
  -o jsonpath='{.data.ConnectionStrings__tmlabs_db}' | base64 -d | cut -c1-30; echo
```

Must start with `Host=`.

## Trap: NetworkPolicy / firewall

A connection refusal from inside the pod is a network problem, not a credentials
problem. Confirm reachability from the pod:

```bash
kubectl -n tmlabs exec deploy/blogapi-deployment -- \
  sh -c 'command -v nc >/dev/null && nc -zv 192.168.1.30 5432 || echo "no nc available"'
```

Next: [11-operations.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/11-operations.md)
