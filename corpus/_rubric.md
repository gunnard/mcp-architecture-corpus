# MCP Server Evaluation Rubric

14 dimensions, each scored 1–5. Total possible score: 70.

**Score interpretation:**
- **60–70**: reference-quality, can be held up as a positive example
- **45–59**: solid, ships in production successfully, minor weaknesses
- **30–44**: works but has friction; the LLM struggles in predictable ways
- **15–29**: ships but bleeds; users will hit confusing failures
- **<15**: actively harmful, do not imitate

**How to use this rubric:**
- Score one MCP server at a time
- Cite specific tools/code as evidence for each dimension
- Note both what's done well and what's done poorly — both feed the patterns/antipatterns docs

---

## 1. Tool granularity (1–5)

**The question:** are tools at the right semantic level for an LLM to reason about?

- **5** — Tools map to user *intents*, not API endpoints. The LLM can express any reasonable
  user goal as 1–3 tool calls.
- **4** — Mostly intent-mapped. A handful of tools are slightly too narrow or too broad.
- **3** — Mixed. Some tools force the LLM to do orchestration in its head.
- **2** — Tools mirror the underlying REST API 1:1. The LLM must chain many calls for simple goals.
- **1** — Either one mega-tool with 15 parameters, OR 30 micro-tools where 5 would do.

**Anti-pattern signals:**
- A tool with `action: enum[20 values]` instead of 20 tools (or 5 tools)
- A tool called `query` that accepts arbitrary SQL/JSON-Logic/etc.
- Two tools that differ only in one parameter

**Evidence to capture:** count of tools, ratio of required-to-optional params, presence
of "god tools."

---

## 2. Tool naming (1–5)

**The question:** does the name predict the behavior closely enough that an LLM picks the
right tool on first try?

- **5** — Verb+object, disambiguated from siblings. `search_tickets`, `get_ticket`, `update_ticket`,
  `list_my_tickets` — no ambiguity.
- **3** — Mostly clear but some collisions or generic verbs. `do_thing`, `helper`, `process`.
- **1** — Names require reading the description to understand. `tool_1`, `execute`, `run`.

**Anti-pattern signals:**
- Generic verbs (`do`, `run`, `execute`, `process`, `handle`)
- Numeric suffixes (`search_2`, `get_v2`)
- Names that match across tools (`get_data` in two namespaces)

---

## 3. Description quality (1–5)

**The question:** does the description tell the LLM *when to use this* vs. *when not to*?

- **5** — Includes: what it does, when to use it, when NOT to use it, common pitfalls,
  example invocation, expected response shape.
- **4** — Has when-to-use guidance but missing pitfalls or counter-examples.
- **3** — Describes what it does but not when to choose it over siblings.
- **2** — Restates the function signature in prose.
- **1** — Missing or trivially short ("Gets a ticket").

**Anti-pattern signals:**
- Description = function name expanded into a sentence
- No mention of failure modes
- No mention of related tools the LLM might confuse this with

**Reference example:** Brodels's `brodels_run` description tells the LLM it's long-running,
expensive, and mentions cost ceilings — the LLM will then default to safer phases first.

---

## 4. Parameter design (1–5)

**The question:** are required params actually required? Are optional params well-defaulted?

- **5** — 1–3 required params max for common operations. Optional params have sane defaults
  documented in the schema.
- **3** — 4–6 required params; some are sometimes-required-sometimes-not (param interdependence).
- **1** — 8+ required params, or params that contradict each other depending on values.

**Anti-pattern signals:**
- "If `mode=foo`, also pass `bar` and `baz`" (conditional requirements)
- Required params that are actually almost always the same value across calls
- Optional params with no documented default behavior

---

## 5. Schema discipline (1–5)

**The question:** are JSON Schema types tight enough that the LLM can't produce garbage?

- **5** — Enums where applicable, regex patterns for IDs, format hints on dates/URLs/emails,
  min/max on numerics, oneOf for variants.
- **3** — Basic types correct but loose. Strings where enums would help.
- **1** — `params: object` or `args: any`. The LLM will fill it with hallucinated keys.

**Anti-pattern signals:**
- Free-form string params for things with finite valid values
- No format on date/time params
- `additionalProperties: true` on a closed schema

---

## 6. Error surface (1–5)

**The question:** when a call fails, what does the LLM see, and can it recover?

- **5** — Structured errors: `code`, human message, *hint to the LLM*, suggested next action.
  Failures are categorized (`invalid_input`, `not_found`, `permission_denied`, `transient`).
- **3** — Errors are returned cleanly but generic. LLM can sometimes recover.
- **1** — Raw stack traces, HTTP 500s, or silent failures. LLM retries blindly.

**Anti-pattern signals:**
- `try/except: pass` patterns
- Returning a 200 with `{success: false}` and a string message
- Errors that don't tell the LLM what input to change

**Reference example:** good error: `{code: "invalid_ticket_id", message: "Ticket ID 'foo'
does not match pattern", hint: "Expected format: PROJECT-1234"}`

---

## 7. Idempotency & side-effect signaling (1–5)

**The question:** does the LLM know which tools mutate state?

- **5** — Mutation is clear from name (`create_*`, `update_*`, `delete_*`). Destructive ops
  require explicit confirmation params or have dry-run modes. Description includes a
  "side effects" line.
- **3** — Most mutations are clear, but a few hide behind innocent names (`refresh_*` that
  triggers a write).
- **1** — No naming convention. `get_user` and `delete_user` look the same shape. No dry-run
  anywhere.

**Anti-pattern signals:**
- `process_*` or `handle_*` tools that mutate
- Destructive tools without `confirm: true` or two-step patterns
- No dry-run mode on any tool

---

## 8. Resources vs. tools split (1–5)

**The question:** is static-ish reference data exposed as MCP resources, or jammed into tools?

- **5** — Reference data (enum values, schemas, available projects, config) is exposed as
  resources the LLM can read into context once. Tools are reserved for actions and queries.
- **3** — Mostly tools, but some obvious resources. Reasonable but inefficient.
- **1** — Everything is a tool, including `list_available_statuses`. LLM wastes tokens
  fetching the same reference data every turn.

**Anti-pattern signals:**
- Tools whose only job is to return a hardcoded or rarely-changing list
- No use of the MCP resources facility at all

---

## 9. Auth model (1–5)

**The question:** how does the server handle credentials, scopes, and multi-tenancy?

- **5** — Per-request auth context with scope minimization. Credentials never logged.
  Rotation/refresh handled internally. Multi-tenant safe.
- **3** — Single-tenant with env-var credentials; works but doesn't scale.
- **1** — Credentials in code, no scope control, leakage risk in logs.

**Anti-pattern signals:**
- API keys logged in error messages
- One key with admin scope used for all operations
- No way to scope the server to a single user/tenant

---

## 10. Observability (1–5)

**The question:** can you reconstruct *why* the LLM made a given tool call and *what happened*?

- **5** — Structured logs (JSON), request IDs, tool-call latency, model decision context
  preserved, errors linked to inputs. You can audit a session end-to-end.
- **3** — Logs exist but are unstructured. Reconstructing a session takes effort.
- **1** — Print-to-stderr or nothing. When something goes wrong, you have no trail.

**Anti-pattern signals:**
- `print()` statements as the entire logging strategy
- No correlation ID across tool calls in a session
- Logs don't include tool inputs/outputs (or log them but in unparseable form)

---

## 11. Composability (1–5)

**The question:** do tool outputs feed cleanly into other tools' inputs?

- **5** — Shared identifier types across tools. Return shapes are consistent. The LLM can
  pass `result.id` from tool A directly to tool B without transformation.
- **3** — Mostly consistent, but a few tools return opaque blobs or wrap IDs differently.
- **1** — Each tool returns a different shape. The LLM has to extract fields with regex.

**Anti-pattern signals:**
- Tool A returns `{ticket: {id: ...}}` and Tool B expects `{ticket_id: ...}`
- Some tools return arrays, others return paginated envelopes, no consistency
- Opaque tokens that only certain tools accept

---

## 12. Pagination & volume handling (1–5)

**The question:** does the server handle large result sets without blowing the context window?

- **5** — Cursor-based pagination with sensible defaults (10–25 items). Summary modes
  (counts only) for exploration. Tools warn the LLM when results are truncated.
- **3** — Pagination exists but defaults are too large, or it's offset-based.
- **1** — Returns all rows always. A `list_users` call returns 50k users and breaks the session.

**Anti-pattern signals:**
- No limit/cursor params
- Default page size > 50
- No "this was truncated" signal in the response

---

## 13. Safety rails for destructive operations (1–5)

**The question:** can the LLM cause irreversible harm with a single tool call?

- **5** — Destructive tools require explicit `confirm` params or a separate "preview" tool
  first. Allow-listing for high-risk operations. Dry-run available everywhere relevant.
- **3** — Some safety on the most dangerous tools, but inconsistent.
- **1** — `delete_all_data` available with no guards. The LLM can wipe production with one call.

**Anti-pattern signals:**
- `force: bool` that defaults to true
- No dry-run on tools that touch production
- Destructive ops not visually distinct from queries

---

## 14. Documentation & onboarding (1–5)

**The question:** can a new developer go from clone to first successful tool call in <10 minutes?

- **5** — README has: install, auth setup, one-line config example, first invocation, common
  troubleshooting. Schema is browsable. There's a `--help` or equivalent.
- **3** — Install works but auth is unclear. Or the docs are accurate but verbose.
- **1** — "See source." No example invocation. Auth setup requires reverse-engineering.

**Anti-pattern signals:**
- README points to "examples/" that's empty
- Auth flow requires reading code
- No version compatibility statement with MCP spec versions

---

## Scoring template

When evaluating a server, copy this block to the corpus entry:

```
| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          |   /5  |          |
| 2. Tool naming               |   /5  |          |
| 3. Description quality       |   /5  |          |
| 4. Parameter design          |   /5  |          |
| 5. Schema discipline         |   /5  |          |
| 6. Error surface             |   /5  |          |
| 7. Idempotency signaling     |   /5  |          |
| 8. Resources vs tools        |   /5  |          |
| 9. Auth model                |   /5  |          |
| 10. Observability            |   /5  |          |
| 11. Composability            |   /5  |          |
| 12. Pagination/volume        |   /5  |          |
| 13. Safety rails             |   /5  |          |
| 14. Docs & onboarding        |   /5  |          |
| **Total**                    | **/70** |        |
```

---

## Rubric maintenance

This rubric is itself a living document. Add dimensions when the corpus reveals a recurring
failure mode that isn't captured. Remove dimensions if they stop discriminating between
good and bad servers.

**Suspected future additions:**
- Streaming/long-running operations (tools that need server-sent events or polling patterns)
- Multi-modal handling (tools that accept/return images, audio, etc.)
- Prompt injection resistance (tools that handle untrusted user content safely)
