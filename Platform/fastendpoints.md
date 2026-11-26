---
title: FastEndpoints Setup Guide
description: This project uses FastEndpoints for API endpoint development in TmLabs.BlogApi and TmLabs.ChatService.
series: API and gateway
seriesOrder: 2
publishedAt: 2026-11-25T08:14:00Z
draft: false
---

This project uses FastEndpoints for API endpoint development in `TmLabs.BlogApi` and `TmLabs.ChatService`.

## Current status

The project now includes:

1. `builder.Services.AddFastEndpoints()` in each service `Program.cs`
2. `app.UseFastEndpoints()` in each service `Program.cs`
3. `Scalar.AspNetCore` package and `MapScalarApiReference()` in each service
4. A health endpoint class per service:
   - `src/TmLabs.BlogApi/Endpoints/HealthEndpoint.cs`
   - `src/TmLabs.ChatService/Endpoints/HealthEndpoint.cs`

This is intentionally minimal for now: only the health route exists, and no business routes were added yet.

## Package install commands

```bash
dotnet add src/TmLabs.BlogApi/TmLabs.BlogApi.csproj package FastEndpoints
dotnet add src/TmLabs.BlogApi/TmLabs.BlogApi.csproj package Scalar.AspNetCore

dotnet add src/TmLabs.ChatService/TmLabs.ChatService.csproj package FastEndpoints
dotnet add src/TmLabs.ChatService/TmLabs.ChatService.csproj package Scalar.AspNetCore
```

## Program.cs wiring

The service entry point should be set up like this:

```csharp
using Scalar.AspNetCore;
using FastEndpoints;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();
builder.Services.AddFastEndpoints();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference(options =>
    {
        options.Title = "TmLabs Blog API";
    });
}

if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
}
app.UseFastEndpoints();

app.Run();
```

This is what enables both the raw OpenAPI document and the Scalar docs UI for the service itself.

## Example endpoint

```csharp
using FastEndpoints;

namespace TmLabs.BlogApi.Endpoints;

public class HealthEndpoint : EndpointWithoutRequest
{
    public override void Configure()
    {
        Get("/health");
        AllowAnonymous();
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        await Send.OkAsync(new { status = "ok", service = "blogapi" }, ct);
    }
}
```

## Gateway route mapping

The gateway prefixes requests:

- `/api/blog/*` -> Blog API
- `/api/chat/*` -> Chat API

The gateway also exposes explicit health routes so they show up in Scalar and can be called without reaching the backend services directly:

- `/api/blog/health`
- `/api/chat/health`
- `/health` (gateway itself)

These are the routes to use when testing through the gateway only.

## Service docs URLs

When each API is running in development mode, the docs are available at:

- Blog API Scalar: `http://localhost:5001/scalar` or `https://localhost:5001/scalar`
- Chat Service Scalar: `http://localhost:5002/scalar` or `https://localhost:5002/scalar`
- Gateway Scalar: `http://localhost:5000/scalar`

The exact port depends on the launchSettings or AppHost configuration, but the route pattern is the same.

When using the combined documents in the gateway Scalar UI, open the gateway URL rather than a downstream service URL. In Development, Blog API and Chat Service keep their internal HTTP endpoints available without HTTPS redirection so YARP can proxy `/api/blog/openapi/v1.json` and `/api/chat/openapi/v1.json` correctly. HTTPS redirection is enabled for non-development deployments.

The Blog API OpenAPI document includes the Keycloak OAuth2 scheme, so its Scalar view also has an **Authorize** button. Scalar presents Gateway, Blog API, and Chat Service as separate documents rather than nested folders; select the service document from the document selector to see that service's endpoints and authorization metadata.

## Blog content sync

Trigger a GitHub-to-PostgreSQL sync through the gateway:

The gateway requires a Keycloak access token with the `admin` realm role. It maps Keycloak's `realm_access.roles` claim to ASP.NET role claims before applying the `Admin` policy.

`GitHub:SyncToken` is compared with the required `syncToken` request field. Configure it with user secrets or deployment configuration; do not leave it empty in a running environment.

```bash
curl -i -X POST \
    -H "Content-Type: application/json" \
    -d '{
        "owner": "TiyiselaniIndustries",
        "repository": "blog",
        "branch": "main",
        "syncToken": "'"$GITHUB_SYNC_TOKEN"'"
    }' \
    http://localhost:<gateway-port>/api/blog/sync
```

An unchanged repository returns a JSON response similar to:

```json
{
    "commitSha": "da5c0ae2e8c33b70575ccc04342f6dd8852b3711",
    "changed": 0,
    "unpublished": 0,
    "skipped": 2
}
```

An invalid or missing `syncToken` returns `401` with a JSON error. GitHub, tree, page, and database progress is logged by Blog API at the `Information` level. A sync failure returns `502` with an error message and is logged with the exception.

## Combined gateway view

The gateway can also aggregate the public microservice OpenAPI documents into one combined spec so that they appear together in a single Scalar page.

This is done by:

1. fetching each service `/openapi/v1.json`
2. merging the `paths` objects under gateway prefixes
3. exposing the merged spec at `/openapi/v1.json`
4. rendering Scalar at `/scalar`

The exposed gateway routes are:

- `/health`
- `/api/blog/health`
- `/api/chat/health`

This gives you one unified docs surface while still keeping each service logically separate.

## Local test commands

Run AppHost:

```bash
dotnet run --project src/TmLabs.AppHost
```

Then test via gateway:

```bash
curl http://localhost:<gateway-port>/api/blog/health
curl http://localhost:<gateway-port>/api/chat/health
```

Or hit the service docs directly in development:

```bash
curl http://localhost:<blogapi-port>/scalar
curl http://localhost:<chatservice-port>/scalar
```

## Next step

Add more endpoint classes under each service `Endpoints/` folder and keep the route structure aligned with the gateway prefixes.
