# Corpus Entry: Memory MCP Server (Knowledge Graph)

**Tier:** Anthropic reference (currently maintained)
**Source:** `github.com/modelcontextprotocol/servers/main/src/memory`
**Status:** Scored from README + index.ts. Modern SDK build.
**Total score:** **54/70** — Middle of the corpus. Strong on tool design discipline. Weak in the dimension where it should be strongest: resources.

---

## What it is

A knowledge-graph persistence server: entities, relations, and observations stored in a
JSON file. Lets the LLM build up structured memory across sessions.

- **Tools (9):** `create_entities`, `create_relations`, `add_observations`,
  `delete_entities`, `delete_observations`, `delete_relations`, `read_graph`,
  `search_nodes`, `open_nodes`
- **Resources:** none
- **Prompts:** none in the server, but README publishes an example System Prompt
  for integrators

## Why this entry matters

Memory is a **state-management archetype** — when a buyer asks the skill *"I want Claude
to remember things across sessions"*, this is the canonical reference. It's also a
notable counterexample: a server whose entire purpose is to be a reference-data
substrate, yet it doesn't use MCP resources at all.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 5/5   | 9 tools, each intent-mapped. CRUD split across entities/relations/observations is correctly factored. `read_graph` (full dump) vs `search_nodes` (filtered) vs `open_nodes` (by-name) is a thoughtful three-way split giving the LLM cost-appropriate retrieval options. |
| 2. Tool naming               | 5/5   | Verb+object throughout. `create_*` / `delete_*` / `add_*` for mutations; `read_*` / `search_*` / `open_*` for reads. Zero ambiguity. |
| 3. Description quality       | 3/5   | Short and accurate (*"Create multiple new entities in the knowledge graph"*) but no "when to use vs alternatives," no examples, no pitfalls. README has rich docs; tool descriptions themselves are thin. Mismatch. |
| 4. Parameter design          | 5/5   | **Batch-by-default**: every tool takes an array (`entities`, `relations`, `observations`, `entityNames`, `deletions`). Spawns P-15. Reduces tool-call count for common multi-entity operations. |
| 5. Schema discipline         | 5/5   | Zod schemas reused across tools (`EntitySchema`, `RelationSchema`). Every field has `.describe()`. Output schemas defined. Reference-quality. |
| 6. Error surface             | 4/5   | `throw new Error(\`Entity with name ${name} not found\`)` is informative. Documented mix of throws and silent-ignores (deletes are silent if missing — idempotent by design). README clarifies. |
| 7. Idempotency signaling     | 3/5   | Verbs are clear. Silent-on-missing semantics make deletes effectively idempotent. **But NO `ToolAnnotations` set** — modern SDK supports them and this server doesn't use them. Missing opportunity. |
| 8. Resources vs tools        | 2/5   | **Weakness — particularly notable.** This server stores reference data the LLM will consult repeatedly across sessions. Yet `read_graph` returns the entire blob via a tool call. Each entity could be a resource (`memory://entity/<name>`). The server has the strongest case in the corpus for using resources and the worst track record. |
| 9. Auth model                | 2/5   | File path via env var. Single-tenant. Single user. For per-Claude-instance memory this is acceptable; doesn't scale to shared/team memory. |
| 10. Observability            | 1/5   | **Weakness.** Nothing visible — no logging, no metrics, no audit trail. State mutations are silent. For a persistent-state server this is a real gap; reconstructing why the LLM remembered something is impossible. |
| 11. Composability            | 5/5   | Returns `structuredContent` (P-09) everywhere. Consistent shapes via shared schemas. The LLM can chain `search_nodes → open_nodes → add_observations` without parsing. Modern. |
| 12. Pagination/volume        | 1/5   | **Weakness.** `read_graph` returns the entire graph. As memory grows, this blows context. No size limit, no pagination, no summary mode. |
| 13. Safety rails             | 3/5   | Cascading delete on `delete_entities` (removes orphan relations) is responsible. But no dry-run, no confirmation, no safety annotation. The LLM can wipe an entity with one call. Acceptable for "memory" (cheap to recreate) but still a gap. |
| 14. Docs & onboarding        | 5/5   | Excellent README: Core Concepts (Entities/Relations/Observations with JSON examples), full API docs, install instructions, **and an example System Prompt** showing real usage (P-16). Reference-quality. |
| **Total**                    | **54/70** | Strong on tool design (granularity, naming, params, schema, composability). Weak where it should be strongest: resources facility unused. |

---

## Patterns extracted (new from this entry)

### P-15: Batch-by-default inputs

**Rule:** Tools that operate on entities should accept an array, not a singleton, as the
default shape. Single-entity calls become array-of-one.

**Solves:**
- Reduces tool-call count for multi-entity operations (LLM doesn't loop)
- Reduces context cost (one tool definition handles N items)
- Eliminates the "should I call this 5 times in parallel or one batched call?" decision

**Observed in:** Memory MCP (every tool takes an array — `entities`, `relations`,
`observations`, `entityNames`, `deletions`).

**Apply when:**
- Tool operates on entities and the LLM commonly works with multiple
- Operation is naturally parallelizable
- Per-item cost is low (large batches don't hit timeout)

**Counter-consideration:** For destructive operations, batching multiplies blast radius.
Pair batch-by-default with confirmation patterns or dry-run on destructive variants.

---

### P-16: Example System Prompt in server docs

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

## Antipatterns reinforced

### AP-01: Reference data exposed as tools — STRONGEST EXAMPLE YET

Memory's entire purpose is to be reference data the LLM consults across sessions.
`read_graph`, `search_nodes`, `open_nodes` are all read-tools where MCP resources would
be the right primitive. Each entity could be `memory://entity/<name>`; each relation set
could be `memory://entity/<name>/relations`.

The skill should cite Memory + Postgres side-by-side: same broad shape (a queryable data
store), Postgres uses resources (5/5), Memory uses tools (2/5). The contrast is the
teaching moment.

---

## Sample evidence quotes

**Batch-by-default (P-15):**
```typescript
inputSchema: {
  entities: z.array(EntitySchema)
}
```

**Reusable Zod schemas (D5):**
```typescript
const EntitySchema = z.object({
  name: z.string().describe("The name of the entity"),
  entityType: z.string().describe("The type of the entity"),
  observations: z.array(z.string()).describe(
    "An array of observation contents associated with the entity"
  )
});
```

**Dual return (P-09):**
```typescript
return {
  content: [{ type: "text", text: JSON.stringify(result, null, 2) }],
  structuredContent: { entities: result }
};
```

**Example System Prompt (P-16):**
```
1. User Identification:
   - You should assume that you are interacting with default_user
2. Memory Retrieval:
   - Always begin your chat by saying only "Remembering..."
   - retrieve all relevant information from your knowledge graph
3. Memory: While conversing, be attentive to: identity, behaviors, preferences, goals, relationships
4. Memory Update:
   - Create entities for recurring organizations, people, events
   - Connect via relations
   - Store facts as observations
```

---

## Notes for the skill

When a buyer says *"I want Claude to remember things across sessions"*:

1. Recommend the **batch-by-default** input pattern (P-15)
2. Recommend the **Postgres resource-per-entity pattern** (P-13) for the data store —
   NOT Memory's everything-is-a-tool approach
3. Recommend publishing an **example System Prompt** (P-16) in the docs
4. Recommend **structured content** dual-return (P-09)
5. Recommend **explicit `ToolAnnotations`** that Memory doesn't set (P-08)
6. Warn against Memory's pattern of `read_graph` returning everything (no pagination)
