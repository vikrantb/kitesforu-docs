# Markdown Viewer Improvement Prompt

Give this to the markdown viewer builder AI. It describes problems and ideas — let them explore creatively.

---

## Prompt

You are improving a custom markdown viewer (`utils/markdown-viewer.html`) that renders technical documentation. The viewer already supports standard CommonMark, GFM tables, mermaid diagrams, KaTeX math, `<details>` collapse, heading fold, syntax-highlighted code blocks, and an optional `<!-- narrate: -->` TTS system.

We need to solve **comprehension and navigation problems** that make long technical documents hard to read. The audience is a technical founder who reads these docs to understand what was built, make decisions, and give feedback. They read on desktop.

### Constraint: No Context Bloating
Any features that add tokens to the markdown source must be minimal. The source files get loaded into LLM context windows. Prefer features that:
- Use existing standard syntax (frontmatter, HTML comments, blockquotes)
- Render from patterns already in the markdown (headings, links, tables)
- Add enrichment at render-time, not write-time

### Problems to Solve

**1. Frontmatter-Driven Status Dashboard**
Technical docs often describe shipped features. The reader wants to know at a glance: Is this deployed? What version? How many components? Currently this is buried in prose.

*Idea:* Parse YAML frontmatter with structured status data and render a visual dashboard bar at the top — like GitHub badges but richer. Example frontmatter:

```yaml
---
title: Content Domain System
status: shipped
date: 2026-03-15
metrics:
  - label: PRs Merged
    value: "6"
  - label: Domains Live
    value: "27"
  - label: API Rev
    value: "434"
  - label: Workers Rev
    value: "250"
repos:
  - name: kitesforu-api
    github: https://github.com/vikrantb/kitesforu-api
    branch: main
  - name: kitesforu-workers
    github: https://github.com/vikrantb/kitesforu-workers
    branch: main
---
```

Render this as a slim, scannable dashboard strip below the title. Status gets a colored badge (green=shipped, yellow=in-progress, red=blocked). Metrics render as a horizontal row of stat cards.

**2. GitHub File Link Resolution**
Technical docs reference code files constantly. Currently authors write relative paths or manually construct GitHub URLs. The viewer should auto-resolve file references.

*Idea:* When the frontmatter defines `repos` with `github` and `branch`, any markdown link whose URL starts with `src/` or matches a known repo path pattern should auto-resolve to a full GitHub URL. Example: `[content_domain.py](src/workers/prompts/content_domain.py)` → resolves to `https://github.com/vikrantb/kitesforu-workers/blob/main/src/workers/prompts/content_domain.py` and opens in new tab with a GitHub icon.

Even better: when a GitHub file link is the sole content of a list item or paragraph, render it as a compact file card showing filename, repo badge, and path breadcrumb.

**3. Callout Blocks with Custom Types**
GitHub already supports `> [!NOTE]`, `> [!WARNING]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!CAUTION]`. Extend the viewer to support additional custom types useful for technical docs:

- `> [!TLDR]` — Blue/teal. For section summaries. Visually prominent.
- `> [!DECISION]` — Purple. For architectural decisions with context.
- `> [!COST]` — Orange. For cost implications.
- `> [!BEFORE_AFTER]` — Split. For showing what changed.

These use standard blockquote syntax so they degrade to readable text in GitHub/VS Code.

**4. Section Anchor Minimap**
Long docs need a persistent visual that shows the reader where they are in the document structure. The existing TOC sidebar helps, but for documents with 5+ major sections, a horizontal "progress bar" minimap could work better.

*Idea:* A thin horizontal bar below the title/dashboard that shows all H2 sections as labeled segments. The current section is highlighted. Clicking a segment scrolls to it. Think of it like a subway map for the document.

**5. Abbreviation Tooltips from Frontmatter**
Technical docs use many abbreviations (LLM, TTS, GCS, SSE, API, etc.). Define them once in frontmatter, get hover tooltips everywhere.

```yaml
abbreviations:
  LLM: Large Language Model
  TTS: Text-to-Speech
  GCS: Google Cloud Storage
  SSE: Server-Sent Events
```

On hover over any occurrence of "LLM" in the text, show a tooltip with "Large Language Model". No extra markup needed in the document body.

**6. Mermaid Diagram Click-to-Section**
When a mermaid diagram has nodes that correspond to document sections, clicking a node should scroll to that section. This requires a convention: if a mermaid node ID matches a heading slug (e.g., `intake` matches `## Intake`), make it clickable.

### Implementation Notes
- All enhancements should be in the single HTML viewer file
- Use the existing markdown-it pipeline for parsing
- New features should degrade gracefully (frontmatter ignored by other renderers, blockquotes show as quotes, links stay as links)
- Keep the viewer fast — no external API calls at render time
- Test with documents that have 5-10 mermaid diagrams and 20+ file references

### Priority Order
1. Frontmatter dashboard (highest impact for status-at-a-glance)
2. GitHub file link resolution (most frequently needed)
3. Custom callout blocks (visual hierarchy)
4. Abbreviation tooltips (comprehension aid)
5. Section minimap (navigation)
6. Mermaid click-to-section (nice-to-have)
