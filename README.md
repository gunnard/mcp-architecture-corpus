# MCP Server Architecture: Corpus + Rubric

A working corpus of **13 production MCP servers** scored against a **14-dimension rubric**,
with **29 design patterns** and **11 antipatterns** distilled and cited.

This repo is the public, free tier of an evidence-based MCP design framework.

> **TL;DR for HN:** I read every official MCP server's source code so you don't have to.
> Findings: GitHub's official MCP scored 49/70 because of god-tool dispatch. The simpler
> Filesystem reference scored 59/70. The SQLite reference has a real security flaw I documented
> as antipattern AP-11. Stripe's MCP is a thin proxy to their hosted backend. The MCP resources
> facility is dramatically underused.

---

## What's in this repo

```
corpus/
├── _rubric.md              14-dimension scoring rubric (the methodology)
├── _patterns.md            29 design patterns with citations
├── _antipatterns.md        11 antipatterns with corpus evidence
├── _working-list.md        Backlog of servers to score
├── reference/              7 official Anthropic reference servers
│   ├── filesystem.md       59/70  ← highest reference-tier score
│   ├── github.md           49/70  ← god-tool antipattern (AP-03)
│   ├── memory.md           54/70
│   ├── postgres.md         41/70  ← AP-04 raw SQL passthrough
│   ├── sqlite.md           35/70  ← AP-11 safety theater (real security flaw)
│   ├── slack.md            45/70  ← pre-modern baseline (no annotations/resources)
│   └── fetch.md            54/70  ← canonical chunked pagination
└── community/              6 community / vendor servers
    ├── sentry.md           65/70  ← highest in corpus · resource resolver (P-26), embedded agent (P-27)
    ├── stripe.md           58/70  ← thin-proxy + hosted-remote OAuth (P-21)
    ├── cloudflare.md       57/70  ← Code Mode (P-29): 2,594 endpoints → 2 sandboxed tools
    ├── brodels.md          56/70  ← internal CI/CD orchestrator (anonymized)
    ├── playwright.md       55/70  ← canonical browser automation
    └── linear.md           50/70  ← first community server using resources correctly
archetypes/
└── _index.md               8 archetypes: catalogue + decision tree
```

Every score has quote-level evidence. Every pattern has at least one corpus citation.
Every antipattern has at least one counter-example.

---

## How to use this corpus

### If you're building an MCP server

1. Read [`corpus/_rubric.md`](corpus/_rubric.md) — score your own design against the 14 dimensions
2. Read [`corpus/_patterns.md`](corpus/_patterns.md) — adopt the patterns that fit
3. Read [`corpus/_antipatterns.md`](corpus/_antipatterns.md) — avoid the failure modes
4. Read [`archetypes/_index.md`](archetypes/_index.md) — find the archetype closest to your domain

### If you're evaluating an MCP server

The 13 corpus entries are templates. Apply the rubric to any server you're considering and
compare scores. Most production servers in the wild score in the 40-55 range.

### If you want a tailored architecture audit

The detailed archetype specifications (8 documents, 2,347 lines) and the actual skill prompt
that produces audits are kept behind a paid tier. See [Paid audit](#paid-audit) below.

---

## The 14-dimension rubric (overview)

Full rubric in [`corpus/_rubric.md`](corpus/_rubric.md). Dimensions:

| # | Dimension | What it measures |
|---|---|---|
| D1 | Tool granularity | Atomic verbs vs god-tool dispatch |
| D2 | Tool naming | Verb+object discipline, namespacing |
| D3 | Schema discipline | Parameter types, descriptions, constraints |
| D4 | Error handling | Structured errors, no secret leakage |
| D5 | Idempotency | Retry safety, side-effect declaration |
| D6 | Annotations | `readOnlyHint` / `idempotentHint` / `destructiveHint` |
| D7 | Dual return | `content` (LLM) + `structuredContent` (programmatic) |
| D8 | Resources facility | Entity-per-resource correctness |
| D9 | Auth | OAuth, scoped keys, audit trail |
| D10 | Observability | Audit logs, cost tracking, error visibility |
| D11 | Output management | Inline vs filename redirect for large outputs |
| D12 | Pagination | Cursor-based, truncation signals |
| D13 | Documentation | README, tool tables, system prompt examples |
| D14 | Safety rails | Multi-gate ops, lockdown modes, prompt-injection warnings |

Each dimension scores 1-5. Max score: 70.

---

## Notable findings

### 30-point spread across 13 production servers (35-65 / 70)

Maturity ≠ design quality. The simpler reference servers (Filesystem, Fetch, Memory) score
higher than the more featureful ones (GitHub, SQLite). Atomic-tool design beats feature-richness
for LLM consumption. The top of the range (Sentry at 65/70) shows what disciplined design plus
modern primitives — OAuth, resources, agent-mode, observability — looks like when they're all
present simultaneously.

### The `actions_run_trigger` god-tool

GitHub MCP's CI tools accept a `method: enum[run_workflow, get_workflow_run, ...]` parameter
and dispatch internally. The description IS the dispatch table. This is antipattern AP-03,
documented across 4 corpus servers. Atomic verb+object naming is the universal mitigation.

### SQLite MCP's safety-theater finding

The reference SQLite server claims read-only enforcement on its `read_query` tool. The
implementation uses `query.strip().upper().startswith(('INSERT', 'UPDATE', ...))`. This is
trivially defeated by:

```sql
SELECT 1; DROP TABLE users;
```

The Postgres MCP, by contrast, wraps queries in `BEGIN TRANSACTION READ ONLY` then
`ROLLBACK` — DB-layer enforcement, not string-prefix matching. Documented as AP-11.

### Hosted-remote MCP convergence across Stripe, Sentry, Cloudflare

The `@stripe/mcp` npm package is ~80 lines that forwards stdio over HTTPS to
`https://mcp.stripe.com`. All tool logic is server-side, OAuth-protected, scoped via
existing Restricted API Keys. Documented as patterns P-21 (hosted remote MCP) and
P-25 (stdio-to-HTTPS proxy).

Sentry and Cloudflare independently converged on the same architecture — three vendors,
three independent codebases, same pattern. When three large independent domains pick the
same shape, that's the signal that the pattern is the right answer for vendor-hosted MCPs.

### MCP resources are dramatically underused

Most servers expose entities (issues, contacts, tickets) as TOOLS that return them. Done
correctly, those entities should be MCP resources — addressable, cacheable, auto-discoverable.
Postgres MCP and Linear MCP are the only corpus servers using resources idiomatically.

---

## Paid audit

The 8 detailed archetype specifications and the actual audit-generating skill are paid:

- **$79 baseline audit** — read-only data sources, simple SaaS reads, internal tools
- **$149 premium audit** — money movement, prod infra, code merges, PII/regulated data,
  multi-archetype hybrids

Each audit is 2,000-6,000 words: tool surface, resources, auth, safety rails, antipatterns
to avoid, suggested file structure, first tool to implement, specific risks to test early.
Every claim cited to the corpus.

**[See pricing, audit options, and skill stack at gunnard.org/mcp-architect →](https://gunnard.org/mcp-architect)**

---

## Methodology notes

### Why 13 servers

After ~10 entries, patterns started repeating across servers — that's the signal a pattern
is real vs. a quirk. We continued scoring past the convergence threshold to span more
archetypes (Sentry for observability with embedded agent tooling, Cloudflare for Code
Mode + sanctioned shell-wrapper mitigations, Slack as the pre-modern Anthropic baseline)
and to surface new patterns (P-26 Resource Resolver, P-27 Embedded Agent, P-29 Code Mode)
rather than to chase numerical breadth.

### Why these 13

Selected to span:
- **Reference vs. community** (7 + 6)
- **Read-only vs. writable** (covered both)
- **Simple vs. featureful** (Filesystem to GitHub)
- **Local stdio vs. hosted remote** (most stdio + Stripe / Sentry / Cloudflare hosted)
- **Different archetypes** (read-only data, writable system, search/fetch, workflow,
  memory, UI automation, telemetry, code/dev-tools)
- **Generational coverage** (Slack as pre-modern baseline through Sentry / Cloudflare
  as 2026-era OAuth + resources + agent-mode references)

### How scores are calibrated

Two independent scorers should land within ±2 per dimension. The rubric is not subjective —
each dimension has explicit 1-5 criteria. Disagreements are documented in the corpus entry's
"verification log."

### Limitations

- 13 entries is still a small N. Bootstrap confidence in any single score is weaker than the
  cross-server pattern signal.
- Some corpus entries scored on documentation only (not source inspection). These are
  marked "partial" or "provisional" in the entry header.
- The archetype catalogue may have gaps. Beta testing will surface them.

---

## License

MIT. Use the rubric, the patterns, the antipatterns, and the corpus entries however
you'd like. Citations appreciated but not required.

The detailed archetype specifications and skill prompt are NOT part of this MIT
release — they're paid IP.

---

## Contributing

If you'd like to:

- **Score a server** that's not in the corpus, open a PR with a draft entry following
  the template structure
- **Dispute a score**, open an issue with the specific dimension and counter-evidence
- **Suggest a pattern or antipattern**, open an issue with corpus citation(s)

I'll merge useful contributions and credit contributors in changelog posts.

---

## Citation

If this corpus informs your work, citations are appreciated:

```
Engebreth, G. (2026). MCP Server Architecture Corpus.
https://github.com/gunnard/mcp-architecture-corpus
```

---

## Status

**v0.2 (May 2026).** Beta. The corpus, rubric, patterns, antipatterns, and archetype
catalogue are stable. The detailed archetype specifications and the audit skill are
in private beta with 10 testers. Public skill launch targeted Q3 2026.
