# Corpus Entry: Playwright MCP Server (Microsoft)

**Tier:** Tier-2 community gold
**Source:** `github.com/microsoft/playwright-mcp`
**Status:** Scored from comprehensive README (auto-generated tool list + config table). Source not deep-read.
**Total score:** **55/70** — Solid Tier-2. Best-in-corpus on tool naming (5) and idempotency signaling (5). Tied with Brodels-class on docs and overall.

---

## What it is

A browser-automation MCP server exposing Playwright's accessibility-tree + coordinate
APIs to LLMs. ~30 tools organized by capability:
- Core automation (~20 tools, always on): `browser_click`, `browser_type`, `browser_navigate`,
  `browser_snapshot`, `browser_take_screenshot`, `browser_fill_form`, etc.
- Tab management (`browser_tabs`)
- Capability-gated (opt-in via `--caps=<name>`):
  - `vision` → coordinate-based interactions (`browser_mouse_click_xy`, etc.)
  - `pdf` → PDF generation
  - `network` → request mocking, offline simulation
  - `testing` → assertion tools
  - `config` → server-config introspection

## Why this entry matters

Playwright is the **browser-automation archetype**. When a buyer asks the skill *"I want
Claude to drive a browser"*, this is the reference. It's also the cleanest implementation
of **toolset gating** (P-10) in the corpus, and the **second server** (after Filesystem)
to publish read-only annotations next to every tool.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 4/5   | 30+ tools, mostly intent-mapped (`browser_click`, `browser_type`, `browser_navigate`). Clean per-verb separation. But `browser_tabs` is an `action` enum god-tool (AP-03 lite). `browser_evaluate` accepts arbitrary JS strings (AP-04). −1 for these. |
| 2. Tool naming               | 5/5   | `browser_<verb>` namespacing throughout. `browser_click` vs `browser_drag` vs `browser_drop`. `browser_snapshot` (accessibility tree) vs `browser_take_screenshot` (image) — distinct because they have different consumption shapes. Zero collisions. Reference-quality. |
| 3. Description quality       | 4/5   | Strong examples: *"Take a screenshot of the current page. **You can't perform actions based on the screenshot, use browser_snapshot for actions.**"* — that's when-NOT-to-use guidance, the gold standard (P-08 cousin). But many tool descriptions are terse one-liners. Mix of 5-quality and 3-quality. |
| 4. Parameter design          | 4/5   | Tools have `element` (human description) + `target` (machine reference) pattern — explicit permission layer (P-20). Multiple optional params with defaults. `--caps` opt-in keeps the surface small by default. `browser_tabs.action` is a god-tool param though. |
| 5. Schema discipline         | 4/5   | Types correct. Enum-typed params (`button`, `action`, `state`, `type`). `target` is a structured selector. Could be 5 with stricter URL/path validation on params. |
| 6. Error surface             | 3/5   | Not visible in README; conservative. The capability gating ensures most error paths are predictable (capability not enabled → tool not exposed). |
| 7. Idempotency signaling     | 5/5   | **Best-in-corpus tied with Filesystem.** Every tool entry in the README has explicit `Read-only: true/false` directly under the parameters. Browser-mutation tools (click, type, navigate) are `Read-only: false`; introspection tools (snapshot, screenshot, console_messages, video_chapter) are `Read-only: true`. P-08 done canonically. |
| 8. Resources vs tools        | 3/5   | No resources visible. Capability-gated toolsets (`--caps`) are the main organizational primitive. Browser state (current page, snapshot, console messages) could arguably be resources but is exposed via tools. |
| 9. Auth model                | 4/5   | `--allowed-hosts`, `--allowed-origins`, `--blocked-origins`, `--allow-unrestricted-file-access` give fine-grained access control. README **explicitly states** these are NOT a security boundary — honest disclaimer. Multi-transport: stdio, SSE, Docker, CDP endpoint. Comprehensive. |
| 10. Observability            | 3/5   | `browser_console_messages` exposes browser console to LLM. Video recording capability lets you replay agent sessions. No structured server-side logging visible. |
| 11. Composability            | 4/5   | The `target` param accepts machine references returned by `browser_snapshot`. Tool chaining works: snapshot → identify element → click target → verify. Predictable. Could be 5 with structured content explicitly documented. |
| 12. Pagination/volume        | 3/5   | `browser_snapshot` has `depth` to limit tree depth. Many tools have `filename` optional param to **redirect output to disk** instead of returning inline — see P-19. But no cursor pagination on lists. |
| 13. Safety rails             | 4/5   | `element` parameter ("Human-readable element description **used to obtain permission** to interact") is per-tool-call consent. Honest README disclaimer (*"not a security boundary"*). Origin/host allowlists + blocklists. `--block-service-workers`, `--ignore-https-errors`, `--allow-unrestricted-file-access` are explicit policy toggles. `browser_evaluate` allows arbitrary JS though. |
| 14. Docs & onboarding        | 5/5   | Exceptional. CLI vs MCP comparison upfront (with honest *"prefer CLI for coding agents"* recommendation), Docker, NPX, programmatic SSE, Docker long-lived service, comprehensive `--cap=*` table, config table with 30+ flags, security disclaimer, robots/UA/proxy customization. |
| **Total**                    | **55/70** | Solid Tier-2. Just below Brodels (56). Notable strengths: naming (5), idempotency annotations (5), docs (5). |

---

## Patterns extracted (new from this entry)

### P-19: Output redirection to file via `filename` param

**Rule:** For tools that return potentially large output (screenshots, accessibility
trees, PDFs, console dumps, video), accept an optional `filename` parameter. When
provided, the tool saves to disk and returns the path; when omitted, returns inline.

**Solves:**
- Avoids blowing the LLM's context window on large binary/structured output
- Lets the LLM keep large artifacts addressable (by filename) without paying token cost
- Composes with file-reading tools downstream

**Observed in:** Playwright MCP — present on `browser_take_screenshot`, `browser_snapshot`,
`browser_console_messages`, `browser_evaluate`, `browser_pdf_save`, and the video tools.

**Apply when:** Tool can return >1KB of structured or binary output that the LLM might
not need to inspect in full.

**Reference shape:**
```
- filename (string, optional): File name to save the screenshot to. Defaults to
  `page-{timestamp}.{png|jpeg}` if not specified. Prefer relative file names to
  stay within the output directory.
```

**Counter-consideration:** Requires the MCP client to expose filesystem access to the
agent so it can read the saved file. Works best when paired with the filesystem MCP.

---

### P-20: Permission via human-readable element description

**Rule:** For tools that act on user-visible elements (DOM, GUI, files in a project),
require a `description` parameter alongside the machine-readable target. The
description is what the LLM types in natural language; the target is what the system
actually acts on.

**Solves:**
- Tool calls become **auditable in plain language** — a session log shows "Click the
  Submit button" not "Click [ref=node_47]"
- Acts as a **per-call consent prompt** — UIs can intercept and ask the user to confirm
  before each action
- Makes the LLM's intent inspectable

**Observed in:** Playwright MCP — almost every interaction tool has both `element`
(human description) and `target` (selector). Description quote: *"Human-readable element
description used to obtain permission to interact with the element."*

**Apply when:** Tool acts on something a human user could also act on (UI elements,
files in a workspace, records in a CRM). Especially valuable for high-trust environments.

**Counter-consideration:** Adds tokens to every tool call. Skip for high-frequency
internal operations.

---

## Antipatterns confirmed

### AP-03: God-tool with method enum — additional citation

`browser_tabs` accepts `action: enum[list, create, close, select]`. Same pattern Microsoft
called out in GitHub MCP. **But Playwright contains only one god-tool**, not pervasive
like GitHub. The skill should note this is a "one-off" failure mode and recommend splitting.

### AP-04: Raw code passthrough — additional citation

`browser_evaluate` accepts `function: () => { /* code */ }` as a string and executes
arbitrary JavaScript in the browser. Mitigated by:
- Honest README disclaimer that the server is "not a security boundary"
- Origin allowlists / blocklists
- Sandbox flags

Not bounded the way Postgres SQL is bounded by `READ ONLY` transaction. The LLM (or a
prompt injector) can do anything the browser can do.

---

## Sample evidence quotes

**Annotation-per-tool (P-08, D7):**
```
- browser_click
  - Title: Click
  - Description: Perform click on a web page
  - Parameters: ...
  - Read-only: false

- browser_console_messages
  - ...
  - Read-only: true
```

**Permission via description (P-20):**
> `element` (string, optional): **Human-readable element description used to obtain permission to interact with the element**
> `target` (string): Exact target element reference from the page snapshot, or a unique element selector

**Output redirection (P-19):**
> `filename` (string, optional): File name to save the screenshot to. Defaults to `page-{timestamp}.{png|jpeg}` if not specified.

**When-not-to-use guidance (D3):**
> `browser_take_screenshot`: Take a screenshot of the current page. **You can't perform actions based on the screenshot, use browser_snapshot for actions.**

**Honest security disclaimer (D9, D13):**
> Playwright MCP is **not** a security boundary. See [MCP Security Best Practices] for guidance on securing your deployment.

**Toolset gating (P-10):**
```
Configuration (opt-in via --caps=config)
Network (opt-in via --caps=network)
Coordinate-based (opt-in via --caps=vision)
PDF generation (opt-in via --caps=pdf)
Test assertions (opt-in via --caps=testing)
```

---

## Notes for the skill

When a buyer says *"I want Claude to drive a browser"* or *"I want Claude to do UI automation"*:

1. Recommend **per-tool `Read-only:` annotation in README** (P-08, Playwright canonical)
2. Recommend **toolset gating via --caps** (P-10) for capability-heavy servers
3. Recommend **`element` + `target` two-param pattern** (P-20) for any action-on-UI tool
4. Recommend **`filename` output redirection** (P-19) for any tool with potentially large output
5. Warn against the `browser_evaluate`-style raw-code passthrough unless the deployment
   has explicit sandbox/policy enforcement
6. Recommend **CLI + Skills over MCP for coding agents** if the use case is high-throughput
   developer work — Microsoft's own README explicitly says this. Honest meta-guidance.

---

## Cross-server position (updated, 8 entries)

| Server | Total | Best dim | Notable |
|---|---|---|---|
| Filesystem | 59 | annotations 5 | Reference standard, weak observability |
| Brodels | 56 | description 5, obs 5 | Multi-domain ops server |
| **Playwright** | **55** | **naming 5, annotations 5, docs 5** | **Browser automation archetype** |
| Memory | 54 | composability 5 | Batch-by-default + System Prompt |
| Fetch | 54 | pagination 5 | One-tool reference |
| GitHub | 49 | auth 5, safety 5 | God-tool dispatch hurts |
| Postgres | 41 | resources 5, safety 5 | Pre-modern reference |
| SQLite | 35 | resources 5 | Safety theater (security finding) |

The 54-56 cluster (Memory, Fetch, Playwright, Brodels) is where good-but-not-great
sits. Each has 1-2 obvious weakness dimensions that prevent breaking into the 59+ tier.
