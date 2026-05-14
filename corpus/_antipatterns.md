# MCP Design Antipatterns (Catalogue)

Distilled from the corpus. Each antipattern includes:
- The name
- The failure mode it causes
- Where it was observed
- The fix
- A "smell" that detects it quickly

---

## AP-01: Reference data exposed as tools

**Failure mode:** LLM wastes tokens (and inference decisions) re-fetching slowly-changing
reference data every session. The data also pollutes the response payload of unrelated tools
because the LLM "looks it up" mid-task.

**Observed in:** Brodels (`brodels_skills_list`, `brodels_db_sites`, `brodels_workspace_list`).

**Fix:** Expose as MCP resources. The LLM reads resources into context once per session,
not per call. Tools are reserved for actions and queries.

**Smell:** Any tool whose output changes <1×/day and is read >5×/session. Or any tool
named `list_*` that returns a finite, slowly-changing set.

---

## AP-02: Offset/top-N pagination without cursors

**Failure mode:** LLM cannot reliably fetch "the next page" of results. When a limit is
hit, the LLM doesn't know it was truncated, so partial results get treated as complete.

**Observed in:** Brodels (`brodels_jira_search`, `brodels_logs`, `brodels_lesson_usage`).

**Fix:**
1. Return an explicit `truncated: true` and `next_cursor: <opaque>` when limits hit
2. Accept the cursor as a param on subsequent calls
3. In the description, tell the LLM how to detect and handle truncation

**Smell:** A list-returning tool with `limit` or `maxResults` but no `cursor` / `page_token` /
`continue_from` companion.

---

## AP-03: God-tool with `method`/`action` enum — CONFIRMED

**Failure mode:** One tool with `method: enum[N values]` instead of N distinct tools.
The LLM must memorize the value-to-behavior mapping. Errors don't surface until the call
returns failure. Conditional required params (AP-08) almost always come along for the ride.

**Observed in:** **GitHub MCP** (`actions_get`, `actions_list`, `actions_run_trigger`,
`label_write`). The description for each god-tool *is the dispatch table in prose*. The
same server has properly-split tools elsewhere (`list_notifications`, `dismiss_notification`,
`manage_notification_subscription`) which proves it's a design choice, not a constraint.

**Fix:** Split into separate atomic tools.
- `actions_get_workflow`, `actions_get_workflow_run`, `actions_get_workflow_run_usage`,
  `actions_download_artifact`, `actions_get_workflow_job`
- Or follow the notifications pattern in the same server

**Smell:** A tool with an `action`, `mode`, `operation`, `method`, or `command` enum param
with >3 values. Or a description that lists conditional behavior per value.

**Reference quote (GitHub `actions_get`):**
```
resource_id: The unique identifier of the resource. This will vary based on the "method":
  - Provide a workflow ID or workflow file name for 'get_workflow' method.
  - Provide a workflow run ID for 'get_workflow_run', 'get_workflow_run_usage', and
    'get_workflow_run_logs_url' methods.
  - Provide an artifact ID for 'download_workflow_run_artifact' method.
  - Provide a job ID for 'get_workflow_job' method.
```
This prose IS the bug. It belongs in tool definitions, not in field descriptions.

---

## AP-04: Raw shell/SQL/code as a tool parameter — CONFIRMED

**Failure mode:**
1. Prompt-injection surface — untrusted content reaches privileged execution
2. No schema discipline — the LLM hallucinates syntax
3. Errors are unparseable to the LLM
4. Tool description has to enumerate dialect support, error categories, etc. in prose
   because the schema carries no information

**Observed in:** **Postgres MCP** and **SQLite MCP** (both Anthropic reference servers,
now archived). Both expose `query: string` with zero schema constraints.

**Severity gradient:**
- **Postgres**: Mitigated by `BEGIN TRANSACTION READ ONLY` + `ROLLBACK` at the DB layer.
  AP-04 still applies but is bounded by transaction semantics.
- **SQLite**: NOT mitigated — see AP-11. String-prefix matching does not enforce the
  scope claimed in tool descriptions. The two antipatterns compound.

**Fix:** Define structured operations. Instead of `query: string`, offer `select_from`,
`filter`, `join`, `aggregate` with typed shapes. If passthrough is truly needed:
1. Wrap in a transaction with explicit read-only semantics (Postgres pattern)
2. Parse the SQL server-side with a real parser (`sqlglot`, `pg_query`)
3. Document the prompt-injection risk explicitly
4. NEVER rely on string-prefix matching for safety

**Smell:** Any tool param named `sql`, `code`, `command`, `script`, or `query` accepting
free-form strings without parser-level validation.

---

## AP-05: Secrets in error messages

**Failure mode:** API keys, tokens, or PII echo back in error responses. The LLM may
log them, surface them to users, or leak them via downstream tools.

**Observed in:** Structurally common across community servers — empirically uncited in
the scored corpus but the absence-of-P-05 (redaction discipline) is the failure mode.
**Counter-citation (POSITIVE pattern):** **Brodels** uses a centralized redaction utility
for secrets in tool inputs *and* error payloads (see P-05 redaction discipline in
`corpus/community/brodels.md`). The antipattern is the negative space: servers that
echo `Authorization: Bearer <token>` headers or `?api_key=...` query strings into
error responses unsanitized.

**Specific risk in the corpus:** **Postgres MCP** strips the password from resource URLs
(positive) but does NOT have explicit handling for `pg_error` payloads that may echo
the connection string. Audit before production.

**Fix:** Sanitize error payloads server-side. Replace credentials with their key name
(`<API_KEY>` not the value). Use a single redaction utility, not per-handler scrubbing.
Brodels' approach: one `redactSecrets()` function called at the error-boundary layer.

**Smell:** Any error response that includes request headers or input params verbatim,
especially with `Authorization`, `X-Api-Key`, `password`, or `token` fields.

---

## AP-06: Generic verbs in tool names

**Failure mode:** LLM must read full description to disambiguate from siblings. Wastes
context; increases wrong-tool-picked rate.

**Observed in:** Brodels (mild — `brodels_run`, `brodels_doctor`).

**Fix:** Verb+object naming. `brodels_run` could be `brodels_pipeline_run`. `brodels_doctor`
could be `brodels_diagnose_install`.

**Smell:** Tool names that are single generic verbs: `run`, `do`, `process`, `handle`,
`execute`, `helper`.

---

## AP-07: No truncation signal on bounded responses

**Failure mode:** When a tool internally limits results (for safety or cost), the LLM
treats partial output as complete output and makes wrong decisions.

**Observed in:** **Brodels** (`brodels_jira_search`, `brodels_logs`, `brodels_lesson_usage`)
— same tools cited in AP-02. The root cause is shared: tools that cap result counts
but don't tell the LLM the cap was hit. Memory MCP's `search_nodes` similarly returns
a bounded list with no `truncated` flag.

**Counter-example (POSITIVE):** **Fetch MCP** (`corpus/reference/fetch.md`, 54/70)
correctly signals: when `start_index + max_length` doesn't cover the full content, the
response explicitly includes a continuation hint with the next `start_index` value.
First 5/5 on D12 pagination because of this discipline.

**Fix:** Always return `{results: [...], truncated: bool, total_available?: int,
next_cursor?: string}`. Pair with description guidance: *"If `truncated` is true, the
result set was capped at `limit`. Call again with `cursor` to continue."*

**Smell:** A list-returning tool whose schema doesn't include any of: `truncated`,
`has_more`, `next_cursor`, `total_count`, `total_available`.

---

## AP-08: Conditional required parameters — CONFIRMED

**Failure mode:** Schema says param X is `optional`, but the description says it's required
when another param has a certain value. The LLM either always provides X (suboptimal) or
guesses wrong and gets errors. Schema validation can't catch the mistake because the
schema is loose by design.

**Observed in:** **GitHub MCP** `actions_run_trigger`. Example:
```
ref: The git reference. **Required for 'run_workflow' method.** (string, optional)
run_id: The workflow run ID. **Required for all methods except 'run_workflow'.** (number, optional)
```

**Fix:** Split into separate tools with non-overlapping required-param shapes. Or use
oneOf in the schema to express the variant. The skill should always recommend the split.

**Smell:** Description text containing phrases like:
- "Required for X method"
- "Required when [condition]"
- "Only used when"
- "If `mode=create`, also pass `name` and `email`"

---

## AP-11: Safety theater — CONFIRMED

**Failure mode:** The server's tool descriptions claim safety properties (read-only,
restricted-scope, sandboxed) that the implementation does NOT enforce. The LLM and the
human reviewer both believe a boundary exists; the boundary does not.

**Observed in:** **SQLite MCP** (Anthropic reference server, now archived). The
`read_query` tool's description says "Execute a SELECT query" but the implementation
passes any SQL through without inspection. The `write_query` description says
"INSERT, UPDATE, or DELETE" but the implementation accepts CREATE/DROP/ALTER too via
a leaky string-prefix check.

**Why it's particularly dangerous:**
- Code reviewers see the description and don't dive into the implementation
- LLMs trust the description and don't probe for boundary violations
- Prompt-injection payloads can deliberately exploit the mismatch
- The "safety" claim provides social cover ("we're read-only, no DBA review needed")
- Multi-statement queries (`SELECT *; DROP TABLE x;`) defeat prefix-based checks entirely

**Fix:**
- Enforce at the layer below the tool (Postgres pattern: `BEGIN TRANSACTION READ ONLY`)
- OR parse the input with a real parser (`sqlglot`, `pg_query`, an AST-based shell parser)
- NEVER rely on string-prefix matching, regex, or "starts with" checks for safety
- Description must match implementation; if they diverge, fix the implementation, not
  the description

**Smell:**
- Implementation uses `startswith()`, `match()`, or prefix matching to "validate" input
- Tool name suggests a scope (`read_*`) but implementation has no check
- Description promises a restriction enforced only by convention

**Reference evidence (SQLite `_execute_query`):**
```python
if query.strip().upper().startswith(('INSERT', 'UPDATE', 'DELETE', 'CREATE', 'DROP', 'ALTER')):
    conn.commit()
# Defeated by: `SELECT *; DROP TABLE users;`
```

---

## AP-09: Mutate-but-look-like-reads

**Failure mode:** Tool name and description suggest a read operation but the
implementation mutates state. The LLM (and the human reviewer) classify it as safe
to retry and chain freely. ToolAnnotations are missing or wrong (`readOnlyHint: true`
when it shouldn't be).

**Observed in:** **GitHub MCP** `actions_run_trigger` reads as a verb-noun trigger but
actually creates a new workflow run (mutation, billable, state-changing). The tool's
description buries the side effect mid-paragraph. Memory MCP's `read_graph` is the
inverse: name suggests pure read and implementation is pure read, so it's fine.

**Fix:**
1. Rename: `trigger_workflow_run` instead of `actions_run_trigger`
2. Set `readOnlyHint: false` explicitly
3. Lead the description with the side effect: *"Creates a new workflow run."*
4. If the tool reads-then-writes, split into two tools

**Smell:**
- Tool name uses neutral verbs (`refresh`, `validate`, `check`, `sync`) but the
  description mentions state change
- `readOnlyHint` not set or set to `true` on a tool that writes
- Verbs that should be unambiguously read (`get`, `list`, `search`) coupled with
  side-effects in the implementation

---

## AP-10: Unbounded mutation blast radius

**Failure mode:** A mutation tool accepts an unbounded array or unbounded query filter
with no count cap. A single LLM call can affect millions of records.

**Observed in:** **Memory MCP** `delete_entities({ entityNames: string[] })` — accepts
an unbounded array of entity names with no cap. A prompt-injection or LLM confusion
can delete the entire graph in one call. Similarly `delete_observations` and
`delete_relations`.

**Severity:** Higher than AP-04 because the LLM doesn't even need to construct a
specific dangerous payload — it just has to be wrong about scope.

**Fix:**
1. Cap array inputs at a sensible max (50, 100, etc.)
2. Require `confirm_count: number` parameter that must equal the array length
   (catches "I meant to update 5 but matched 50" mistakes)
3. Default to dry-run for any tool affecting >10 records
4. For query-filter-based mutations, require explicit `where_clause_max_matches: number`
5. Document the cap in the tool description

**Counter-citations (POSITIVE):** Brodels' `prune_lessons` caps deletion via
`observation_floor_days` and `min_age_days` gates (P-03 multi-gate). Worked example 03
(HubSpot) caps `bulk_update_contacts` at 50 with `confirm_count` echo.

**Smell:**
- Mutation tool accepting `string[]`, `id[]`, `entities[]`, etc. with no max
- Filter-based mutation (`delete_where_status=closed`) with no count preview
- `bulk_*` tools without explicit caps

---

## Pending antipatterns (not yet observed in corpus, structural concerns)

- [ ] **No idempotency token on create operations** — LLM retries cause duplicates.
  Stripe MCP's API supports this via the underlying API; check whether `@stripe/mcp`
  surfaces it in tool descriptions.
- [ ] **Boolean-flag explosion** — 8+ boolean params where an enum would carry semantics.
  Watch for this in cloud-infra MCPs (AWS, Cloudflare) once scored.
