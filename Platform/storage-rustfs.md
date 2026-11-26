---
title: Object storage (RustFS)
description: RustFS provides S3-compatible object storage for things that must outlive a pod
series: Platform operations
seriesOrder: 2
publishedAt: 2026-11-25T08:23:00Z
draft: false
---

RustFS provides S3-compatible object storage for things that must outlive a pod:
user avatars today, attachments later. Nothing is written to the container
filesystem, because a rollout replaces the container.

## Endpoint

| Port | Purpose |
| --- | --- |
| `9000` | S3 API — what applications and SDKs use |
| `9001` | Web console — buckets, access keys, metrics |

Current instance: `http://192.168.1.19:9000` (console on `:9001`).

The port mix-up is the most common cause of "connection refused" — the console
URL is not the API URL.

## Bucket

`tmlabs-avatars` — private. The browser never reaches RustFS directly; the Next
server proxies reads, so the bucket needs no public policy.

### Create it

Console: **Buckets → Create Bucket**, name `tmlabs-avatars`, default region,
versioning and object locking off.

Script (idempotent, safe to re-run):

```bash
export S3_ENDPOINT=http://192.168.1.19:9000
export S3_BUCKET=tmlabs-avatars
export S3_ACCESS_KEY=...
export S3_SECRET_KEY=...
./scripts/ensure-s3-bucket.sh
```

Raw curl, if you prefer no script:

```bash
export REGION=us-east-1
curl -sS -o /dev/null -w '%{http_code}\n' \
  -X PUT "$S3_ENDPOINT/$S3_BUCKET" \
  --aws-sigv4 "aws:amz:$REGION:s3" \
  --user "$S3_ACCESS_KEY:$S3_SECRET_KEY"
```

`200` = created, `409` = already exists (both fine). Requires curl ≥ 7.75 for
`--aws-sigv4`.

Access keys are created in the console (**Access Keys → Create Access Key**) —
S3 has no API for that. The secret is shown once.

### Verify

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -I "$S3_ENDPOINT/$S3_BUCKET" \
  --aws-sigv4 "aws:amz:us-east-1:s3" \
  --user "$S3_ACCESS_KEY:$S3_SECRET_KEY"

curl -sS "$S3_ENDPOINT/" \
  --aws-sigv4 "aws:amz:us-east-1:s3" \
  --user "$S3_ACCESS_KEY:$S3_SECRET_KEY"
```

`HEAD` returning `200` proves endpoint, region, signature and bucket name all
agree — the single most useful check.

## Configuration the app will need

Wired as of the profile feature. `S3_ACCESS_KEY` and `S3_SECRET_KEY` come from
OpenBao (`secret/tmlabs/storage` → `access_key` / `secret_key`) through
`web-secrets`; the rest are non-secret and land in the web ConfigMap:

| Variable | Source | Value |
| --- | --- | --- |
| `S3_ENDPOINT` | chart config | `http://192.168.1.19:9000` |
| `S3_BUCKET` | chart config | `tmlabs-avatars` |
| `S3_REGION` | chart config | `us-east-1` (RustFS accepts any consistent value) |
| `S3_ACCESS_KEY` | `web-secrets` | from the console |
| `S3_SECRET_KEY` | `web-secrets` | from the console |

Create the OpenBao entry once:

```bash
bao kv put secret/tmlabs/storage access_key="<key>" secret_key="<secret>"
```

Per [AGENTS.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/AGENTS.md) rule 3, adding a variable means touching the AppHost,
the workflow's `Parameters__*` placeholders, the ExternalSecret, and the tables in
`argocd-deployment.md` and `blueprint/07-configuration-and-secrets.md`.

## Design constraints

- **Store a key, not a URL.** `chat.users.AvatarKey` (`avatars/{userId}/{hash}.webp`)
  so the serving strategy can change without a data migration.
- **Serve through the app.** `/api/avatar/[userId]` streams from RustFS with
  `Cache-Control: public, max-age=31536000, immutable`. Publishing the bucket is
  the alternative, but it widens the public surface for no benefit at this scale.
- **Validate before storing.** Magic-byte check, ~2 MB cap, re-encode to 256×256
  webp with `sharp`. Storing what the browser uploads means storing whatever an
  attacker uploads.
- **Fallback avatars are generated, not stored.** DiceBear renders an identicon
  from the immutable user id (see `components/user-avatar.tsx`), so a user with
  no upload costs no storage and no request.

## When a request fails

| Response | Cause |
| --- | --- |
| `403 SignatureDoesNotMatch` | Wrong secret, or the signed host differs from the URL (don't mix `localhost` and the LAN IP) |
| `400 AuthorizationHeaderMalformed` | Region mismatch — use `us-east-1` consistently |
| `403 AccessDenied` | Key lacks permission on the bucket |
| `409 BucketNotEmpty` | Deleting a bucket that still holds objects |
| `Connection refused` | Console port used instead of `9000` |
