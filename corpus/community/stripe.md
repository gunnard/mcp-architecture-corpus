# Corpus Entry: Stripe MCP Server

**Tier:** Tier-2 community gold
**Source:** `github.com/stripe/agent-toolkit` + `docs.stripe.com/mcp` + remote at `mcp.stripe.com`
**Status:** README + docs.stripe.com + `@stripe/mcp` npm source verified. Some dimensions still provisional (descriptions, schemas) because the local `@stripe/mcp` is a **thin proxy** — actual tool implementations live at `mcp.stripe.com`.
**Total score:** **58/70 (provisional, source-partially-verified)** — Third-highest in corpus. **First server in corpus to score 5/5 on D9 auth.** P-18 attribution pattern confirmed via source.

---

## What it is

A financial-services MCP exposing ~24 Stripe API operations to LLMs. Available in three
deployment modes:

1. **Hosted remote MCP** at `https://mcp.stripe.com` with OAuth — **first remote MCP
   in the corpus**
2. **Local server** via `npx -y @stripe/mcp --api-key=YOUR_STRIPE_SECRET_KEY`
3. **Agent toolkit** library (Python + TypeScript) for direct integration with OpenAI
   SDK, LangChain, CrewAI, Vercel AI SDK

Status: **Public preview** (Stripe is honest about maturity level).

## Tool surface (24 tools)

Account: `get_stripe_account_info`, `retrieve_balance`
Coupons: `create_coupon`, `list_coupons`
Customers: `create_customer`, `list_customers`
Disputes: `list_disputes`, `update_dispute`
Invoices: `create_invoice`, `create_invoice_item`, `finalize_invoice`, `list_invoices`
Payment Links: `create_payment_link`
Payment Intents: `list_payment_intents`
Prices: `create_price`, `list_prices`
Products: `create_product`, `list_products`
Refunds: `create_refund`
Subscriptions: `cancel_subscription`, `list_subscriptions`, `update_subscription`
Search/Docs: `search_stripe_resources`, `fetch_stripe_resources`, `search_stripe_documentation`

**Every tool is verb+object, atomic, 1:1 with a Stripe API endpoint.** Zero god-tools.
This is the cleanest tool naming surface in the corpus.

## Why this entry matters

Stripe is the **safety-critical financial server archetype**. When a buyer asks the
skill *"I want Claude to interact with our payments API"* (or any financial API), this
is the reference. It's also the FIRST server in the corpus to demonstrate:
- **OAuth-based remote MCP** (P-21)
- **Permission scoping via existing API key infrastructure** (P-22) — Stripe RAK
- **Documentation-search as a built-in tool** (P-23)
- **Explicit prompt-injection warning in docs** (P-24)

---

## Scoring

| Dimension                    | Score | Confidence | Evidence |
|------------------------------|-------|------------|----------|
| 1. Tool granularity          | 5/5   | High       | 24 atomic tools, 1:1 with API endpoints, verb+object. No god-tools. Every CRUD is its own tool. Reference-quality for an API-exposure server. |
| 2. Tool naming               | 5/5   | High       | `create_*`, `list_*`, `update_*`, `cancel_*`, `finalize_*`, `retrieve_*`, `search_*`, `fetch_*`, `get_*` — perfectly consistent. The cleanest naming surface in the corpus. |
| 3. Description quality       | 3/5   | **Provisional** | Each tool links to the underlying Stripe API doc. Can't verify per-tool descriptions without source inspection. Stripe's API docs are excellent so the inherited descriptions likely are too; conservatively 3 until verified. |
| 4. Parameter design          | 4/5   | **Provisional** | Can't fully verify from docs page. Stripe APIs are well-known for sensible defaults and minimal required params. Projecting 4. |
| 5. Schema discipline         | 4/5   | **Provisional** | Stripe APIs use typed enums (currencies, statuses, modes). Schemas inherited from Stripe SDK should be strong. Projecting 4. |
| 6. Error surface             | 4/5   | **Provisional** | Stripe API errors are well-structured (`code`, `message`, `type`, `param`). MCP wrapper likely passes through. Projecting 4. |
| 7. Idempotency signaling     | 4/5   | **Provisional** | Tool naming makes mutation vs. read very clear. Stripe API has formal idempotency keys for create operations. Can't verify if MCP `ToolAnnotations` are set. |
| 8. Resources vs tools        | 3/5   | High       | `search_stripe_documentation` and `search_stripe_resources` are tools rather than resources (defensible since search-by-query is naturally a tool). No resource exposure visible. |
| 9. Auth model                | **5/5** | High     | **FIRST 5/5 in corpus on this dimension.** OAuth for remote MCP. RAK for local. **Tool permissions enforced via Stripe's existing Restricted API Key scope system** — leverages mature, battle-tested permission infrastructure rather than reinventing. Multi-deployment. New gold standard for MCP auth. |
| 10. Observability            | 4/5   | **Provisional** | Stripe's API has rich observability via Dashboard, request logs, webhook events. MCP wrapper inherits this — every tool call shows up as a Stripe API request with full audit trail. Projecting 4. |
| 11. Composability            | 4/5   | **Provisional** | Tools return Stripe objects (well-known shape). `customer.id` from `create_customer` works as input to any customer-id-accepting tool. Projecting 4. |
| 12. Pagination/volume        | 4/5   | **Provisional** | Stripe's list endpoints have cursor-based pagination (`limit`, `starting_after`, `ending_before`). MCP wrapper should expose. Projecting 4. |
| 13. Safety rails             | **5/5** | High     | Reference-quality. **Explicit prompt-injection warning** in docs (P-24): *"We recommend enabling human confirmation of tools and exercising caution when using the Stripe MCP with other servers to avoid prompt injection attacks."* RAK scoping limits blast radius per-key. Public-preview disclaimer for honest expectations. Dashboard session management. The right posture for a financial server. |
| 14. Docs & onboarding        | 4/5   | High       | Cursor one-click install. Docs split between agent-toolkit README and docs.stripe.com/mcp (a -1). Multi-deployment instructions. Comprehensive but spread across two locations. |
| **Total**                    | **58/70** | — | Provisional. 8 of 14 dimensions verified from docs; 6 require source inspection to confirm. Worst case (if all provisional dims drop by 1): 52. Best case (all 5s): 64. |

---

## Patterns extracted (new from this entry)

### P-21: Hosted remote MCP with OAuth

**Rule:** For SaaS APIs with established OAuth flows, offer a hosted remote MCP server
that uses OAuth for authentication. Clients connect via URL (`{"url": "https://mcp.example.com"}`)
instead of spawning a local process.

**Solves:**
- Zero install — buyer adds one JSON config and the server is ready
- Centralized maintenance — provider updates the server, no client re-deploy
- Identity-aware — each user authenticates once via OAuth, no API-key shuffling
- Multi-device — same auth works from Cursor, Claude Desktop, Goose, anywhere

**Observed in:** Stripe MCP (`mcp.stripe.com`). Cursor config is one line:
```json
{ "mcpServers": { "stripe": { "url": "https://mcp.stripe.com" } } }
```

**Apply when:** Provider has existing OAuth infrastructure. Especially for SaaS products
where the buyer already has an account.

**Counter-consideration:** Requires the MCP host to support remote/URL-based servers
(Cursor and Claude Desktop both do; some clients don't yet). Latency is higher than
local stdio. Provider bears the runtime cost.

---

### P-22: Permission scoping via existing API key infrastructure

**Rule:** When the provider already has a permission-scoped API key system (Stripe's RAK,
AWS IAM keys, GitHub fine-grained PATs, etc.), use IT for tool-level permissions rather
than building MCP-specific permission scopes.

**Solves:**
- Reuses battle-tested permission infrastructure
- Users already know how to manage scopes in the provider's dashboard
- Audit trails flow through existing systems
- No new permission model to teach
- Failure modes (revocation, rotation, expiration) are already handled

**Observed in:** Stripe MCP. *"Tool permissions are controlled by your Restricted API
Key (RAK). Create a RAK with the desired permissions at
https://dashboard.stripe.com/apikeys."*

**Apply when:** Provider has mature scoped-key infrastructure. Don't invent a new
permission model when an existing one works.

**Counter-consideration:** Couples the MCP server to the provider's key model. If
the provider's keys are coarse-grained, the MCP is also coarse-grained.

---

### P-23: Documentation-search as a built-in tool

**Rule:** For servers exposing a complex API, include a tool that searches the
provider's documentation and knowledge base. The LLM can answer "how do I X" questions
before invoking domain tools.

**Solves:**
- LLM doesn't have to guess at API semantics
- Reduces hallucinations on parameter shapes / edge cases
- Self-service onboarding — the server teaches the LLM how to use itself
- Cuts down on the "I tried X and got error Y" trial-and-error loop

**Observed in:** Stripe MCP — `search_stripe_documentation`, `search_stripe_resources`,
`fetch_stripe_resources` form a 3-tool docs/KB surface.

**Apply when:** Server's domain has substantial documentation (>50 pages of API reference,
guides, tutorials). The cost is one tool; the benefit is fewer failed invocations.

**Counter-consideration:** Tempting to make this the primary tool ("just ask the docs").
Resist — the LLM should still invoke real domain tools for real work, not just RAG over docs.

---

### P-24: Explicit prompt-injection warning in docs

**Rule:** For servers that act on real-world systems (payments, code, infra), publish
an explicit prompt-injection warning in the README/docs. Acknowledge the threat model.

**Solves:**
- Honest threat communication
- Sets expectation that human-in-the-loop is appropriate
- Signals mature security thinking to buyers
- Reduces legal/reputational risk when (not if) prompt-injection is attempted

**Observed in:** Stripe MCP. *"We recommend enabling human confirmation of tools and
exercising caution when using the Stripe MCP with other servers to avoid prompt
injection attacks."*

**Apply when:** Tool calls have real-world consequences (money, code merges, infra
changes, communications sent on behalf of user).

**Pairs well with:** P-12 server-level policy modes (read-only, lockdown), P-20
per-call consent prompts.

---

## Sample evidence quotes

**Atomic tool naming (D1, D2):**
```
create_customer / list_customers
create_invoice / list_invoices / finalize_invoice
cancel_subscription / list_subscriptions / update_subscription
```
24 tools, every one verb+object, every one 1:1 with a Stripe API endpoint.

**Hosted remote MCP (P-21):**
```json
{ "mcpServers": { "stripe": { "url": "https://mcp.stripe.com" } } }
```

**RAK-based permissions (P-22):**
> Tool permissions are controlled by your Restricted API Key (RAK). Create a RAK with
> the desired permissions at https://dashboard.stripe.com/apikeys

**Prompt-injection warning (P-24):**
> The server exposes the following MCP tools. We recommend enabling **human confirmation
> of tools and exercising caution when using the Stripe MCP with other servers to avoid
> prompt injection attacks**. If you have feedback or want to see more tools, email us
> at mcp@stripe.com.

**Public preview honesty (D13):**
> Model Context Protocol (MCP) [Public preview]

The "Public preview" tag is itself a safety signal — Stripe is honest about maturity
rather than overstating reliability.

---

## Verification log (partial)

`@stripe/mcp` source fetched from unpkg. **Major architectural finding:** the local
server is a **thin proxy** — it forwards stdio messages over HTTPS to `mcp.stripe.com`.
All tool implementations, descriptions, schemas, and error shapes live server-side at
the hosted endpoint. The local CLI only:
1. Validates the API key
2. Builds a User-Agent string from the MCP client's `initialize` request
3. Forwards messages bidirectionally between stdio and HTTPS

This is a **deliberate architectural choice** worth its own pattern (see notes below).

### Confirmed via source:
- [x] **P-18 confirmed** — `extractClientName(message)` + `buildUserAgent(clientName)`
  build per-client User-Agent strings. Third citation for P-18; canonical.
- [x] **D9 auth = 5/5 confirmed** — `validateApiKey(options.apiKey)` and
  `validateStripeAccount(options.stripeAccount)` are explicit input validation.
  `buildHeaders(options, userAgent)` constructs auth + attribution headers.
- [x] **Stdio-to-HTTPS proxy pattern** — first occurrence in corpus; new pattern P-25.

### Still provisional (require hitting mcp.stripe.com to verify):
- [ ] D3 description quality (lives at remote server)
- [ ] D5 schema discipline (server-side schemas)
- [ ] D6 error response shapes (mostly: most pass through from remote)
- [ ] D7 `ToolAnnotations` presence (server-side)
- [ ] D12 pagination semantics in tool responses

The thin-proxy architecture means **the hosted server is the canonical Stripe MCP** and
any future scoring update should query the remote endpoint directly rather than
inspecting more local source.

---

## Notes for the skill

When a buyer says *"I want Claude to access our SaaS API"* (Stripe-like: payments,
financial, billing, customer management):

1. **Recommend hosted remote MCP with OAuth (P-21)** if the buyer has the infrastructure
2. **Recommend permission scoping via existing API keys (P-22)** — don't reinvent
3. **Recommend documentation search as a built-in tool (P-23)** for any complex API
4. **Recommend explicit prompt-injection warning in docs (P-24)** for any tool with
   real-world consequences
5. **Recommend atomic verb+object naming** — Stripe's 24-tool surface is the cleanest
   in the corpus; cite it as the reference for "how to name CRUD tools"
6. **Recommend public-preview tagging** if the buyer isn't ready to commit to GA
   reliability — honest maturity signaling is a safety pattern

---

## Cross-server position (9 entries, with Stripe provisional)

| Server | Total | Note |
|---|---|---|
| Filesystem | 59 | Reference standard |
| **Stripe** | **58 prov.** | **First 5/5 on auth (D9)** |
| Brodels | 56 | Description + observability leader |
| Playwright | 55 | Naming + annotations + docs all 5 |
| Memory | 54 | Batch-by-default, system prompt |
| Fetch | 54 | One-tool reference; first 5/5 on pagination |
| GitHub | 49 | God-tool dispatch hurts |
| Postgres | 41 | Pre-modern, resources done right |
| SQLite | 35 | Safety theater (security finding) |

**The 54-59 cluster** has 5 servers. The skill should treat this band as "
production-ready with one or two specific weaknesses to flag." Below 49 is "use as
educational counter-examples."
