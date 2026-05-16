# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This is a system design wiki maintained by an LLM (you). It is not a code project. There is no build step, no tests, no linter. The "codebase" is a growing collection of markdown files covering system design concepts, patterns, and reference material.

Your job: maintain the wiki. Read sources, extract knowledge, write and update pages, keep cross-references accurate, keep the index current.

## Directory layout

```
wiki/
  CLAUDE.md          ← this file (the schema)
  index.md           ← catalog of all wiki pages (you maintain this)
  log.md             ← append-only operation log (you maintain this)
  raw/               ← immutable source documents (you read, never modify)
    assets/          ← locally downloaded images referenced by sources
    foundation_concept/  ← buildmoat 系統設計基礎課程（16 份 PDF）
  pages/             ← wiki pages you write and maintain
    concepts/        ← system design concepts (e.g. CAP theorem, consistent hashing)
    patterns/        ← design patterns (e.g. saga, CQRS, circuit breaker)
    components/      ← specific technologies and components (e.g. Kafka, Redis, Zookeeper)
    comparisons/     ← side-by-side comparisons between related things
    case-studies/    ← analyses of real-world systems
```

Create directories as needed. The structure above is a starting point, not a constraint.

## Core operations

### Ingest

When the user drops a source into `raw/` and asks you to process it:

1. Read the source fully. If it references images locally in `raw/assets/`, view them too.
2. Discuss key takeaways with the user before writing anything. Confirm emphasis.
3. Write a summary page in `pages/` under the appropriate subdirectory.
4. Update all relevant existing pages — add cross-references, revise claims contradicted by the new source, strengthen or challenge the synthesis.
5. Update `index.md` with the new page(s).
6. Append an entry to `log.md` in the format: `## [YYYY-MM-DD] ingest | <Source Title>` followed by a 2–3 line summary of what changed.

A single source commonly touches 5–15 pages. Don't shortcut on cross-references.

### Query

When the user asks a question:

1. Read `index.md` to find the relevant pages.
2. Read those pages and synthesize an answer with citations (link to the wiki pages, not raw sources).
3. If the answer is non-trivial and reusable — a comparison, an analysis, a synthesis — offer to file it as a new wiki page. Good answers compound the knowledge base.

Output format depends on the question: prose, a comparison table, or a Marp slide deck if the user wants a presentation. Ask if unclear.

### Lint

When the user asks you to health-check the wiki:

- Find contradictions between pages.
- Find stale claims superseded by newer sources.
- Find orphan pages (no inbound links from other pages).
- Find important concepts mentioned but lacking their own page.
- Find missing cross-references between pages that discuss the same thing.
- Suggest new sources to look for or questions worth exploring.

Report findings as a list. Fix what you can; flag what needs the user's input.

## Page conventions

**Frontmatter** — include YAML frontmatter on every wiki page:

```yaml
---
title: <page title>
tags: [concept, distributed-systems]   # use existing tags when possible
sources: 2                              # number of raw sources this page draws from
updated: YYYY-MM-DD
---
```

**Cross-references** — use `[[Page Title]]` wikilink syntax (Obsidian-compatible). Always link on first mention of a concept that has its own page.

**Page structure** — no rigid template, but prefer: a one-paragraph summary at the top (readable standalone), then sections for depth. End with a `## See also` section listing related pages.

**Contradictions** — when two sources conflict, note it explicitly in the page body: `> **Contradiction:** Source A claims X; Source B claims Y. Likely explained by Z.`

## index.md format

Group entries by category. Each entry is one line:

```
- [[Page Title]] — one-line description. (N sources)
```

## log.md format

Append-only. Each entry starts with a header parseable by grep:

```
## [YYYY-MM-DD] <operation> | <label>

Short description of what changed.
```

Operations: `ingest`, `query`, `lint`, `edit`.

## Working style

- Never modify files in `raw/`. They are the source of truth.
- Update `index.md` and `log.md` on every operation that changes wiki content.
- When in doubt about page placement or emphasis, ask the user before writing.
- Prefer updating an existing page over creating a new one for minor additions.
- Keep page titles stable — they are the targets of wikilinks across the wiki.
