# Corpus Entry: Brodels MCP

**Tier:** Community gold (worked example)
**Status:** Verified against live server (2026-05-14). All provisional scores resolved.
**Total score:** **56/70** — reference-quality with three specific weaknesses now confirmed.

---

## What it is

Brodels is a multi-domain MCP server that exposes:
- Jira ticket operations (fetch, search, comment, linked tickets)
- GitHub PR operations (create, comment, state, smoke-test, convert-to-draft)
- Git operations (diff, recent commits)
- Database introspection (describe, query, list sites — multi-tenant)
- Workspace management (create, list, cleanup — git worktree-based isolation)
- AI pipeline orchestration (the `brodels_run` phase pipeline)
- Knowledge base operations (query, learn, prune, lesson usage tracking)
- Operational utilities (cost tracking, logs, smoke testing, account unlock, skills sync)

~50 tools total across 8 functional domains.

## Why this is a useful corpus entry

- Demonstrates a **mature multi-domain server** — most MCPs in the wild are single-domain
- Shows how to handle **safety-critical operations** (prune with multi-gate guards,
  block-PR-creation, dry-run defaults)
- Strong **observability story** worth imitating
- Has visible weaknesses worth flagging (resources/tools split, pagination)

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 5/5   | Tools map 1:1 to user intents. Clear separation: `pr_create` vs `pr_state` vs `pr_comments`. No god-tools. |
| 2. Tool naming               | 4/5   | Excellent `brodels_<domain>_<action>` namespacing. Minor wobbles: `brodels_run` is a generic verb requiring description-read; `brodels_doctor` is ambiguous to a fresh LLM. |
| 3. Description quality       | 5/5   | Best-in-class. Descriptions include cost ceilings, variance warnings, scaling defaults, setup links, when-to-use guidance. See `brodels_pr_smoke_test` for the gold standard. |
| 4. Parameter design          | 4/5   | Most tools have 0–3 required params. Optional params have documented defaults. Some conditional requirements exist but are described well. |
| 5. Schema discipline         | 4/5   | Good use of enums (`phase`, `provisioner`, `which`, `category`). Numeric constraints with `exclusiveMinimum`/`maximum`. Some free-form strings where enums could tighten further (e.g., `mode` in `brodels_learn`). |
| 6. Error surface             | 4/5   | Content is high-quality. SQL typo returned `"Only read-only queries allowed (SELECT, SHOW, DESCRIBE, DESC, EXPLAIN). Got: SELCT"` — exact allowed forms + echo of received input. Fake Jira ID returned the upstream Jira 404 verbatim. Concern: errors surface as exceptions, not structured tool returns; LLM frameworks must propagate them cleanly. |
| 7. Idempotency signaling     | 4/5   | Clear verbs (`create`, `register`, `comment`, `unlock` for mutations; `fetch`, `query`, `search`, `state` for reads). Destructive ops named explicitly (`workspace_cleanup`, `prune_lessons`). |
| 8. Resources vs tools        | 2/5   | **Weakness — confirmed.** `list_resources` returned `"server brodels does not support resources"`. Static/reference data exposed as tools: `brodels_skills_list`, `brodels_db_sites`, `brodels_workspace_list`, `brodels_help`. All are MCP resource candidates. |
| 9. Auth model                | 4/5   | Per-developer identity via `brodels_register`. Personal anthropic + GitHub token override. Workspace-level isolation. Concern: shared-server-token fallback is a potential leak path; no explicit scope minimization in descriptions. |
| 10. Observability            | 5/5   | Best-in-class. `brodels_logs` (episodic, phase-filterable), `brodels_cost` (per-ticket + total), `brodels_lesson_usage` (KB ROI tracking), `brodels_rejection_learning` (failure pattern surfacing). Structured JSON/jsonl throughout. |
| 11. Composability            | 3/5   | **Weakness.** Identifier types ARE consistent (`ticketId`, `prNumber`). But return shapes are **prose strings**, not structured JSON. Every tool returns differently-formatted text — markdown headers (`brodels_help`), ASCII tables (`brodels_lesson_usage`), bullet lists (`brodels_db_sites`), key-value blocks (`brodels_whoami`), section dividers (`brodels_doctor`). The LLM must regex-parse to extract fields for downstream calls. See P-07 (prose-first design tradeoffs). |
| 12. Pagination/volume        | 3/5   | **Weakness.** Limit-style controls present (`limit`, `maxResults`, `lastN`, `topN`) with reasonable defaults, but no cursor-based pagination. Long lists can lose data without the LLM knowing. |
| 13. Safety rails             | 5/5   | Reference-quality. `prune_lessons` has triple safety: dry-run default, observation-floor gate, bullet-age gate. `block_pr_creation` is itself an escalation rail. `clear_policy_flag` requires `operator + reason` for audit. PR creation defaults to non-draft with safety-net `pr_convert_to_draft`. Smoke tests have cost ceilings with scaling defaults. |
| 14. Docs & onboarding        | 4/5   | Self-documenting via `brodels_help`, `brodels_doctor`, `brodels_whoami`, `brodels_skills_list`. Rich per-tool descriptions. Negative: 50-tool surface is intimidating; realistic onboarding is 30–60 min, not <10. |
| **Total**                    | **56/70** | Strong overall; three confirmed weaknesses (resources facility unused, prose-only outputs hurt composability, inconsistent pagination signals). |

---

## Patterns worth extracting

### Pattern: cost-aware tool descriptions

`brodels_pr_smoke_test` and `brodels_run` both publish their typical cost in the
description. This lets the LLM make a budget decision before invoking. Reference
quote (paraphrased): *"Each PR costs ~$0.01 in LLM tokens at sonnet-4 rates."*

**Generalizable rule:** for any tool with non-trivial cost (compute, money, or wall-clock),
publish the expected cost in the description.

### Pattern: scaled defaults

`brodels_pr_smoke_test` has `maxCost` and `maxTurns` defaults that scale with the
selected suite size: *"Defaults scale with suite size: Tier 1 only=$2, +1 Tier-2 suite=$2,
+full=$6."*

**Generalizable rule:** when a tool has wildly different cost profiles depending on inputs,
make defaults a function of inputs rather than a single constant.

### Pattern: multi-gate destructive operations

`brodels_prune_lessons` enforces three independent gates:
1. Dry-run is the default (`apply=false`)
2. Observation-floor gate (refuses to act if log history is too short)
3. Per-bullet age gate (`minBulletAgeDays`)

Even with `apply=true`, gates 2 and 3 still enforce.

**Generalizable rule:** for destructive ops, dry-run-default is not enough; layer
independent safety conditions so a single misconfigured flag can't cause harm.

### Pattern: audit-required policy clears

`brodels_clear_policy_flag` requires `operator` and `reason` as required params, and
appends a record to the episodic log. The audit trail is built into the tool contract.

**Generalizable rule:** for tools that override safety policies, require the *reason*
as a structured input, not optional metadata.

### Pattern: self-documenting fleet

`brodels_help`, `brodels_doctor`, `brodels_whoami`, `brodels_skills_list` are tools whose
purpose is to teach the LLM about the server. The LLM can self-orient on first contact.

**Generalizable rule:** every MCP server with >10 tools should include a help/discovery tool.

---

## Antipatterns worth extracting

### Antipattern: reference data exposed as tools

`brodels_skills_list`, `brodels_db_sites`, `brodels_workspace_list` return data that
changes rarely and is consulted often. Each call costs tokens (response payload) and
inference (decision to call). These are MCP resource candidates.

**Generalizable rule:** if a tool's output changes <1×/day and is read >5×/session,
it should be a resource, not a tool.

### Antipattern: offset/top-N pagination without cursors

`brodels_jira_search`, `brodels_logs`, `brodels_lesson_usage` all use limit-style
controls (`maxResults`, `lastN`, `topN`) without cursor continuation. The LLM cannot
know whether results were truncated or how to fetch the next page.

**Generalizable rule:** any list-returning tool needs (a) an explicit truncation signal
in the response when limits hit, and (b) a way to continue from where it stopped.

---

## Notes for the skill

When the skill generates an architecture for a complex multi-domain server, Brodels is the
best citable reference for:
- Namespacing convention (`<server>_<domain>_<action>`)
- Cost-aware descriptions
- Safety-rail layering
- Self-documenting fleet

Avoid imitating:
- The everything-is-a-tool pattern (use resources for reference data)
- Offset-only pagination (use cursors with truncation signals)

---

## Verification log (2026-05-14)

All TODOs resolved against live server.

- [x] **D6 error shapes** — Probed `brodels_db_query` with typo SQL and `brodels_jira_fetch` with fake ID. Error content is high-quality (exact allowed forms enumerated, echo of received input). Concern: surfaced as exceptions, not structured returns. **Provisional 3 → confirmed 4.**
- [x] **D11 return shapes** — Probed `brodels_help`, `brodels_whoami`, `brodels_skills_list`, `brodels_workspace_list`, `brodels_doctor`, `brodels_cost`, `brodels_lesson_usage`, `brodels_db_sites`, `brodels_jira_search`. All return prose strings in distinct formats. **Provisional 4 → confirmed 3.** Discovered prose-first design choice — promoted to P-07.
- [x] **D12 pagination** — `brodels_db_sites` returned `"... and 82 more (use filter to narrow)"` — strong truncation signal with remediation hint. `brodels_jira_search` returned `"2 result(s):"` with no signal of total. Same server, inconsistent pattern. **3/5 confirmed.**
- [x] **D8 resources facility** — `list_resources` returned `"server brodels does not support resources"`. **2/5 confirmed.** AP-01 fully validated.

## Sample evidence quotes (for skill output citations)

**Good error (D6):**
```
Only read-only queries allowed (SELECT, SHOW, DESCRIBE, DESC, EXPLAIN). Got: SELCT
```

**Good truncation signal (D12):**
```
132 site(s) matching "PROJ":
  ...
  ... and 82 more (use filter to narrow)
```

**Self-documenting tool (P-05):**
```
# Brodels MCP Tools (33)

## Core
- brodels_run — Run the Brodels AI pipeline on a Jira ticket
  brodels_run({ ticketId: "PROJ-1234" })
[...]
Use `brodels_help({ tool: "tool_name" })` for detailed usage.
```

**Real cost surface (P-01 evidence):**
- 28 tickets total cost: $34.97 → ~$1.25/ticket average
- Range: $0.14 (single-phase) to $8.50 (9-phase, ~9 phases at sonnet-4 rates)
- Smoke tests are tracked separately as `pr-{N}-smoke` entries
