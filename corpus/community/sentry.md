# Sentry MCP

**Source:** https://github.com/getsentry/sentry-mcp (`packages/mcp-core/src/tools/`)
**Variant inspected:** `main` branch, remote-MCP edition (Cloudflare-hosted). Companion stdio package at `getsentry/sentry-mcp-stdio` is documented as superseded.
**Vendor:** Sentry (vendor-maintained, like Stripe and Playwright).
**Archetype fit:** Primary **07 — Observability / Telemetry**. Secondary **02 — Writable System** (write surface is intentionally narrow: `update_issue`, `update_project`, `create_team`, `create_project`, `create_dsn`).
**Status:** Source verified — read `tools/index.ts` (canonical export), `tools/get-sentry-resource.ts`, `tools/use-sentry/handler.ts`, `tools/search-events/handler.ts`, plus `AGENTS.md` and architecture/security doc references. Live invocation NOT performed (would need a Sentry org). Two dimensions provisional.
**Total score: 65/70** — **highest in corpus**, narrowly edging out Filesystem (59) and Stripe (58). First entry to score 5/5 on observability (D10) and on safety rails (D13) simultaneously.

---

## What it is

Sentry's official MCP exposes error tracking, performance monitoring, replays, profiles, and visual snapshots to coding-assistant LLMs. Tool selection is explicitly scoped to **developer workflows** — not a general-purpose Sentry surface. The README states this up front: *"primarily designed for human-in-the-loop coding agents... rather than providing a general-purpose MCP server for all Sentry functionality."*

Three deployment modes:

1. **Hosted remote MCP** at `mcp.sentry.dev` via Cloudflare Workers + OAuth (the canonical mode)
2. **Stdio** via `npm i @sentry/mcp-server` (also supports OAuth device-code flow with token caching — see `docs/stdio-auth.md`)
3. **Claude Code Plugin** with shared skills auto-installed via `npx @sentry/dotagents install`

Tool count is **27 imports** in `index.ts`, but the AGENTS.md tool budget is documented as *"Target ≤20, hard limit 25"* — they enforce the limit by **filtering some tools from external MCP surfaces**. Five detail handlers (`get_issue_details`, `get_trace_details`, `get_replay_details`, `get_profile_details`, `get_snapshot_details`) stay reachable internally for code reuse but are hidden from the LLM, which uses the unified `get_sentry_resource` instead.

This is the first server in the corpus that explicitly **budgets its tool slot count** as an architectural constraint and rebuilds its surface to fit.

## Tool surface (27 imports, ~21 LLM-visible after mode-filtering)

Identity & discovery (6): `whoami`, `find_organizations`, `find_teams`, `find_projects`, `find_releases`, `find_dsns`

Reads (5 detail handlers, internally-only): `get_issue_details`, `get_trace_details`, `get_replay_details`, `get_event_attachment`, `get_profile_details` — all dispatched through:

Resource resolver (1, externally-visible): `get_sentry_resource` — accepts EITHER a Sentry URL OR a `(resourceType, resourceId)` pair, internally routes to the appropriate detail handler. Supports types `issue`, `event`, `trace`, `span`, `breadcrumbs`, `replay`, plus partial coverage of `monitor`, `release`, `profile`, `snapshot`.

Search (4): `search_events`, `search_issues`, `search_issue_events`, `search_docs` — and `get_doc` for documentation fetch (the **two-step pattern** P-13 done canonically).

Writes (5, narrow): `update_issue`, `update_project`, `create_team`, `create_project`, `create_dsn`. **No delete tools.** Deletion is not exposed to the LLM at all.

Issue tags (1): `get_issue_tag_values`

Visual snapshots (2): `get_snapshot_details`, `get_latest_base_snapshot`

AI-powered analysis (2): `analyze_issue_with_seer` (Sentry's RCA agent), and the agent-mode meta-tool:

Agent mode (1, gated): `use_sentry` — natural-language interface that runs an embedded LLM agent against the rest of the server. `agentOnly: true`, `destructiveHint: true`, recursion-prevented (excludes itself from the inner tool list).

## Why this entry matters

Sentry is the **observability archetype** the working-list flagged as the largest gap. With this entry the corpus now has at least one scored representative of every archetype (01–08).

Beyond filling the gap, three architecturally novel patterns emerge that no other corpus entry has:

1. **Tool-count budgeting via mode-filtered registry** — they explicitly target ≤20 tools and rebuild detail tools as composable internal helpers behind `get_sentry_resource`.
2. **Resource-resolver tool** — one polymorphic `get_resource(url | type+id)` substituting for N specific `get_X_details(id)` tools.
3. **Embedded agent tool** — `use_sentry` composes the entire MCP server as a sub-agent, with explicit recursion prevention, scope gating, and `destructiveHint: true` so callers know what they're inviting.

These promote to **P-26**, **P-27**, **P-28** below.

---

## Scoring

| Dimension | Score | Confidence | Evidence |
|---|---|---|---|
| 1. Tool granularity | 5/5 | High | 27 imports → ~21 LLM-visible. Three deliberate granularity layers: low-level CRUD (`create_team`, `find_releases`), mid-level resolver (`get_sentry_resource`), high-level agent (`use_sentry`). Each layer maps to a different LLM intent. No god-tools at any layer. |
| 2. Tool naming | 5/5 | High | `verb_object` throughout: `find_*` (5 tools, all sibling-disambiguated), `get_*`, `search_*`, `create_*`, `update_*`, `analyze_*`. Zero generic verbs. The `find_*` family alone proves discipline: organizations / teams / projects / releases / dsns each have their own tool, never a single `find(type, ...)`. |
| 3. Description quality | 5/5 | High | `use_sentry` description is the strongest in the corpus: structured into intent (`Use this tool when you need to...`), capability gating (`Capabilities depend on granted skills:`), `<examples>` block, `<hints>` block. `search_events` has dataset-disambiguation prose: *"If the user says logs, log messages, error logs, or warning logs, choose logs instead of errors."* — explicitly teaches the outer LLM how to route. |
| 4. Parameter design | 5/5 | High | Zod with `.describe()` on every field. Defaults thoughtful: `trace: z.boolean().nullable().default(null)`. Reused param schemas (`ParamOrganizationSlug`, `ParamRegionUrl`, `ParamProjectSlug`) — DRY across tools. |
| 5. Schema discipline | 5/5 | High | AGENTS.md mandates *"Prefer strict types over `any` - they catch bugs and improve tooling. Use `unknown` for truly unknown types."* Quality gate: `pnpm run tsc && pnpm run lint && pnpm run test`. Strict types enforced via lint. |
| 6. Error surface | 5/5 | High | Three named error classes used: `UserInputError` (clear bad-input messages with remediation: *"Either `url` or `resourceType` must be provided"*), `ApiNotFoundError` (404 from upstream Sentry), `enhanceNotFoundError` (wraps 404 with next-step guidance for the LLM). Reference-quality. |
| 7. ToolAnnotations | 5/5 | Provisional | Verified for `use_sentry`: `{ readOnlyHint: false, destructiveHint: true, openWorldHint: true }` — correctly annotated as the most powerful tool. Need to spot-check write tools (`update_issue` etc.) for `destructiveHint`. Projecting 5 conditional on that holding. |
| 8. Resources vs tools | 4/5 | High | `get_sentry_resource` is a TOOL that mimics MCP-resources semantics (URI-keyed dispatch with internal type discrimination). Defensible choice — search-by-URL is naturally a tool, not a resource. But not a true MCP `Resource` exposure. -1 vs. Postgres/SQLite which expose actual `Resource`s. |
| 9. Auth model | 5/5 | High | Two-token architecture documented: *"the upstream token and downstream token do not mean the same thing"* — downstream MCP token is validated separately from the upstream Sentry OAuth token. **Per-tool scope gating** via `requiredScopes: ["event:read"]` AND **per-tool skill gating** via `skills: ["inspect", "triage", "seer"]` (skills act as named permission bundles). Granular, auditable, layered. Ties Stripe (5/5 on this dim). |
| 10. Observability | 5/5 | High | First 5/5 in corpus on observability. They ARE Sentry — and they instrument their own server: `import { getActiveSpan, setTag } from "@sentry/core"` in tool handlers. Plus dedicated `docs/oauth-signout-playbook.md` for diagnostic runbooks. Reference exemplar. |
| 11. Composability | 5/5 | High | `get_sentry_resource` IS composition — it explicitly imports and dispatches to 5 detail handlers. `use_sentry` IS composition at the next level — embedded agent over the whole server with `Object.fromEntries(...filter(...))` to remove `use_sentry` itself and obey mode visibility. Composition is named, structured, and tested (`use-sentry/agent.ts` is exported separately for testing). |
| 12. Pagination/volume | 4/5 | Provisional | `search_events` has dataset selection + sort + limit semantics inherited from Sentry's API; pagination cursor structure not directly verified in handler.ts excerpt. Sentry's underlying API uses Link-header cursor pagination, so the bones are there. Projecting 4 pending verification. |
| 13. Safety rails | 5/5 | High | `agentOnly: true` per-tool flag (server policy mode P-12), `experimentalMode` discrimination (`isToolVisibleInMode(tool, context.experimentalMode)`), recursion prevention (`toolsToExclude.add("use_sentry")` before agent build), `destructiveHint: true` on `use_sentry`, **no delete tools** (deletion not exposed to LLM at all), per-tool scope+skill gating. Defense in depth. |
| 14. Docs & onboarding | 5/5 | High | `search_docs` + `get_doc` tools mean **the LLM can self-onboard** — agent finds its own docs at runtime. AGENTS.md is comprehensive and links to `docs/adding-tools.md`, `docs/common-patterns.md`, `docs/error-handling.md`, `docs/security.md`, `docs/oauth-signout-playbook.md`, `docs/embedded-agents.md`. Documentation is a deliverable. |
| **Total** | **65/70** | — | Highest in corpus. Two dimensions (D7, D12) provisional; worst case 63 if both drop a point, best case 67 if D8 reads as 5/5 to a strict-tool-vs-resource reviewer. |

---

## Patterns extracted (new from this entry)

### P-26: Resource Resolver Tool

**Rule:** When you have N tools of shape `get_X_details(id)` for related entity types (issues, traces, replays, profiles, snapshots), consolidate them into ONE polymorphic `get_resource(url | type+id)` tool. Keep the specific handlers as **internal-only** helpers for code reuse, but **filter them out of the externally-advertised tool list**.

**Why it works:**
- LLM tool slots are scarce (`AGENTS.md`: target ≤20, hard limit 25). One resolver replaces five details.
- The natural input the LLM has is usually a URL (a developer says *"look at sentry.io/.../issues/ABC-123"*); URL parsing inside the resolver is trivial and avoids forcing the LLM to extract IDs and pick the right `get_X_details` variant.
- Specific handlers stay testable in isolation — they're imported from the resolver but not exposed.

**Observed in:** Sentry MCP — `tools/get-sentry-resource.ts:resolveResourceParams()` accepts `url`, `resourceType`, `resourceId`, `organizationSlug`, `projectSlug`; switches on type to call `getIssueDetails`, `getTraceDetails`, `getProfileDetails`, `getReplayDetails`, `getSnapshotDetails`. Index comment: *"Legacy detail handlers stay available for internal composition behind `get_sentry_resource`, but are filtered from all external MCP surfaces."*

**Apply when:** A domain has ≥3 entity types reachable via stable URLs and the LLM-side workflow nearly always starts from a copy-pasted URL. Don't apply when each entity type has materially different parameters or different auth scopes — the resolver becomes a god-tool in disguise (AP-03).

**Counter-consideration:** Resolver responses are heterogeneous by type. Document the per-type response shapes in the tool description, OR use `structuredContent` (P-09) with a discriminated union schema so callers can branch.

---

### P-27: Embedded Agent Tool ("agent mode")

**Rule:** For complex multi-step workflows on top of a tool surface, expose a single high-level tool that hosts an embedded LLM agent which calls the rest of the server's tools as an inner loop. Gate it behind a server policy mode (`agentOnly: true`); annotate it with `destructiveHint: true`; prevent recursion by removing it from the inner tool list.

**Why it works:**
- An outer LLM with limited tool slots can express a complex intent as `use_sentry(request: "find unresolved errors from yesterday and create a triage team")` instead of orchestrating 5+ tool calls.
- The embedded agent has full visibility into the server's tools and can chain them — saving tokens on the outer model and letting the inner model specialize on Sentry-specific reasoning (search syntax, dataset choice).
- Server policy gating (`agentOnly: true` → tool hidden in non-agent mode) lets organizations turn it off if they want explicit step-by-step control.

**Observed in:** Sentry MCP — `tools/use-sentry/handler.ts`. Description states: *"Pass the user's request verbatim - do not interpret or rephrase. The agent can chain multiple tool calls automatically. Use trace=true parameter to see which tools were called."* Recursion prevention: `const toolsToExclude = new Set<string>(["use_sentry"])`. Mode visibility: `isToolVisibleInMode(tool, context.experimentalMode ?? false)`.

**Apply when:** Tool surface is large (≥10 tools), workflows often span 3+ tools, AND the inner agent has clear scope boundaries (Sentry-only, in this case). Always pair with: explicit recursion prevention, `destructiveHint: true`, and a `trace` parameter for caller debugging.

**Counter-consideration:** This is a god-tool from one angle (AP-03). The line between *legitimate composition tool* and *god-tool* is whether the description **reads as a single coherent intent** ("ask Sentry") rather than a generic dispatcher ("do anything"). Sentry's `use_sentry` reads as the former; a tool called `do_thing(action_type, params)` would read as the latter and IS a god-tool.

---

### P-28: Embedded LLM Query Translator (candidate)

**Rule:** When a tool's input is a complex domain DSL (Sentry search syntax, SQL, JQL), and users will sometimes pass natural language instead, use an **internal** LLM agent to translate NL → DSL before calling the upstream API. This LLM is invisible to the outer LLM-of-record — it's a server-side capability.

**Observed in:** Sentry MCP — `search_events/handler.ts` defines `buildSearchRepairPrompt()` and imports `searchEventsAgent` for cases where the query parses as natural language. The outer LLM passes either real Sentry syntax or English; the inner agent decides which and rewrites if needed. Description states: *"`query` can be natural language or Sentry search syntax. With an agent configured, it fixes dataset, query, fields, and sort before running."*

**Status:** Candidate (1 evidence point). Promote when a second corpus entry shows the same pattern (likely candidates: any MCP fronting a complex query DSL — Snowflake, BigQuery, Elastic).

**Counter-consideration:** Hidden LLM calls inside a tool inflate latency and cost in ways the caller cannot predict. Document the agent presence in the tool description (Sentry does: *"With an agent configured..."*) and consider a `repair: false` opt-out parameter for callers who want determinism.

---

## Patterns confirmed (existing patterns this entry strengthens)

- **P-08 (ToolAnnotations table)** — `use_sentry` annotations `{ readOnlyHint: false, destructiveHint: true, openWorldHint: true }` are textbook. Promote any provisional rating on this pattern.
- **P-12 (Server policy modes)** — `agentOnly: true` and `experimentalMode` are server-policy flags that hide/show tools. Adds a second confirmed example after Brodels' panel modes.
- **P-13 (Two-step search/fetch)** — `search_events` + `get_sentry_resource`, `search_issues` + `get_sentry_resource`, `search_docs` + `get_doc` — three pairs in one server. Strongest single confirmation in the corpus.
- **P-21 (Hosted remote MCP with OAuth)** — Sentry runs on Cloudflare Workers with OAuth, mirroring Stripe. Pattern now has **2 cross-domain examples** (payments + observability), promoting from "interesting" to "confirmed pattern".
- **P-22 (Permission scoping via existing infrastructure)** — Sentry uses its existing per-token-scope system (`event:read`, etc.) instead of inventing new MCP-specific scopes. Second example after Stripe's RAK system.

## Antipatterns: none observed

I looked for AP-03 (god-tool with method enum) and AP-08 (conditional required params) specifically. The closest call is `use_sentry` (one input parameter `request: string`, behaves like a god-tool from one angle), but it's correctly framed as a single-purpose composition tool with `destructiveHint: true` and recursion prevention. The closest input-shape candidate is `get_sentry_resource` accepting `url XOR (resourceType, resourceId)`, but the tool throws `UserInputError` with clear remediation guidance when both are absent — that's correct UX, not AP-08.

---

## Notes for the skill

When a buyer says *"I want Claude to access our observability platform"* or *"I want to expose error tracking / APM / logs to coding agents"*:

1. **Recommend the resource-resolver pattern (P-26)** for any domain with multiple entity types reachable by URL — saves tool slots and gives the LLM a natural URL-paste workflow.
2. **Consider an embedded agent tool (P-27)** ONLY if the buyer's tool surface will exceed 10 tools AND workflows commonly span 3+ tools. Pair with mandatory `destructiveHint: true`, recursion prevention, and a `trace` debug parameter.
3. **Use Sentry's `<examples>` + `<hints>` description format** as the template for tool descriptions — the structured prompt-format outperforms paragraph descriptions for LLM tool-selection accuracy.
4. **Strict tool-count budget** — write the budget into AGENTS.md (or CLAUDE.md) as a hard limit, and enforce it via mode-filtering rather than refusing to add features.
5. **Two-token auth architecture (P-21+P-22 combined)** — a downstream MCP token is NOT the same as the upstream platform's OAuth token. Validate them separately. Each tool declares `requiredScopes`. Skill bundles (`skills: ["inspect"]`) group scopes into named permission tiers callers can grant.
6. **Self-documenting fleet (P-05)** — expose `search_docs` + `get_doc` as actual tools so the LLM can onboard itself when it hits an unfamiliar tool.
7. **No delete tools** — Sentry's write surface is `update_*` and `create_*` only. If your domain has destructive ops the LLM should never call from agent flows, simply don't expose them. Out-of-band deletion via the dashboard is fine.

---

## Cross-server position (11 entries with Sentry)

| Server | Total | Note |
|---|---|---|
| **Sentry** | **65** | **Highest in corpus.** First 5/5 on observability + safety rails simultaneously. P-26, P-27 originate here. |
| Filesystem | 59 | Reference standard for read-only data sources. |
| Stripe | 58 prov. | First 5/5 on auth (D9). P-21+P-22+P-23+P-24 origin. |
| Brodels | 56 | Description + observability leader (workflow-orchestrator archetype). |
| Playwright | 55 | Naming + annotations + docs all 5/5 (browser-automation archetype). |
| Memory | 54 | Batch-by-default, system prompt (state/memory archetype). |
| Fetch | 54 | One-tool reference; first 5/5 on pagination. |
| Linear | 50 | Modern SaaS API; community-grade P-13 example. |
| GitHub | 49 | God-tool dispatch hurts (code/dev-tools archetype). |
| Postgres | 41 | Pre-modern, resources done right. |
| SQLite | 35 | Safety theater — counter-example. |

**The new top of the corpus** is now Sentry → Filesystem → Stripe (all 58+). Below them is the 49–56 production-ready cluster. Below 49 is the educational counter-example tier.

**Score distribution check:** 65, 59, 58, 56, 55, 54, 54, 50, 49, 41, 35 — 30-point range, healthy discrimination, no plateau. The rubric is working.
