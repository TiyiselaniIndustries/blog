---
title: User Linking and Database Setup
description: This guide connects Keycloak identities to TmLabs application users and sets up the PostgreSQL database used by the Blog API and Chat Service.
series: Identity and access
seriesOrder: 6
publishedAt: 2026-11-25T08:20:00Z
draft: false
---

This guide connects Keycloak identities to TmLabs application users and sets up the PostgreSQL database used by the Blog API and Chat Service.

## What owns what

Keep identity data and application data separate:

| Data | Owner |
| --- | --- |
| Passwords, social logins, passkeys, MFA, sessions | Keycloak |
| Blog posts and revisions | Blog API, in the `blog` schema |
| Chat users and comments | Chat Service, in the `chat` schema |
| The link to a signed-in person | The Keycloak JWT `sub` claim |

A Keycloak user may have several linked login methods. GitHub, Google, Apple, a password, and a passkey should all resolve to the same Keycloak user before the application creates or loads a local profile.

Do not use email, username, or a provider-specific numeric ID as the permanent application identity. Those values can change or be unavailable. The opaque `sub` claim is the stable key for the Keycloak user in this realm.

## 1. Start PostgreSQL

### Option A: Aspire local database

The AppHost creates a PostgreSQL resource with a persistent data volume when no external `tmlabs-db` connection string is configured:

```bash
dotnet run --project src/TmLabs.AppHost
```

Open the Aspire dashboard and use the generated connection information for troubleshooting. Aspire passes the `tmlabs-db` connection to both services.

### Option B: Existing PostgreSQL

Start PostgreSQL, then configure the AppHost user secret:

```bash
dotnet user-secrets set \
  "ConnectionStrings:tmlabs-db" \
  "Host=localhost;Port=5432;Database=tmlabs;Username=postgres;Password=postgres" \
  --project src/TmLabs.AppHost
```

For a disposable local instance:

```bash
docker run --name tmlabs-postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=tmlabs \
  -p 5432:5432 \
  -d postgres:16
```

Never commit a real password or production connection string. In deployment, provide `ConnectionStrings__tmlabs-db` or `TMLABS_DB` through the secret manager used by the environment.

## 2. Apply the database schema

The services use EF Core migrations and PostgreSQL. Each service owns its own schema and migration history:

```bash
dotnet ef database update \
  --project src/TmLabs.BlogApi \
  --startup-project src/TmLabs.BlogApi

dotnet ef database update \
  --project src/TmLabs.ChatService \
  --startup-project src/TmLabs.ChatService
```

The current Blog API and Chat Service also call `Database.Migrate()` during startup. That makes local development convenient, but production deployments should still apply migrations as an explicit release step and should run only one migration job at a time.

Create a migration after changing a model:

```bash
dotnet ef migrations add DescribeTheChange \
  --project src/TmLabs.ChatService \
  --startup-project src/TmLabs.ChatService
```

Use the Blog API project instead when changing blog models. Commit the generated migration files.

### Verify the schemas

Using `psql` or a database client, confirm that the migration histories and tables exist:

```sql
select schemaname, tablename
from pg_tables
where schemaname in ('blog', 'chat')
order by schemaname, tablename;

select * from blog."__EFMigrationsHistory";
select * from chat."__EFMigrationsHistory";
```

The exact table-name quoting is required because EF Core uses `__EFMigrationsHistory` with capital letters.

## 3. Link a user in Keycloak

There are two links to establish, in this order.

### Link login methods inside Keycloak

1. Create or select the user's Keycloak account.
2. Enable the GitHub, Google, or Apple identity provider in the `non-prod` realm.
3. Have the user sign in with the existing account.
4. In the Keycloak Account Console, open **Account security > Linked accounts**.
5. Link the additional provider and complete the requested verification.
6. Sign in again through the new provider and verify that the access token has the same `sub` value.

Alternatively, keep Keycloak's **First broker login** flow configured to require email verification or re-authentication when an existing account is found. Never merge accounts solely because their email strings match.

After the merge, every linked login method produces the same Keycloak `sub`. The TmLabs database should therefore contain one local profile for that person.

### Link the Keycloak user to a TmLabs profile

The application should provision a local profile on the first authenticated request:

```text
Keycloak login
  -> access token with sub
  -> gateway validates the token
  -> Chat Service reads sub
  -> Chat Service finds or creates one local user
```

The find-or-create operation must:

- require an authenticated principal and a non-empty `sub`
- look up the local row by a unique `KeycloakSubject`
- copy display name, username, avatar, and email only as initial profile values
- never replace user-edited profile values on every login
- use a unique database index to handle two simultaneous first requests
- never accept a user ID from a profile request body or URL when determining the current user

A frontend should call the authenticated `GET /users/me` endpoint immediately after login. That endpoint can both provision the profile and return the current user's profile.

## 4. Read the current Keycloak user

The current logged-in user is represented by the validated access token. The most important claims are:

| Value | Token claim | Meaning |
| --- | --- | --- |
| Keycloak user ID | `sub` | Stable identity key for this realm; use it as `KeycloakSubject` |
| Username | `preferred_username` | Display/login name; not a permanent identity key |
| Email | `email` | Profile data; it may change or be absent |
| Roles | `realm_access.roles` or `resource_access.<client>.roles` | Keycloak permissions represented as roles |
| Scopes | `scope` | Space-delimited OAuth scopes such as `openid profile email` |

### From the backend

The gateway can expose a development-only diagnostic endpoint after JWT authentication is configured:

```csharp
using System.Security.Claims;

app.MapGet("/api/auth/me", (ClaimsPrincipal user) =>
{
    var roles = user.Claims
        .Where(claim => claim.Type == ClaimTypes.Role)
        .Select(claim => claim.Value)
        .Distinct(StringComparer.Ordinal)
        .ToArray();

    var scopes = user.FindFirst("scope")?.Value?
        .Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
        ?? [];

    return Results.Ok(new
    {
        Id = user.FindFirstValue("sub"),
        Username = user.FindFirstValue("preferred_username"),
        Email = user.FindFirstValue("email"),
        Roles = roles,
        Scopes = scopes
    });
}).RequireAuthorization();
```

Call it through the gateway with the access token:

```bash
curl -s \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:<gateway-port>/api/auth/me
```

Example response:

```json
{
  "id": "4a1c...",
  "username": "alex",
  "email": "alex@example.com",
  "roles": ["user", "chat:read"],
  "scopes": ["openid", "profile", "email"]
}
```

The gateway must flatten Keycloak's nested role claims into `ClaimTypes.Role` claims before this endpoint or application policies read them. Do not treat a role as a scope, and do not authorize from `preferred_username` or `email`.

### From the frontend

The frontend may decode the access token to display non-sensitive UI state, but it must not treat decoded browser data as authorization. Always enforce permissions in the gateway or service using the validated token.

For profile data, call Keycloak's UserInfo endpoint with the access token:

```bash
curl -s \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://auth.example.com/realms/non-prod/protocol/openid-connect/userinfo
```

UserInfo typically returns profile claims such as `sub`, `preferred_username`, `name`, and `email`. It is not the source of application permissions; use the access token's role claims for that, and use the backend's `/api/auth/me` response for the normalized application view.

Never expose the access token in URLs, logs, or a public profile response. Remove or restrict `/api/auth/me` before production if it returns raw claims.

## 5. Current repository migration note

The current Chat Service model still has a unique `GitHubUserId` column in `chat.users`. That is provider-specific and does not yet implement the Keycloak design above. Before enabling Google, Apple, passkeys, or local Keycloak accounts for existing users:

1. Add a nullable `KeycloakSubject` column and unique index.
2. Deploy the find-or-create logic keyed by `sub`.
3. For an existing GitHub user, link the row only after a verified GitHub login. Use a Keycloak GitHub mapper to carry the verified GitHub numeric ID as a temporary `github_id` claim.
4. Backfill `KeycloakSubject` on the matching row instead of creating a duplicate profile.
5. Check for duplicate or unmatched accounts.
6. After the migration is complete, remove `GitHubUserId` in a separate migration.

Do not backfill by matching email or username. Do not automatically match a new social login to an existing row based only on an email address.

The target user entity should contain a field similar to:

```csharp
public string KeycloakSubject { get; set; } = string.Empty;
```

And the EF configuration should enforce the identity invariant:

```csharp
entity.Property(x => x.KeycloakSubject).IsRequired().HasMaxLength(128);
entity.HasIndex(x => x.KeycloakSubject).IsUnique();
```

Keep the local `Guid` as the foreign-key target for comments. The Keycloak subject identifies the person; `User.Id` identifies the local profile row.

## 6. Test the complete path

1. Start PostgreSQL or the Aspire AppHost.
2. Apply both migration sets.
3. Sign in to Keycloak with a test user.
4. Call the gateway with the bearer access token.
5. Confirm the Chat Service receives the token and reads `sub`.
6. Call the authenticated profile endpoint twice.
7. Confirm the first call creates one row and the second call returns the same row.
8. Sign in through a linked provider and confirm it returns the same `sub` and local profile.
9. Create a comment and verify its `UserId` points to the local `chat.users` row.
10. Confirm an unauthenticated request cannot create a profile or comment.

Never log access tokens, refresh tokens, client secrets, or full authorization headers while testing.

## Related documentation

- [Keycloak authentication guide](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/keycloak-auth-guide.md)
- [Authentication checklist](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/auth-checklist.md)
- [Keycloak and Scalar setup](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/keycloak-scalar-setup.md)
- [Project README](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/README.md)
