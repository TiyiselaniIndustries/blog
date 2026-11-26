---
title: Keycloak Authentication and Scalar Setup
description: This guide adapts the Keycloak and Scalar approach to the TmLabs backend.
series: Identity and access
seriesOrder: 3
publishedAt: 2026-11-25T08:17:00Z
draft: false
---

This guide adapts the Keycloak and Scalar approach to the TmLabs backend.

## Target architecture

```text
Next.js frontend
      |
      | OIDC Authorization Code + PKCE
      v
Keycloak
      |
      | JWT access token
      v
TmLabs Gateway
      |
      +--> Blog API
      |
      +--> Chat Service
```

Keycloak is the identity provider, not another TmLabs business microservice.

- Keycloak owns users, credentials, social login, passkeys, sessions, and tokens.
- The gateway validates bearer access tokens.
- Blog API and Chat Service enforce domain authorization.
- Scalar documents and tests the API; it is not an authentication provider.

Use Authorization Code + PKCE for the Next.js application. Do not use the OAuth Implicit Flow for a new application because it exposes tokens in the browser redirect and is no longer the preferred flow.

## 1. Keycloak realm

Create a realm for this application, for example:

```text
tmlabs
```

Record the issuer URL. For a local Keycloak instance it will look like:

```text
http://<keycloak-host>:8080/realms/tmlabs
```

For production, use HTTPS:

```text
https://auth.example.com/realms/tmlabs
```

The issuer URL must match the `iss` claim in access tokens exactly.

## 2. Keycloak clients

Create separate clients for the frontend and API.

### API client

Create an OpenID Connect client for the API:

```text
Client ID: tmlabs-api
Client authentication: On or confidential, depending on the Keycloak version
Standard flow: Off
Direct access grants: Off
```

The API does not perform browser login. Its audience is used to identify tokens intended for the TmLabs backend.

### Frontend client

Create a public OpenID Connect client for Next.js:

```text
Client ID: tmlabs-web
Client authentication: Off
Standard flow: On
Direct access grants: Off
Valid redirect URIs: http://localhost:3000/*
Web origins: http://localhost:3000
```

For production, replace the localhost values with the real HTTPS frontend origin. Never put a client secret in a browser application.

## 2.1 Add GitHub sign-in through Keycloak

GitHub should be configured as an identity provider in Keycloak. The browser does not sign in directly to GitHub for this application:

```text
Scalar or Next.js
  |
  v
Keycloak -> GitHub
  |
  v
Keycloak access token for tmlabs-api
```

### Create the GitHub OAuth app

In GitHub, open **Settings > Developer settings > OAuth Apps > New OAuth App**. If this is an organization-managed account, use the organization policy and approval process required by that organization.

For the current local/non-production Keycloak instance, use:

```text
Application name: TmLabs Keycloak non-prod
Homepage URL: http://192.168.1.17:8080
Authorization callback URL: http://192.168.1.17:8080/realms/non-prod/broker/github/endpoint
```

The callback URL is a Keycloak URL, not the Scalar URL. GitHub redirects to Keycloak first; Keycloak then redirects to Scalar or the frontend.

Click **Register application**, then generate a client secret. Copy the GitHub **Client ID** and **Client secret** immediately. Store the secret in a password manager or Kubernetes Secret. Do not commit it to this repository, `appsettings.json`, or a browser client.

For production, use the public HTTPS Keycloak hostname:

```text
Homepage URL: https://auth.example.com
Authorization callback URL: https://auth.example.com/realms/non-prod/broker/github/endpoint
```

The callback URL must use the exact Keycloak hostname that appears in the browser and in the GitHub OAuth app configuration. A callback through an IP address and a callback through a DNS name are different URLs.

### Configure GitHub in Keycloak

In the Keycloak Admin Console:

1. Select the `non-prod` realm.
2. Open **Identity providers**.
3. Click **Add provider** and choose **GitHub**.
4. Set **Alias** to `github`.
5. Set **Display name** to `GitHub`.
6. Paste the GitHub OAuth app **Client ID**.
7. Paste the GitHub OAuth app **Client secret**.
8. Leave the provider enabled.
9. Set **Trust email** only if GitHub email verification is acceptable for your account policy.
10. Set **Store tokens** to disabled unless the application must call GitHub after login.
11. Save the provider.

Keycloak displays a **Redirect URI** or **Callback URL** for the provider. It must match the GitHub OAuth app callback exactly:

```text
http://192.168.1.17:8080/realms/non-prod/broker/github/endpoint
```

Use the scopes `read:user` and `user:email` if the provider exposes a scope field. `read:user` supplies the GitHub profile and `user:email` allows Keycloak to retrieve the account email when it is not public.

### Test GitHub sign-in

1. Open Scalar at `http://localhost:5020/scalar` or open the frontend.
2. Click **Authorize** and start the Keycloak Authorization Code + PKCE flow.
3. On the Keycloak login page, choose **Sign in with GitHub**.
4. Approve the GitHub consent screen.
5. Confirm GitHub redirects back to Keycloak, and Keycloak redirects back to Scalar.
6. Execute `/api/auth/me`.
7. Confirm the response contains a new `sub`, GitHub username or display name, and email when available.
8. Confirm the access token still contains `aud` with `tmlabs-api`.
9. Assign the resulting Keycloak user the required realm roles under **Users > [user] > Role mapping**.
10. Authorize again and confirm the expected roles appear in `/api/auth/me`.

The first GitHub login normally creates a Keycloak user. Do not assume a GitHub account and an existing Keycloak user are linked automatically. Configure Keycloak account linking or link them from the user's account settings after verifying that the email belongs to the same person. Never use email alone as proof of identity when linking accounts unless the provider and verification policy support it.

If GitHub reports `redirect_uri_mismatch`, compare the callback URL character-for-character with the URL shown by Keycloak. If Keycloak reports an invalid client or the provider returns to an error page, verify the GitHub client ID and secret and make sure the OAuth app is not disabled.

## 2.2 Add Google sign-in through Keycloak

Google should also be configured as a Keycloak identity provider. Scalar or the frontend redirects to Keycloak, Keycloak redirects to Google, and Google returns to Keycloak before Keycloak issues the TmLabs access token.

### Create Google OAuth credentials

Open Google Cloud Console:

```text
https://console.cloud.google.com/
```

Select or create the TmLabs Google Cloud project. The current Google Cloud OAuth console uses these sections in the left navigation:

```text
Google Cloud Console
  -> Branding
  -> Audience
  -> Clients
```

Configure them in this order:

1. Open **Branding**.
2. Set the application name to `TmLabs`.
3. Set the user support email and developer contact email.
4. Save the branding configuration.
5. Open **Audience**.
6. Choose **External**, unless only users in your Google Workspace organization should be allowed.
7. Add your Google account under **Test users** while the app is in testing mode.
8. Save the audience configuration.
9. Open **Clients**.
10. Click **Create OAuth client**.
11. Choose **Web application**.
12. Set the name to `TmLabs Keycloak non-prod`.

The equivalent older navigation, still shown in some Google Cloud projects, is:

```text
Google Cloud Console
  -> APIs & Services
  -> OAuth consent screen
```

After configuring Branding and Audience, create the OAuth client from:

```text
Google Cloud Console
  -> Clients
  -> Create OAuth client
```

Some projects expose the same page at:

```text
Google Cloud Console
  -> APIs & Services
  -> Credentials
  -> Create credentials
  -> OAuth client ID
```

Choose **Web application**. The Google Cloud credentials page is also available at:

```text
https://console.cloud.google.com/apis/credentials
```

Before creating the Google credential, open Keycloak in another tab:

```text
http://192.168.1.17:8080/admin/
```

Then navigate:

```text
Keycloak Admin Console
  -> select the non-prod realm
  -> Identity providers
  -> Add provider
  -> Google
```

Copy the **Redirect URI** shown on this Keycloak page. Keep this page open, because you will return to it to paste the Google client ID and secret.

After clicking **Create OAuth client** in Google Cloud, paste the Keycloak redirect URI into **Authorized redirect URIs**. Do not use the Scalar URL here. Google redirects to Keycloak first:

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

After Google creates the client, copy its **Client ID** and **Client secret**. Return to the open Keycloak Google identity-provider page and paste both values, then save.

The complete Google Cloud sequence is:

1. Select the TmLabs Google Cloud project.
2. Configure **Branding**.
3. Configure **Audience** as **External** and add your test user.
4. Open **Clients**.
5. Click **Create OAuth client**.
6. Choose **Web application**.
7. Set the name to `TmLabs Keycloak non-prod`.

In Keycloak, first open **Identity providers > Add provider > Google** and copy the **Redirect URI** displayed by Keycloak. Paste that exact value into Google Cloud under **Authorized redirect URIs**. For your current realm, Keycloak will normally display:

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

Do not use the Scalar URL as the Google callback. Google redirects to Keycloak first. Keycloak then redirects to Scalar using the original OAuth state.

Google requires an exact redirect URI. A raw private IP such as `http://192.168.1.17:8080/...` may be rejected because Google does not generally allow arbitrary IP hosts for OAuth web credentials. Use an HTTPS DNS name for Keycloak, such as `https://auth.example.com`, with a certificate trusted by the browser. For local-only experiments, `http://localhost` is treated specially by Google, but it points to the local computer running the browser, not to a separate Keycloak LXC. Do not substitute the LXC IP for `localhost`.

The Google callback host and the Keycloak `Authority` must use the same public Keycloak hostname. If you introduce `https://auth.example.com`, update the realm issuer and gateway configuration to use:

```text
https://auth.example.com/realms/non-prod
```

Copy the Google client ID and client secret. Store the secret in a password manager or Kubernetes Secret. Do not commit it to this repository, `appsettings.json`, or a browser application.

For production, create or update the Google credential with the public HTTPS Keycloak hostname:

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

Google redirect URIs must match exactly, including the scheme, host, realm name, path, and trailing characters. If you use both local and production Keycloak instances, add each callback as a separate authorized redirect URI or create separate Google OAuth credentials.

### Configure Google in Keycloak

In the Keycloak Admin Console:

1. Select the `non-prod` realm.
2. Open **Identity providers**.
3. Click **Add provider** and choose **Google**.
4. Set **Alias** to `google`.
5. Set **Display name** to `Google`.
6. Paste the Google OAuth **Client ID**.
7. Paste the Google OAuth **Client secret**.
8. Leave the provider enabled.
9. Set **Trust email** only if Google email verification is acceptable for your account policy.
10. Set **Store tokens** to disabled unless the application needs to call Google after login.
11. Save the provider.

Keycloak displays the provider redirect URI. Confirm it is exactly the URI copied into Google Cloud. With the production-style hostname above, it is:

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

The default Google provider normally requests the identity information needed for this flow. If Keycloak exposes a scope setting, use:

```text
openid email profile
```

### Test Google sign-in

1. Open Scalar at `http://localhost:5020/scalar` or open the frontend.
2. Click **Authorize** and start the Keycloak Authorization Code + PKCE flow.
3. On the Keycloak login page, choose **Sign in with Google**.
4. Select a Google test user authorized on the OAuth consent screen.
5. Approve consent if Google asks for it.
6. Confirm Google redirects to Keycloak, then Keycloak redirects to Scalar.
7. Execute `/api/auth/me`.
8. Confirm the response contains the Google subject, email, and profile claims when available.
9. Confirm the access token contains `aud` with `tmlabs-api`.
10. Assign the resulting Keycloak user the required realm roles.
11. Authorize again and confirm the expected roles appear in `/api/auth/me`.

The first Google login normally creates a Keycloak user. Do not assume a Google account and an existing Keycloak user are linked automatically. Use Keycloak account linking only after verifying the identity according to your account policy.

If Google reports `redirect_uri_mismatch`, compare the Google credential's authorized redirect URI with Keycloak's displayed provider redirect URI character-for-character, including scheme, host, port, path, and trailing slash. If Google rejects the raw IP host, move Keycloak behind an HTTPS DNS hostname. If Google reports `access blocked`, add the account as a test user or publish the consent screen. If Keycloak reports an invalid client, verify the Google client ID and secret.

## 2.3 Configure passkeys

Passkeys are configured in Keycloak through WebAuthn. They are not another OAuth identity provider. Keycloak registers and validates the passkey, then issues the normal TmLabs access token.

### Keycloak passkey setup

In the `non-prod` realm:

1. Go to **Authentication > Required actions**.
2. Enable `Webauthn Register Passwordless`.
3. Enable `Webauthn Register` if passkeys will also be used as a second factor.
4. Go to **Authentication > Policies > WebAuthn Passwordless Policy**.
5. Set **Relying Party Entity Name** to `TmLabs`.
6. Set **Relying Party ID** to the effective Keycloak domain, for example `auth.example.com`.
7. Set **User Verification Requirement** to `Preferred` or `Required`.
8. Set **Discoverable Credential** to `Required` for passwordless passkeys.
9. Go to **Realm settings > Login** and enable **Passkeys** if that switch is available in your Keycloak version.
10. Confirm the browser authentication flow includes **WebAuthn Passwordless Authenticator**.

Users register from the Keycloak Account Console:

```text
https://auth.example.com/realms/non-prod/account

Account security
  -> Signing in
  -> Set up a Passkey
```

Passkeys require a secure browser context. Use HTTPS with a real hostname for deployed testing. `localhost` is generally treated as a secure development origin, but `http://192.168.1.17` should not be used as the long-term passkey origin.

### Passkey testing

1. Log in through Scalar, Google, GitHub, or a local Keycloak account.
2. Open the Keycloak Account Console.
3. Register a passkey using the browser, device, phone, or security key.
4. Log out.
5. Start login again and select the passkey option.
6. Confirm `/api/auth/me` returns the same Keycloak subject and expected roles.

## 2.4 Configure authenticator apps with TOTP

Authenticator apps use TOTP. This is not an OAuth provider. Keycloak generates a shared secret, displays a QR code, and validates the six-digit codes generated by the authenticator app.

Compatible apps include Google Authenticator, Microsoft Authenticator, 1Password, Bitwarden, and Authy.

### Keycloak TOTP setup

In the `non-prod` realm:

1. Go to **Authentication > Policies > OTP Policy**.
2. Configure:

```text
Type: totp
Algorithm: SHA256 or SHA1 for maximum compatibility
Digits: 6
Period: 30 seconds
```

3. Go to **Authentication > Required actions**.
4. Enable `Configure OTP`.
5. In the **Browser** authentication flow, locate the conditional 2FA flow.
6. Set `OTP Form` to `Alternative` when users may choose OTP or passkey.
7. Set `OTP Form` to `Required` when every user must use an authenticator app.
8. Enable `Recovery Authentication Codes` as a fallback if required.

Users register an authenticator app from:

```text
Account security
  -> Signing in
  -> Set up Authenticator application
```

The user scans the QR code, enters the current six-digit code, and Keycloak stores the TOTP credential. Do not store the TOTP secret in Blog API, Chat Service, or the frontend.

### Authenticator app testing

1. Log in to the Keycloak Account Console.
2. Register an authenticator application.
3. Log out and start a new login.
4. Enter the generated six-digit code when prompted.
5. Execute `/api/auth/me` through Scalar.
6. Confirm that the JWT issuer, audience, subject, and roles are unchanged.

## 2.5 Configure Apple Sign in with Apple

Apple sign-in requires an Apple Developer account and is best configured after Keycloak has a stable public HTTPS hostname. Apple requires an exact callback URL and uses a short-lived signed JWT as the client secret.

Required Apple values:

```text
Team ID
Key ID
Apple private key (.p8)
Services ID
```

The broker callback should be based on the public Keycloak hostname:

```text
https://auth.example.com/realms/non-prod/broker/apple/endpoint
```

### Apple setup

1. Open the **Apple Developer portal**.
2. Create or select an **App ID** and enable **Sign in with Apple**.
3. Create a **Services ID** for web sign-in.
4. Configure the Services ID return URL with the exact Keycloak callback above.
5. Create a Sign in with Apple key.
6. Record the Team ID and Key ID.
7. Download the `.p8` private key once and store it in a Kubernetes Secret or a password manager.
8. In Keycloak, open **Identity providers**.
9. Add the Apple provider if your Keycloak version includes it. Otherwise use the generic **OpenID Connect v1.0** provider with Apple’s authorization, token, and user-info endpoints.
10. Configure the Apple client ID as the Services ID.
11. Configure the Apple client secret as a signed JWT generated with the Apple private key, Team ID, Key ID, and the Apple token endpoint as the audience.
12. Use scopes `openid email name` where supported.
13. Save and test the provider.

Apple client-secret JWTs expire and must be rotated. Do not commit the `.p8` key or generated client secret to this repository or place them in a browser client.

Apple may provide a private relay email address. Treat it as the user’s email identifier, but do not assume it can be used to link accounts automatically without an explicit account-linking flow.

## 2.6 Account management and an in-app user directory

Keycloak and TmLabs have separate responsibilities.

Keycloak should manage:

- password changes and recovery
- GitHub, Google, and Apple links
- passkeys, TOTP, and recovery codes
- active sessions and logout
- identity-provider users, roles, and groups

Users can manage these through the Keycloak Account Console:

```text
https://auth.example.com/realms/non-prod/account
```

Your application should manage its own domain data, such as chat profiles, comments, blog ownership, preferences, and moderation data. Link that data to the authenticated Keycloak subject.

### Listing users inside TmLabs

Do not call the Keycloak Admin API directly from the browser. Do not give normal users an admin token. Instead, create an admin-only endpoint in the service that owns the application user table, then call it through the gateway:

```text
Next.js admin page
  -> GET /api/chat/users
  -> Gateway authorization policy
  -> Chat Service user query
```

A suitable endpoint contract is:

```text
GET /api/chat/users?page=1&pageSize=25&search=alex
Authorization: Bearer <user-access-token>
```

Require the `admin` role or a dedicated `users:read` permission. Prefer a dedicated permission for a public application:

```text
users:read
users:manage
```

Return a safe public/admin view, not raw identity-provider data or tokens:

```json
{
  "items": [
    {
      "id": "local-user-id",
      "username": "alex",
      "displayName": "Alex Example",
      "avatarUrl": "https://...",
      "createdAtUtc": "2026-09-19T12:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 25,
  "totalCount": 1
}
```

Do not return these fields from a general user-list endpoint:

- access tokens or refresh tokens
- passwords or password metadata
- passkey credentials
- TOTP secrets
- provider client secrets
- the complete raw JWT claims collection

Use pagination, a bounded `pageSize`, and server-side search. If users can view a public directory, create a separate public response model with only fields such as display name and avatar. Keep email, roles, moderation state, and login metadata admin-only.

### Linking a request to a local user

Use the JWT `sub` claim as the stable identity key. It is the same application identity whether the user logged in with a password, GitHub, Google, Apple, or a passkey.

The Chat Service currently has a provider-specific `GitHubUserId`. Replace that concept with a provider-neutral field:

```text
KeycloakSubject: string, required, unique, indexed
```

Treat the subject as opaque. Do not use email, username, display name, or a GitHub/Google provider ID as the primary key.

On an authenticated request, the service should:

1. Read `sub` from the validated `ClaimsPrincipal`.
2. Find or create the local user by `KeycloakSubject`.
3. Use the local `User.Id` for comments, posts, memberships, and ownership.
4. Check roles and resource ownership before returning or changing data.

For example, a protected user-list endpoint should conceptually be configured as:

```csharp
Get("/users");
Roles("admin");
```

The exact syntax depends on the FastEndpoints endpoint class. The important rule is that authorization is enforced in the service, not only hidden in the frontend.

### Keycloak users versus application users

These are not necessarily the same list:

- **Keycloak user list:** every identity known to the realm, including users who have never used TmLabs.
- **TmLabs user list:** users who have signed in and have a local application profile.

For normal in-app features, list the local TmLabs users. If an administrator truly needs to manage every Keycloak account, create a separate backend-only integration using a confidential service account with narrowly scoped Keycloak Admin API roles. Never expose that service account or Admin API token to the frontend.

### Ownership and deletion

Store local foreign keys on domain records:

```text
Comment.AuthorUserId -> User.Id
BlogPost.OwnerUserId -> User.Id
```

Never accept an owner ID from the request body for an ownership-sensitive operation. Derive the current user from the bearer token. Decide separately whether deleting a Keycloak account should anonymize, soft-delete, or remove the user’s TmLabs data.

## 3. Scopes and roles

Use roles first because ASP.NET Core maps Keycloak roles predictably when they are present in `realm_access.roles`. In Keycloak, open the `tmlabs` realm, choose **Realm roles**, and create:

```text
user
admin
```

For the first implementation, create the API permissions as realm roles too. Open **Realm roles** and create:

```text
blog:read
blog:write
chat:read
chat:write
```

Assign roles to a test user through **Users > [test user] > Role mapping**. Assign `user` and `blog:read` first. Assign `blog:write`, `chat:write`, or `admin` only when needed.

Confirm that the built-in **roles** client scope is assigned to the API client. The access token should then contain:

```json
{
  "realm_access": { "roles": ["user", "admin"] },
  "resource_access": {
    "tmlabs-api": { "roles": ["blog:read", "blog:write"] }
  }
}
```

Use `RequireRole("admin")` and `RequireRole("blog:read")` for this first version. These roles will be read from `realm_access.roles` by ASP.NET Core. Do not assume a policy such as `RequireClaim("scope", "blog:read")` works: Keycloak normally emits `scope` as one space-delimited value, for example `"openid profile blog:read"`. If you later move permissions to client roles, they normally appear under `resource_access.tmlabs-api.roles` and require a claims transformation before `RequireRole()` can see them.

To request custom scopes, open **Clients > tmlabs-api > Client scopes**, create `blog` and `chat` scopes if needed, add them as optional or default scopes, and map them into the token. Keep the first implementation role-based until token claims are confirmed.

### Add the API audience to the web token

The gateway is configured with:

```json
"Audience": "tmlabs-api"
```

Therefore, the access token obtained through `tmlabs-web` must contain `tmlabs-api` in its `aud` claim. If Scalar returns `401` with an error such as `The audience '(null)' is invalid`, the token has no audience.

Add the audience in Keycloak:

1. Select the `non-prod` realm.
2. Open **Client scopes** and click **Create client scope**.
3. Set **Name** to `tmlabs-api-audience`.
4. Set **Type** to `Default` and **Protocol** to `openid-connect`.
5. Save the client scope.
6. Open its **Mappers** tab and click **Configure a new mapper**.
7. Choose **Audience** or **Audience mapper**.
8. Set **Included Client Audience** to `tmlabs-api`.
9. Enable **Add to access token** and save the mapper.
10. Open **Clients > tmlabs-web > Client scopes**.
11. Add `tmlabs-api-audience` as a **Default** client scope.
12. Log out of Scalar and authorize again to obtain a fresh token.

Depending on the Keycloak version, the mapper may be named **Audience** and the field may be **Included Client Audience**, or it may be under **Configure a new mapper > By configuration > Audience**. The configured audience value must be exactly `tmlabs-api`.

Decode the new access token and confirm that it contains:

```json
{
  "aud": ["tmlabs-api"]
}
```

Keycloak may serialize one audience as either a string or a one-item array. Both are valid. The important point is that `tmlabs-api` is present. Do not edit the token manually; obtain a new token after changing the mapper.

## 4. Gateway configuration

Add JWT bearer authentication to `TmLabs.Gateway`:

```bash
dotnet add src/TmLabs.Gateway/TmLabs.Gateway.csproj package Microsoft.AspNetCore.Authentication.JwtBearer
```

Configure the gateway with environment variables or user secrets. Do not commit production values:

```text
Authentication__Authority=https://auth.example.com/realms/tmlabs
Authentication__Audience=tmlabs-api
Authentication__RequireHttpsMetadata=true
```

The committed appsettings files contain dummy, non-routable values so they are safe to keep in source control:

```json
{
  "Authentication": {
    "Authority": "https://keycloak.example.invalid/realms/tmlabs",
    "Audience": "tmlabs-api",
    "RequireHttpsMetadata": true
  }
}
```

For local HTTP-only Keycloak, `appsettings.Development.json` uses a dummy HTTP authority and sets `RequireHttpsMetadata` to `false`. Replace that authority with the real URL of the Keycloak LXC, for example `http://192.168.1.50:8080/realms/tmlabs`. The URL must be reachable from the gateway process and must exactly match the token issuer.

For Kubernetes, leave the JSON placeholders in the image and override them with environment variables. ASP.NET Core converts double underscores to nested configuration sections:

```yaml
env:
  - name: Authentication__Authority
    value: "https://auth.example.com/realms/tmlabs"
  - name: Authentication__Audience
    value: "tmlabs-api"
  - name: Authentication__RequireHttpsMetadata
    value: "true"
```

Prefer a Kubernetes `ConfigMap` for the authority and audience and a `Secret` for any confidential server-side client value. This gateway only validates bearer tokens, so it does not need a Keycloak client secret. Do not put access tokens, refresh tokens, or Keycloak admin credentials in `appsettings.json`.

The three settings mean:

| Setting | Value | Purpose |
| --- | --- | --- |
| `Authentication:Authority` | `https://auth.example.com/realms/tmlabs` | Keycloak issuer/realm. ASP.NET downloads signing keys from its discovery endpoint. |
| `Authentication:Audience` | `tmlabs-api` | Accept only tokens intended for this API. |
| `Authentication:RequireHttpsMetadata` | `true` in Kubernetes/production | Require HTTPS for Keycloak discovery and signing-key metadata. |

The current gateway does not yet read these settings because JWT registration has not been added to `Program.cs`. After adding the registration below, a wrong or unreachable authority will prevent token validation and may prevent the service from starting or validating requests.

Register authentication and authorization before building the application:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Authentication:Authority"];
        options.Audience = builder.Configuration["Authentication:Audience"];
        options.RequireHttpsMetadata = builder.Configuration.GetValue(
            "Authentication:RequireHttpsMetadata", true);
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("BlogRead", policy =>
        policy.RequireClaim("scope", "blog:read"));

    options.AddPolicy("BlogWrite", policy =>
        policy.RequireClaim("scope", "blog:write"));

    options.AddPolicy("ChatWrite", policy =>
        policy.RequireClaim("scope", "chat:write"));

    options.AddPolicy("Admin", policy =>
        policy.RequireRole("admin"));
});
```

Add the middleware before endpoint mapping:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Protect routes explicitly. For example, health checks can remain public, while write routes require a policy:

```csharp
app.MapGet("/api/blog/posts", ...)
    .RequireAuthorization("BlogRead");
```

YARP routes also need authorization metadata if the gateway is responsible for blocking unauthenticated requests before forwarding. Apply authorization to protected proxy routes, or expose only protected downstream endpoints and enforce the same policy in each service.

## 5. Service authorization

The gateway is not the only security boundary. Blog API and Chat Service should validate tokens and authorize their own business operations, especially if they can be reached without the gateway in another deployment.

Each service should:

1. Configure the same `Authority` and `Audience`.
2. Call `UseAuthentication()` and `UseAuthorization()`.
3. Require policies on protected FastEndpoints.
4. Check ownership and resource-level permissions in application code.

Do not add password storage or passkey storage to either service.

Also avoid combining `AllowAnonymous()` with a policy. `AllowAnonymous()` bypasses authorization.

## 6. Scalar API documentation

Scalar can display three separate OpenAPI documents:

```text
Gateway:      http://localhost:<gateway-port>/openapi/v1.json
Blog API:     http://localhost:<gateway-port>/api/blog/openapi/v1.json
Chat Service: http://localhost:<gateway-port>/api/chat/openapi/v1.json
```

The current gateway UI is:

```text
http://localhost:<gateway-port>/scalar
```

The gateway does not merge the documents. Scalar presents the Gateway, Blog API, and Chat Service documents as separate selectable entries. The Blog and Chat documents are served through the gateway's existing YARP routes, so Blog API and Chat Service must be running before those entries can load.

For local Aspire development, open the **Gateway Scalar URL** shown by Aspire. The gateway talks to Blog API and Chat Service over their internal HTTP endpoints. The downstream services intentionally do not redirect HTTP to HTTPS in Development; otherwise a request such as `/api/blog/openapi/v1.json` is redirected to a service HTTPS port and Scalar cannot load the document through the gateway. HTTPS redirection remains enabled outside Development.

The gateway rewrites the downstream OpenAPI `servers` value to `/api/blog` or `/api/chat`. This keeps Scalar's operation requests on the same gateway origin and ensures `/health` in the Blog document calls `/api/blog/health` rather than the standalone Blog API port.

If the Blog API document shows `could not be loaded`, verify these URLs from the gateway host:

```bash
curl http://localhost:<gateway-port>/api/blog/openapi/v1.json
curl http://localhost:<gateway-port>/api/chat/openapi/v1.json
```

Both requests should return JSON with an `openapi` property. Do not use the Blog API's direct `/scalar` URL when testing the combined Gateway, Blog API, and Chat Service documents.

### 6.1 Add a temporary identity test endpoint

Add this endpoint to `src/TmLabs.Gateway/Program.cs` after `var app = builder.Build();` and after the authentication middleware is registered. It returns the authenticated subject, useful identity claims, roles, and scopes:

```csharp
using System.Security.Claims;

app.MapGet("/api/auth/me", (ClaimsPrincipal user) =>
{
  var scopes = user.FindFirst("scope")?.Value?
    .Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
    ?? [];

  var roles = user.Claims
    .Where(claim => claim.Type is ClaimTypes.Role or "role" or "roles")
    .Select(claim => claim.Value)
    .Distinct(StringComparer.Ordinal)
    .ToArray();

  return Results.Ok(new
  {
    IsAuthenticated = user.Identity?.IsAuthenticated ?? false,
    Subject = user.FindFirst("sub")?.Value,
    Username = user.FindFirst("preferred_username")?.Value,
    Email = user.FindFirst("email")?.Value,
    Roles = roles,
    Scopes = scopes,
    Claims = user.Claims.Select(claim => new
    {
      claim.Type,
      claim.Value
    })
  });
})
.RequireAuthorization();
```

Call it through the gateway:

```bash
curl -i \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:<gateway-port>/api/auth/me
```

Expected results:

- No token: `401 Unauthorized`.
- Valid token: `200 OK` with `Subject`, `Roles`, `Scopes`, and `Claims`.
- Valid token without the required role: `200 OK` for this diagnostic endpoint; use a role policy on real business endpoints to test `403 Forbidden`.

This endpoint is for development and testing. It returns claims and should not be exposed publicly in production. Remove it or protect it behind an administrator-only policy before deployment. Do not return raw claims in a public user profile endpoint.

The Gateway document is generated automatically by `app.MapOpenApi()`. Therefore gateway-only endpoints such as `/api/auth/me` appear there automatically when they have endpoint metadata such as `.RequireAuthorization()`. Blog and Chat endpoints appear in their respective service documents.

To show an authentication button in Scalar, the aggregated OpenAPI document must contain a bearer security scheme. The document should describe the API scheme like this:

```json
{
  "components": {
    "securitySchemes": {
      "Bearer": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT"
      }
    }
  }
}
```

Protected operations should include a security requirement:

```json
{
  "security": [
    {
      "Bearer": []
    }
  ]
}
```

### 6.2 Configure the Scalar authorization button

There are two ways to test with Scalar:

1. **Bearer token mode:** configure the bearer security scheme above, click **Authorize** in Scalar, and paste the access token. This is the simplest first test.
2. **OAuth login mode:** configure Scalar with the public `tmlabs-web` client and Authorization Code + PKCE. This lets Scalar redirect to Keycloak instead of requiring a pasted token.

For bearer token mode, Scalar only needs the OpenAPI `Bearer` security scheme and a `security` entry on protected operations. The `/api/auth/me` endpoint must be marked as protected in the OpenAPI document, otherwise Scalar will not show it as requiring authorization.

The gateway currently adds a Keycloak OAuth2 authorization-code scheme to its generated OpenAPI document in `TmLabs.Gateway/Program.cs`. The scheme is what tells Scalar that the **Authorize** button can start an OAuth login instead of only accepting a pasted bearer token.

For the current non-production Keycloak instance, the URLs are:

```text
http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/auth
```

Token URL:

```text
http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/token
```

The discovery document for this realm is:

```text
http://192.168.1.17:8080/realms/non-prod/.well-known/openid-configuration
```

Open the discovery URL in a browser first. It must return JSON and must list the same authorization and token endpoints. The gateway's JWT `Authority` setting must be the issuer base URL, without `/protocol/openid-connect`:

```text
http://192.168.1.17:8080/realms/non-prod
```

Do not write `https://http://192.168.1.17:8080`. The scheme appears exactly once. Use `http` only for this local/non-production setup; use `https` in Kubernetes and production.

#### Keycloak client required by Scalar

In the `non-prod` realm, create or edit the public client:

```text
Client ID: tmlabs-web
Client authentication: Off
Standard flow: On
Direct access grants: Off
``` 

The client must not have a secret because Scalar runs in the browser. Authorization Code + PKCE uses a code verifier instead of a client secret.

Register the URL where the gateway Scalar UI runs. For the standard gateway HTTP profile:

```text
Valid redirect URIs: http://localhost:5020/scalar/*
Web origins: http://localhost:5020
```

These two settings do different jobs:

- **Valid redirect URIs** controls where Keycloak may redirect the browser after login.
- **Web origins** controls whether browser JavaScript served by Scalar may call Keycloak's token endpoint.

The Web Origins value must be the origin only: scheme, host, and port, with no path and no trailing slash. For this setup it is exactly:

```text
http://localhost:5020
```

If Web Origins is missing or incorrect, login may complete in Keycloak but Scalar fails during the authorization-code exchange with a browser error such as `No 'Access-Control-Allow-Origin' header` on the `/protocol/openid-connect/token` request.

If Aspire assigns a different gateway port, replace `5020` with that port. If using HTTPS, register the HTTPS origin separately, for example:

```text
Valid redirect URIs: https://localhost:7020/*
Web origins: https://localhost:7020
```

Use the narrowest callback path accepted by the Scalar version in use. A wildcard is convenient for local development but should be replaced with the exact Scalar callback URL for production.

Add these client scopes or make them available to the client:

```text
openid
profile
email
```

Ensure the access token also has the `tmlabs-api` audience. Configure an audience mapper or client scope in Keycloak if `aud` does not contain `tmlabs-api`.

For local testing, Scalar may be configured with a public client and a redirect URI accepted by Keycloak. Keep this separate from production credentials and origins. If the gateway uses another port, replace `5020` in both settings with that port. If using HTTPS, register the HTTPS origin separately:

```text
Valid redirect URIs: https://localhost:7020/scalar/*
Web origins: https://localhost:7020
```

If Scalar uses a different callback path, register that exact callback URI instead of the wildcard. Check the browser network log if Keycloak rejects the redirect with `invalid_redirect_uri`.

The OAuth settings must use the public browser client, not the confidential `tmlabs-api` client. The browser client has no secret. The values used by the generated OpenAPI document should be:

```text
Authorization endpoint:
http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/auth

Token endpoint:
http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/token

Scopes:
openid profile email

Client ID:
tmlabs-web

PKCE:
Enabled
```

Request the API audience as well if it is configured as a client scope or audience mapper in Keycloak. After login, decode the returned access token and confirm:

```text
iss = http://192.168.1.17:8080/realms/non-prod
aud contains tmlabs-api
realm_access.roles contains user and/or blog:read
```

#### Test the login from Scalar

1. Start the complete system with `dotnet run --project src/TmLabs.AppHost`.
2. Open `http://localhost:5020/scalar` or the actual gateway URL shown by Aspire.
3. Select **Gateway API** from the document selector.
4. Select the protected `/api/auth/me` operation.
5. Click **Authorize**.
6. Choose the Keycloak OAuth scheme if Scalar displays more than one scheme.
7. Enter `tmlabs-web` if Scalar asks for the client ID.
8. Ensure PKCE is enabled and request `openid profile email`.
9. Complete login in Keycloak.
10. Approve consent if Keycloak asks for it.
11. Return to Scalar and execute `/api/auth/me`.
12. Confirm the response contains the expected `Subject`, `Roles`, and `Claims`.

If the browser console reports a CORS error on the Keycloak token request:

1. Confirm **Web origins** on `tmlabs-web` is exactly `http://localhost:5020`.
2. Confirm **Valid redirect URIs** includes `http://localhost:5020/scalar/*`.
3. Save the Keycloak client configuration.
4. Clear Scalar's authorization state or retry in a private browser window.
5. Inspect the `/token` response in browser Developer Tools. It should include `Access-Control-Allow-Origin: http://localhost:5020`.

Do not add a trailing slash to Web Origins. Do not put the `/scalar/` path in Web Origins; the path belongs only in Valid redirect URIs.

If Scalar asks for a client secret, the wrong Keycloak client was selected. Use the public `tmlabs-web` client and do not enter a secret.

To verify the OpenAPI configuration directly, open:

```text
http://localhost:5020/openapi/v1.json
```

Search the JSON for `securitySchemes`, `oauth2`, `authorizationCode`, `authorizationUrl`, and `tokenUrl`. Search the `/api/auth/me` operation for a `security` entry referencing the OAuth scheme. If the scheme exists but the operation has no `security` metadata, Scalar may not show the operation as requiring authorization.

If Scalar shows no **Authorize** button, the gateway document is missing `components.securitySchemes` or the operation is missing its `security` array. If login succeeds but the API returns `401`, inspect `iss`, `aud`, expiry, and the gateway's `Authentication:Authority` and `Authentication:Audience` values. If the API returns `403`, the token is valid but does not contain the role required by the endpoint.

## 7. Aspire and local URLs

When launched through `TmLabs.AppHost`, Aspire supplies the gateway's Blog and Chat URLs. Start the complete application with:

```bash
dotnet run --project src/TmLabs.AppHost
```

Do not launch the gateway binary alone and expect the Aspire hostnames `blogapi` and `chatservice` to resolve. A direct gateway run requires Blog API and Chat Service to run on their localhost ports and requires matching development configuration.

The gateway should log calls to actual service endpoints, not literal unresolved names such as:

```text
http://blogapi/openapi/v1.json
```

## 8. Test sequence

1. Open Keycloak's discovery document and confirm it returns JSON.
2. Create or use a test user.
3. Obtain an access token for the `tmlabs-api` audience.
4. Decode the token locally and verify `iss`, `aud`, expiry, roles, and scopes.
5. Request a public health endpoint without a token.
6. Request a protected endpoint without a token and expect `401`.
7. Request it with a valid token and expect success.
8. Request it with a valid token missing the required permission and expect `403`.
9. Open Scalar and authorize with the test client.
10. Confirm Blog and Chat requests pass through the gateway with the bearer token.

Example request:

```bash
curl -i \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:<gateway-port>/api/blog/posts
```

## Production rules

- Use HTTPS for Keycloak, frontend, gateway, and services.
- Use Authorization Code + PKCE, not Implicit Flow.
- Do not commit client secrets or access tokens.
- Use a confidential client only on a trusted server.
- Validate issuer, audience, signature, and lifetime.
- Keep service authorization checks even when traffic normally goes through the gateway.
- Configure CORS for the exact frontend origin.
- Do not expose Keycloak administration publicly without network and MFA controls.
- Do not log bearer tokens or full authorization headers.
- Use short-lived access tokens and rotate refresh tokens according to the Keycloak policy.
