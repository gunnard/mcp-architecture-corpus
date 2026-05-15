# Cloudflare MCP

**Source:** https://github.com/cloudflare/mcp (canonical), live at `mcp.cloudflare.com/mcp`
**Vendor:** Cloudflare (vendor-maintained).
**Archetype fit:** Primary **02 — Writable System** (manages Workers, KV, R2, D1, DNS, Firewall, etc.). Secondary **08 — Code/Dev-Tools** (Workers deployment surface).
**Status:** Source verified from `README.md`, `Token Comparison` table, `Build a Code Mode MCP Server` section, and the linked Code Mode SDK docs. Live invocation NOT performed.
**Total score: 57/70** — sits in the 54–58 production-ready cluster. Below Sentry/Filesystem/Stripe but architecturally **the most distinctive entry in the corpus**.

---

## What it is

The official Cloudflare MCP — a remote MCP server fronting **2,594 Cloudflare API endpoints** for AI agents. The standout decision: instead of registering 2,594 individual tools (or even a "list of resources" pattern), the server exposes **only 2 tools** that accept arbitrary JavaScript and execute it in sandboxed Cloudflare Worker isolates.

This is published as the **"Code Mode" pattern** — Cloudflare ships it as a documented architectural choice, not an accident. The server is the canonical reference implementation.

### The token-economics decision

From the README's `Token Comparison` table:

| Approach | Tools | Token cost | % of 200K context |
|---|---|---|---|
| Raw OpenAPI spec in prompt | — | ~2,000,000 | 977% |
| Native MCP (full schemas) | 2,594 | 1,170,523 | 585% |
| Native MCP (minimal — required params only) | 2,594 | 244,047 | 122% |
| **Code mode** | **2** | **1,069** | **0.5%** |

The "do it the obvious way" answer (one tool per endpoint) **does not fit** in a 200K context window. Even the minimal-schema variant takes 122% of context — leaving no room for actual reasoning. Code Mode reduces tool-list overhead by 99.6%, at the cost of letting the agent execute arbitrary JS.

## Tool surface (2 tools)

| Tool | Description |
|---|---|
| `search` | Agent writes JavaScript to query `spec.paths` (the Cloudflare OpenAPI spec living server-side) and find relevant endpoints. |
| `execute` | Agent writes JavaScript to call `cloudflare.request(...)` against the discovered endpoints. |

Both tools take a single `code: string` parameter. The agent's workflow is:

1. `search({ code: "...iterate spec.paths, filter by tag..." })` → returns endpoint summaries
2. `execute({ code: "...await cloudflare.request({method, path, body})..." })` → returns API response

Code runs in **Cloudflare's Dynamic Worker Loader API** — a V8 isolate with no filesystem, no network beyond the bound `cloudflare` client, and a per-request lifecycle. The spec.json is read-only inside the isolate; the API binding is scoped to the OAuth/token permissions the user granted at connection time.

### Fallback mode

Setting `?codemode=false` on the connection URL switches to traditional one-tool-per-endpoint registration — exactly the design Code Mode replaces. The README explicitly recommends this **only** when composing with other code-mode-aware servers, and warns about the 244k-token cost.

## Why this entry matters

This is the corpus's **case-study entry on deliberate antipattern violation**. By a strict rubric reading, both `search` and `execute` are textbook **AP-03 god-tools** (single tool, arbitrary code input, infinite degrees of freedom). The architects know this and chose it anyway, because the alternative (2,594 tools) blows the context window on tool definitions alone.

The right takeaway for the skill is **not** "Cloudflare violates AP-03, scores low." It's:

- **AP-03 has a genuine exception:** when API surface size makes per-endpoint tools physically impossible.
- **The exception requires three mitigations:** sandboxed execution environment, scoped auth from existing infrastructure, AND a dual-mode fallback so the design choice is visible.
- **The new pattern is P-29 below.**

The corpus also gains a third confirmed cross-domain example of P-21 (hosted remote MCP via OAuth): payments (Stripe) + observability (Sentry) + edge infrastructure (Cloudflare). P-21 promotes from "interesting" to "established pattern."

---

## Scoring

| Dimension | Score | Confidence | Evidence |
|---|---|---|---|
| 1. Tool granularity | 3/5 | High | 2 tools is GREAT for token economics, but each tool has unbounded input space (arbitrary JS). Reads as AP-03 from a strict perspective. -2 vs. 5 because the design tradeoff is real, not free. The `?codemode=false` mode (~2,500 tools, all atomic verb_object) demonstrates Cloudflare CAN do granular tools and chose not to — that's one notch back to 3 instead of 2. |
| 2. Tool naming | 5/5 | High | `search` + `execute`. Verb. Single-purpose. Zero ambiguity. The naming carries the entire design intent. |
| 3. Description quality | 4/5 | Provisional | README documentation is reference-quality (Token Comparison table, dual-mode explanation, JS code examples for both tools). Per-tool MCP descriptions not directly verified — projecting 4 conditional on inheriting the README quality. |
| 4. Parameter design | 4/5 | High | Minimal: `code: string`, plus optional `account_id` on `execute` for token-type disambiguation. Both decisions are correct for the design. |
| 5. Schema discipline | 4/5 | Provisional | TypeScript codebase, Cloudflare's typical engineering rigor. Schema can't validate JS code semantics — that's a known limitation of Code Mode, not a discipline failure. Projecting 4. |
| 6. Error surface | 3/5 | Provisional | Arbitrary code → arbitrary failure modes. Some failures are JS runtime errors (TypeError, ReferenceError); some are API errors propagated from `cloudflare.request()`; some are isolate-level (timeout, memory, sandbox violations). Surfacing all three coherently is hard. Projecting 3 pending source verification — could be 4 if the implementation has good error wrapping. |
| 7. ToolAnnotations | 3/5 | Provisional | `execute` is unambiguously destructive (arbitrary write API access) — `destructiveHint: true` should be set. Need source verification to confirm. The dual-mode fallback complicates annotations: in code mode `execute` is destructive; in `?codemode=false` mode the per-endpoint tools have specific annotations. Projecting 3 pending verification. |
| 8. Resources vs tools | 3/5 | High | The Cloudflare OpenAPI spec is a textbook MCP `Resource` (~2M tokens of structured data the agent reads). Exposing it as the implicit input to a `search(code)` tool rather than a real `Resource` is defensible (search-by-code IS a tool action) but it sidesteps the resources protocol entirely. -2 vs Postgres/SQLite. |
| 9. Auth model | 5/5 | High | OAuth at `mcp.cloudflare.com` (P-21) **plus** API token fallback for CI/CD, **plus** permission scoping via Cloudflare's existing token permissions (P-22) — same pattern as Stripe's RAK. Honest constraint documentation: "API tokens with Client IP Address Filtering enabled are not currently supported." Ties Stripe and Sentry at 5/5. |
| 10. Observability | 4/5 | Provisional | Code execution in Worker isolates can emit telemetry via Cloudflare's standard Workers observability (logs, metrics, tail workers). Per-tool-call observability not directly verified. Projecting 4. |
| 11. Composability | 5/5 | High | The entire design IS composition — `search` discovers, `execute` invokes. Two tools chained handle 2,594 endpoints. The agent composes endpoints via JS (loops, conditionals, parallel calls). Highest composability story in the corpus. |
| 12. Pagination / volume | 5/5 | High | The whole design is a pagination strategy at the **meta-tool level**: instead of listing 2,594 tools, return endpoint summaries via search. Within `execute`, agents can iterate paginated API responses naturally with JS. Token-economical at every layer. First server in the corpus to score 5/5 here on a pure-volume basis. |
| 13. Safety rails | 3/5 | High | **The architecture's main tension.** Sandboxing via Worker isolates is genuine — V8 boundaries, no filesystem, scoped `cloudflare.request()` binding. BUT: the agent can issue any API call within token scope, including destructive ones, with no per-call human-in-the-loop. Compare to Sentry (no delete tools at all) or Stripe (RAK + explicit prompt-injection warning): Cloudflare's mitigation is "trust the sandbox + trust the token scope." That works, but it's a 3, not a 5. The dual-mode fallback (`?codemode=false`) is a partial mitigation — orgs that want per-call review can disable code mode, accept the 244k token cost, and get individually-annotated tools. |
| 14. Docs & onboarding | 5/5 | High | Token Comparison table is the single most-cited reference in the corpus for the tool-count vs. context-window tradeoff. Code Mode SDK is published as a reusable building block. Dual-mode JSON config examples. Honest constraint notes. Reference-quality. |
| **Total** | **57/70** | — | Sits in the 54–58 cluster. Provisional on 5 dimensions (D3, D5, D6, D7, D10); worst case 53 if all drop a point, best case 60 if D6/D7/D10 read as 4–5 to a generous reviewer. |

---

## New pattern extracted

### P-29: Code Mode Tool (sandboxed-execution meta-tool)

**Rule:** When an upstream API has a surface so large that registering it as N MCP tools would exceed the LLM context budget (rough threshold: ≥500 endpoints or ≥100k tokens of tool definitions), expose **2 meta-tools** instead:

1. A `search(code)` tool that runs sandboxed code against a server-side spec.json (or equivalent metadata).
2. An `execute(code)` tool that runs sandboxed code against a scoped API client binding.

The "sandbox" must be a real isolation boundary (V8 isolate, WebAssembly, gVisor, etc.) — NOT just `eval()` in the MCP server process.

**Why it works:**

- Tool-definition tokens drop from O(N×schema_size) to O(2×schema_size). Cloudflare's data: 244k tokens → 1,069 tokens (99.6% reduction).
- The agent's compositional power increases — JS for-loops over endpoint sets, parallel calls, server-side filtering before return.
- Auth scoping inherits from the upstream platform's existing token permissions (P-22) — the sandbox runs with the binding the user granted at connection time, no per-tool scope inflation.

**Required mitigations:**

- **Sandbox isolation must be real.** No `eval()` in the server process. Cloudflare uses Worker isolates; alternatives are V8 contexts, WebAssembly modules, or out-of-process execution.
- **Auth scope must be enforced at the binding, not the input.** The agent's code cannot escape the platform's existing permission model.
- **Dual-mode fallback.** Provide a `?codemode=false` (or equivalent) mode that registers per-endpoint tools for compositions that need them. This is also a security release valve: orgs that want per-call human review can opt out of code mode at the cost of context size.
- **Annotations:** the execute tool must be `destructiveHint: true`. There is no way to prove a code blob is read-only without analyzing it.

**Observed in:** Cloudflare MCP — `tools: [search, execute]`, `mcp.cloudflare.com/mcp`. Sandbox: Cloudflare Dynamic Worker Loader API (V8 isolates). Spec lives server-side; agent receives only execution results.

**Apply when:** Upstream API has ≥500 endpoints OR ≥100k tokens of full-fidelity tool schemas, AND the platform already has a robust token-scope permission model the sandbox can inherit.

**Don't apply when:** Upstream API is small enough for traditional tools (≤50 endpoints), OR the platform has no scoped-token model (then the sandbox can do anything any token can do — too much trust on the LLM).

**Counter-consideration / when this is dangerous:** Per-call human-in-the-loop review is essentially impossible on a code-mode tool. A reviewer would have to read JS to know what the agent is about to do. For high-stakes domains (finance, prod infra mutation, PII), prefer the `?codemode=false` path even with the token cost — the per-tool annotations + descriptions give human reviewers something legible.

**Relationship to other patterns:**

- P-27 (Embedded Agent Tool) is the LLM-as-inner-runtime version of this pattern. Sentry's `use_sentry` runs an LLM that calls atomic tools. Cloudflare's `execute` runs JS that calls APIs directly. Use P-27 when reasoning matters; use P-29 when token economics dominate.
- AP-03 (god-tool) is the antipattern this pattern deliberately violates. The mitigations above are what differentiate a legitimate P-29 from an AP-03.

---

## Patterns confirmed (existing patterns this entry strengthens)

- **P-21 (Hosted remote MCP with OAuth)** — promotes from "interesting" (1 example) to **confirmed cross-domain pattern** (3 examples: payments + observability + edge infrastructure).
- **P-22 (Permission scoping via existing infrastructure)** — Cloudflare's "use the existing API token permission system" is a near-perfect mirror of Stripe's RAK approach. Two cross-domain examples confirm this is the right answer for vendor MCPs.
- **P-25 (Dual-mode connection URL)** — `?codemode=false` query param flips behavior at connection time. Mirrors stdio-vs-remote in Stripe and stdio-vs-OAuth in Sentry. Worth tracking as a candidate connection-time configuration pattern.

## Antipatterns intentionally violated

- **AP-03 (god-tool with unbounded input space)** — `search(code)` and `execute(code)` ARE god-tools by strict definition. The README's Token Comparison table is the case for the violation; the Code Mode SDK's sandbox guarantees are the mitigation. Use this entry as the corpus's primary discussion of when AP-03 is the right call.

## Antipatterns avoided

- The dual-mode fallback (`?codemode=false`) is a deliberate hedge against any future security disclosure that breaks code-mode safety assumptions. If the sandbox is ever compromised, orgs can fail open to per-tool mode without server changes.

---

## Notes for the skill

When a buyer says *"my domain has hundreds of API endpoints"* (Cloudflare-like: AWS, GCP, Azure, Salesforce, Twilio, anything with ≥500 endpoints):

1. **Don't list every endpoint as a tool.** Calculate the token budget first — if minimal-schema-per-endpoint exceeds 100k tokens, traditional MCP design is wrong for this surface.
2. **Recommend P-29 (Code Mode Tool) IF the platform has scoped-token auth** — and only if you can guarantee a real sandbox boundary. Cloudflare Workers, AWS Lambda + STS-scoped roles, browser-based JS sandboxes (`vm2` is broken — don't use it).
3. **Always pair P-29 with `?codemode=false` fallback** for orgs that want per-call review.
4. **Annotate `execute`-style tools as destructiveHint: true.** There is no static analysis for "this code blob is read-only" — assume the worst.
5. **Document the trade-off explicitly in the README** — Cloudflare's Token Comparison table is the model. Show the math. Give buyers' security teams something they can audit.
6. **Watch for the inverse trap:** P-29 is wrong when API surface is small (<50 endpoints). Then it's just AP-03 with extra steps — sandbox overhead, harder review, no token-budget benefit.

---

## Cross-server position (12 entries with Cloudflare)

| Server | Total | Note |
|---|---|---|
| **Sentry** | 65 | Highest in corpus. P-26, P-27 origin (observability). |
| Filesystem | 59 | Reference standard for read-only data sources. |
| Stripe | 58 prov. | First 5/5 on auth (D9). P-21+P-22+P-23+P-24 origin. |
| **Cloudflare** | **57** | **P-29 origin (Code Mode).** Case study in deliberate AP-03 violation. First 5/5 on D12 (volume) on a pure-scaling basis. |
| Brodels | 56 | Description + observability leader. |
| Playwright | 55 | Naming + annotations + docs all 5/5. |
| Memory | 54 | Batch-by-default, system prompt. |
| Fetch | 54 | One-tool reference; first 5/5 on pagination (text-chunking). |
| Linear | 50 | Modern SaaS API; community-grade P-13 example. |
| GitHub | 49 | God-tool dispatch hurts (and now has a published, sanctioned alternative in P-29). |
| Postgres | 41 | Pre-modern, resources done right. |
| SQLite | 35 | Safety theater — counter-example. |

**Score distribution:** 65, 59, 58, 57, 56, 55, 54, 54, 50, 49, 41, 35 — 30-point range, dense top cluster (5 entries within 4 points), discrimination still healthy.
