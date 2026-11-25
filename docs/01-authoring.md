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
| `publishedAt` | no | none | also accepts `published`; ISO 8601 |
| `draft` | no | `false` | drafts are stored but hidden from the public list |

Parser constraints worth knowing:

- It is **not** a YAML parser. Only flat `key: value` pairs. Nested keys, lists,
  and multi-line values are ignored, not rejected.
- Surrounding `"` or `'` are stripped.
- Keys are case-insensitive.
- Anything before the first `---` is ignored; no front matter means "derive
  everything from the path".

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
