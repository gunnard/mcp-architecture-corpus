# Corpus Entry: SQLite MCP Server

**Tier:** Anthropic reference (now archived)
**Source:** `github.com/modelcontextprotocol/servers-archived/main/src/sqlite`
**Status:** Scored from full README + complete `server.py` source.
**Total score:** **35/70** — **Lowest in corpus.** Contains a real security finding worth a new antipattern entry (AP-11: safety theater).

---

## What it is

A Python MCP server exposing SQLite database operations. Uses ALL THREE MCP primitives:
- **Tools** (6): `read_query`, `write_query`, `create_table`, `list_tables`, `describe-table`, `append_insight`
- **Resources** (1): `memo://insights` — a "living document" that updates dynamically as `append_insight` is called
- **Prompts** (1): `mcp-demo` — a multi-step interactive prompt that walks users through a business-intelligence demo

This was the canonical "kitchen sink" reference that showed off all three MCP primitives in one server.

## Why it scores lowest in the corpus

Despite being a richer, more ambitious reference than Postgres, SQLite scores 6 points lower because:

1. **Inconsistent tool naming in the same file** (`read_query` / `describe-table` — underscore vs hyphen)
2. **Safety theater** (D13 — see security finding below)
3. **No read/write isolation** — the same DB handle accepts everything
4. **Conditional dispatch in `write_query`** (accepts INSERT, UPDATE, OR DELETE — that's a method enum hidden in prose)
5. **AP-04 raw SQL throughout**, with worse enforcement than Postgres

---

## Security finding (drives D13 = 1/5)

The server uses string-prefix matching to decide whether to commit or roll back:

```python
if query.strip().upper().startswith(('INSERT', 'UPDATE', 'DELETE', 'CREATE', 'DROP', 'ALTER')):
    conn.commit()
    affected = cursor.rowcount
    return [{"affected_rows": affected}]
```

This has **three independent problems**:

1. **`read_query` has no enforcement.** The tool description says "Execute a SELECT query"
   but the implementation passes the query through unchanged. An LLM (or attacker via prompt
   injection) can call `read_query` with `INSERT INTO ...` and it executes. The prefix check
   above only runs in `write_query`'s code path, not `read_query`'s.

2. **`write_query` description lies about scope.** Description says
   *"Execute an INSERT, UPDATE, or DELETE query"*. Implementation accepts `CREATE`, `DROP`,
   `ALTER` too. The LLM may decide "I'll create a table via `write_query`" because
   nothing technically stops it.

3. **Multi-statement queries bypass the prefix check.** SQLite supports
   `SELECT * FROM users; DROP TABLE users;` as a single query string. The prefix check
   sees `SELECT` and the statement still executes as a destructive operation.

**This is the canonical AP-11 (safety theater) example.** The server *describes* restrictions
that the implementation does not enforce. The LLM and the human reviewer both believe
the safety boundary is real.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 3/5   | `read_query` / `write_query` is a reasonable split. But `write_query` accepts INSERT/UPDATE/DELETE (method-like overload). `create_table` is split out separately. Inconsistent: why isn't there `update_query` or `delete_query`? Why ISN'T `create_table` part of `write_query`? |
| 2. Tool naming               | 2/5   | **Mixed casing in the same file.** `read_query`, `write_query`, `create_table`, `list_tables` use underscores. `describe-table` uses a hyphen. Same server, same handler list. This is a real-world signal of incoherent design discipline. |
| 3. Description quality       | 2/5   | `"Execute an INSERT, UPDATE, or DELETE query"` is a dispatch table in prose (AP-03 lite). Read-query description: `"Execute a SELECT query on the SQLite database"` — better but doesn't warn about the safety theater finding. |
| 4. Parameter design          | 3/5   | Simple params (`query` string for most). `describe-table` has a clean `table_name`. Reasonable shape. |
| 5. Schema discipline         | 1/5   | **AP-04 canonical.** `query: string` for all SQL tools. No constraints, no examples, no syntax hint. |
| 6. Error surface             | 2/5   | `raise ValueError("Unknown prompt")` is generic. Database errors propagate via `logger.error` + re-raise. The LLM gets raw Python tracebacks in some paths. |
| 7. Idempotency signaling     | 2/5   | No MCP `ToolAnnotations`. `write_query` is destructive but you only know from the description. `create_table` is destructive. `append_insight` mutates a resource — and its name doesn't hint at mutation. |
| 8. Resources vs tools        | **5/5** | **Reference-quality, in a different way than Postgres.** Uses all three MCP primitives: Resources (`memo://insights`), Prompts (`mcp-demo`), Tools. The `memo://insights` resource is *dynamic* — it updates as `append_insight` is called. Canonical "resources as living documents" example. |
| 9. Auth model                | 2/5   | DB path via CLI. No auth (SQLite is local). Appropriate for the domain but minimal. |
| 10. Observability            | 3/5   | Has `logger.debug`, `logger.info`, `logger.error`. Structured Python logging. Better than Postgres but no request IDs and no per-tool latency. |
| 11. Composability            | 4/5   | Consistent shapes: writes return `[{"affected_rows": N}]`, reads return list of row dicts. Predictable. Not using `structuredContent` but the conventions are clean. |
| 12. Pagination/volume        | 1/5   | None. `read_query` returns all rows. Same problem as Postgres. |
| 13. Safety rails             | 1/5   | **Worst in corpus.** Safety theater: prefix-string matching does not enforce the boundaries the descriptions claim. See security finding above. AP-11 canonical example. |
| 14. Docs & onboarding        | 4/5   | README is decent. Shows Resources + Prompts + Tools architecture clearly. `uv` + Docker setup documented. Could be 5 with security warnings. |
| **Total**                    | **35/70** | Lowest in corpus. Two 5/5 dimensions (resources, docs) can't overcome four ≤2/5 dimensions. Inconsistent design discipline manifests at every layer. |

---

## Patterns extracted (new from this entry)

### P-14: Dynamic resource (living document)

**Rule:** Resources can update over the course of a session. Tools that mutate state can
trigger the resource to be re-read by the LLM (via the resource subscription mechanism)
or simply expose updated content on the next `read_resource` call.

**Solves:** Long-running analyses where insights/state accumulate. Avoids stuffing
intermediate results into the tool response payload.

**Observed in:** SQLite MCP (`memo://insights` — synthesized fresh on every read from
`db.insights[]`; `append_insight` tool extends the list).

**Apply when:** The LLM is doing iterative analysis and accumulating findings worth
preserving across the session.

**Counter-consideration:** The resource's "current state" is server-side; the LLM has
to re-`read_resource` to see updates. Doesn't compose with stateless tool chains.

---

## Antipatterns confirmed (with strong evidence)

### AP-11: Safety theater — NEW antipattern

**Failure mode:** The server's tool descriptions claim safety properties (read-only,
restricted-scope) that the implementation does NOT enforce. The LLM and the human
reviewer both believe a boundary exists; the boundary does not exist.

**Observed in:** SQLite MCP `read_query` / `write_query`. See full analysis above.

**Why it's particularly dangerous:**
- Code reviewers see the description and don't dive into the implementation
- LLMs trust the description and don't probe for boundary violations
- Prompt-injection payloads can deliberately exploit the mismatch
- The "safety" claim provides social cover ("we're a read-only server, no DBA review needed")

**Fix:**
- Parse the SQL with a real parser (e.g., `sqlglot`, `pg_query`) and reject statements
  outside the declared scope
- OR enforce at the transaction layer (Postgres's `BEGIN TRANSACTION READ ONLY` pattern)
- NEVER rely on string-prefix matching for SQL safety
- For multi-statement support: reject queries containing `;` outside string literals

**Smell:**
- Implementation uses `startswith()`, regex, or prefix matching to "validate" SQL
- Description promises a restriction that's enforced only by convention
- Tool name suggests a scope (`read_*`) but implementation doesn't check

### AP-04: Raw SQL as a tool parameter — REINFORCED

SQLite's `query: string` pattern combined with the safety-theater failure makes this a
*worse* AP-04 example than Postgres (which at least has DB-layer enforcement).

---

## Sample evidence quotes

**Safety theater (AP-11) — the smoking gun:**
```python
if query.strip().upper().startswith(('INSERT', 'UPDATE', 'DELETE', 'CREATE', 'DROP', 'ALTER')):
    conn.commit()
```

**Mixed-style tool names in the same handler list:**
```python
types.Tool(name="read_query", ...),       # underscore
types.Tool(name="write_query", ...),      # underscore
types.Tool(name="create_table", ...),     # underscore
types.Tool(name="list_tables", ...),      # underscore
types.Tool(name="describe-table", ...),   # HYPHEN — same file!
types.Tool(name="append_insight", ...),   # underscore
```

**Dispatch-via-description on `write_query`:**
```python
description="Execute an INSERT, UPDATE, or DELETE query on the SQLite database"
# But implementation accepts CREATE, DROP, ALTER too. Description lies.
```

**Dynamic resource (P-14):**
```python
@server.read_resource()
async def handle_read_resource(uri: AnyUrl) -> str:
    # ...
    return db._synthesize_memo()  # synthesized FRESH each read from db.insights[]
```

---

## Notes for the skill output

When a buyer says *"I want Claude to access my SQL database"*, the skill MUST:

1. Recommend the **Postgres pattern** (read-only transaction enforcement) for query tools
2. **Reject the SQLite pattern** as a worked example of what NOT to do — even though it's
   an Anthropic reference server, it has a confirmed security finding
3. Recommend **structured query tools** (`select_from`, `filter_by`, `aggregate`) over
   raw SQL pass-through when the use case allows
4. Recommend exposing **schema metadata as resources** (the Postgres pattern), not as tools

The juxtaposition of these two siblings (Postgres 41/70 vs SQLite 35/70) is one of the
skill's best teaching moments: **two reference servers from the same source, doing the
same domain, with very different design discipline and one with a real security bug.**

---

## Cross-server position (updated)

| Server | Total | Resources | Schema | Safety | Naming |
|---|---|---|---|---|---|
| Filesystem | 59 | 3 | 5 | 5 | 5 |
| Brodels | 56 | 2 | 4 | 5 | 4 |
| GitHub | 49 | 3 | 3 | 5 | 3 |
| Postgres | 41 | 5 | 1 | 5 | 3 |
| **SQLite** | **35** | **5** | **1** | **1** | **2** |
