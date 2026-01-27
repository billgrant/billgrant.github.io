# AI Coding Guidelines — billgrant.io Blog

> Instructions for Claude and other AI assistants when contributing to this Jekyll site.

## Blog Documentation

Each demo-app phase gets a blog post at https://billgrant.io

### Blog Details
- **Repo:** `~/code/billgrant.github.io` (Jekyll site)
- **Theme:** Minimal Mistakes (`mmistakes/minimal-mistakes@4.24.0`), dark skin
- **Posts directory:** `_posts/`
- **Filename format:** `YYYY-MM-DD-title.md`
- **Tag:** `#demo-app`

### Front Matter

**IMPORTANT:** Do NOT set `layout: post` in front matter. The site uses Minimal Mistakes theme which uses `layout: single`. The `_config.yml` already sets this as the default for all posts — no explicit layout is needed.

Use YAML lists for categories and tags, not space-separated strings.

```markdown
---
title: "Demo App: [Phase/Topic]"
date: YYYY-MM-DD
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - learning-in-public
---
```

### Writing Style & Workflow

**The `*** ***` markers serve two purposes:**
1. Bill's voice — intro sets context, outro reflects on learnings
2. **AI transparency disclaimer** — signals to readers that the main content below was written by Claude, not Bill. This is intentional transparency about AI use.

**Workflow:**
1. Claude drafts the full post using DEVLOG.md session notes
2. Bill rewrites the intro/outro sections in his own voice
3. Bill reviews main content, adds screenshots or additional context
4. Claude spell-checks and grammar-checks Bill's edits
5. Bill commits and pushes to the blog repo

**Content guidelines:**
- Documents successes AND failures — the learning journey matters
- Include code snippets, decisions made, and lessons learned
- Main content is Claude's synthesis of the sessions, not purely Bill's writing

### Blog Post Structure (suggested)
```markdown
---
title: "Demo App: [Phase/Topic]"
date: YYYY-MM-DD
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - learning-in-public
---

*** Bill's intro — sets context, what we set out to do ***

[Main content — what happened, code examples, decisions]
```

Note: Intro only, no outro. The `*** ***` markers signal AI transparency — content below is Claude's synthesis of the sessions.

### Jekyll/Template Code in Posts
When showing Go templates or any curly-brace syntax in blog posts, wrap with raw tags to prevent Jekyll from processing:

```
{% raw %}
{{ .SomeVariable }} or {{ if .Condition }}{{ end }}
{% endraw %}
```

This applies to:
- Go `html/template` or `text/template` examples
- Any `{{ }}` syntax that Jekyll might try to interpret

## Known Issues / Lessons Learned

### Do NOT use `layout: post` (2026-01-27)
Phases 7 and 8 blog posts were missing CSS because their front matter had `layout: post`. Minimal Mistakes uses `single`, not `post`. Since `_config.yml` sets the default layout for all posts, omit the layout field entirely.

---

*Last updated: 2026-01-27*
