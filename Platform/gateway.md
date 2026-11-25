---
title: API Gateway
description: This project uses a YARP-based gateway so the frontend and clients can talk to a single entry point instead of calling each service directly.
publishedAt: 2026-11-25T08:01:00Z
draft: false
---

This project uses a YARP-based gateway so the frontend and clients can talk to a single entry point instead of calling each service directly.

## Why add a gateway?

The backend is split into small services:

- `TmLabs.BlogApi`
- `TmLabs.ChatService`

A gateway gives you one public route surface:

- `/api/blog/*` -> Blog API
- `/api/chat/*` -> Chat API

This makes the system easier to reason about during learning, and it mirrors how larger distributed apps typically expose services in production.

## How it was implemented

1. A new project was created:
   - `src/TmLabs.Gateway`
2. The project references the YARP package:
   - `Yarp.ReverseProxy`
3. The app registers a reverse proxy with route and cluster configuration in `Program.cs`.
4. The AppHost adds the gateway project alongside the services.
5. The gateway uses environment-based values for the downstream URLs:
   - `Gateway__BlogApiUrl`
   - `Gateway__ChatApiUrl`

## Route setup

The gateway maps:

- `/api/blog/{**catch-all}` to the blog cluster
- `/api/chat/{**catch-all}` to the chat cluster

The `PathRemovePrefix` transform strips the route prefix before forwarding the request to the downstream service.

## AppHost wiring

In `TmLabs.AppHost/AppHost.cs`, the gateway is attached with:

```csharp
var gateway = builder.AddProject<Projects.TmLabs_Gateway>("gateway")
    .WithEnvironment("Gateway__BlogApiUrl", "http://blogapi")
    .WithEnvironment("Gateway__ChatApiUrl", "http://chatservice");
```

This keeps the gateway route targets aligned with the Aspire service names used during local orchestration.

## Local access

Once the app host is running, the gateway becomes the single entry point for service traffic.

Examples:

- `https://localhost:<gateway-port>/api/blog/...`
- `https://localhost:<gateway-port>/api/chat/...`

The exact port is assigned by the local runtime, but the route structure remains stable.

## Notes for learning

This is intentionally simple and explicit so it is easy to understand:

- one project for the gateway
- route-based forwarding
- downstreams separated by path prefix
- no business logic inside the gateway itself

That makes the gateway a great place to learn about reverse proxying, service composition, and microservice entry points.

## Auth guidance for the public app

For a public app with a Next.js frontend, the gateway should not be the source of user identity. It should enforce auth for public traffic while the identity provider owns user accounts, social sign-ins, and passkeys.

Recommended pattern:

1. User signs in with GitHub, Google, or a passkey through an identity provider such as Keycloak.
2. The identity provider issues a JWT access token.
3. The Next.js app sends that token to the gateway.
4. The gateway validates issuer, audience, signature, and expiry.
5. The gateway forwards the request only if the token is valid.
6. Blog API and Chat Service authorize based on claims like role or scope.

This keeps the gateway focused on authentication and routing while each microservice stays responsible for business-level authorization.
