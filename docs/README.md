# Blog docs

How content moves from this repository into the TmLabs database and onto the site.

| File | Covers |
| --- | --- |
| [01-authoring.md](01-authoring.md) | File layout, front matter, drafts, what the parser accepts |
| [02-sync-pipeline.md](02-sync-pipeline.md) | What runs on push, the auth chain, the workflow internals |
| [03-troubleshooting.md](03-troubleshooting.md) | Interpreting failures: 401, 403, 502, cache, port-forward |

The wider application (Kubernetes, Argo CD, secrets, keycloak) is documented in
the backend repo: `tmlabs-backend/docs/blueprint/`. These files only cover the
content side.

## The short version

```
push *.md → sync-blog.yml → POST /api/blog/sync → database → POST /api/revalidate → site
```

The API pulls the markdown from GitHub itself. The workflow does not upload
content — it only tells the API "go look now".
