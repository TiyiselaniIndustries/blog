---
title: Authentication and Authorization Flow
description: This project is designed around a public frontend, a gateway, and multiple backend services. The recommended architecture is
publishedAt: 2026-11-25T08:03:00Z
draft: false
---

This project is designed around a public frontend, a gateway, and multiple backend services. The recommended architecture is:

- Next.js frontend for the public app
- Identity provider for user management and login
- API gateway for public entry and JWT validation
- Blog and Chat services for business authorization

## Recommended identity model

Use a dedicated identity provider such as Keycloak.

It should handle:

- GitHub sign-in
- Google sign-in
- passkey / WebAuthn registration and login
- user storage
- roles and groups
- access token issuance

Do not keep user accounts or credentials in the microservices themselves.

## Public login flow

1. The user opens the Next.js app.
2. The frontend redirects the user to the identity provider.
3. The user signs in using GitHub, Google, or a passkey.
4. The identity provider issues an access token and refresh token.
5. The frontend stores the access token securely and includes it in API requests.
6. The frontend calls the gateway with `Authorization: Bearer <token>`.

## Gateway validation flow

The gateway is the trust boundary for public traffic.

It should validate:

- signature
- issuer
- audience
- expiry
- token format

If validation succeeds, the gateway can continue the request. If validation fails, it should reject the request with `401 Unauthorized`.

The gateway is where public auth is enforced.

## Downstream authorization

Once the gateway accepts the request, the downstream services should still enforce authorization rules based on claims.

Examples of claims:

- `sub`
- `email`
- `role`
- `groups`
- `scope`

Recommended service rules:

- `blog:read` required to read posts
- `blog:write` required to create or update blog content
- `chat:write` required to comment or post
- `admin` required for moderation or administrative actions

The services should not trust the caller without checking the identity claims.

## Recommended architecture

### Frontend

The Next.js app should be responsible for:

- login UI
- redirecting to the identity provider
- storing session tokens securely
- making authenticated requests to the gateway

### Identity provider

The identity provider should be responsible for:

- user accounts
- social login linking
- passkey registration and login
- MFA if needed
- role and group assignment

### Gateway

The gateway should be responsible for:

- public API entry point
- validation of access tokens
- enforcement of protected routes
- routing to Blog and Chat services

### Microservices

The Blog and Chat services should be responsible for:

- business logic
- domain authorization
- ownership checks
- data access constraints

## Why this is the right fit

This design is ideal for a public app because it keeps user identity and credential management centralized and standards-based.

It also makes the app easier to extend later with:

- more social providers
- MFA
- organization-level roles
- user administration
- access revocation

## Suggested implementation path

1. Add Keycloak or another OIDC provider.
2. Configure GitHub and Google identity providers.
3. Enable passkeys / WebAuthn in the identity provider.
4. Add JWT authentication to the gateway.
5. Add policies for anonymous, user, and admin access.
6. Keep the service-specific rules inside Blog API and Chat Service.

## Summary

The recommended flow is:

1. User signs in to the identity provider.
2. Identity provider issues JWT.
3. Next.js sends JWT to the gateway.
4. Gateway validates JWT.
5. Gateway routes to Blog/Chat service.
6. Services authorize using claims and policies.

This gives you a simple, secure, public-facing auth flow that scales well as the app grows.
