---
title: Welcome to the TmLabs blog
description: What lives here, how it is organised, and where to start reading.
pinned: true
publishedAt: 2026-11-25T07:00:00Z
draft: false
---

This is the TmLabs engineering log. Everything here is written while building
the platform, so it documents decisions as they were made rather than after the
fact.

## Where to start

If you are new, read the **Deployment blueprint** series below. It follows one
application from `dotnet run` on a laptop to a Helm chart applied by Argo CD, in
order, including the things that broke along the way.

Part 1 is the overview; parts 2–13 walk through architecture, images, CI, the
chart pipeline, GitOps, configuration and secrets, networking, identity,
database, operations, and finally a checklist for starting a new app from the
same pattern.

## How this site is organised

Everything is grouped into series, which `/blog` renders as ordered sections with
numbered parts:

| Series | Parts | Contents |
| --- | --- | --- |
| Deployment blueprint | 13 | One app from `dotnet run` to a Helm chart applied by Argo CD |
| API and gateway | 2 | The YARP gateway and FastEndpoints setup |
| Identity and access | 7 | Auth model, Keycloak, Scalar, Cloudflare tunnel, user linking, checklist |
| Platform operations | 2 | Argo CD deployment and RustFS object storage |

`Coding` and `Archery` hold posts that are not part of a series.

## How posts get here

Markdown lives in a separate content repository. Pushing it triggers a sync that
pulls the files over the GitHub API into PostgreSQL, then revalidates the pages so
changes appear immediately. Nothing is hand-edited on the site.

Posts carry front matter for title, description, and optional `series` and
`seriesOrder` values, which is what produces the numbered parts you see above.

## What is deliberately missing

There are no comments, no email capture, and no analytics. If something here is
wrong or has aged badly, the source repository is the place to fix it.
