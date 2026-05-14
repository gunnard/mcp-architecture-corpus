# Corpus Entry: Filesystem MCP Server

**Tier:** Anthropic reference
**Source:** `github.com/modelcontextprotocol/servers/tree/main/src/filesystem`
**Status:** Scored from README + index.ts. Live testing optional (server is stateless).
**Total score:** **59/70** — Reference-quality. The highest score in the corpus so far.

---

## What it is

A 13-tool MCP server exposing filesystem operations: read (text, media, batch), write,
edit, list, search, move, get-info, plus directory discovery. Uses MCP Roots protocol for
dynamic access control. ~12 tools, single domain.

## Why it scores so high

Three architectural choices put this above Brodels (56/70) despite Brodels being a more
complex product:

1. **Explicit MCP `ToolAnnotations`** — every tool has `readOnlyHint` / `idempotentHint` /
   `destructiveHint`. The README publishes a full table. The LLM (and human) knows the
   side-effect profile of every tool without reading the description.
2. **Zod schemas with `.describe()` calls** — type discipline + inline field documentation
   in one place.
3. **`structuredContent` dual return** — tools return both human-readable text AND a
   machine-parsable structured object via the modern MCP SDK pattern.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 5/5   | 13 tools, each maps to a distinct user intent. `read_text_file` / `read_media_file` / `read_multiple_files` are correctly split by return-shape. `edit_file` vs `write_file` is the right granularity (granular edit vs full replace). |
| 2. Tool naming               | 5/5   | Verb+object throughout. `list_directory` vs `list_directory_with_sizes` — extension in the name. `list_allowed_directories` for discovery. Zero collisions. |
| 3. Description quality       | 4/5   | Each tool explains what, when to use, constraints (`Only works within allowed directories`), and feature lists for complex ones (`edit_file`). Includes deprecation notice on `read_file`. Could be 5 with "when NOT to use" guidance. |
| 4. Parameter design          | 5/5   | Most tools have 1 required param (`path`). Optional params well-defaulted. `read_text_file` has mutex rule (`head` xor `tail`) explicitly documented. `edit_file` has `dryRun: false` default. |
| 5. Schema discipline         | 5/5   | Zod with `.describe()` calls. Enums where appropriate (`sortBy: enum(['name', 'size'])`). Array `.min(1)` constraints. Defaults specified. Reference-quality. |
| 6. Error surface             | 4/5   | Descriptive errors (`"Cannot specify both head and tail parameters simultaneously"`). Centralized path validation. Couldn't verify full error shape but content is good. |
| 7. Idempotency signaling     | 5/5   | **Reference standard.** Every tool has explicit `readOnlyHint` / `idempotentHint` / `destructiveHint`. README publishes the mapping as a table. Example: `write_file` is `destructive:true, idempotent:true`; `edit_file` is `destructive:true, idempotent:false`. The LLM gets this for free from MCP metadata. |
| 8. Resources vs tools        | 3/5   | `list_allowed_directories` is a tool that smells like a resource. However, the server uses MCP **Roots** for dynamic directory access — sophisticated. Net: mixed. |
| 9. Auth model                | 4/5   | Allowed-directories whitelist enforced via centralized path validation. Symlink resolution at startup (`fs.realpath`). Roots protocol for dynamic permissions. Single-tenant by design (appropriate for filesystem). |
| 10. Observability            | 2/5   | **Weakness.** `console.error()` for warnings only. No structured logging, no request IDs, no metrics. The weakest dimension. Acceptable for a single-user reference server, weak by the rubric. |
| 11. Composability            | 4/5   | Returns `structuredContent: { content }` alongside text content. Output schemas defined (`outputSchema: { content: z.string() }`). Modern MCP dual-return pattern. Path identifier consistent across all tools. |
| 12. Pagination/volume        | 3/5   | `read_text_file` has `head`/`tail` for partial reads. But `directory_tree` and `list_directory` return everything; a deep tree blows context. No truncation signal. |
| 13. Safety rails             | 5/5   | Path validation prevents escape. Symlink resolution at startup. Allowed-directories whitelist enforced everywhere. `edit_file` has explicit `dryRun` mode. Write tools annotated `destructive:true`. README explicitly says "exercise caution with this" on `write_file`. Multi-layer defense. |
| 14. Docs & onboarding        | 5/5   | README has: features, access control (CLI args + Roots), full tools API, MCP hints table, Claude Desktop config, VS Code config, Docker + NPX both ways, build instructions. New dev can be running in <10 min. |
| **Total**                    | **59/70** | Reference-quality. Single notable weakness: observability. |

---

## Patterns extracted (new from this entry)

### P-08: MCP ToolAnnotations table

**Rule:** Publish a table in the README mapping every tool to its `readOnlyHint`,
`idempotentHint`, `destructiveHint` values.

**Observed:** Filesystem README has the canonical example.

**Why it matters:** The LLM gets side-effect metadata via the MCP protocol; the human
reviewer gets the same info via the table. Tool-picking discipline improves on both sides.

### P-09: Dual return (text + structuredContent)

**Rule:** Return tools with both `content: [{type: "text", text: ...}]` AND
`structuredContent: { ... }`. The text channel is for LLM consumption; the structured
channel is for programmatic clients and downstream tool composition.

**Observed:** Filesystem index.ts uses this pattern via the modern MCP SDK.

**Why it matters:** Solves the Brodels prose-vs-structured tension (P-07) by doing both.

---

## Sample evidence quotes

**Zod schema discipline (D5):**
```typescript
const ReadTextFileArgsSchema = z.object({
  path: z.string(),
  tail: z.number().optional().describe('If provided, returns only the last N lines of the file'),
  head: z.number().optional().describe('If provided, returns only the first N lines of the file')
});
```

**ToolAnnotations table from README (D7):**
```
| Tool          | readOnlyHint | idempotentHint | destructiveHint |
| write_file    | false        | true           | true            |
| edit_file     | false        | false          | true            |
| move_file     | false        | false          | true            |
```

**Dual return pattern (D11):**
```typescript
return {
  content: [{ type: "text" as const, text: content }],
  structuredContent: { content }
};
```

**Path-escape safety (D13):**
```typescript
// Security: Resolve symlinks in allowed directories during startup
// This ensures we know the real paths and can validate against them later
const resolved = await fs.realpath(absolute);
```

---

## Comparison vs Brodels

| Dimension | Filesystem | Brodels | Notes |
|---|---|---|---|
| Tool granularity | 5 | 5 | Tied — both intent-mapped |
| Tool naming | 5 | 4 | Filesystem cleaner (no `run` or `doctor`) |
| Schema discipline | 5 | 4 | Filesystem's Zod approach is tighter |
| Idempotency | 5 | 4 | Filesystem has explicit annotations |
| Resources vs tools | 3 | 2 | Filesystem uses Roots; Brodels uses nothing |
| Observability | 2 | 5 | Brodels wins by a mile here |
| Composability | 4 | 3 | Filesystem has structuredContent |
| Safety rails | 5 | 5 | Tied — both excellent in their domain |

**Takeaway:** Brodels is a bigger product with much more observability investment, but the
core tool-design discipline (granularity, schema, annotations) is where Filesystem pulls
ahead. The skill should recommend Filesystem patterns for tool definitions and Brodels
patterns for operational concerns.
