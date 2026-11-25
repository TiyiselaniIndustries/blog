---
title: Technical Auth Stack and Flow
description: This document describes the recommended implementation for a public-facing app with
publishedAt: 2026-11-25T08:04:00Z
draft: false
---

This document describes the recommended implementation for a public-facing app with:

- Next.js frontend
- GitHub login
- Google login
- passkeys / WebAuthn
- ASP.NET Core gateway
- microservice APIs built with FastEndpoints

## Recommended architecture

Use a dedicated identity provider for all public authentication concerns.

Recommended choice:

- Keycloak

Why:

- supports OIDC/OAuth2
- supports GitHub and Google social login
- supports WebAuthn / passkeys
- manages users, roles, groups, and sessions
- issues JWT access tokens in a standard format

The gateway should validate those JWTs. The backend microservices should authorize based on claims and policies.

## Frontend stack

For the Next.js app, the usual stack is:

- Next.js
- Auth.js / NextAuth.js

Packages:

```bash
npm install next-auth
```

Or, if using a lower-level OIDC client:

```bash
npm install oidc-client-ts
```

Typical providers:

- GitHub
- Google
- Keycloak OIDC

The frontend should not manage password or passkey credentials directly in the backend services. It should simply log the user in through the identity provider and send the resulting access token to the backend gateway.

## Identity provider flow

1. User clicks Sign in with GitHub or Google.
2. Next.js redirects to Keycloak.
3. Keycloak handles the social login or passkey flow.
4. Keycloak issues a JWT access token.
5. Frontend sends the bearer token to the gateway.

This keeps passkeys, OAuth, and social identity logic out of the .NET API code.

## Gateway stack

For the .NET gateway, use:

- ASP.NET Core
- `Microsoft.AspNetCore.Authentication.JwtBearer`
- `Microsoft.IdentityModel.Tokens`

Package install:

```bash
dotnet add src/TmLabs.Gateway/TmLabs.Gateway.csproj package Microsoft.AspNetCore.Authentication.JwtBearer
```

### Example gateway configuration

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://keycloak.example.com/realms/tmlabs";
        options.Audience = "tmlabs-api";
        options.RequireHttpsMetadata = true;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("BlogReader", policy =>
        policy.RequireClaim("scope", "blog:read"));

    options.AddPolicy("Admin", policy =>
        policy.RequireRole("admin"));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

This is the standard JWT validation pattern for an API gateway.

## Microservice stack

For the Blog API and Chat Service, use:

- FastEndpoints
- policy-based authorization

Package install:

```bash
dotnet add src/TmLabs.BlogApi/TmLabs.BlogApi.csproj package FastEndpoints
dotnet add src/TmLabs.ChatService/TmLabs.ChatService.csproj package FastEndpoints
```

### Example service authorization

```csharp
using FastEndpoints;

public class GetPostsEndpoint : EndpointWithoutRequest
{
    public override void Configure()
    {
        Get("/posts");
        Policies("BlogReader");
        AllowAnonymous();
    }

    public override Task HandleAsync(CancellationToken ct)
    {
        return Send.OkAsync(new { posts = Array.Empty<string>() }, ct);
    }
}
```

The service should enforce business authorization based on claims, not user management.

## User management responsibilities

User management belongs in the identity provider.

It should own:

- social account linking
- passkeys / WebAuthn credentials
- roles and groups
- MFA configuration
- account recovery
- user deactivation

The .NET services should not be storing user passwords or passkey credentials.

## Recommended security split

### Frontend

- login UX
- token storage
- API calls

### Identity provider

- users
- social auth
- passkeys
- roles and claims

### Gateway

- JWT validation
- public route enforcement
- route protection

### Services

- business policy checks
- resource ownership checks
- feature-level authorization

## Recommended user and role model

Suggested roles/scopes:

- `anonymous`
- `user`
- `admin`
- `blog:read`
- `blog:write`
- `chat:read`
- `chat:write`

This gives you a simple, clear permission model without embedding auth logic in every endpoint.

## Recommended flow for this project

1. User signs in through Next.js using GitHub or Google.
2. Keycloak issues a JWT.
3. Next.js sends the JWT to the gateway.
4. Gateway validates the JWT.
5. Gateway routes to the Blog or Chat service.
6. Service checks claims such as `role` or `scope`.
7. Service allows or denies the action.

This is the cleanest design for a public microservice app.

## Why not build auth directly in the microservices?

Because social login and passkeys are hard to implement safely and consistently across services.

A dedicated identity provider gives you:

- better security
- easier compliance
- standard OAuth/OIDC behavior
- less duplicated auth code
- easier passkey management

## Summary

For this project, the most practical and robust stack is:

- Next.js frontend
- Auth.js / NextAuth.js
- Keycloak as identity provider
- ASP.NET Core JWT bearer auth in the gateway
- FastEndpoints with policy-based authorization in microservices

This is the standard way to support public login, social providers, and passkeys without scattering auth logic across the backend.

For the repository-specific setup sequence and validation checklist, see:

- `docs/keycloak-scalar-setup.md`
- `docs/auth-checklist.md`
