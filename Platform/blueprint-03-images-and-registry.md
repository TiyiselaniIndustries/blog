---
title: 3. Images and registry
description: Five images ship: three .NET services, the frontend, and (originally) the Aspire
series: Deployment blueprint
seriesOrder: 4
publishedAt: 2026-11-25T08:03:00Z
draft: false
---

Five images ship: three .NET services, the frontend, and (originally) the Aspire
dashboard — dropped, see below.

## Registry

A self-hosted **Zot** registry at `192.168.1.10:8080`, namespace `tmlabs`.

```
192.168.1.10:8080/tmlabs/gateway:<tag>
192.168.1.10:8080/tmlabs/blogapi:<tag>
192.168.1.10:8080/tmlabs/chatservice:<tag>
192.168.1.10:8080/tmlabs/web:<tag>
```

`<tag>` is `<short-sha>-<run-number>`, e.g. `ebdf805-6`. Unique per run, so a
deployment always references an exact build and rolling back is just a tag change.

## .NET services

Built with the SDK's container publish — no Dockerfile per service:

```bash
dotnet publish "src/TmLabs.BlogApi" -c Release -r linux-x64 \
  -t:PublishContainer \
  -p:ContainerRegistry="192.168.1.10:8080" \
  -p:ContainerRepository="tmlabs/blogapi" \
  -p:ContainerImageTags="ebdf805-6"
```

Notes:

- `-r linux-x64` matters on an ARM laptop; without it the image won't run on the
  cluster's amd64 nodes. If you build arm64 images locally, they'll fail on the
  nodes with `exec format error`.
- The registry is plain HTTP, so the building machine needs it trusted. On the
  runner, `DOTNET_CONTAINER_INSECURE_REGISTRIES` is set in the workflow env;
  locally, use `--allow-insecure-registry` or configure the Docker daemon.
- `ContainerImageTags` accepts semicolons for multiple tags (`ebdf805-6;latest`)
  — deliberately not used here, because `latest` invites `IfNotPresent` mistakes.

## Frontend

A real Dockerfile, because Next.js needs a build stage:

- `deps` → `npm ci`
- `builder` → declares `ARG NEXT_PUBLIC_*`, exports them as `ENV`, runs `npm run build`
- `runner` → `node:22-alpine`, copies `.next/standalone` and `.next/static`

```bash
docker build \
  --build-arg NEXT_PUBLIC_API_BASE_URL="http://gateway-service:8080" \
  --build-arg NEXT_PUBLIC_KEYCLOAK_URL="https://auth.tiyiselani.com" \
  --build-arg NEXT_PUBLIC_KEYCLOAK_REALM="non-prod" \
  --build-arg NEXT_PUBLIC_KEYCLOAK_CLIENT_ID="tmlabs-frontend" \
  --build-arg NEXT_PUBLIC_KEYCLOAK_REDIRECT_URI="https://blog-test.tiyiselani.com/api/auth/callback/keycloak" \
  -t 192.168.1.10:8080/tmlabs/web:ebdf805-6 tmlabs-frontend
```

## Trap: Next.js prerendering at build time

`next build` runs with **no cluster access**. Any page that fetches data during
prerender will fail the build when the API is unreachable — and the gateway is
not deployed yet when the image is built.

Symptom:

```
⨯ Error: NEXT_PUBLIC_API_BASE_URL is not configured.
Error: When using Cache Components, all `generateStaticParams` functions must return at least one result.
```

Both pages and routes were fixed the same way: defer the fetch to request time.

```tsx
import { Suspense } from "react";
import { connection } from "next/server";

export default function Page() {
  return <Suspense fallback={<Loading />}><Content /></Suspense>;
}

async function Content() {
  await connection();          // opt out of prerendering
  const posts = await getBlogList();
}
```

For dynamic routes, `generateStaticParams` must still return something:

```tsx
export function generateStaticParams() {
  return [{ id: "build-placeholder" }];   // real ids render on demand
}
```

## Trap: the Aspire dashboard image

The generated chart deploys
`mcr.microsoft.com/aspire/nightly/dashboard:17.0`. It is a **nightly** build and
segfaulted on this cluster (exit code 139), and the URL is hardcoded in the
template so a values override can't replace it.

The chart pipeline deletes it (`rm -rf chart/templates/k8s-dashboard`) and blanks
the OTLP endpoint so services stop exporting to a collector that no longer
exists. If you want telemetry later, pin a stable dashboard tag instead.

## Verification

```bash
curl -s http://192.168.1.10:8080/v2/tmlabs/web/tags/list
# {"name":"tmlabs/web","tags":["ebdf805-6", ...]}
```

Next: [04-ci-pipeline.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/04-ci-pipeline.md)
