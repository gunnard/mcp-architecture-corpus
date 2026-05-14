# Corpus Entry: GitHub MCP Server

**Tier:** Anthropic reference (now maintained by GitHub Inc.)
**Source:** `github.com/github/github-mcp-server`
**Status:** Scored from README. Live testing pending.
**Total score:** **49/70** — Solid but the god-tool dispatch pattern hurts it significantly.

---

## What it is

A large-surface MCP server exposing GitHub's API across ~12 toolsets: `context`,
`actions`, `code_security`, `copilot`, `dependabot`, `discussions`, `gists`, `git`,
`issues`, `labels`, `notifications`, `orgs`, `projects`, `pull_requests`, `repos`,
`secret_protection`, `security_advisories`, `stargazers`, `users`. Toolsets can be
enabled selectively via `--toolsets` flag or env var. Read-only mode and Lockdown mode
available at the server level.

## Why it scores lower than expected

The GitHub MCP server is built by GitHub Inc. with significant engineering resources,
yet it scores **10 points below** the much simpler Filesystem reference. The reason is
a single architectural choice: **god-tool dispatch**.

Tools like `actions_get`, `actions_list`, `actions_run_trigger`, `label_write` accept
a `method` enum parameter that internally dispatches to 5–8 different operations.
The LLM must:
1. Pick the right "outer" tool
2. Pick the right `method` value
3. Provide *conditionally required* params that depend on `method`

This is exactly AP-03 (god-tool with `action` enum) and AP-08 (conditional required params)
in production.

**Counterbalanced by**: best-in-class auth, safety modes, and docs — which is why the
score is 49 and not 35.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 3/5   | Toolset organization is excellent. But many tools use the `method: enum[...]` dispatch antipattern. `actions_get` takes method ∈ {`get_workflow`, `get_workflow_run`, `get_workflow_run_usage`, `get_workflow_run_logs_url`, `download_workflow_run_artifact`, `get_workflow_job`}. Compare to filesystem's clean split: `read_text_file` / `read_media_file` are separate tools. |
| 2. Tool naming               | 3/5   | Mixed. Clean: `get_label`, `list_label`, `dismiss_notification`, `list_notifications`. God-tools: `actions_get`, `label_write` hide their dispatch behind generic verbs. |
| 3. Description quality       | 3/5   | Atomic tools have good descriptions. God-tools have descriptions that ARE the dispatch table: *"Provide a workflow ID or workflow file name (e.g. ci.yaml) for 'get_workflow' method. Provide a workflow run ID for 'get_workflow_run'..."* — this is the antipattern in plain text. |
| 4. Parameter design          | 2/5   | **Confirmed AP-08.** God-tools have conditional required params: *"Required for 'get_workflow_run' method"*, *"Only used when method is 'list_workflow_jobs'"*. The schema says one thing; the prose says another. The LLM has to dispatch in its head. |
| 5. Schema discipline         | 3/5   | Types correct. Pagination constraints present (`per_page: max 100`). OAuth scopes documented per tool. But conditional-required pattern undermines schema rigor. |
| 6. Error surface             | 3/5   | Read-only mode returns errors when write tools requested. Lockdown mode returns errors for non-push-access content. Well-designed error paths. Couldn't verify full shapes from README. |
| 7. Idempotency signaling     | 4/5   | Most mutation tools clearly named (`actions_run_trigger`, `dismiss_notification`). Read-only mode is a server-level filter that removes write tools entirely. But god-tools (`label_write` with method=delete) hide destructive ops behind innocuous names. |
| 8. Resources vs tools        | 3/5   | README states toolsets include "MCP Resources and Prompts where applicable" — so the server DOES use the resources facility. Extent unclear without source inspection. Some reference data still exposed as tools (`list_label`). |
| 9. Auth model                | 5/5   | **Reference-quality.** OAuth scopes documented per tool. Token security best practices section. Multi-host support (Cloud, Enterprise Server, ghe.com data residency). Remote vs Local deployment. PAT and OAuth flows both supported. This is what enterprise-grade looks like. |
| 10. Observability            | 3/5   | Provisional. Can't verify from README alone. Conservative score given the scale of the product. |
| 11. Composability            | 3/5   | Identifier types (`owner`/`repo`/`issue_number`) are consistent across tools. But god-tools with `method` enums break clean composition — you can't pass tool A's output directly to tool B because tool B's required params shift based on the method picked. |
| 12. Pagination/volume        | 4/5   | `page` and `per_page` consistently used. Max 100, default 30. Standard GitHub pagination semantics. No cursor (but GitHub's API itself uses page-based; this matches upstream). |
| 13. Safety rails             | 5/5   | **Reference-quality.** `--read-only` mode at server level filters out write tools entirely. `--lockdown-mode` filters non-push-access content from public repos. Both can be combined with toolset gating. Specific tools listed for each mode's effect. Defense in depth. |
| 14. Docs & onboarding        | 5/5   | Exhaustive README: remote vs local install, multiple MCP host configs (VS Code, Copilot, others), Docker, build-from-source, toolset configuration, security modes, dynamic discovery, i18n/description overrides. New developer can choose their deployment path in <10 min. |
| **Total**                    | **49/70** | Solid. God-tool pattern is the main drag; auth and safety lift it back up. |

---

## Patterns extracted (new from this entry)

### P-10: Toolset gating

**Rule:** For large-surface servers, group tools into named toolsets and let users enable
only what they need via flag/env var. Default to a "context" toolset that gives basic
orientation.

**Observed:** GitHub MCP `--toolsets repos,issues,pull_requests` exposes only those
groups. Each toolset can include tools, resources, AND prompts.

**Why it matters:**
- Reduces context cost for the LLM (fewer tool definitions in the prompt)
- Reduces tool-picking error rate (smaller decision space)
- Allows policy-based deployment (a read-only deployment can disable `actions` toolset)

### P-11: Dynamic tool discovery (beta pattern)

**Rule:** Instead of exposing all tools at startup, expose a discovery mechanism (`list_available_toolsets`, `enable_toolset`, `get_toolset_tools`) that the LLM can use
to load more tools when the user's request requires them.

**Observed:** GitHub MCP `--dynamic-toolsets` flag.

**Why it matters:** Solves the "100+ tools confuses the model" problem by letting the
LLM negotiate its own tool surface. Beta — watch for adoption.

**Counter-consideration:** Adds round-trips and complexity. Only worth it past ~30 tools.

### P-12: Server-level policy modes (read-only, lockdown)

**Rule:** Offer global modes that filter the available tool surface based on
deployment policy. Modes compose with toolset gating.

**Observed:** GitHub MCP `--read-only`, `--lockdown-mode`.

**Why it matters:** Lets ops/security teams deploy a "safe by default" version of the
same binary without re-implementing tool definitions.

---

## Antipatterns confirmed (with strong evidence)

### AP-03: God-tool with method enum — CONFIRMED

`actions_get`, `actions_list`, `actions_run_trigger`, `label_write` all dispatch via
`method` parameter. The README description for each is *the dispatch table in prose*.

**Better design (counter-example):** Split into atomic tools.
- `actions_get_workflow`, `actions_get_workflow_run`, `actions_get_workflow_run_usage`, etc.
- Or follow the `notifications` pattern in the same server: `list_notifications`,
  `dismiss_notification`, `manage_notification_subscription` are properly split.

The fact that **the same server has both patterns** (notifications done well, actions
done poorly) is itself the evidence — it's not a technical constraint, it's a choice
worth questioning.

### AP-08: Conditional required params — CONFIRMED

From the `actions_run_trigger` description:
> `ref`: The git reference for the workflow. The reference can be a branch or tag name. **Required for 'run_workflow' method.** (string, optional)
> `run_id`: The ID of the workflow run. **Required for all methods except 'run_workflow'.** (number, optional)

The schema says both are optional. The prose says they are conditionally required based
on `method`. The LLM has to do this dispatch in-context.

---

## Sample evidence quotes

**God-tool dispatch (AP-03):**
```
actions_run_trigger — Trigger GitHub Actions workflow actions
  - method: The method to execute (string, required)
  - run_id: The ID of the workflow run. Required for all methods except 'run_workflow'. (number, optional)
  - workflow_id: The workflow ID (numeric) or workflow file name. Required for 'run_workflow' method. (string, optional)
```

**Read-only mode (D13):**
```bash
./github-mcp-server --read-only
```
*"This will only offer read-only tools, preventing any modifications to repositories,
issues, pull requests, etc."*

**Lockdown mode (D13):**
*"Lockdown mode limits the content that the server will surface from public repositories.
When enabled, the server checks whether the author of each item has push access to the
repository."*

**Toolset gating (P-10):**
```bash
GITHUB_TOOLSETS="repos,issues,pull_requests,actions,code_security" ./github-mcp-server
```

---

## Comparison: Filesystem vs GitHub vs Brodels

| Dimension | Filesystem | GitHub | Brodels |
|---|---|---|---|
| Tool granularity | **5** | 3 | 5 |
| Tool naming | **5** | 3 | 4 |
| Description quality | 4 | 3 | **5** |
| Parameter design | **5** | 2 | 4 |
| Schema discipline | **5** | 3 | 4 |
| Error surface | 4 | 3 | 4 |
| Idempotency | **5** | 4 | 4 |
| Resources vs tools | 3 | 3 | 2 |
| Auth model | 4 | **5** | 4 |
| Observability | 2 | 3 | **5** |
| Composability | 4 | 3 | 3 |
| Pagination | 3 | **4** | 3 |
| Safety rails | **5** | **5** | **5** |
| Docs/onboarding | **5** | **5** | 4 |
| **Total** | **59** | **49** | **56** |

**Cross-server takeaway:**
- The simplest server (Filesystem) scores highest because its discipline is uniform
- The largest enterprise server (GitHub) scores lower than the smaller community server
  (Brodels) because of the god-tool pattern
- Brodels wins observability by a wide margin; the skill should cite this for ops-heavy
  domains
- All three score 5/5 on safety rails — the bar for production MCP servers is high here
- Resources facility is underused everywhere (max score: 3) — there's an opening for
  servers that use it well
