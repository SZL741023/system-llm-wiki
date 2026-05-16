# System Design Wiki

A personal system design knowledge base maintained by an LLM. Based on the [LLM Wiki](llm-wiki.md) pattern.

## What this is

Instead of querying raw documents every time, this wiki compiles knowledge into a persistent, interlinked set of markdown pages. When a new source is added, the LLM reads it, extracts key insights, writes or updates pages, and maintains cross-references throughout the wiki. Knowledge compounds with each source instead of being re-derived on every question.

The human curates sources and asks questions. The LLM writes and maintains everything else.

## Directory layout

```
wiki/
  README.md          ← this file
  CLAUDE.md          ← schema and workflow instructions for the LLM
  llm-wiki.md        ← description of the LLM Wiki pattern
  pages/             ← LLM-maintained wiki pages
    concepts/        ← core system design concepts
    patterns/        ← design patterns
    components/      ← specific technologies
    comparisons/     ← side-by-side comparisons
    case-studies/    ← real-world system analyses
```

## How to use

**Browse** — open in [Obsidian](https://obsidian.md) and use the graph view to navigate links between pages. All pages use `[[wikilink]]` syntax.

**Ingest a new source** — drop a document into `raw/` and ask Claude Code to process it. It will read the source, discuss key takeaways, and write or update relevant pages.

**Query** — ask Claude Code a question. It finds relevant pages, synthesizes an answer with citations, and can file good answers back as new wiki pages.

**Lint** — ask Claude Code to health-check the wiki for contradictions, orphan pages, missing cross-references, and stale claims.

## Tools

- **Claude Code** — LLM agent that writes and maintains the wiki
- **Obsidian** — markdown editor and graph viewer (optional but recommended)
- **Git** — version history and branching for free
