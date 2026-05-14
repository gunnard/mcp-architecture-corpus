# Corpus Entry: Fetch MCP Server

**Tier:** Anthropic reference (currently maintained)
**Source:** `github.com/modelcontextprotocol/servers/main/src/fetch`
**Status:** Scored from full README. Python-based, single-tool server.
**Total score:** **54/70** — Tied with Memory by total but very different shape. First server in the corpus to score 5/5 on D12 (pagination).

---

## What it is

A single-tool MCP server that fetches a URL and returns its content as markdown (or raw).
Stateless HTTP gateway. ~1 tool, ~1 prompt of the same name.

- **Tool (1):** `fetch` — URL, with optional `max_length`, `start_index`, `raw`
- **Prompt (1):** `fetch` — user-initiated counterpart to the tool

## Why this entry matters

Fetch is the **HTTP gateway archetype** — when a buyer asks the skill *"I want Claude to
read web content"*, this is the canonical pattern. It's also a teaching example for two
things: (a) a one-tool server can score well if the tool is meticulously designed, and
(b) chunked pagination via `start_index + max_length` is a pattern more servers should
adopt.

The same name being used for both a Tool AND a Prompt (model-initiated vs user-initiated)
is itself a notable design choice worth extracting.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 3/5   | One tool. Same shape as Postgres `query`. Appropriate for HTTP fetch but doesn't decompose user intents (parse HTML, extract structured data, follow redirects). The LLM has to do its own analysis. |
| 2. Tool naming               | 4/5   | `fetch` is the clearest possible name for the domain. Generic verb but maps directly to HTTP semantics. For a one-tool server this is fine. |
| 3. Description quality       | 4/5   | *"Fetches a URL from the internet and extracts its contents as markdown"* — clear scope, mentions the markdown transformation. Could be 5 with rate-limit / robots.txt notes inline. |
| 4. Parameter design          | 5/5   | 1 required (`url`). 3 optional with sensible defaults: `max_length: 5000`, `start_index: 0`, `raw: false`. **Pagination is built into the tool itself** via `start_index`. Spawns P-17. |
| 5. Schema discipline         | 4/5   | Types correct (string/integer/boolean). Numeric bounds via integer typing. Could tighten with URL format validation. |
| 6. Error surface             | 3/5   | README hints at thoughtful error paths (robots.txt blocks return errors). Conservative; couldn't verify shapes from README alone. |
| 7. Idempotency signaling     | 4/5   | `fetch` is clearly a read (HTTP GET). Idempotent by HTTP semantics. No `ToolAnnotations` set — trend across the corpus is that even modern servers don't use them. |
| 8. Resources vs tools        | 3/5   | Exposes itself as both a Tool AND a Prompt (model-initiated vs user-initiated entry points). No resources, but for a stateless HTTP gateway resources don't naturally apply. The Tool/Prompt duality is itself a design choice (P-18). |
| 9. Auth model                | 4/5   | **User-agent differentiation** between model-initiated and user-initiated requests is a notable pattern. robots.txt respect by default (with `--ignore-robots-txt` opt-out). Proxy support via `--proxy-url`. No per-request auth but the design choices around request attribution are mature. |
| 10. Observability            | 2/5   | Not detailed in README. Likely minimal. Conservative. |
| 11. Composability            | 4/5   | Returns markdown by default (or raw HTML if `raw=true`). `start_index + max_length` gives the LLM chunked reads. Predictable. Could be 5 with structured content (markdown + metadata about the fetch). |
| 12. Pagination/volume        | **5/5** | **First server in corpus at 5/5.** `start_index` + `max_length` is the chunked-pagination pattern the rubric calls for. Default 5000 chars is small enough to not blow context. The LLM can do multi-call fetches of long pages. Spawns P-17. |
| 13. Safety rails             | 4/5   | robots.txt respect by default. User-agent attribution lets external operators know it's model-initiated traffic. Proxy support for restricted environments. Defaults are responsible. No URL allow-listing visible (would push to 5). |
| 14. Docs & onboarding        | 5/5   | Excellent README: install via uvx/Docker/pip, VS Code one-click installs, customization options (robots, UA, proxy), debugging notes. Reference-quality. |
| **Total**                    | **54/70** | Tied with Memory on total but inverted shape: tiny surface, exquisite per-tool design. The one-tool server done right. |

---

## Patterns extracted (new from this entry)

### P-17: Built-in chunked pagination via `start_index` + `max_length`

**Rule:** For tools that return large textual content (web pages, file contents, log
dumps, document bodies), expose chunked-read params directly on the tool: a `start_index`
(offset) and `max_length` (limit). The LLM does its own multi-call paging without
needing a separate "continue" tool.

**Solves:**
- Content too large for one tool response
- Avoids the "cursor pagination requires a stateful tool pair" complexity
- LLM can selectively read the middle or end of large content
- Stateless — each call is independent

**Observed in:** Fetch MCP (`max_length: 5000` default, `start_index: 0` default).

**Apply when:** Tool returns variable-length content where the LLM might want a slice
rather than the whole thing.

**Reference shape:**
```
- url (string, required)
- max_length (integer, optional, default: 5000)  // characters per call
- start_index (integer, optional, default: 0)    // offset for resumption
- raw (boolean, optional, default: false)        // skip markdown conversion
```

**Counter-consideration:** Works for character-indexed text content. For structured
data (paginated lists), cursor-based pagination is still the right choice.

---

### P-18: Model vs user-initiated request attribution

**Rule:** When a tool makes external HTTP requests, differentiate model-initiated traffic
from user-initiated traffic (via user-agent header, request log tag, or similar).
External operators can then reason about LLM-driven access patterns.

**Solves:**
- External services can rate-limit or block model traffic separately
- robots.txt compliance can be applied selectively (e.g., respect for model traffic,
  bypass for user-initiated)
- Operators gain visibility into LLM access without instrumenting the LLM

**Observed in:** Fetch MCP. Two user-agent strings:
```
ModelContextProtocol/1.0 (Autonomous; +https://github.com/modelcontextprotocol/servers)
ModelContextProtocol/1.0 (User-Specified; +https://github.com/modelcontextprotocol/servers)
```

The "Autonomous" tag is sent when the model invokes the `fetch` tool. The "User-Specified"
tag is sent when the user invokes the `fetch` *prompt*. Same server, two paths,
distinguishable traffic.

**Apply when:** Tool issues external requests on behalf of an LLM. Especially relevant
for tools that scrape web content, hit third-party APIs, or send messages.

---

### Bonus pattern (worth noting, not promoting yet)

The Tool/Prompt duality in Fetch — same name, different entry shape — is an interesting
MCP-protocol-level pattern. Promote when 2+ more servers exhibit it.

---

## Sample evidence quotes

**Built-in chunked pagination (P-17):**
```
- max_length (integer, optional): Maximum number of characters to return (default: 5000)
- start_index (integer, optional): Start content from this character index (default: 0)
- raw (boolean, optional): Get raw content without markdown conversion (default: false)
```

**Request attribution (P-18):**
> *"By default, depending on if the request came from the model (via a tool), or was
> user initiated (via a prompt), the server will use either the user-agent
> [Autonomous] or [User-Specified]."*

**Responsible defaults (D13):**
> *"By default, the server will obey a websites robots.txt file if the request came from
> the model (via a tool), but not if the request was user initiated (via a prompt). This
> can be disabled by adding the argument `--ignore-robots-txt`."*

The split — model traffic respects robots, user traffic doesn't by default — is a
defensible policy choice and a worked example of how Tool vs Prompt differ semantically,
not just structurally.

---

## Notes for the skill

When a buyer says *"I want Claude to read web content"* or *"I want Claude to call an
external API"*:

1. Recommend the **single-tool with chunked-pagination params** pattern (P-17)
2. Recommend **model vs user request attribution** via user-agent (P-18)
3. Recommend **robots.txt respect by default** with a documented override
4. Recommend the **Tool/Prompt duality** when the same operation has different defaults
   for model vs user invocation
5. Caveat: for structured-API access, recommend specific endpoint tools rather than a
   generic HTTP gateway — fetch is the right pattern for *unstructured content*, not
   for "everything HTTP"

---

## Updated cross-server position

| Server | Total | Top strength | Weakest |
|---|---|---|---|
| Filesystem | 59 | Annotations, schema, safety, docs | Observability (2) |
| Brodels | 56 | Description, observability, safety | Resources (2) |
| Memory | **54** | Granularity, naming, params, schema, composability, docs | Observability (1), pagination (1), resources (2) |
| Fetch | **54** | Pagination (first 5/5), params, docs | Granularity (3), observability (2) |
| GitHub | 49 | Auth, safety, docs | Params (2) |
| Postgres | 41 | Resources, safety | Description (1), schema (1), obs (1), page (1) |
| SQLite | 35 | Resources, docs | Safety (1) — security finding |

**The 54 cluster.** Memory and Fetch both score 54 by following completely different
philosophies: Memory has 9 well-designed tools; Fetch has 1 meticulously-designed tool.
The skill should note that **good MCP design has multiple valid shapes** depending on
the domain.
