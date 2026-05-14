# MCP Server Archetypes — Index

An archetype is a **recurring shape of MCP server design** keyed to a buyer's intent.
When the skill receives a domain description, it maps the description to one (or a
hybrid of) these archetypes and emits an architecture spec.

The archetype catalogue captures: which patterns apply, which antipatterns to flag,
and which corpus entries provide the canonical worked examples.

---

## The archetype catalogue

| # | Archetype | Buyer intent (paraphrase) | Reference servers | Status |
|---|-----------|---------------------------|-------------------|--------|
| 01 | **Read-only data source** | *"I want Claude to read my data."* (DB, files, web content, API search) | Postgres (canonical), Fetch (HTTP), Filesystem (files) | **drafted** |
| 02 | **Writable system** | *"I want Claude to create/update/delete records in my SaaS."* | Stripe (canonical), Linear, GitHub | **drafted** |
| 03 | **Search and fetch** | *"I want Claude to search a body of content, then drill into specific results."* | Memory (canonical), Stripe, Brodels | **drafted** |
| 04 | **Workflow orchestrator** | *"I want Claude to drive a multi-phase pipeline."* | Brodels (canonical), GitHub (CI), Playwright | **drafted** |
| 05 | **State-management / memory** | *"I want Claude to remember things across sessions."* | Memory (canonical), Brodels (KB), SQLite | **drafted** |
| 06 | **UI / browser automation** | *"I want Claude to drive a browser or GUI."* | Playwright (canonical) | **drafted** |
| 07 | **Observability / telemetry** | *"I want Claude to investigate errors, logs, or metrics."* | Brodels (canonical), Sentry, Postgres | **drafted** |
| 08 | **Code / dev-tools integration** | *"I want Claude to interact with code repos, PRs, CI."* | Brodels (canonical), GitHub (counter-example), Stripe | **drafted** |

---

## How the skill picks an archetype

Decision tree, applied in order. The first match wins; hybrids are common.

```
1. Does the LLM need to CHANGE state in an external system?
   YES → archetype 02 (writable system)
   NO  → continue

2. Does the LLM need state PERSISTED across sessions?
   YES → archetype 05 (state-management)
   NO  → continue

3. Does the LLM need to drive a UI or browser?
   YES → archetype 06 (UI automation)
   NO  → continue

4. Is the domain primarily DRIVING A MULTI-STEP PROCESS?
   YES → archetype 04 (workflow orchestrator)
   NO  → continue

5. Is the primary purpose INVESTIGATING errors, logs, traces?
   YES → archetype 07 (observability)
   NO  → continue

6. Does the LLM SEARCH first, then DRILL INTO results?
   YES → archetype 03 (search and fetch)
   NO  → continue

7. Default: archetype 01 (read-only data source)
```

**Hybrid examples:**
- *"I want Claude to read my database AND create reports as PRs"* → 01 + 02 + 08
- *"I want Claude to investigate a Sentry error AND remember what it learned"* → 07 + 05
- *"I want Claude to drive a browser AND log what it found"* → 06 + 07
- *"I want Claude to search docs AND apply edits to files"* → 03 + 01 (files) + 02 (writes)

The skill should detect hybrid intent and emit a layered architecture spec that
combines the relevant archetypes.

---

## What an archetype document contains

Each archetype file has the following sections:

1. **Buyer intent signals** — phrases/keywords that map to this archetype
2. **The canonical architecture** — tools, resources, prompts, auth, transport
3. **Required patterns** — which P-IDs apply (with citations to corpus)
4. **Antipatterns to flag** — which AP-IDs are common pitfalls in this archetype
5. **Decision branches** — variations and when to apply each
6. **Worked skeleton** — code-shaped template for the architecture spec
7. **Corpus citations** — which corpus entries to reference in skill output
8. **Common buyer mistakes** — things the skill should proactively warn about

---

## Archetype maturity & corpus coverage

| Archetype | Corpus citations available | Confidence |
|---|---|---|
| 01 Read-only data source | Postgres (canonical), Fetch (text content), Filesystem (files), Linear (entity-per-resource) | **high** |
| 02 Writable system | Stripe (canonical), Linear, GitHub (with caveats), Brodels (ops domain) | **high** |
| 03 Search and fetch | Memory (search_nodes/open_nodes pair), Fetch, Stripe (search_*/fetch_*) | medium |
| 04 Workflow orchestrator | Brodels (phase pipeline), GitHub (actions toolset) | medium |
| 05 State-management | Memory (knowledge graph), SQLite (memo://insights), Brodels (KB) | medium |
| 06 UI automation | Playwright (canonical) | medium (one server) |
| 07 Observability | Brodels (best-in-corpus on D10), Sentry (need source scoring) | medium |
| 08 Code/dev-tools | GitHub, Brodels | medium |

**Archetypes 01 and 02 are ready to ship.** They cover the bulk of expected buyer
intent (estimated ~60-70% of incoming questions). Archetypes 03-08 should be drafted
before the skill goes live, but in priority order.

---

## When NOT to use an archetype

The archetype catalogue captures patterns. It does NOT cover:

- Highly novel domains where no corpus server is analogous
- Custom-protocol servers that don't follow MCP conventions
- Performance-sensitive servers where the standard patterns add unacceptable overhead

For these, the skill should fall back to the **rubric-only mode**: emit a 14-dimension
architecture review against the rubric, without claiming archetype fit.
