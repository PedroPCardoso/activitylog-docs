# activitylog-docs

Content source for the [activitylog](https://activitylog.readme.io) hosted documentation site, powered by [readme.io](https://readme.com).

## What this is

This repo holds the Markdown files that readme.io reads to render the public docs at **https://activitylog.readme.io**.  
It is **not** the source of truth for the API — that lives in [activitylog](https://github.com/PedroPCardoso/activitylog). When the API docs change there, this repo is updated to reflect them.

## Branch structure

| Branch | Purpose |
|--------|---------|
| `v1.0` | Active content — what readme.io syncs from |
| `main`  | Meta / tooling only (this README, scripts) |

## File layout (v1.0)

```
docs/
  Getting Started/
    getting-started.md   ← mirrors activitylog docs (English)
    _order.yaml
  _order.yaml
reference/
  ReadMeConfig/          ← readme.io reference pages
```

Each Markdown file starts with a readme.io frontmatter block:

```yaml
---
title: <page title>
excerpt: <short description>
hidden: false
---
```

## How to update

After merging doc changes into **activitylog**, push the updated content here:

```bash
# 1. Get the current SHA of the file to update
gh api "repos/PedroPCardoso/activitylog-docs/contents/docs/Getting%20Started/getting-started.md?ref=v1.0" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['sha'])"

# 2. Push the new content via the GitHub API
```

## Related

- [activitylog](https://github.com/PedroPCardoso/activitylog) — monorepo (activitylog-core + activitylog-nestjs + activitylog-nextjs)
- [activitylog.readme.io](https://activitylog.readme.io) — live docs site
