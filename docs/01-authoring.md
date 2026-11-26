# 1. Authoring

## File layout is part of the schema

The sync walks the repository tree and accepts a file only when its path has
**exactly two segments** and ends in `.md`:

```
Archery/my-post.md      ✅ theme = "Archery", slug = "Archery/my-post"
Coding/sql-joins.md     ✅ theme = "Coding"
README.md               ❌ one segment — ignored
Archery/2024/post.md    ❌ three segments — ignored
.github/x.md            ❌ excluded explicitly
```

The **first segment is the theme**. There is no `theme` front-matter key — moving
a file between folders changes its theme.

`README.md` at the root is therefore documentation only; it never becomes a post.

## Front matter

A leading `---` block, then simple `key: value` lines, closed by `---`:

```markdown
---
title: Joins explained
slug: coding/sql-joins
description: How SQL joins actually work.
author: Morris
publishedAt: 2026-11-24T09:00:00Z
draft: false
---

Body markdown starts here.
```

| Key | Required | Default | Notes |
| --- | --- | --- | --- |
| `title` | no | file name | shown in lists and headings |
| `slug` | no | `<Theme>/<file name>` | must be unique; used in URLs |
| `description` | no | first 220 chars of the body | used for excerpts |
| `author` | no | none | displayed under the title |
| `series` | no | none | groups posts into an ordered series |
| `seriesOrder` | no | none | position within `series`; `series_order` also accepted |
| `pinned` | no | `false` | renders the post above everything else on /blog, labelled "Start here" |
| `publishedAt` | no | none | also accepts `published`; ISO 8601 |
| `draft` | no | `false` | drafts are stored but hidden from the public list |

Parser constraints worth knowing:

- It is **not** a YAML parser. Only flat `key: value` pairs. Nested keys, lists,
  and multi-line values are ignored, not rejected.
- Surrounding `"` or `'` are stripped.
- Keys are case-insensitive.
- Anything before the first `---` is ignored; no front matter means "derive
  everything from the path".

## Series

Give related posts the same `series` value and number them with `seriesOrder`:

```markdown
---
title: 3. Images and registry
series: Deployment blueprint
seriesOrder: 4
publishedAt: 2026-11-25T08:16:00Z
---
```

Then:

- `/blog` groups them into one section, ordered by `seriesOrder`, each card
  labelled `Part N`.
- The post page shows `Part N of M`, the series name, and previous/next links.

Current series, in the order `/blog` renders them (groups follow the publication
date of their earliest post):

| Series | Posts |
| --- | --- |
| Deployment blueprint | 13 |
| API and gateway | 2 |
| Identity and access | 7 |
| Platform operations | 2 |

`documentation-index.md` is standalone; `welcome.md` is pinned.

Notes:

- `series` is matched exactly, so keep the spelling consistent across files.
- A post with `series` but no `seriesOrder` still joins the series; those sort
  last, alphabetically by title.
- Series membership is stored per post by the sync, so it only appears after a
  sync runs — a front-matter change alone does not update the database.
- Renaming a series orphans the old grouping until every member is re-synced.

## Pinned posts

`pinned: true` puts a post at the top of `/blog`, above the series and standalone
sections, with a "Start here" badge:

```markdown
---
title: Welcome to the TmLabs blog
pinned: true
publishedAt: 2026-11-25T07:00:00Z
---
```

Multiple pinned posts render in publication order. Currently
`Platform/welcome.md` is the only one — it explains the site's structure and
points at the deployment series.

## Publishing

`publishedAt` is informational — the API publishes a post when `draft` is not
`true`. A post with no date still appears; it just sorts by
`lastModifiedAtUtc` instead.

## Body

Rendered with `react-markdown` + `remark-gfm`, so standard CommonMark and GFM
tables/task lists work. Raw HTML is not rendered.

Code fences are syntax-highlighted client-side; use a language tag for best
results:

````markdown
```csharp
var x = 1;
```
````

## Changing a published post

Commit the change. The sync stores a revision per commit SHA and compares content
checksums, so only changed files are rewritten. Revision history is kept, which
is why the site can show "revision abc1234" in the footer.

## Deleting or unpublishing

- **Delete the file** — the next sync marks the post unpublished.
- **Set `draft: true`** — same effect, history preserved.

Either way the existing record stays in the database; nothing is hard-deleted by
a sync.

Next: [02-sync-pipeline.md](02-sync-pipeline.md)
