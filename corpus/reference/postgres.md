# Corpus Entry: PostgreSQL MCP Server

**Tier:** Anthropic reference (now archived)
**Source:** `github.com/modelcontextprotocol/servers-archived/main/src/postgres`
**Status:** Scored from full README + complete `index.ts` source. Server is small (~140 lines).
**Total score:** **41/70** — Pre-modern reference. Notable for being **first server in corpus to use MCP resources correctly (5/5)**, but suffers from raw-SQL antipattern and bare-minimum description quality.

---

## What it is

A minimal MCP server exposing one tool (`query` — read-only SQL) and dynamic per-table
schema resources at `postgres://<host>/<table>/schema`. Uses `BEGIN TRANSACTION READ ONLY`
+ `ROLLBACK` to enforce read-only semantics at the database layer.

## Why it scores low despite being a reference

This server was an early canonical example. It predates:
- MCP `ToolAnnotations` (read-only / idempotent / destructive hints)
- The modern SDK `structuredContent` dual-return pattern
- Zod-style schema discipline with `.describe()`
- Pagination conventions in the community

The design discipline that was state-of-the-art in 2024 is now below the bar. It's a useful
artifact for understanding how MCP design has evolved, and for one specific pattern done
extremely well: **resources for schema metadata**.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 3/5   | One tool (`query`). Appropriate for "expose SQL DB," but most LLM intents (count, find duplicates, get schema) don't decompose into "write SQL." Forces the LLM to be a SQL author. |
| 2. Tool naming               | 3/5   | `query` is a generic verb. For a single-tool server tolerable, but doesn't predict behavior beyond "do something with SQL." Same shape as the `brodels_run` / `actions_run_trigger` family. |
| 3. Description quality       | 1/5   | **Verbatim quote:** *"Run a read-only SQL query"*. That's the entire description. No mention of: read-only enforcement mechanism, supported dialects, how to discover schema (which IS exposed but only via resources), errors to expect. |
| 4. Parameter design          | 4/5   | One required param. Simple. Could be tighter (no max length on SQL string). |
| 5. Schema discipline         | 1/5   | **AP-04 canonical.** `sql: { type: "string" }` — no constraints, no examples, no format hint. The LLM hallucinates dialect-specific syntax with no guardrails. |
| 6. Error surface             | 3/5   | `throw error` propagates DB errors verbatim — good for LLM self-correction on syntax issues. But no error categorization. |
| 7. Idempotency signaling     | 3/5   | Tool is read-only by transaction wrapping. But NO `ToolAnnotations` set. The LLM must read the description to learn it's read-only. Pre-annotation era. |
| 8. Resources vs tools        | **5/5** | **Reference-quality.** Schema metadata exposed as MCP resources (`postgres://<host>/<table>/schema`), not as a tool. Auto-discovered from `information_schema.tables`. Each table becomes a resource the LLM can `read_resource` once and keep in context. This is the canonical pattern. |
| 9. Auth model                | 3/5   | DB URL via CLI arg. Password stripped from resource URLs (`resourceBaseUrl.password = ""`) — small but thoughtful detail. No multi-tenant. |
| 10. Observability            | 1/5   | **Weakness.** `console.warn` on rollback error is the entire logging strategy. No request IDs, no structured logging. |
| 11. Composability            | 4/5   | Returns `JSON.stringify(rows, null, 2)` as text content with `isError: false` envelope. Predictable shape. Pre-`structuredContent` SDK. |
| 12. Pagination/volume        | 1/5   | **Weakness.** None. `SELECT * FROM users` returns 10M rows; server happily JSON-stringifies all of them and blows the context window. |
| 13. Safety rails             | 5/5   | `BEGIN TRANSACTION READ ONLY` + `ROLLBACK` after every call — enforced at the DB layer, not at server validation. Strong defense for the chosen surface. Cannot be bypassed by SQL injection because the transaction is read-only. |
| 14. Docs & onboarding        | 4/5   | README is concise: Docker, NPX, Claude Desktop config, VS Code config. Covers all the basics. Could be 5 with a troubleshooting section. |
| **Total**                    | **41/70** | Pre-modern reference. Two 5/5 dimensions (resources, safety); two 1/5 dimensions (description, schema discipline). Wide variance is the signal. |

---

## Patterns extracted (new from this entry)

### P-13: Resource-per-entity for slowly-changing reference data

**Rule:** For metadata that the LLM might want to consult repeatedly (table schemas, API
endpoints, config values, user preferences), expose each entity as a separate MCP
resource at a deterministic URI pattern.

**Solves:**
- LLM can read once, cache in context
- No tool-call overhead per schema lookup
- Each resource has its own URI, so the LLM can reference it later in conversation
- Auto-discoverable via `list_resources`

**Observed in:** Postgres MCP (`postgres://<host>/<table>/schema` — one resource per
table, auto-generated from `information_schema.tables`).

**Apply when:**
- You have N entities of the same shape (tables, projects, users, etc.)
- Each entity's "schema" or "metadata" is queryable but rarely changes
- The LLM may need to reference any of N entities

**Reference implementation:**
```typescript
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const result = await client.query(
    "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'"
  );
  return {
    resources: result.rows.map((row) => ({
      uri: new URL(`${row.table_name}/${SCHEMA_PATH}`, resourceBaseUrl).href,
      mimeType: "application/json",
      name: `"${row.table_name}" database schema`,
    })),
  };
});
```

---

## Antipatterns confirmed (with strong evidence)

### AP-04: Raw SQL as a tool parameter — CONFIRMED

The `query` tool accepts `sql: string` with zero schema constraints. This is the
canonical AP-04 example, **shipped as an Anthropic reference server**. Mitigated only
by the `BEGIN TRANSACTION READ ONLY` wrapper.

**Note for skill output:** when a buyer says "I want Claude to query my database," the
skill should recommend AGAINST this pattern unless paired with read-only transaction
enforcement. Even then, recommend structured query tools (`select_from`, `filter`,
`aggregate`) for predictability.

---

## Sample evidence quotes

**Resource discovery (P-13):**
```typescript
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  // queries information_schema.tables to find all tables
  // returns each as a resource with its own URI
});
```

**Read-only enforcement at the DB layer (D13):**
```typescript
await client.query("BEGIN TRANSACTION READ ONLY");
const result = await client.query(sql);
// ... finally ...
client.query("ROLLBACK").catch(...)
```

**Password stripping for safety (D9):**
```typescript
const resourceBaseUrl = new URL(databaseUrl);
resourceBaseUrl.protocol = "postgres:";
resourceBaseUrl.password = "";  // credentials never leak into resource URIs
```

**The whole tool definition (D3, D5):**
```typescript
{
  name: "query",
  description: "Run a read-only SQL query",
  inputSchema: {
    type: "object",
    properties: {
      sql: { type: "string" },
    },
  },
}
```
This is what the LLM sees. Every weakness in D3 and D5 is visible in this 8-line block.

---

## Cross-server position

| Server | Total | Resources | Schema | Annotations | Observability |
|---|---|---|---|---|---|
| Filesystem | 59 | 3 | 5 | 5 | 2 |
| Brodels | 56 | 2 | 4 | 4 | 5 |
| GitHub | 49 | 3 | 3 | 4 | 3 |
| **Postgres** | **41** | **5** | **1** | **3** | **1** |

Postgres is the **first server in the corpus to score 5/5 on resources-vs-tools**. The
skill should cite this entry whenever a buyer's domain has "many entities of the same
shape with queryable metadata."
