---
title: Keycloak Authentication, Scalar Setup, and .NET Integration for TmLabs
description: This guide covers identity setup in Keycloak, wiring the TmLabs gateway and services to validate tokens, testing with Scalar, and linking authenticate...
series: Identity and access
seriesOrder: 4
publishedAt: 2026-11-25T08:18:00Z
draft: false
---

This guide covers identity setup in Keycloak, wiring the TmLabs gateway and services to validate tokens, testing with Scalar, and linking authenticated users to local TmLabs profiles from a Next.js frontend and .NET backend.

One realm name is used throughout: `non-prod`. If you use a different realm name, replace it consistently — the issuer URL in every service and client must match the token's `iss` claim exactly.

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

Create a realm for this application:

```text
non-prod
```

Record the issuer URL. For a local Keycloak instance it will look like:

```text
http://<keycloak-host>:8080/realms/non-prod
```

For production, use HTTPS:

```text
https://auth.example.com/realms/non-prod
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

Create a public OpenID Connect client for Next.js and Scalar:

```text
Client ID: tmlabs-web
Client authentication: Off
Standard flow: On
Direct access grants: Off
Valid redirect URIs: http://localhost:3000/*, http://localhost:5020/scalar/*
Web origins: http://localhost:3000, http://localhost:5020
```

For production, replace the localhost values with the real HTTPS origins and use the narrowest redirect path your Scalar version supports rather than a wildcard. Never put a client secret in a browser application.

## 2.1 Allow public signups

Signup should be handled by Keycloak. Do not create a password-registration endpoint in the gateway, Blog API, or Chat Service.

### Enable local Keycloak registration

In the `non-prod` realm, open:

```text
Realm settings
  -> Login
```

Enable:

```text
User registration: On
Email as username: Optional
Verify email: On for production
Login with email: On
```

When **User registration** is enabled, Keycloak adds a **Register** link to its login page. The user creates the account in Keycloak and then returns to the frontend or Scalar with a TmLabs access token.

For local development, email verification may remain off until SMTP is configured. For a public deployment, configure SMTP under:

```text
Realm settings
  -> Email
```

Then enable email verification. Do not mark an unverified email as trusted merely because it came from a signup form.

### Configure first-login behavior

Social signups through GitHub, Google, or Apple are handled by the identity provider's **First Login Flow**. The default flow can create a Keycloak user when no matching account exists and can ask the user to review or complete their profile.

Review it under:

```text
Authentication
  -> First broker login
```

Keep the default steps in place:

```text
Review Profile
Create User If Unique
  Handle Existing Account
    Confirm Link Existing Account
    Account verification options
      Verify existing account by Email      (Alternative)
      Verify Existing Account by Re-authentication (Alternative)
```

With this, a person who logs in with GitHub or Google and already has a Keycloak account with the same email is asked to prove they own the existing account — via an email link or by logging in — before the accounts are merged. This needs SMTP configured (above).

Recommended public-app behavior:

- keep **Review Profile** enabled when profile data may be incomplete
- keep account linking protected by email verification or re-authentication
- do not automatically link accounts solely because two providers return the same email — this lets anyone who controls a provider account with a matching email take over an existing account
- assign only the baseline `user` role to new accounts

Only enable **Trust email** on an identity provider if that provider verifies emails. GitHub allows unverified emails on some accounts, so leave it off for GitHub unless you add your own verification.

### Assign a baseline role to new signups

Create a `user` realm role, then configure it as a default role for new local and brokered users. The exact menu label varies by Keycloak version, but it is commonly under:

```text
Realm settings
  -> User registration
  -> Default roles
```

If the version does not expose Default roles there, assign the role in the First Broker Login flow or use a post-login flow. Do not assign `admin`, write, moderation, or management roles automatically. Once you've created the API roles in section 3, also add `blog:read` and `chat:read` to this default set — a new signup with no roles at all can authenticate but cannot call any policy-protected endpoint.

### Create the local TmLabs profile

Keycloak signup creates the identity. The TmLabs service creates its local profile the first time the user calls an authenticated endpoint:

```text
User signs up in Keycloak
  -> Keycloak issues JWT
  -> frontend calls the gateway
  -> service reads JWT sub
  -> service creates local User row if missing
```

Use the opaque JWT `sub` claim as `KeycloakSubject`, with a unique database index. Copy `preferred_username`, `name`, `email`, or `picture` only as display/profile data, and only at creation time. Do not use those fields as the identity key, and do not overwrite them on every login — see section 12.

### Public signup protections

Before enabling public registration in production:

- use HTTPS for the frontend and Keycloak
- enable email verification and configure SMTP
- enable brute-force detection in **Realm settings > Security defenses**
- use a strong password policy for local accounts
- consider CAPTCHA or rate limiting if automated signup becomes a problem
- keep new users on least-privilege roles
- log signup and first-login failures without logging passwords or tokens
- define account deletion, abuse reporting, and email-change policies (section 16)

The application may offer a **Sign up** button, but it should redirect to Keycloak's registration flow rather than collecting credentials itself. Clients can request the standard Keycloak authorization endpoint with `prompt=create` when they need to open registration directly.

## 2.2 Add GitHub sign-in through Keycloak

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
9. Set **Trust email** to off unless you have your own verification (see 2.1).
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

The first GitHub login normally creates a Keycloak user. Do not assume a GitHub account and an existing Keycloak user are linked automatically — see section 11 for how linking works and section 11.4 for migrating existing GitHub-keyed data.

If GitHub reports `redirect_uri_mismatch`, compare the callback URL character-for-character with the URL shown by Keycloak. If Keycloak reports an invalid client or the provider returns to an error page, verify the GitHub client ID and secret and make sure the OAuth app is not disabled.

## 2.3 Add Google sign-in through Keycloak

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

1. Open **Branding**. Set the application name to `TmLabs`. Set the user support email and developer contact email. Save.
2. Open **Audience**. Choose **External**, unless only users in your Google Workspace organization should be allowed. Add your Google account under **Test users** while the app is in testing mode. Save.
3. Open **Clients**. Click **Create OAuth client**. Choose **Web application**. Set the name to `TmLabs Keycloak non-prod`.

The equivalent older navigation, still shown in some Google Cloud projects, is **APIs & Services > OAuth consent screen**, and the client can also be created from **APIs & Services > Credentials > Create credentials > OAuth client ID**, or directly at `https://console.cloud.google.com/apis/credentials`. Choose **Web application** in every case.

Before creating the Google credential, open Keycloak in another tab (`http://192.168.1.17:8080/admin/`), select the `non-prod` realm, go to **Identity providers > Add provider > Google**, and copy the **Redirect URI** shown on that page. Keep it open — you will paste the Google client ID and secret back into it.

After clicking **Create OAuth client** in Google Cloud, paste the Keycloak redirect URI into **Authorized redirect URIs**. Do not use the Scalar URL here. Google redirects to Keycloak first:

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

Google requires an exact redirect URI. A raw private IP such as `http://192.168.1.17:8080/...` may be rejected because Google does not generally allow arbitrary IP hosts for OAuth web credentials. Use an HTTPS DNS name for Keycloak, such as `https://auth.example.com`, with a certificate trusted by the browser. For local-only experiments, `http://localhost` is treated specially by Google, but it points to the local computer running the browser, not to a separate Keycloak LXC — do not substitute the LXC IP for `localhost`.

The Google callback host and the Keycloak `Authority` must use the same public Keycloak hostname. If you introduce `https://auth.example.com`, update the realm issuer and gateway configuration to use `https://auth.example.com/realms/non-prod`.

Copy the Google client ID and client secret. Store the secret in a password manager or Kubernetes Secret. Do not commit it to this repository, `appsettings.json`, or a browser application.

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

Keycloak displays the provider redirect URI. Confirm it is exactly the URI copied into Google Cloud:

```text
https://auth.example.com/realms/non-prod/broker/google/endpoint
```

If Keycloak exposes a scope setting, use `openid email profile`.

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

The first Google login normally creates a Keycloak user. Do not assume a Google account and an existing Keycloak user are linked automatically — see section 11.

If Google reports `redirect_uri_mismatch`, compare the Google credential's authorized redirect URI with Keycloak's displayed provider redirect URI character-for-character, including scheme, host, port, path, and trailing slash. If Google rejects the raw IP host, move Keycloak behind an HTTPS DNS hostname. If Google reports `access blocked`, add the account as a test user or publish the consent screen. If Keycloak reports an invalid client, verify the Google client ID and secret.

## 2.4 Configure passkeys

Passkeys are configured in Keycloak through WebAuthn. They are not another OAuth identity provider. Keycloak registers and validates the passkey, then issues the normal TmLabs access token.

### Keycloak passkey setup

In the `non-prod` realm:

1. Go to **Authentication > Required actions**. Enable `Webauthn Register Passwordless`. Enable `Webauthn Register` if passkeys will also be used as a second factor.
2. Go to **Authentication > Policies > WebAuthn Passwordless Policy**.
3. Set **Relying Party Entity Name** to `TmLabs`.
4. Set **Relying Party ID** to the effective Keycloak domain, for example `auth.example.com`.
5. Set **User Verification Requirement** to `Preferred` or `Required`.
6. Set **Discoverable Credential** to `Required` for passwordless passkeys.
7. Go to **Realm settings > Login** and enable **Passkeys** if that switch is available in your Keycloak version.
8. Confirm the browser authentication flow includes **WebAuthn Passwordless Authenticator**.

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

## 2.5 Configure authenticator apps with TOTP

Authenticator apps use TOTP. This is not an OAuth provider. Keycloak generates a shared secret, displays a QR code, and validates the six-digit codes generated by the authenticator app.

Compatible apps include Google Authenticator, Microsoft Authenticator, 1Password, Bitwarden, and Authy.

### Keycloak TOTP setup

In the `non-prod` realm:

1. Go to **Authentication > Policies > OTP Policy**. Configure `Type: totp`, `Algorithm: SHA256 or SHA1 for maximum compatibility`, `Digits: 6`, `Period: 30 seconds`.
2. Go to **Authentication > Required actions**. Enable `Configure OTP`.
3. In the **Browser** authentication flow, locate the conditional 2FA flow. Set `OTP Form` to `Alternative` when users may choose OTP or passkey, or `Required` when every user must use an authenticator app.
4. Enable `Recovery Authentication Codes` as a fallback if required.

Users register an authenticator app from **Account security > Signing in > Set up Authenticator application**. The user scans the QR code, enters the current six-digit code, and Keycloak stores the TOTP credential. Do not store the TOTP secret in Blog API, Chat Service, or the frontend.

### Authenticator app testing

1. Log in to the Keycloak Account Console.
2. Register an authenticator application.
3. Log out and start a new login.
4. Enter the generated six-digit code when prompted.
5. Execute `/api/auth/me` through Scalar.
6. Confirm that the JWT issuer, audience, subject, and roles are unchanged.

## 2.6 Configure Apple Sign in with Apple

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
9. Add the Apple provider if your Keycloak version includes it. Otherwise use the generic **OpenID Connect v1.0** provider with Apple's authorization, token, and user-info endpoints.
10. Configure the Apple client ID as the Services ID.
11. Configure the Apple client secret as a signed JWT generated with the Apple private key, Team ID, Key ID, and the Apple token endpoint as the audience.
12. Use scopes `openid email name` where supported.
13. Save and test the provider.

Apple client-secret JWTs expire and must be rotated. Do not commit the `.p8` key or generated client secret to this repository or place them in a browser client.

Apple may provide a private relay email address. Treat it as the user's email identifier, but do not assume it can be used to link accounts automatically without an explicit account-linking flow.

## 2.7 Account management and an in-app user directory

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

Your application should manage its own domain data, such as chat profiles, comments, blog ownership, preferences, and moderation data. Link that data to the authenticated Keycloak subject, as described in sections 11 to 14.

### Listing users inside TmLabs

Do not call the Keycloak Admin API directly from the browser. Do not give normal users an admin token. Instead, create an admin-only endpoint in the service that owns the application user table, then call it through the gateway:

```text
Next.js admin page
  -> GET /api/chat/users
  -> Gateway authorization policy
  -> Chat Service user query
```

Require the `admin` role or a dedicated `users:read` permission. Return a safe public/admin view, not raw identity-provider data or tokens. Section 13.5 gives the full endpoint.

Do not return these fields from a general user-list endpoint:

- access tokens or refresh tokens
- passwords or password metadata
- passkey credentials
- TOTP secrets
- provider client secrets
- the complete raw JWT claims collection

Use pagination, a bounded `pageSize`, and server-side search. If users can view a public directory, create a separate public response model with only fields such as display name and avatar. Keep email, roles, moderation state, and login metadata admin-only.

### Keycloak users versus application users

These are not necessarily the same list:

- **Keycloak user list:** every identity known to the realm, including users who have never used TmLabs.
- **TmLabs user list:** users who have signed in and have a local application profile.

For normal in-app features, list the local TmLabs users. If an administrator truly needs to manage every Keycloak account, create a separate backend-only integration using a confidential service account with narrowly scoped Keycloak Admin API roles. Never expose that service account or Admin API token to the frontend.

## 3. Scopes and roles

Use realm roles rather than scope claims: Keycloak emits `scope` as a single space-delimited string (for example `"openid profile blog:read"`), so a policy built on `RequireClaim("scope", "blog:read")` never matches a single value in it. Roles are the reliable mechanism, once mapped into role claims (section 10.2).

In Keycloak, open the `non-prod` realm, choose **Realm roles**, and create:

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

If you later move permissions to client roles instead of realm roles, they appear under `resource_access.tmlabs-api.roles` instead of `realm_access.roles`. The claims transformation in section 10.2 reads both, so either layout works without further changes.

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

### Seeding an avatar from GitHub or Google

If you want to seed a profile picture at signup, add mappers in Keycloak:

1. **Identity providers > github > Mappers > Add mapper**: type **Attribute Importer**, claim `avatar_url`, user attribute `picture`. For Google, use claim `picture` on the Google provider's mapper.
2. **Client scopes > tmlabs-api-audience > Mappers > By configuration > User Attribute**: user attribute `picture`, token claim name `picture`, add to access token.

Treat the `picture` claim as a starting value only; let users replace it in their TmLabs profile (section 13.3).

## 4. Gateway configuration

Add JWT bearer authentication to `TmLabs.Gateway`:

```bash
dotnet add src/TmLabs.Gateway/TmLabs.Gateway.csproj package Microsoft.AspNetCore.Authentication.JwtBearer
```

Configure the gateway with environment variables or user secrets. Do not commit production values:

```text
Authentication__Authority=https://auth.example.com/realms/non-prod
Authentication__Audience=tmlabs-api
Authentication__RequireHttpsMetadata=true
```

The committed appsettings files contain dummy, non-routable values so they are safe to keep in source control:

```json
{
  "Authentication": {
    "Authority": "https://keycloak.example.invalid/realms/non-prod",
    "Audience": "tmlabs-api",
    "RequireHttpsMetadata": true
  }
}
```

For local HTTP-only Keycloak, `appsettings.Development.json` uses a dummy HTTP authority and sets `RequireHttpsMetadata` to `false`. Replace that authority with the real URL of the Keycloak LXC, for example `http://192.168.1.50:8080/realms/non-prod`. The URL must be reachable from the gateway process and must exactly match the token issuer.

For Kubernetes, leave the JSON placeholders in the image and override them with environment variables. ASP.NET Core converts double underscores to nested configuration sections:

```yaml
env:
  - name: Authentication__Authority
    value: "https://auth.example.com/realms/non-prod"
  - name: Authentication__Audience
    value: "tmlabs-api"
  - name: Authentication__RequireHttpsMetadata
    value: "true"
```

Prefer a Kubernetes `ConfigMap` for the authority and audience and a `Secret` for any confidential server-side client value. This gateway only validates bearer tokens, so it does not need a Keycloak client secret. Do not put access tokens, refresh tokens, or Keycloak admin credentials in `appsettings.json`.

| Setting | Value | Purpose |
| --- | --- | --- |
| `Authentication:Authority` | `https://auth.example.com/realms/non-prod` | Keycloak issuer/realm. ASP.NET downloads signing keys from its discovery endpoint. |
| `Authentication:Audience` | `tmlabs-api` | Accept only tokens intended for this API. |
| `Authentication:RequireHttpsMetadata` | `true` in Kubernetes/production | Require HTTPS for Keycloak discovery and signing-key metadata. |

Rather than registering JWT bearer auth directly in each service's `Program.cs`, put the registration in one shared library (section 10) and call one extension method from each service. This keeps the three services from drifting out of sync, and it's where the role-claims fix lives.

```csharp
builder.Services.AddTmLabsAuth(builder.Configuration);

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
```

Protect routes explicitly. For example, health checks can remain public, while write routes require a policy:

```csharp
app.MapGet("/api/blog/posts", ...)
    .RequireAuthorization("BlogRead");
```

YARP routes also need authorization metadata if the gateway is responsible for blocking unauthenticated requests before forwarding — see section 10.4 for the route-level config, including how to keep specific routes (health checks, OpenAPI documents) deliberately public.

## 5. Service authorization

The gateway is not the only security boundary. Blog API and Chat Service should validate tokens and authorize their own business operations, especially if they can be reached without the gateway in another deployment.

Each service should:

1. Reference the shared `TmLabs.Auth` library and call `AddTmLabsAuth` (section 10).
2. Call `UseAuthentication()` and `UseAuthorization()`.
3. Require policies on protected FastEndpoints.
4. Check ownership and resource-level permissions in application code (section 14).

Do not add password storage or passkey storage to either service.

Also avoid combining `AllowAnonymous()` with a policy. `AllowAnonymous()` bypasses authorization entirely — an endpoint with both will be open to everyone regardless of the policy.

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

The Gateway document is generated automatically by `app.MapOpenApi()`. Gateway-only endpoints such as `/api/auth/me` appear there automatically when they have endpoint metadata such as `.RequireAuthorization()`. Blog and Chat endpoints appear in their respective service documents.

To show an authentication button in Scalar, the aggregated OpenAPI document must contain a bearer security scheme:

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
Authorization endpoint:
http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/auth

Token endpoint:
http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/token

Discovery document:
http://192.168.1.17:8080/realms/non-prod/.well-known/openid-configuration
```

Open the discovery URL in a browser first. It must return JSON and must list the same authorization and token endpoints. The gateway's JWT `Authority` setting must be the issuer base URL, without `/protocol/openid-connect`:

```text
http://192.168.1.17:8080/realms/non-prod
```

Do not write `https://http://192.168.1.17:8080`. The scheme appears exactly once. Use `http` only for this local/non-production setup; use `https` in Kubernetes and production.

#### Keycloak client required by Scalar

Use the same `tmlabs-web` public client created in section 2. It must not have a secret because Scalar runs in the browser. Authorization Code + PKCE uses a code verifier instead of a client secret.

Register the URL where the gateway Scalar UI runs:

```text
Valid redirect URIs: http://localhost:5020/scalar/*
Web origins: http://localhost:5020
```

These two settings do different jobs:

- **Valid redirect URIs** controls where Keycloak may redirect the browser after login.
- **Web origins** controls whether browser JavaScript served by Scalar may call Keycloak's token endpoint.

The Web Origins value must be the origin only: scheme, host, and port, with no path and no trailing slash. For this setup it is exactly `http://localhost:5020`.

If Web Origins is missing or incorrect, login may complete in Keycloak but Scalar fails during the authorization-code exchange with a browser error such as `No 'Access-Control-Allow-Origin' header` on the `/protocol/openid-connect/token` request.

If Aspire assigns a different gateway port, replace `5020` with that port. If using HTTPS, register the HTTPS origin separately:

```text
Valid redirect URIs: https://localhost:7020/scalar/*
Web origins: https://localhost:7020
```

Use the narrowest callback path accepted by the Scalar version in use. A wildcard is convenient for local development but should be replaced with the exact Scalar callback URL for production.

Add these client scopes or make them available to the client: `openid`, `profile`, `email`. Ensure the access token also has the `tmlabs-api` audience via the mapper from section 3.

The OAuth settings used by the generated OpenAPI document should be:

```text
Authorization endpoint: http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/auth
Token endpoint:         http://192.168.1.17:8080/realms/non-prod/protocol/openid-connect/token
Scopes:                 openid profile email
Client ID:              tmlabs-web
PKCE:                   Enabled
```

After login, decode the returned access token and confirm:

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

To verify the OpenAPI configuration directly, open `http://localhost:5020/openapi/v1.json` and search for `securitySchemes`, `oauth2`, `authorizationCode`, `authorizationUrl`, and `tokenUrl`. Search the `/api/auth/me` operation for a `security` entry referencing the OAuth scheme. If the scheme exists but the operation has no `security` metadata, Scalar may not show the operation as requiring authorization.

If Scalar shows no **Authorize** button, the gateway document is missing `components.securitySchemes` or the operation is missing its `security` array. If login succeeds but the API returns `401`, inspect `iss`, `aud`, expiry, and the gateway's `Authentication:Authority` and `Authentication:Audience` values. If the API returns `403`, the token is valid but does not contain the role required by the endpoint.

## 7. Aspire and local URLs

When launched through `TmLabs.AppHost`, Aspire supplies the gateway's Blog and Chat URLs. Start the complete application with:

```bash
dotnet run --project src/TmLabs.AppHost
```

Do not launch the gateway binary alone and expect the Aspire hostnames `blogapi` and `chatservice` to resolve. A direct gateway run requires Blog API and Chat Service to run on their localhost ports and requires matching development configuration.

The gateway should log calls to actual service endpoints, not literal unresolved names such as `http://blogapi/openapi/v1.json`.

## 8. Shared authentication library for .NET

Every service (Gateway, Blog API, Chat Service) validates tokens the same way. Put the code in one shared project, `src/TmLabs.Auth`, and reference it from each service. That way there is only one place to get it right, and the fixes below apply everywhere at once.

Two things in a typical first pass at this don't work and are corrected here:

- **ASP.NET Core does not read `realm_access.roles` automatically.** Keycloak puts roles in a nested JSON object, and the JWT handler leaves it as a single claim named `realm_access`. `RequireRole("admin")` returns `403` until you flatten the roles into `ClaimTypes.Role` claims. The transformation below does that.
- **Scope-based policies don't fit a role-based setup.** Keycloak emits `scope` as one space-delimited string, so `RequireClaim("scope", "blog:read")` never matches a single value. Use role-based policies against the realm roles created in section 3.

```bash
dotnet new classlib -o src/TmLabs.Auth
dotnet add src/TmLabs.Auth package Microsoft.AspNetCore.Authentication.JwtBearer
```

In `TmLabs.Auth.csproj`, make the library able to use ASP.NET Core types:

```xml
<ItemGroup>
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

### 8.1 Registration extension

```csharp
// src/TmLabs.Auth/AuthExtensions.cs
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace TmLabs.Auth;

public static class AuthExtensions
{
    public static IServiceCollection AddTmLabsAuth(
        this IServiceCollection services, IConfiguration config)
    {
        services
            .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.Authority = config["Authentication:Authority"];
                options.Audience = config["Authentication:Audience"];
                options.RequireHttpsMetadata =
                    config.GetValue("Authentication:RequireHttpsMetadata", true);

                // Keep claim names exactly as they appear in the token
                // ("sub" stays "sub" instead of becoming a long URI).
                options.MapInboundClaims = false;

                options.TokenValidationParameters.NameClaimType = "preferred_username";
                options.TokenValidationParameters.RoleClaimType = ClaimTypes.Role;
                options.TokenValidationParameters.ClockSkew = TimeSpan.FromSeconds(30);
            });

        services.AddTransient<IClaimsTransformation, KeycloakRolesTransformation>();

        services.AddAuthorizationBuilder()
            .AddPolicy("BlogRead",  p => p.RequireRole("blog:read"))
            .AddPolicy("BlogWrite", p => p.RequireRole("blog:write"))
            .AddPolicy("ChatRead",  p => p.RequireRole("chat:read"))
            .AddPolicy("ChatWrite", p => p.RequireRole("chat:write"))
            .AddPolicy("Admin",     p => p.RequireRole("admin"));

        return services;
    }
}
```

### 8.2 Claims transformation for Keycloak roles

This flattens `realm_access.roles` and `resource_access.<audience>.roles` into normal role claims, so `RequireRole`, `Roles("admin")` and `User.IsInRole` all work.

```csharp
// src/TmLabs.Auth/KeycloakRolesTransformation.cs
using System.Security.Claims;
using System.Text.Json;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Configuration;

namespace TmLabs.Auth;

public sealed class KeycloakRolesTransformation(IConfiguration config) : IClaimsTransformation
{
    private const string Marker = "tmlabs:roles-mapped";

    public Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        if (principal.Identity is not ClaimsIdentity identity || !identity.IsAuthenticated)
            return Task.FromResult(principal);

        // IClaimsTransformation can run more than once per request.
        if (identity.HasClaim(c => c.Type == Marker))
            return Task.FromResult(principal);

        var audience = config["Authentication:Audience"];
        var roles = new HashSet<string>(StringComparer.Ordinal);

        ReadRoles(identity.FindFirst("realm_access")?.Value, null, roles);
        ReadRoles(identity.FindFirst("resource_access")?.Value, audience, roles);

        foreach (var role in roles)
            identity.AddClaim(new Claim(ClaimTypes.Role, role));

        identity.AddClaim(new Claim(Marker, "1"));
        return Task.FromResult(principal);
    }

    // realm_access:    { "roles": [ ... ] }
    // resource_access: { "tmlabs-api": { "roles": [ ... ] } }
    private static void ReadRoles(string? json, string? clientId, HashSet<string> into)
    {
        if (string.IsNullOrWhiteSpace(json)) return;

        try
        {
            using var doc = JsonDocument.Parse(json);
            var node = doc.RootElement;

            if (clientId is not null &&
                !node.TryGetProperty(clientId, out node)) return;

            if (node.ValueKind == JsonValueKind.Object &&
                node.TryGetProperty("roles", out var arr) &&
                arr.ValueKind == JsonValueKind.Array)
            {
                foreach (var item in arr.EnumerateArray())
                    if (item.GetString() is { Length: > 0 } role)
                        into.Add(role);
            }
        }
        catch (JsonException)
        {
            // Malformed claim: ignore and grant no roles from it.
        }
    }
}
```

### 8.3 Wire it into each service

In the Gateway, Blog API and Chat Service `Program.cs`:

```csharp
builder.Services.AddTmLabsAuth(builder.Configuration);

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
```

Each service gets the same `Authentication:Authority`, `Authentication:Audience` and `Authentication:RequireHttpsMetadata` settings from section 4.

### 8.4 Gateway: protect YARP routes and forward the token

YARP forwards the `Authorization` header to the destination by default, so services can validate the token themselves. To also reject unauthenticated calls at the gateway, add an authorization policy to the routes in `appsettings.json`:

```json
{
  "ReverseProxy": {
    "Routes": {
      "blog-write": {
        "ClusterId": "blog",
        "AuthorizationPolicy": "BlogWrite",
        "Match": { "Path": "/api/blog/{**catch-all}", "Methods": [ "POST", "PUT", "PATCH", "DELETE" ] }
      },
      "blog-read": {
        "ClusterId": "blog",
        "AuthorizationPolicy": "BlogRead",
        "Match": { "Path": "/api/blog/{**catch-all}", "Methods": [ "GET" ] }
      },
      "chat": {
        "ClusterId": "chat",
        "AuthorizationPolicy": "ChatRead",
        "Match": { "Path": "/api/chat/{**catch-all}" }
      }
    }
  }
}
```

Anything that must stay public — health checks, OpenAPI documents for Scalar, public blog reads if you want them — needs its own route with `"AuthorizationPolicy": "anonymous"`. Do not leave a public route out of the config and expect it to be open: be deliberate about which routes are public and which require a policy.

If both the gateway and the services enforce policies, a token missing a role is rejected at the gateway first. The services still enforce their own rules in case they are reached directly.

## 9. Test sequence

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

## 10. Production rules

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

---

# Part 2: Linking users and building profiles

## 11. Linking users: how identities map to TmLabs profiles

"Linking" means two different things here. Keep them separate.

| Kind of link | Where it lives | Who manages it |
| --- | --- | --- |
| Keycloak user <-> GitHub / Google / Apple / passkey / TOTP | Keycloak | Keycloak |
| Keycloak user <-> TmLabs profile | Your database | Your service |

The rule that ties them together: the JWT `sub` is the Keycloak user ID. It stays the same whether the person logs in with a password, GitHub, Google, Apple or a passkey, as long as all those logins are linked to the same Keycloak user. Your database therefore only ever stores one key per person: `KeycloakSubject`.

### 11.1 Linking a social login to an existing Keycloak account

By default, if someone signs up with a password and later clicks "Sign in with GitHub" using the same email, Keycloak does not silently merge them. It runs the first broker login flow from section 2.1. Two safe options:

**Option A: user links from the Account Console (simplest, safest).** The user logs in with their existing method, opens the Account Console, goes to **Account security > Linked accounts**, and links GitHub, Google or Apple. Nothing needs configuring beyond having the provider enabled.

**Option B: link during first social login with verification.** With the **First broker login** flow left at its defaults (section 2.1), a person who logs in with GitHub or Google and already has a Keycloak account with the same email is asked to prove they own the existing account — via an email link or by logging in — before the accounts are merged.

Do not use "automatically set existing user" without verification; it lets anyone who controls an identity-provider account with a matching email take over the existing account.

### 11.2 What happens on the TmLabs side

Because linking happens in Keycloak, your app does nothing special. After a merge the user's `sub` is unchanged, so their existing local profile continues to work. If two Keycloak users are never merged, they are two separate people as far as TmLabs is concerned, each with their own profile.

### 11.3 Migrating existing GitHub-based users

If existing rows are keyed by `GitHubUserId`, you have to connect them to Keycloak subjects once. The safest approach is to link on first login only after the person proves control of the GitHub account:

1. Add GitHub as an identity provider in Keycloak (section 2.2).
2. Add an **Attribute Importer** mapper on the GitHub provider that stores the GitHub numeric ID as a user attribute `github_id`, and a **User Attribute** protocol mapper that puts it in the token as `github_id`.
3. In the find-or-create logic (section 12.3), before inserting a new user, look for an existing row with `GitHubUserId == github_id` and no `KeycloakSubject`. If found, set `KeycloakSubject = sub` on that row instead of creating a new one.

Only do step 3 for logins that came through the GitHub provider. Never match on email or username, since either can be spoofed or reassigned.

## 12. Data model for local profiles

Put the user table in the service that owns it (Chat Service in this guide). Other services can reference users by `KeycloakSubject` or by an API call, but should not each keep their own copy of profile data.

### 12.1 Entity

```csharp
public sealed class User
{
    public Guid Id { get; set; } = Guid.NewGuid();

    // JWT "sub". Opaque. Unique. Never change it, never reuse another field as the key.
    public string KeycloakSubject { get; set; } = default!;

    // Public handle, chosen by the user. Unique, case-insensitive.
    public string Username { get; set; } = default!;
    public string UsernameNormalized { get; set; } = default!;

    public string DisplayName { get; set; } = default!;
    public string? Bio { get; set; }
    public string? AvatarUrl { get; set; }

    // Copied from the token on creation for admin use only. Not a source of truth.
    public string? Email { get; set; }

    public bool IsDeleted { get; set; }
    public DateTime CreatedAtUtc { get; set; }
    public DateTime UpdatedAtUtc { get; set; }
    public DateTime LastSeenAtUtc { get; set; }
}
```

### 12.2 EF Core configuration

```csharp
public sealed class ChatDbContext(DbContextOptions<ChatDbContext> options) : DbContext(options)
{
    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<User>(e =>
        {
            e.HasKey(x => x.Id);

            e.Property(x => x.KeycloakSubject).IsRequired().HasMaxLength(64);
            e.HasIndex(x => x.KeycloakSubject).IsUnique();

            e.Property(x => x.Username).IsRequired().HasMaxLength(32);
            e.Property(x => x.UsernameNormalized).IsRequired().HasMaxLength(32);
            e.HasIndex(x => x.UsernameNormalized).IsUnique();

            e.Property(x => x.DisplayName).IsRequired().HasMaxLength(64);
            e.Property(x => x.Bio).HasMaxLength(500);
            e.Property(x => x.AvatarUrl).HasMaxLength(500);
            e.Property(x => x.Email).HasMaxLength(256);

            e.HasQueryFilter(x => !x.IsDeleted);
        });
    }
}
```

Create the migration:

```bash
dotnet ef migrations add AddUsers --project src/TmLabs.ChatService
dotnet ef database update --project src/TmLabs.ChatService
```

If the Chat Service currently has a `GitHubUserId` column, add `KeycloakSubject` first, backfill it using section 11.3, then drop `GitHubUserId` in a later migration.

### 12.3 Find-or-create on first authenticated request

This is the code behind "service reads JWT `sub`, creates local User row if missing" from section 2.1. It is written as a scoped service so it runs at most once per request and handles two simultaneous first requests from the same user.

```csharp
using System.Security.Claims;
using System.Text.RegularExpressions;
using Microsoft.EntityFrameworkCore;

public sealed partial class CurrentUserService(
    IHttpContextAccessor http, ChatDbContext db, TimeProvider clock)
{
    private User? _cached;

    public async Task<User> GetAsync(CancellationToken ct = default)
    {
        if (_cached is not null) return _cached;

        var principal = http.HttpContext?.User
            ?? throw new InvalidOperationException("No HTTP context.");
        var sub = principal.FindFirstValue("sub")
            ?? throw new UnauthorizedAccessException("Token has no sub claim.");

        var now = clock.GetUtcNow().UtcDateTime;

        var user = await db.Users.IgnoreQueryFilters()
            .SingleOrDefaultAsync(u => u.KeycloakSubject == sub, ct);

        if (user is { IsDeleted: true })
            throw new UnauthorizedAccessException("Account has been deleted.");

        if (user is not null)
        {
            // Avoid a write on every request.
            if (now - user.LastSeenAtUtc > TimeSpan.FromMinutes(10))
            {
                user.LastSeenAtUtc = now;
                await db.SaveChangesAsync(ct);
            }
            return _cached = user;
        }

        return _cached = await CreateAsync(principal, sub, now, ct);
    }

    private async Task<User> CreateAsync(
        ClaimsPrincipal principal, string sub, DateTime now, CancellationToken ct)
    {
        var preferred = principal.FindFirstValue("preferred_username");
        var email = principal.FindFirstValue("email");
        var name = principal.FindFirstValue("name") ?? preferred ?? "New user";

        for (var attempt = 0; attempt < 5; attempt++)
        {
            var username = await PickUsernameAsync(preferred ?? email, attempt, ct);

            var user = new User
            {
                KeycloakSubject = sub,
                Username = username,
                UsernameNormalized = username.ToLowerInvariant(),
                DisplayName = name.Length > 64 ? name[..64] : name,
                AvatarUrl = principal.FindFirstValue("picture"),
                Email = email,
                CreatedAtUtc = now,
                UpdatedAtUtc = now,
                LastSeenAtUtc = now,
            };

            db.Users.Add(user);
            try
            {
                await db.SaveChangesAsync(ct);
                return user;
            }
            catch (DbUpdateException)
            {
                // Either another request created this user first, or the username
                // collided. Drop our copy, then check which one it was.
                db.Entry(user).State = EntityState.Detached;

                var existing = await db.Users.IgnoreQueryFilters()
                    .SingleOrDefaultAsync(u => u.KeycloakSubject == sub, ct);
                if (existing is not null) return existing;
                // Otherwise it was a username collision: loop and try a new suffix.
            }
        }

        throw new InvalidOperationException("Could not allocate a username.");
    }

    private async Task<string> PickUsernameAsync(string? seed, int attempt, CancellationToken ct)
    {
        // "alex@example.com" -> "alex"
        var baseName = (seed ?? "user").Split('@')[0];
        baseName = InvalidChars().Replace(baseName.ToLowerInvariant(), "");
        if (baseName.Length < 3) baseName = "user";
        if (baseName.Length > 24) baseName = baseName[..24];

        var candidate = attempt == 0
            ? baseName
            : $"{baseName}{Random.Shared.Next(1000, 9999)}";

        var taken = await db.Users.IgnoreQueryFilters()
            .AnyAsync(u => u.UsernameNormalized == candidate, ct);

        return taken ? $"{baseName}{Random.Shared.Next(1000, 9999)}" : candidate;
    }

    [GeneratedRegex("[^a-z0-9_-]")]
    private static partial Regex InvalidChars();
}
```

Register it:

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<CurrentUserService>();
```

Design choices worth keeping:

- The token seeds the profile once, at creation. Later logins never overwrite what the user edited. If you overwrite on every login, a user who changes their display name in TmLabs will see it revert.
- `Email` is stored for admin use and support. If a user changes their email in Keycloak, your copy goes stale. Treat it as informational. The identity key is `sub`, not email.
- The `IgnoreQueryFilters` calls are deliberate: they let a deleted user be recognised (and refused) instead of silently recreated with a new profile.

## 13. Profile endpoints (FastEndpoints)

Routes below are relative to the Chat Service. If your YARP route strips `/api/chat`, the gateway URL is `/api/chat/users/me`. If it does not, set the endpoints to the full path.

### 13.1 Response models

```csharp
public sealed record MyProfileResponse(
    Guid Id, string Username, string DisplayName, string? Bio,
    string? AvatarUrl, string? Email, string[] Roles, DateTime CreatedAtUtc);

public sealed record PublicProfileResponse(
    string Username, string DisplayName, string? Bio, string? AvatarUrl, DateTime CreatedAtUtc);

public sealed record AdminUserResponse(
    Guid Id, string Username, string DisplayName, string? Email,
    string? AvatarUrl, DateTime CreatedAtUtc, DateTime LastSeenAtUtc);

public sealed record PagedResponse<T>(IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount);
```

The three user shapes are separate on purpose. `Email` and roles appear only for the owner or an admin. The public one exposes nothing private.

### 13.2 Get my profile (also creates it on first call)

```csharp
using FastEndpoints;

public sealed class GetMeEndpoint(CurrentUserService current)
    : EndpointWithoutRequest<MyProfileResponse>
{
    public override void Configure()
    {
        Get("/users/me");
        Policies("ChatRead");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var u = await current.GetAsync(ct);
        var roles = User.FindAll(System.Security.Claims.ClaimTypes.Role)
                        .Select(c => c.Value).Distinct().ToArray();

        await SendOkAsync(new MyProfileResponse(
            u.Id, u.Username, u.DisplayName, u.Bio, u.AvatarUrl, u.Email, roles, u.CreatedAtUtc), ct);
    }
}
```

Have the frontend call this right after login. That single call provisions the profile.

### 13.3 Update my profile

```csharp
public sealed class UpdateProfileRequest
{
    public string? DisplayName { get; set; }
    public string? Username { get; set; }
    public string? Bio { get; set; }
    public string? AvatarUrl { get; set; }
}

public sealed class UpdateProfileValidator : Validator<UpdateProfileRequest>
{
    public UpdateProfileValidator()
    {
        RuleFor(x => x.DisplayName).MaximumLength(64).NotEmpty().When(x => x.DisplayName is not null);

        RuleFor(x => x.Username)
            .Matches("^[a-zA-Z0-9_-]{3,32}$")
            .WithMessage("3-32 characters: letters, numbers, underscore, hyphen.")
            .When(x => x.Username is not null);

        RuleFor(x => x.Bio).MaximumLength(500);

        // Only https URLs, so the avatar can't be a javascript: or data: URL.
        RuleFor(x => x.AvatarUrl)
            .MaximumLength(500)
            .Must(u => Uri.TryCreate(u, UriKind.Absolute, out var uri) && uri.Scheme == Uri.UriSchemeHttps)
            .When(x => !string.IsNullOrEmpty(x.AvatarUrl));
    }
}

public sealed class UpdateProfileEndpoint(CurrentUserService current, ChatDbContext db, TimeProvider clock)
    : Endpoint<UpdateProfileRequest, MyProfileResponse>
{
    public override void Configure()
    {
        Put("/users/me");
        Policies("ChatWrite");
    }

    public override async Task HandleAsync(UpdateProfileRequest req, CancellationToken ct)
    {
        var u = await current.GetAsync(ct);
        // Attach: the cached entity is already tracked by the same scoped DbContext.

        if (req.Username is not null &&
            !string.Equals(req.Username, u.Username, StringComparison.OrdinalIgnoreCase))
        {
            var normalized = req.Username.ToLowerInvariant();
            if (await db.Users.IgnoreQueryFilters().AnyAsync(x => x.UsernameNormalized == normalized, ct))
            {
                AddError(r => r.Username!, "That username is taken.");
                ThrowIfAnyErrors();
            }
            u.Username = req.Username;
            u.UsernameNormalized = normalized;
        }

        if (req.DisplayName is not null) u.DisplayName = req.DisplayName.Trim();
        if (req.Bio is not null) u.Bio = req.Bio.Trim();
        if (req.AvatarUrl is not null) u.AvatarUrl = string.IsNullOrWhiteSpace(req.AvatarUrl) ? null : req.AvatarUrl;
        u.UpdatedAtUtc = clock.GetUtcNow().UtcDateTime;

        try { await db.SaveChangesAsync(ct); }
        catch (DbUpdateException)
        {
            // Lost a race on the unique username index.
            AddError(r => r.Username!, "That username is taken.");
            ThrowIfAnyErrors();
        }

        var roles = User.FindAll(System.Security.Claims.ClaimTypes.Role).Select(c => c.Value).Distinct().ToArray();
        await SendOkAsync(new MyProfileResponse(
            u.Id, u.Username, u.DisplayName, u.Bio, u.AvatarUrl, u.Email, roles, u.CreatedAtUtc), ct);
    }
}
```

Notice there is no `UserId` in the request. The user being edited always comes from the token, never from the body or URL. This is what stops one user editing another's profile.

### 13.4 Public profile

```csharp
public sealed class GetPublicProfileRequest { public string Username { get; set; } = default!; }

public sealed class GetPublicProfileEndpoint(ChatDbContext db)
    : Endpoint<GetPublicProfileRequest, PublicProfileResponse>
{
    public override void Configure()
    {
        Get("/users/{Username}");
        AllowAnonymous();   // No policy here: AllowAnonymous bypasses policies.
    }

    public override async Task HandleAsync(GetPublicProfileRequest req, CancellationToken ct)
    {
        var key = req.Username.ToLowerInvariant();
        var u = await db.Users.AsNoTracking()
            .FirstOrDefaultAsync(x => x.UsernameNormalized == key, ct);

        if (u is null) { await SendNotFoundAsync(ct); return; }

        await SendOkAsync(new PublicProfileResponse(
            u.Username, u.DisplayName, u.Bio, u.AvatarUrl, u.CreatedAtUtc), ct);
    }
}
```

`/users/me` is a literal route, so it wins over `/users/{Username}`. Do not let anyone register the username `me`: add it to a reserved-names list in the validator (also `admin`, `api`, `system`, and similar).

### 13.5 Admin user list

```csharp
public sealed class ListUsersRequest
{
    [QueryParam] public int Page { get; set; } = 1;
    [QueryParam] public int PageSize { get; set; } = 25;
    [QueryParam] public string? Search { get; set; }
}

public sealed class ListUsersEndpoint(ChatDbContext db)
    : Endpoint<ListUsersRequest, PagedResponse<AdminUserResponse>>
{
    public override void Configure()
    {
        Get("/users");
        Roles("admin");
    }

    public override async Task HandleAsync(ListUsersRequest req, CancellationToken ct)
    {
        var page = Math.Max(1, req.Page);
        var size = Math.Clamp(req.PageSize, 1, 100);

        var q = db.Users.AsNoTracking();
        if (!string.IsNullOrWhiteSpace(req.Search))
        {
            var s = req.Search.Trim().ToLowerInvariant();
            q = q.Where(u => u.UsernameNormalized.Contains(s) || u.DisplayName.ToLower().Contains(s));
        }

        var total = await q.CountAsync(ct);
        var items = await q.OrderBy(u => u.UsernameNormalized)
            .Skip((page - 1) * size).Take(size)
            .Select(u => new AdminUserResponse(
                u.Id, u.Username, u.DisplayName, u.Email, u.AvatarUrl, u.CreatedAtUtc, u.LastSeenAtUtc))
            .ToListAsync(ct);

        await SendOkAsync(new PagedResponse<AdminUserResponse>(items, page, size, total), ct);
    }
}
```

Never use `Policies` and `AllowAnonymous` together on the same endpoint.

## 14. Using the local user from other domain code

Everything that has an owner references the local `User.Id`:

```csharp
public sealed class BlogPost
{
    public Guid Id { get; set; }
    public Guid OwnerUserId { get; set; }   // FK -> User.Id
    // ...
}
```

Always derive the owner from the token and check ownership before changing anything:

```csharp
public override async Task HandleAsync(UpdatePostRequest req, CancellationToken ct)
{
    var me = await current.GetAsync(ct);
    var post = await db.Posts.FindAsync([req.Id], ct);

    if (post is null) { await SendNotFoundAsync(ct); return; }

    var isOwner = post.OwnerUserId == me.Id;
    var isAdmin = User.IsInRole("admin");
    if (!isOwner && !isAdmin) { await SendForbiddenAsync(ct); return; }

    // update...
}
```

If Blog API and Chat Service are separate databases, Blog API can store `OwnerSubject` (the Keycloak `sub`) instead of a chat-service user ID. Both services see the same `sub` in the token, so you avoid a cross-service lookup. Fetch display names and avatars from the Chat Service (or from a cached public-profile endpoint) when rendering.

## 15. Next.js integration

The frontend gets the access token from Keycloak and sends it to the gateway as a bearer token. This example uses Auth.js (NextAuth v5) with the existing public `tmlabs-web` client and PKCE, so no secret is needed.

### 15.1 Configuration

```bash
npm install next-auth@beta
```

`.env.local`:

```text
AUTH_SECRET=<generate with: npx auth secret>
AUTH_KEYCLOAK_ID=tmlabs-web
AUTH_KEYCLOAK_ISSUER=https://auth.example.com/realms/non-prod
API_BASE_URL=https://api.example.com
```

`auth.ts`:

```ts
import NextAuth from "next-auth";
import Keycloak from "next-auth/providers/keycloak";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Keycloak({
      clientId: process.env.AUTH_KEYCLOAK_ID!,
      issuer: process.env.AUTH_KEYCLOAK_ISSUER!,
      // Public client: no secret, PKCE only.
      client: { token_endpoint_auth_method: "none" },
      checks: ["pkce", "state"],
    }),
  ],
  session: { strategy: "jwt" },
  callbacks: {
    async jwt({ token, account }) {
      if (account) {
        // First sign-in: keep the tokens on the server-side session cookie.
        token.accessToken = account.access_token;
        token.refreshToken = account.refresh_token;
        token.expiresAt = account.expires_at; // seconds
      }
      if (Date.now() / 1000 < ((token.expiresAt as number) ?? 0) - 30) return token;
      return await refresh(token);
    },
    async session({ session, token }) {
      (session as any).accessToken = token.accessToken;
      (session as any).error = token.error;
      return session;
    },
  },
});

async function refresh(token: any) {
  try {
    const res = await fetch(
      `${process.env.AUTH_KEYCLOAK_ISSUER}/protocol/openid-connect/token`,
      {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          grant_type: "refresh_token",
          client_id: process.env.AUTH_KEYCLOAK_ID!,
          refresh_token: token.refreshToken,
        }),
      },
    );
    const data = await res.json();
    if (!res.ok) throw data;
    return {
      ...token,
      accessToken: data.access_token,
      refreshToken: data.refresh_token ?? token.refreshToken,
      expiresAt: Math.floor(Date.now() / 1000) + data.expires_in,
      error: undefined,
    };
  } catch {
    return { ...token, error: "RefreshFailed" };
  }
}
```

`app/api/auth/[...nextauth]/route.ts`:

```ts
import { handlers } from "@/auth";
export const { GET, POST } = handlers;
```

If the session shows `error: "RefreshFailed"`, sign the user out and back in.

### 15.2 Sign in, sign up and sign out

```tsx
// Sign in
<form action={async () => { "use server"; await signIn("keycloak"); }}>
  <button>Sign in</button>
</form>

// Sign up: opens Keycloak's registration page (needs a recent Keycloak with prompt=create support).
<form action={async () => { "use server"; await signIn("keycloak", undefined, { prompt: "create" }); }}>
  <button>Sign up</button>
</form>
```

Do not build your own password form. Both buttons redirect to Keycloak.

### 15.3 Calling the gateway

Call the API from server components or route handlers so the token never reaches browser JavaScript:

```ts
import { auth } from "@/auth";

export async function apiFetch(path: string, init: RequestInit = {}) {
  const session = await auth();
  const token = (session as any)?.accessToken;
  return fetch(`${process.env.API_BASE_URL}${path}`, {
    ...init,
    headers: { ...init.headers, Authorization: `Bearer ${token}` },
    cache: "no-store",
  });
}

// In a server component:
const me = await (await apiFetch("/api/chat/users/me")).json();
```

The first call to `/users/me` after sign-up creates the local profile.

Add the Next.js origin to the gateway's CORS policy only if the browser calls the gateway directly. With the server-side pattern above, no browser CORS is needed.

Also add these to `tmlabs-web` in Keycloak (in addition to the Scalar redirect from section 2):

```text
Valid redirect URIs: http://localhost:3000/api/auth/callback/keycloak
Valid post logout redirect URIs: http://localhost:3000/*
Web origins: http://localhost:3000
```

## 16. Account deletion, email changes and cleanup

### 16.1 Deleting an account

Keycloak owns the login; TmLabs owns the content. Decide what happens to each:

1. **Local data (TmLabs):** soft-delete and anonymize, so comments and posts keep working without personal data.

```csharp
public sealed class DeleteMeEndpoint(CurrentUserService current, ChatDbContext db, TimeProvider clock)
    : EndpointWithoutRequest
{
    public override void Configure() { Delete("/users/me"); Policies("ChatRead"); }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var u = await current.GetAsync(ct);
        var now = clock.GetUtcNow().UtcDateTime;

        u.IsDeleted = true;
        u.DisplayName = "Deleted user";
        u.Username = $"deleted-{u.Id:N}"[..16];
        u.UsernameNormalized = u.Username;
        u.Bio = null; u.AvatarUrl = null; u.Email = null;
        u.UpdatedAtUtc = now;
        await db.SaveChangesAsync(ct);

        await SendNoContentAsync(ct);
    }
}
```

2. **The Keycloak user:** either let the user delete it from the Account Console (enable **Authentication > Required actions > Delete Account**, then grant the `delete-account` role of the `account` client to your `user` role), or delete it from a backend service using a confidential service-account client with only the `manage-users` role from `realm-management`. Never expose that credential to the browser.

Keep `KeycloakSubject` on the soft-deleted row so a returning token from the same subject is refused instead of creating a fresh profile.

### 16.2 Email changes

If a user changes their email in Keycloak, your stored `Email` copy is stale. Either ignore it (it is informational), or refresh it from the token on login when it differs and the token's `email_verified` claim is true.

### 16.3 Orphaned profiles

If an admin deletes a user in Keycloak, the local profile remains. That is harmless because nobody can log in as that subject. If you want it tidy, run an occasional admin job that lists local users and checks them against Keycloak's Admin API, from the backend only.

## 17. Full test sequence for profiles and linking

1. Start everything: `dotnet run --project src/TmLabs.AppHost`.
2. Register a new user through Keycloak's **Register** link, or sign in with GitHub or Google.
3. Confirm the access token contains `sub`, `aud: tmlabs-api`, and `realm_access.roles` including your default roles.
4. Call `GET /api/auth/me`: `Roles` should list the realm roles (this proves the claims transformation works).
5. Call `GET /api/chat/users/me`: expect `200` and a new row in the `Users` table, keyed by `KeycloakSubject`.
6. Call it again: expect the same `Id`, no second row.
7. Fire two requests at once with a brand-new user: expect exactly one row.
8. `PUT /api/chat/users/me` with a new display name and bio, then `GET` again to confirm the change stuck.
9. Log out and back in: confirm your edited display name was not overwritten.
10. `GET /api/chat/users` as a normal user: expect `403`. As an `admin`: expect `200` with a page of users.
11. Link a second provider (Account Console > Linked accounts), log in with it, and confirm `sub` and the profile are unchanged.
12. `PUT /api/chat/users/me` with another user's username: expect a validation error, not a `500`.
13. Try to update another user's post: expect `403`.
14. Delete your own account, confirm it's refused if you try to log in again with a cached token whose subject matches.

### Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `403` on an endpoint that requires `admin` although the token has the role | Claims transformation not registered, or `MapInboundClaims` and `RoleClaimType` not set as in 8.1 |
| `Roles` is empty in `/api/auth/me` but the token has `realm_access` | `AddTmLabsAuth` is not called in that service |
| `401` with audience error | Audience mapper missing on `tmlabs-web` (section 3) |
| `401` after changing Keycloak URL | `Authority` does not match the token `iss` exactly |
| Second row created for the same person | Not using `sub` as the key, or two Keycloak users that were never linked |
| New user has no permissions | Default roles not set (section 2.1) |
| Duplicate-key error in logs on first login | Expected under a race; the find-or-create code recovers from it |
| Deleted user reappears with a fresh profile | Query filter hides deleted rows and `IgnoreQueryFilters` was not used in the lookup |
| CORS error on Keycloak token request from Scalar or Next.js | Web Origins mismatch — see section 6.2 |

## 18. Final checklist before going public

- HTTPS on Keycloak, frontend, gateway and services; `RequireHttpsMetadata` true.
- One realm name everywhere; issuer in tokens equals `Authentication:Authority`.
- Email verification and SMTP configured; brute-force detection on.
- Default roles are minimal (`user`, `blog:read`, `chat:read`); write and admin roles are manual.
- First broker login requires verification before linking to an existing account.
- `Trust email` off for providers that do not verify email.
- The `/api/auth/me` diagnostic endpoint removed or restricted to `admin`.
- Every endpoint has a policy or an explicit `AllowAnonymous`, never both.
- Owner IDs come from the token, never from a request body.
- List endpoints have a bounded `PageSize`.
- No tokens, secrets or `.p8` keys in the repository or logs.
- Account deletion and abuse-report processes defined.
