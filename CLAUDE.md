# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A pure Markdown knowledge base — no build system, no package manager, no tests.
All content lives in four directories: `case-studies/`, `tools/`, `lessons/`, and the root.
"Working on this repo" means writing or editing `.md` files and committing them.

> Note: `CLAUDE-ajk.md` in the root is a stale file from a different project (LLM Council).
> It does not describe this repository and can be deleted.

## Content Structure

| Directory | What goes there |
|---|---|
| `case-studies/` | End-to-end stories: a goal, tools tried in order, failures + successes |
| `tools/` | Single-tool verdict pages: what it is, gotchas, working config |
| `lessons/` | Cross-cutting principles distilled from multiple case studies |

Every entry cross-links related files using relative Markdown paths
(e.g. `[tool page](../tools/n8n-ce.md)`).

## Adding a Case Study

Copy `case-studies/TEMPLATE.md` to `case-studies/NNN-short-slug.md` where `NNN`
is the next sequential number (check existing files). Fill in every section of
the template. Use `> TODO` for gaps rather than omitting sections.

## Verdicts

All tool evaluations use exactly one of three icons — no variations:

- `✅ Worked`
- `⚠️ Partial`
- `❌ Failed`

## Commit Message Convention

```
add: <short description>        # new file(s)
update: <what changed>          # editing existing content
fix: <what was wrong>           # typo / broken link / factual correction
```

## Privacy Rule

This is a public repo. Before committing any content drawn from internal projects:

- Replace specific server domains/IPs with generic placeholders (`your-server.com`)
- Replace user IDs in table names with the pattern (`u{user_id}_vectors`, not `u42_vectors`)
- Remove email addresses that include product or company names
- Remove client names, pricing figures, and private repo URLs
- Keep: version numbers, error messages, commands, architecture patterns, timing data
