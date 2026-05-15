# MCP Design Patterns (Catalogue)

Distilled from the corpus. Each pattern includes:
- The name and the rule
- The problem it solves
- Where it was observed
- When to apply it
- Counter-considerations

**Status:** this document grows as corpus entries accumulate. After ~10 entries, patterns
should start repeating across servers — that's the signal a pattern is "real" vs. a quirk.

---

## P-01: Cost-aware tool descriptions

**Rule:** Tools with non-trivial cost (money, compute, wall-clock) publish expected cost
in their description.

**Solves:** LLM makes budget-aware decisions before invocation; user-facing prompts can
surface cost expectations.

**Observed in:** Brodels (`brodels_pr_smoke_test`, `brodels_run`).

**Apply when:** any tool that incurs >$0.005 per call, takes >5s wall-clock, or has
quota implications.

**Counter-consideration:** stale cost estimates erode trust. Either auto-derive from
recent usage or version the estimate.

---

## P-02: Scaled defaults

**Rule:** When a tool's cost profile varies wildly by input, make defaults a function of
inputs rather than a single constant.

**Solves:** LLM doesn't have to override defaults for every non-trivial call.

**Observed in:** Brodels (`brodels_pr_smoke_test` `maxCost` scales by suite size).

**Apply when:** the same tool has both lightweight and heavyweight invocation modes.

---

## P-03: Multi-gate destructive operations

**Rule:** For destructive ops, layer independent safety conditions so a single
misconfigured flag can't cause harm.

**Solves:** LLM accidents on destructive tools. Reduces blast radius of prompt-injection
attacks targeting destructive ops.

**Observed in:** Brodels (`brodels_prune_lessons` — dry-run default + observation-floor +
bullet-age, all enforced even with `apply=true`).

**Apply when:** any tool that performs irreversible writes or deletes.

**Counter-consideration:** too many gates becomes friction for legitimate use. Cap at 3
independent gates; document the override path for each.

---

## P-04: Audit-required policy overrides

**Rule:** Tools that override safety policies require the *reason* as a structured input,
not optional metadata.

**Solves:** Forensic reconstruction of why a policy was bypassed.

**Observed in:** Brodels (`brodels_clear_policy_flag` requires `operator` + `reason`).

**Apply when:** any tool whose purpose is to override or bypass a guardrail.

---

## P-05: Self-documenting fleet

**Rule:** Servers with >10 tools include explicit discovery tools.

**Solves:** LLM can self-orient on first contact without a giant system-prompt dump.

**Observed in:** Brodels (`brodels_help`, `brodels_doctor`, `brodels_whoami`, `brodels_skills_list`).

**Apply when:** tool count >10, or when categories aren't obvious from naming alone.

---

## P-07: Prose-first outputs (tradeoff pattern)

**Rule:** Tool outputs may return formatted prose (markdown, ASCII tables, bullet lists)
instead of structured JSON when the primary consumer is an LLM.

**Solves:**
- Token efficiency — bulleted prose is ~30–50% denser than JSON-equivalent
- LLM comprehension — modern models read prose better than nested JSON for human-shaped data
- Human debuggability — log inspection doesn't require a JSON formatter

**Observed in:** Brodels (all 33 tools return prose: markdown headers, ASCII tables,
bullet lists, key-value blocks, section dividers).

**Apply when:**
- Server is LLM-first, not program-first
- Token budget matters more than programmatic consumability
- Output is consumed once per turn, not piped between tools

**Do NOT apply when:**
- Tool outputs feed directly into other tools (composability suffers — see Brodels D11 = 3/5)
- Output schema needs to be machine-validated
- Server has non-LLM clients

**Mitigation when applying:** preserve consistent identifier markers within the prose
(e.g., always render ticket IDs as `[PROJECT-1234]` so regex extraction is reliable).
Brodels does this with project keys in `jira_search` output (`PROJ-26:`) but inconsistently.

---

## P-06: Namespacing convention

**Rule:** For multi-domain servers, use `<server>_<domain>_<action>` naming.

**Solves:** Tool-picking ambiguity when multiple domains have similar verbs.

**Observed in:** Brodels (`brodels_jira_*`, `brodels_pr_*`, `brodels_db_*`, `brodels_workspace_*`).

**Counter-consideration:** single-domain servers shouldn't use this; it adds noise.

---

## P-08: MCP ToolAnnotations table

**Rule:** Publish a table in the README mapping every tool to its `readOnlyHint`,
`idempotentHint`, `destructiveHint` values. Set these annotations in the tool registration.

**Solves:** Side-effect metadata is available to the LLM via MCP protocol and to humans
via documentation. Tool-picking discipline improves on both sides.

**Observed in:** Filesystem (canonical example; full table in README).

**Apply when:** Any server, but especially those with >5 tools or any destructive ops.

**Reference quote (filesystem README):**
```
| Tool          | readOnlyHint | idempotentHint | destructiveHint |
| write_file    | false        | true           | true            |
| edit_file     | false        | false          | true            |
```

---

## P-09: Dual return (text + structuredContent)

**Rule:** Return tools with both `content: [{type: "text", text: ...}]` AND
`structuredContent: { ... }`. Text channel for LLM consumption; structured channel for
programmatic clients and downstream tool composition.

**Solves:** The prose-vs-structured tension (P-07) by doing both.

**Observed in:** Filesystem (modern MCP SDK pattern).

**Apply when:** Any MCP server built on the current MCP SDK. Should be default.

**Counter-consideration:** Doubles token cost on the wire (mitigation: keep structured
content terse; let the text channel carry the long-form).

---

## P-10: Toolset gating

**Rule:** For large-surface servers, group tools into named toolsets and let users enable
only what they need via flag/env var. Default to a "context" toolset that gives basic
orientation.

**Solves:**
- Reduces context cost for the LLM (fewer tool definitions in the prompt)
- Reduces tool-picking error rate (smaller decision space)
- Allows policy-based deployment (a read-only deployment can disable specific toolsets)

**Observed in:** GitHub MCP (`--toolsets repos,issues,pull_requests`).

**Apply when:** Server has >15 tools, or when distinct user personas need disjoint subsets.

---

## P-11: Dynamic tool discovery (beta pattern)

**Rule:** Instead of exposing all tools at startup, offer discovery tools
(`list_available_toolsets`, `enable_toolset`, `get_toolset_tools`) that let the LLM
negotiate its own surface based on user request.

**Solves:** The "100+ tools confuses the model" problem.

**Observed in:** GitHub MCP `--dynamic-toolsets` (beta).

**Counter-consideration:** Adds round-trips and complexity. Only worth it past ~30 tools.

---

## P-12: Server-level policy modes

**Rule:** Offer global modes (`--read-only`, `--lockdown`) that filter the available tool
surface based on deployment policy. Modes compose with toolset gating.

**Solves:** Lets ops/security teams deploy "safe by default" versions of the same binary
without re-implementing tool definitions.

**Observed in:** GitHub MCP (`--read-only`, `--lockdown-mode`).

**Apply when:** Server has destructive tools and will be deployed in policy-sensitive
environments.

---

## P-13: Resource-per-entity for slowly-changing reference data

**Rule:** For metadata the LLM might consult repeatedly (table schemas, API endpoints,
config, user preferences), expose each entity as a separate MCP resource at a
deterministic URI pattern. Auto-discover via `list_resources`.

**Solves:**
- LLM reads once, caches in context
- No tool-call overhead per metadata lookup
- Each resource has a stable URI for cross-reference
- Discovery is built into the protocol

**Observed in:** Postgres MCP (`postgres://<host>/<table>/schema` — one resource per
table, auto-generated from `information_schema.tables`).

**Apply when:**
- N entities of the same shape (tables, projects, users, etc.)
- Each entity's "schema" or "metadata" is queryable but rarely changes
- The LLM may need to reference any of N

**Reference implementation:** see Postgres index.ts — `ListResourcesRequestSchema` handler
that queries the schema catalog and emits one resource per row.

**This solves AP-01** (reference data exposed as tools) at the architectural level.
Brodels's `skills_list`, `db_sites`, `workspace_list` should all be this pattern.

---

## P-15: Batch-by-default inputs

**Rule:** Tools that operate on entities should accept an array as the default shape,
not a singleton. Single-entity calls become array-of-one.

**Solves:**
- Reduces tool-call count for multi-entity operations (LLM doesn't loop)
- Reduces context cost (one tool definition handles N items)
- Eliminates the "should I call this 5 times in parallel or one batched call?" decision

**Observed in:** Memory MCP (every tool: `entities[]`, `relations[]`, `observations[]`,
`entityNames[]`, `deletions[]`).

**Apply when:**
- Tool operates on entities and the LLM commonly works with multiple
- Operation is naturally parallelizable
- Per-item cost is low (large batches don't hit timeout)

**Counter-consideration:** For destructive operations, batching multiplies blast radius.
Pair batch-by-default with confirmation patterns or dry-run on destructive variants.

---

## P-16: Example System Prompt in server docs

**Rule:** For servers that benefit from explicit prompt scaffolding (memory, knowledge,
analysis), publish an example System Prompt in the README showing how to integrate the
server's tools into the user's Claude setup.

**Solves:**
- Buyer understands *how to actually use* the server, not just what it does
- Reduces "I installed it but Claude doesn't know to use it" failures
- Establishes a usage convention the community can iterate on

**Observed in:** Memory MCP README (3-step "Memory Retrieval / Memory / Memory Update"
prompt scaffold).

**Apply when:** Server's value depends on the LLM proactively choosing to use it.
Memory and state-management servers benefit most.

---

## P-17: Built-in chunked pagination via `start_index` + `max_length`

**Rule:** For tools that return large textual content (web pages, file contents, log
dumps, documents), expose chunked-read params directly: `start_index` (offset) +
`max_length` (limit). The LLM does its own multi-call paging without needing a separate
"continue" tool.

**Solves:**
- Content too large for one tool response
- Avoids the "cursor pagination requires a stateful tool pair" complexity
- LLM can selectively read the middle or end of large content
- Stateless — each call is independent

**Observed in:** Fetch MCP (`max_length: 5000` default, `start_index: 0` default).

**Apply when:** Tool returns variable-length content where the LLM might want a slice
rather than the whole thing.

**Counter-consideration:** Works for character-indexed text. For structured paginated
lists, cursor-based pagination is still the right choice.

---

## P-18: Model vs user-initiated request attribution

**Rule:** When a tool makes external HTTP requests, differentiate model-initiated traffic
from user-initiated traffic (via user-agent header, request log tag, or similar).
External operators can then reason about LLM-driven access patterns.

**Solves:**
- External services can rate-limit or block model traffic separately
- robots.txt compliance can be applied selectively
- Operators gain visibility into LLM access without instrumenting the LLM

**Observed in:** Fetch MCP. Sends `ModelContextProtocol/1.0 (Autonomous; ...)` for
tool-initiated requests; sends `... (User-Specified; ...)` for prompt-initiated
requests. Same server, two paths, distinguishable traffic.

**Apply when:** Tool issues external requests on behalf of an LLM. Especially relevant
for tools that scrape web content, hit third-party APIs, or send messages.

---

## P-19: Output redirection to file via `filename` param

**Rule:** For tools that return potentially large output (screenshots, accessibility
trees, PDFs, console dumps, video), accept an optional `filename` parameter. When
provided, the tool saves to disk and returns the path; when omitted, returns inline.

**Solves:**
- Avoids blowing the LLM's context window on large binary/structured output
- Lets the LLM keep large artifacts addressable (by filename) without paying token cost
- Composes with file-reading tools downstream

**Observed in:** Playwright MCP (`browser_take_screenshot`, `browser_snapshot`,
`browser_console_messages`, `browser_evaluate`, `browser_pdf_save`, video tools).

**Apply when:** Tool can return >1KB of structured or binary output that the LLM might
not need to inspect in full.

**Counter-consideration:** Requires the MCP client to expose filesystem access to the
agent so it can read the saved file. Works best paired with the filesystem MCP.

---

## P-20: Permission via human-readable element description

**Rule:** For tools that act on user-visible elements (DOM, GUI, files in a project),
require a `description` parameter alongside the machine-readable target. The
description is what the LLM types in natural language; the target is what the system
actually acts on.

**Solves:**
- Tool calls become auditable in plain language (session log shows intent, not selectors)
- Acts as a per-call consent prompt — UIs can intercept and ask the user to confirm
- Makes the LLM's intent inspectable

**Observed in:** Playwright MCP — almost every interaction tool has `element` (human
description) + `target` (selector). Description quote: *"Human-readable element
description used to obtain permission to interact with the element."*

**Apply when:** Tool acts on something a human user could also act on (UI elements,
files in a workspace, records in a CRM). Especially valuable for high-trust environments.

---

## P-21: Hosted remote MCP with OAuth

**Rule:** For SaaS APIs with established OAuth flows, offer a hosted remote MCP server.
Clients connect via URL (`{"url": "https://mcp.example.com"}`) instead of spawning a
local process.

**Solves:**
- Zero install — one JSON config line and the server is ready
- Centralized maintenance — provider updates the server, no client re-deploy
- Identity-aware — each user authenticates once via OAuth
- Multi-device — same auth works from any MCP client

**Observed in:** Stripe MCP (`mcp.stripe.com`).

**Apply when:** Provider has existing OAuth infrastructure. SaaS products where the
buyer already has an account.

**Counter-consideration:** Requires MCP host to support remote/URL servers. Latency is
higher than local stdio. Provider bears runtime cost.

---

## P-22: Permission scoping via existing API key infrastructure

**Rule:** When the provider already has a permission-scoped API key system
(Stripe RAK, AWS IAM keys, GitHub fine-grained PATs), use IT for tool-level permissions
rather than building MCP-specific permission scopes.

**Solves:**
- Reuses battle-tested permission infrastructure
- Users already know how to manage scopes in the provider's dashboard
- Audit trails flow through existing systems
- No new permission model to teach

**Observed in:** Stripe MCP — *"Tool permissions are controlled by your Restricted API
Key (RAK). Create a RAK with the desired permissions."*

**Apply when:** Provider has mature scoped-key infrastructure. Don't invent a new
permission model when an existing one works.

**Counter-consideration:** Couples the MCP server to the provider's key model. If the
provider's keys are coarse-grained, the MCP is also coarse-grained.

---

## P-23: Documentation-search as a built-in tool

**Rule:** For servers exposing a complex API, include a tool that searches the
provider's documentation and knowledge base. The LLM can answer "how do I X" questions
before invoking domain tools.

**Solves:**
- LLM doesn't have to guess at API semantics
- Reduces hallucinations on parameter shapes / edge cases
- Self-service onboarding — the server teaches the LLM how to use itself
- Cuts trial-and-error loops

**Observed in:** Stripe MCP — `search_stripe_documentation`, `search_stripe_resources`,
`fetch_stripe_resources` form a 3-tool docs/KB surface.

**Apply when:** Server's domain has substantial documentation (>50 pages of API
reference, guides, tutorials).

**Counter-consideration:** Tempting to make this the primary tool — resist. The LLM
should still invoke real domain tools for real work.

---

## P-24: Explicit prompt-injection warning in docs

**Rule:** For servers that act on real-world systems (payments, code, infra), publish
an explicit prompt-injection warning in the README/docs. Acknowledge the threat model.

**Solves:**
- Honest threat communication
- Sets expectation that human-in-the-loop is appropriate
- Signals mature security thinking to buyers
- Reduces legal/reputational risk when prompt-injection is attempted

**Observed in:** Stripe MCP — *"We recommend enabling human confirmation of tools and
exercising caution when using the Stripe MCP with other servers to avoid prompt
injection attacks."*

**Apply when:** Tool calls have real-world consequences (money, code merges, infra
changes, communications sent on behalf of user).

**Pairs well with:** P-12 server-level policy modes, P-20 per-call consent prompts.

---

## P-25: Stdio-to-HTTPS proxy as a local MCP CLI

**Rule:** When the canonical server is a hosted remote MCP, ship a thin local CLI that
forwards stdio messages over HTTPS. The local CLI handles authentication and client
attribution; all tool logic stays server-side.

**Solves:**
- Clients that don't yet support remote/URL MCP transports get access via stdio
- Server provider maintains single source of truth for tool definitions
- Tool descriptions, schemas, error shapes update without buyers re-installing
- Authentication can be enforced at the local CLI before forwarding
- User-Agent can be enriched with MCP client identity (pairs with P-18)

**Observed in:** Stripe (`@stripe/mcp` is a ~80-line proxy forwarding to
`mcp.stripe.com`). Local CLI does:
1. Parse args, validate API key
2. Extract client name from MCP initialize request
3. Build User-Agent including client identity (P-18)
4. Open stdio transport ↔ HTTPS transport with `requestInit: { headers }`
5. Bidirectionally forward messages

**Apply when:** Provider operates a hosted MCP server (P-21) and wants stdio-protocol
support for clients that don't speak URL transports yet.

**Reference code shape:**
```typescript
const stdioTransport = new StdioServerTransport();
const httpTransport = new StreamableHTTPClientTransport(
  new URL(MCP_SERVER_URL),
  { requestInit: { headers: buildHeaders(options, userAgent) } }
);
stdioTransport.onmessage = msg => httpTransport.send(msg);
httpTransport.onmessage = msg => stdioTransport.send(msg);
```

**Counter-consideration:** Adds a network hop. For latency-sensitive tools, consider
returning a Server-Sent Events transport directly to the client without the local proxy.

---

## P-14: Dynamic resource (living document)

**Rule:** Resources can be *synthesized* per-read from server state. Tools that mutate the
underlying state cause subsequent `read_resource` calls to return updated content.

**Solves:** Long-running analyses where insights/state accumulate across many tool calls.
Avoids stuffing intermediate results into the tool response payload.

**Observed in:** SQLite MCP (`memo://insights` — synthesized fresh on every read from
`db.insights[]`; `append_insight` tool extends the list).

**Apply when:** LLM is doing iterative analysis and accumulating findings worth preserving
across the session.

**Counter-consideration:** The resource's "current state" is server-side; the LLM has to
re-`read_resource` to see updates. Doesn't compose cleanly with stateless tool chains.

---

## P-26: Resource Resolver Tool

**Rule:** When a domain has N entity types reachable via stable URLs (issues, traces,
replays, profiles, …), expose ONE polymorphic `get_resource(url | type+id)` tool instead
of N specific `get_X_details(id)` tools. Keep the per-type handlers as **internal helpers**
for code reuse, but **filter them out of the externally-advertised tool list** so they
do not consume LLM tool slots.

**Why it works:**

- LLM tool slots are scarce (Sentry's AGENTS.md states `Target ≤20, hard limit 25`).
  One resolver replaces five details.
- The natural input the LLM has is usually a URL (developer pastes
  `sentry.io/.../issues/ABC-123`); URL parsing inside the resolver avoids forcing the
  LLM to extract IDs and pick the right `get_X_details` variant.
- Per-type handlers stay testable and reusable internally — they are just no longer in
  the public tool index.

**Observed in:** Sentry MCP — `tools/get-sentry-resource.ts` accepts `url` OR
`(resourceType, resourceId, organizationSlug)`, switches on type, dispatches to
`getIssueDetails`, `getTraceDetails`, `getProfileDetails`, `getReplayDetails`,
`getSnapshotDetails`. Index comment: *"Legacy detail handlers stay available for
internal composition behind `get_sentry_resource`, but are filtered from all external
MCP surfaces."*

**Apply when:** Domain has ≥3 entity types reachable via stable URLs, AND the LLM-side
workflow nearly always starts from a copy-pasted URL.

**Don't apply when:** Per-type entities have materially different parameter sets, auth
scopes, or rate limits — the resolver becomes a god-tool in disguise (AP-03).

**Counter-consideration:** Resolver responses are heterogeneous by type. Document
per-type response shapes in the tool description, OR use `structuredContent` (P-09) with
a discriminated union schema so callers can branch.

---

## P-27: Embedded Agent Tool ("agent mode")

**Rule:** For complex multi-step workflows on a large tool surface, expose a single
high-level tool that hosts an embedded LLM agent which calls the rest of the server's
tools as an inner loop. Gate it behind a server policy mode (`agentOnly: true`); annotate
it with `destructiveHint: true`; prevent recursion by removing it from the inner tool list.

**Why it works:**

- An outer LLM with limited tool slots can express a complex intent as a single call
  (e.g. `use_sentry(request: "find unresolved errors from yesterday and triage them")`)
  instead of orchestrating 5+ tool calls and burning tokens on intermediate reasoning.
- The embedded agent has full visibility into the server's tools and specializes on the
  domain's reasoning (search syntax, dataset choice, dispatch routing).
- Server policy gating lets organizations turn it off if they want explicit step-by-step
  control with audit trails per call.

**Observed in:** Sentry MCP — `tools/use-sentry/handler.ts`. Recursion prevention:
`const toolsToExclude = new Set<string>(["use_sentry"])`. Mode visibility:
`isToolVisibleInMode(tool, context.experimentalMode ?? false)`. Description states:
*"Pass the user's request verbatim - do not interpret or rephrase. The agent can chain
multiple tool calls automatically. Use trace=true parameter to see which tools were called."*

**Apply when:** Tool surface is large (≥10 tools), workflows commonly span 3+ tools, AND
the inner agent has clear scope boundaries (one platform, one domain).

**Always pair with:** explicit recursion prevention, `destructiveHint: true`, a `trace`
parameter for caller debugging, AND scope/skill gating per inner tool.

**Counter-consideration:** This LOOKS like a god-tool from one angle (AP-03). The line
between *legitimate composition tool* and *god-tool* is whether the description **reads
as a single coherent intent** ("ask Sentry") rather than a generic dispatcher ("do
anything"). A tool called `do_thing(action_type, params)` IS a god-tool; a tool called
`use_sentry(request: string)` with explicit Sentry-only scope is not.

---

## P-29: Code Mode Tool (sandboxed-execution meta-tool)

**Rule:** When an upstream API has a surface so large that registering it as N MCP tools
would exceed the LLM context budget (rough threshold: ≥500 endpoints OR ≥100k tokens of
tool definitions), expose **2 meta-tools** instead:

1. A `search(code)` tool that runs sandboxed code against a server-side spec.json (or
   equivalent metadata).
2. An `execute(code)` tool that runs sandboxed code against a scoped API client binding.

The "sandbox" must be a real isolation boundary (V8 isolate, WebAssembly, gVisor, etc.) —
NOT just `eval()` in the MCP server process.

**Why it works:** Tool-definition tokens drop from O(N×schema_size) to O(2×schema_size).
Cloudflare's published data: 244,047 tokens (minimal-schema-per-endpoint, 2,594 tools) →
1,069 tokens (code mode, 2 tools). 99.6% reduction.

**Required mitigations** (all four — without them this is just AP-03 with extra steps):

- **Real sandbox isolation.** No `eval()` in the server process. V8 isolates, WebAssembly
  modules, or out-of-process execution.
- **Auth scope enforced at the binding, not the input.** The agent's code cannot escape
  the platform's existing permission model.
- **Dual-mode fallback.** Provide a `?codemode=false` (or equivalent) mode that registers
  per-endpoint tools. This is the security release valve for orgs that need per-call
  human review at the cost of context size.
- **`destructiveHint: true` on `execute`.** There is no static analysis for "this code
  blob is read-only" — assume the worst.

**Observed in:** Cloudflare MCP — `tools: [search, execute]`, hosted at
`mcp.cloudflare.com/mcp`. Sandbox: Cloudflare Dynamic Worker Loader API (V8 isolates).
Spec lives server-side; agent receives only execution results.

**Apply when:** Upstream API has ≥500 endpoints OR ≥100k tokens of tool schemas, AND the
platform already has a robust token-scope permission model the sandbox can inherit.

**Don't apply when:** Upstream API is small enough for traditional tools (≤50 endpoints).
Then this pattern is just AP-03 with extra steps — sandbox overhead, harder review, no
token-budget benefit.

**Counter-consideration:** Per-call human-in-the-loop review is essentially impossible on
a code-mode tool. A reviewer would have to read JS to know what the agent is about to do.
For high-stakes domains (finance, prod infra mutation, PII), prefer the dual-mode fallback
even with the token cost.

**Relationship to other patterns:**
- P-27 (Embedded Agent Tool) is the LLM-as-inner-runtime version of this. Sentry's
  `use_sentry` runs an LLM that calls atomic tools. Cloudflare's `execute` runs JS that
  calls APIs directly. Use P-27 when reasoning matters; use P-29 when token economics
  dominate.
- AP-03 (god-tool) is the antipattern this pattern deliberately violates with mitigations.
  See the AP-03 mitigation-spectrum table for the three-point comparison
  (mcp-bash unmitigated → cli-mcp-server mitigated → Cloudflare sanctioned).

---

## P-28: Embedded LLM Query Translator (candidate, 1 evidence point)

**Rule:** When a tool's input is a complex domain DSL (Sentry search syntax, SQL, JQL,
KQL) and users will sometimes pass natural language instead, use an **internal** LLM
agent to translate NL → DSL before calling the upstream API. This LLM is invisible to
the outer LLM-of-record — it is a server-side capability, not a tool the caller sees.

**Observed in:** Sentry MCP — `search_events/handler.ts` defines `buildSearchRepairPrompt()`
and imports `searchEventsAgent` for cases where the query parses as natural language.
Description states: *"`query` can be natural language or Sentry search syntax. With an
agent configured, it fixes dataset, query, fields, and sort before running."*

**Status:** Candidate (1 evidence point). Promote to confirmed when a second corpus
entry exhibits the same pattern. Likely candidates: any MCP fronting a complex query DSL
(Snowflake, BigQuery, Elasticsearch, Splunk).

**Counter-consideration:** Hidden LLM calls inside a tool inflate latency and cost in
ways the caller cannot predict. Always document the agent presence in the tool
description (Sentry does), and consider a `repair: false` opt-out parameter for callers
who need determinism.

---

## Pending patterns (await more corpus entries)

- [x] **Pagination patterns** — P-17 (chunked text via start_index/max_length, Fetch)
  covers text. Still need a clean cursor-pagination example for structured paginated lists.
- [x] **Resource-vs-tool decision tree** — Postgres + SQLite both 5/5 on D8; P-13
  (entity-per-resource) and P-14 (dynamic resource) capture the two main shapes. P-26
  (resource-resolver tool) is the third shape: tool-as-resource-router.
- [ ] **Multi-tenant auth context** — GitHub has multi-deployment; still need a true
  multi-tenant example with per-request auth contexts.
- [ ] **Streaming / long-running tools** — need to see real implementations of
  server-sent events or polling-pattern tools.
- [ ] **Prompt-injection-safe tools handling untrusted content** — need real defenses
  beyond the broken SQLite example (AP-11). Worth looking for in Stripe or Linear servers.
- [ ] **Tool/Prompt duality** (same name, different entry shape, different defaults).
  Fetch exhibits it. Watch for 2+ more examples before promoting.
- [ ] **Tool-count budget enforcement** — Sentry's `AGENTS.md` declares ≤20 target,
  hard limit 25, and enforces via mode-filtering. Watch for a second example before
  promoting to a pattern (currently a strong observation rather than a rule).
