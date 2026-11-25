---
title: TmLabs Authentication Checklist
description: [ ] Keycloak owns users, passwords, passkeys, social login, sessions, and tokens.
publishedAt: 2026-11-25T08:05:00Z
draft: false
---

## Architecture

- [ ] Keycloak is used as the identity provider.
- [ ] No authentication microservice is added to TmLabs.
- [ ] Keycloak owns users, passwords, passkeys, social login, sessions, and tokens.
- [ ] The gateway validates access tokens.
- [ ] Blog API and Chat Service enforce domain authorization.

## Keycloak

- [ ] Create the `tmlabs` realm.
- [ ] Record the realm issuer URL.
- [ ] Create the `tmlabs-api` API client.
- [ ] Set the API audience to `tmlabs-api`.
- [ ] Add an audience mapper for `tmlabs-api` to the `tmlabs-web` access token.
- [ ] Create the `tmlabs-web` frontend client.
- [ ] Make the browser client public; do not use a browser client secret.
- [ ] Enable Authorization Code flow.
- [ ] Do not enable Implicit Flow for the new frontend.
- [ ] Add local redirect URI: `http://localhost:3000/*`.
- [ ] Add local web origin: `http://localhost:3000`.
- [ ] Add production HTTPS redirect URI and web origin before deployment.
- [ ] Create `user` and `admin` roles.
- [ ] Configure `blog:read`, `blog:write`, `chat:read`, and `chat:write` permissions.
- [ ] Ensure required roles and scopes are mapped into access tokens.
- [ ] Create a GitHub OAuth App.
- [ ] Set its callback to `http://192.168.1.17:8080/realms/non-prod/broker/github/endpoint`.
- [ ] Configure the GitHub provider under Keycloak **Identity providers** with alias `github`.
- [ ] Store the GitHub OAuth secret in a password manager or Kubernetes Secret.
- [ ] Configure GitHub scopes `read:user` and `user:email` when available.
- [ ] Test GitHub login through Keycloak and Scalar.
- [ ] Confirm the GitHub-created Keycloak user receives the `tmlabs-api` audience.
- [ ] Assign required realm roles to the GitHub-created Keycloak user.
- [ ] Create a Google Cloud OAuth consent screen.
- [ ] Configure Google Cloud **Branding** with the TmLabs app name and contact emails.
- [ ] Configure Google Cloud **Audience** as External, unless this is Workspace-only.
- [ ] Add the Google test user while the consent screen is in testing mode.
- [ ] Navigate to Google Cloud **Clients > Create OAuth client**.
- [ ] Create a **Web application** OAuth client named `TmLabs Keycloak non-prod`.
- [ ] Copy the Google Redirect URI from Keycloak **Identity providers > Google**.
- [ ] Add that exact URI to Google Cloud **Authorized redirect URIs**.
- [ ] Return to Keycloak **Identity providers > Google** and paste the Google client ID and secret.
- [ ] Use an HTTPS DNS hostname for Keycloak; do not rely on the raw LXC IP as a Google OAuth web callback.
- [ ] Configure the Google provider under Keycloak **Identity providers** with alias `google`.
- [ ] Store the Google OAuth secret in a password manager or Kubernetes Secret.
- [ ] Configure Google scopes `openid`, `email`, and `profile`.
- [ ] Test Google login through Keycloak and Scalar.
- [ ] Confirm the Google-created Keycloak user receives the `tmlabs-api` audience.
- [ ] Assign required realm roles to the Google-created Keycloak user.
- [ ] Link users to local records using the opaque Keycloak `sub` claim.
- [ ] Replace provider-specific `GitHubUserId` with a unique indexed `KeycloakSubject` field.
- [ ] Create a service-owned, admin-protected `GET /api/chat/users` endpoint for the in-app user directory.
- [ ] Add pagination and bounded server-side search to the user list.
- [ ] Do not expose Keycloak Admin API tokens or raw identity-provider credentials to the browser.
- [ ] Keep emails, roles, and account metadata out of public directory responses unless explicitly required.
- [ ] Link comments and ownership records to the local `User.Id`, not provider IDs or display names.
- [ ] Define behavior for deleted or disabled Keycloak users.
- [ ] Enable `Webauthn Register Passwordless` in Keycloak.
- [ ] Configure the WebAuthn Passwordless Policy.
- [ ] Set a deployed HTTPS hostname as the passkey relying-party origin.
- [ ] Register and test a passkey from the Keycloak Account Console.
- [ ] Configure the Keycloak OTP Policy for TOTP.
- [ ] Enable `Configure OTP` as a required action.
- [ ] Configure OTP as Alternative or Required in the Browser flow.
- [ ] Enable recovery authentication codes if required.
- [ ] Register and test an authenticator app.
- [ ] Create the Apple App ID, Services ID, and Sign in with Apple key if Apple login is required.
- [ ] Register the exact Apple callback with Apple.
- [ ] Configure Apple in Keycloak or through the generic OIDC provider.
- [ ] Store the Apple `.p8` key and client-secret signing values outside the repository.
- [ ] Create a non-admin test user.

## Gateway implementation

- [ ] Add `Microsoft.AspNetCore.Authentication.JwtBearer` to `TmLabs.Gateway`.
- [ ] Configure `Authentication:Authority`.
- [ ] Configure `Authentication:Audience` as `tmlabs-api`.
- [ ] Set `RequireHttpsMetadata=true` outside local HTTP development.
- [ ] Register JWT bearer authentication.
- [ ] Register authorization policies.
- [ ] Call `UseAuthentication()`.
- [ ] Call `UseAuthorization()`.
- [ ] Keep health endpoints public where appropriate.
- [ ] Protect Blog and Chat routes that require a signed-in user.
- [ ] Apply scope and role policies to protected routes.
- [ ] Confirm YARP forwards the `Authorization` header.
- [ ] Avoid logging authorization headers or tokens.

## Services

- [ ] Configure JWT validation in Blog API.
- [ ] Configure JWT validation in Chat Service.
- [ ] Add service-level authorization policies.
- [ ] Protect write operations with the appropriate scopes.
- [ ] Enforce ownership checks in business logic.
- [ ] Keep service checks even when traffic normally goes through the gateway.
- [ ] Do not add passwords, passkeys, or refresh tokens to service databases.
- [ ] Do not use `AllowAnonymous()` on an endpoint that should be protected.

## Scalar and OpenAPI

- [ ] Confirm Blog API exposes `/openapi/v1.json`.
- [ ] Confirm Chat Service exposes `/openapi/v1.json`.
- [ ] Confirm Scalar lists Gateway API, Blog API, and Chat Service as separate documents.
- [ ] Confirm Blog API and Chat Service are running before loading gateway Scalar.
- [ ] Confirm the gateway has exactly one Scalar route.
- [ ] Confirm the gateway has exactly one generated `/openapi/v1.json` route.
- [ ] Add a bearer security scheme to the aggregated document.
- [ ] Add security requirements to protected operations.
- [ ] Configure Scalar's OAuth authorization-code flow with PKCE if interactive login is needed.
- [ ] Use the `tmlabs-web` public client for browser-based Scalar testing.
- [ ] Configure the `non-prod` realm authorization URL at `http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/auth`.
- [ ] Configure the `non-prod` realm token URL at `http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/token`.
- [ ] Configure gateway `Authentication:Authority` as `http://192.168.1.17:8080/realms/non-prod`.
- [ ] Configure the `tmlabs-web` client with authentication off and Standard Flow on.
- [ ] Configure the `tmlabs-web` client for Authorization Code + PKCE with no client secret.
- [ ] Verify Scalar's redirect URI is registered in Keycloak.
- [ ] Register `http://localhost:5020/scalar/*` as a Valid redirect URI, or use the actual Scalar URL.
- [ ] Register `http://localhost:5020` as the `tmlabs-web` Web origin, without a path or trailing slash.
- [ ] Verify the Keycloak token response includes `Access-Control-Allow-Origin` for `http://localhost:5020`.
- [ ] Retry in a private browser window after changing Keycloak CORS settings.
- [ ] Add the temporary `/api/auth/me` endpoint to inspect the authenticated user and claims.
- [ ] Mark `/api/auth/me` as protected with `.RequireAuthorization()`.
- [ ] Confirm `/api/auth/me` appears in the generated Gateway API document.
- [ ] Confirm Scalar's `Bearer` scheme and operation `security` metadata are present.
- [ ] Test `http://localhost:<gateway-port>/scalar`.

## Aspire and local development

- [ ] Start the full system with `dotnet run --project src/TmLabs.AppHost`.
- [ ] Confirm Aspire supplies reachable Blog and Chat endpoint URLs.
- [ ] Do not rely on `blogapi` or `chatservice` DNS names when running the gateway binary directly.
- [ ] If running services directly, use their localhost HTTP ports in development configuration.
- [ ] Verify gateway logs contain actual reachable service URLs.
- [ ] Verify PostgreSQL is running and migrations are applied.

## Verification

- [ ] Keycloak discovery endpoint returns successfully.
- [ ] Public health endpoint works without a token.
- [ ] Protected endpoint returns `401` without a token.
- [ ] Protected endpoint succeeds with a valid token.
- [ ] Missing scope or role returns `403`.
- [ ] Wrong issuer is rejected.
- [ ] Wrong audience is rejected.
- [ ] Expired token is rejected.
- [ ] Scalar can load the aggregated document.
- [ ] Scalar can authorize and call a protected endpoint.
- [ ] Scalar redirects to Keycloak and returns to the gateway without `invalid_redirect_uri`.
- [ ] `/api/auth/me` returns the expected subject, roles, scopes, and claims.
- [ ] Blog API and Chat Service receive the caller identity claims.

## Production hardening

- [ ] Use HTTPS everywhere.
- [ ] Store secrets in deployment secret storage.
- [ ] Restrict Keycloak administration access.
- [ ] Enable MFA for administrators.
- [ ] Configure exact CORS origins.
- [ ] Use short-lived access tokens.
- [ ] Configure refresh-token rotation.
- [ ] Review token claims and role mappings.
- [ ] Add rate limiting and audit logging where required.
- [ ] Add automated authentication and authorization tests.
