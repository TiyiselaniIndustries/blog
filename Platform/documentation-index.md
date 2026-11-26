---
title: TmLabs backend: documentation index
description: This directory holds learning notes and architecture documentation for the backend.
publishedAt: 2026-11-25T08:24:00Z
draft: false
---

This directory holds learning notes and architecture documentation for the backend.

## Contents

- `gateway.md` — how the API gateway was added and why it is useful for this project.
- `auth-flow.md` — the recommended public auth architecture for Next.js, GitHub/Google sign-in, passkeys, JWT validation, and service-level authorization.
- `auth-technical.md` — the package-level and implementation-level guide for the frontend, gateway, and microservice auth stack.
- `keycloak-scalar-setup.md` — the implementation guide for Keycloak, JWT validation, Aspire, and Scalar in this repository.
- `auth-checklist.md` — the setup, verification, and production hardening checklist.
- `user-linking-and-database-setup.md` — how Keycloak identities map to local users and how to provision and migrate the PostgreSQL schemas.
- `argocd-deployment.md` — deploying to Kubernetes with Argo CD: CI pipeline, GitOps layout, secrets, Ingress, and the first-deploy checklist.
- `storage-rustfs.md` — object storage: bucket setup, credentials, and the routing/validation constraints for uploads.
- `blueprint/` — a step-by-step trail of how this app is built and deployed, written as a template for future apps. Start at [`blueprint/README.md`](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/README.md).
