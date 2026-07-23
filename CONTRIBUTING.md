# Contributing

Thanks for helping keep these recipes accurate. Changes land here as Markdown; [james-barnett.com/homelab](https://www.james-barnett.com/homelab) picks them up on the next site build.

## Recipe files

- One recipe per file under [`recipes/`](recipes/)
- Filename is the URL slug: `recipes/tailscale.md` → `/homelab/tailscale`
- Prefer editing an existing recipe for corrections; open an issue first if you want to propose a wholly new recipe number/title

## Frontmatter

Every recipe needs YAML frontmatter:

```yaml
---
num: "01"           # sort key, zero-padded string
title: "…"
desc: "…"           # one-line summary
tags: ["Tag", "…"]
color: teal         # teal | purple | accent | pink | warn | success
pubDate: 2026-05-17 # YYYY-MM-DD
---
```

## Body conventions

- Start with `## Context`
- Prefer ending with `## When it breaks` (failure modes and recovery)
- Cross-link other recipes with site paths: `/homelab/{slug}`
- Mermaid fences are supported on the site
- Keep tone practical and specific — what to install, in what order, what breaks

## Pull requests

1. Fork and branch from `main`
2. Edit the relevant `recipes/*.md` file(s)
3. Open a PR with a short description of what was wrong and what you changed

No deploy or CI setup is required in this repository.
