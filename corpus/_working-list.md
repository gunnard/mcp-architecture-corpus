# Corpus Working List

MCP servers to evaluate, prioritized by ROI for the skill corpus.

**Target:** 15–20 scored entries. Mix of reference, community-gold, and counter-examples.

**Process per entry:**
1. Read the server's source (or its tool list at minimum)
2. Try invoking 3–5 tools to feel the LLM-side ergonomics
3. Fill in the rubric scoring template
4. Capture 3–5 specific quotes from the source as evidence
5. Tag the entry with patterns it exhibits (good and bad)

---

## Tier 1 — Anthropic reference servers (canonical baselines)

These set the implicit "official" style. Even when imperfect, they shape community
expectations. Score these *first* because they're the comparison baseline for everything else.

| # | Server | Repo / Path | Status | Priority |
|---|--------|-------------|--------|----------|
| 1 | filesystem | `modelcontextprotocol/servers/src/filesystem` | **scored 59/70** | must |
| 2 | github | `github/github-mcp-server` (moved out of monorepo) | **scored 49/70** | must |
| 3 | postgres | `modelcontextprotocol/servers-archived/main/src/postgres` (archived) | **scored 41/70** | must |
| 4 | slack | `modelcontextprotocol/servers-archived/main/src/slack` (likely archived) | pending | must |
| 5 | memory | `modelcontextprotocol/servers/src/memory` | **scored 54/70** | high |
| 6 | fetch | `modelcontextprotocol/servers/src/fetch` | **scored 54/70** | high |
| 7 | sequentialthinking | `modelcontextprotocol/servers/src/sequentialthinking` | pending | high |
| 8 | sqlite | `modelcontextprotocol/servers-archived/main/src/sqlite` (archived) | **scored 35/70** | medium |
| 9 | brave-search | `modelcontextprotocol/servers/src/brave-search` | pending | medium |
| 10 | google-drive | `modelcontextprotocol/servers/src/gdrive` | pending | medium |

---

## Tier 2 — Community gold (real production servers worth imitating)

These are public community servers shipped by teams that ship real product. Higher signal
on what works in production.

| # | Server | Owner | Status | Priority | Notes |
|---|--------|-------|--------|----------|-------|
| 11 | **Brodels** | (you) | **scored 56/70** | must | Worked example; complex multi-domain; ~33 tools |
| 12 | Playwright MCP | Microsoft | **scored 55/70** | must | Browser automation; toolset gating + annotations 5/5 |
| 13 | Stripe MCP | Stripe | **scored 58/70 (prov.)** | high | First 5/5 on D9 auth; remote MCP via OAuth; RAK scoping |
| 14 | Linear MCP | jerhadf community | **scored 50/70** | high | Modern SaaS API; community-grade P-13 example |
| 15 | Cloudflare MCP | Cloudflare | pending | medium | Infra/edge platform exposure |
| 16 | Sentry MCP | Sentry | pending | medium | Observability domain |
| 17 | Notion MCP | community | pending | medium | Watch for which one is canonical (multiple exist) |
| 18 | AWS MCP suite | community | pending | low | Pick one representative server |

---

## Tier 3 — Counter-examples (cite for antipatterns)

Don't score these in full — just capture the specific antipattern they illustrate.

| # | Type | Example to find | Status | What it illustrates |
|---|------|-----------------|--------|---------------------|
| 19 | "raw SQL" tool | early community sqlite MCPs | pending | Prompt-injection surface, no schema discipline |
| 20 | CLI-wrapper mega-tool | various community shell MCPs | pending | God-tool antipattern |
| 21 | unbounded list returns | community CRM wrappers | pending | Pagination antipattern |
| 22 | secret-in-error-message | (find from GH issues) | pending | Auth-leak antipattern |

---

## Status legend

- **pending** — not yet started
- **drafted** — initial pass written, needs verification
- **scored** — rubric filled in with evidence
- **published** — included in skill examples

---

## Notes on what to optimize for

The corpus is **not** about completeness. It is about:

1. **Discriminating power** — the rubric should produce visibly different scores for
   visibly different servers. If everything scores 50/70, the rubric is broken.
2. **Citable evidence** — every claim in the skill output should be able to point at a
   real server's real tool definition.
3. **Pattern emergence** — after ~10 entries, recurring patterns should jump out. If they
   don't, the rubric is missing dimensions.

Stop at 15 entries unless gaps remain. Quality > quantity.
