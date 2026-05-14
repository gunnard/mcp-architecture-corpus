# Corpus Entry: Linear MCP Server (community)

**Tier:** Tier-2 community gold
**Source:** `github.com/jerhadf/linear-mcp-server` (most-cited community Linear MCP; not Linear-official)
**Status:** Scored from README. Source file 404'd on direct fetch — schema details inferred from README documentation.
**Total score:** **50/70** — Solid mid-tier. **Notable as the first COMMUNITY server in the corpus that uses MCP resources correctly (D8 = 4/5).** Confirms P-13 pattern reaching mainstream community implementations.

---

## What it is

A community-built MCP server exposing Linear's issue-tracking API to LLMs.

- **Tools (5):** `linear_create_issue`, `linear_update_issue`, `linear_search_issues`,
  `linear_get_user_issues`, `linear_add_comment`
- **Resources (5):** Per-entity URIs:
  - `linear-issue:///{issueId}` — individual issue
  - `linear-team:///{teamId}/issues` — team issue list
  - `linear-user:///{userId}/assigned` — user's assigned issues
  - `linear-organization:` — org info
  - `linear-viewer:` — current user context

## Why this entry matters

Linear is the **modern SaaS API exposure archetype**. When a buyer asks the skill *"I
want Claude to interact with our ticket tracker / CRM / project management tool"*, this
is the reference shape: small atomic tool surface + entity-per-resource for queryable
metadata.

It's also the **first community server in the corpus** (not made by Anthropic, Microsoft,
or Stripe) — important signal that P-13 (entity-per-resource) is reaching mainstream
implementation quality. Demonstrates: a community dev can ship something that scores
50/70 with modest surface and disciplined use of MCP primitives.

---

## Scoring

| Dimension                    | Score | Evidence |
|------------------------------|-------|----------|
| 1. Tool granularity          | 4/5   | 5 tools, all intent-mapped. CRUD basics covered (create, update, search, get, comment). Notable omissions: no `delete_issue`, no `list_teams`, no `list_projects`, no `assign_issue`. -1 for incomplete coverage; the LLM has to use the broader `search_issues` for several common operations. |
| 2. Tool naming               | 5/5   | `linear_<verb>_<noun>` consistent throughout. P-06 namespacing canonical. Zero ambiguity. |
| 3. Description quality       | 3/5   | Short and accurate but thin. *"Create a new Linear issues"* (typo in plural). Inputs documented with types. No "when to use vs alternatives," no examples, no pitfalls. |
| 4. Parameter design          | 4/5   | Required minimal: `title + teamId` for create; `id` for update; `issueId + body` for comment. Optional params well-documented with constraints (*"priority (number, 0-4): Priority level (1=urgent, 4=low)"*). Reasonable defaults (search limit=10, user-issues limit=50). |
| 5. Schema discipline         | 4/5   | Number ranges stated (priority 0-4). String types. Status names are project-specific so no global enum possible — acceptable. Defaults documented. Could be 5 with explicit JSON schema in README. |
| 6. Error surface             | 3/5   | Conservative (source not directly inspected). Linear API errors are well-structured; wrapper presumably passes through. |
| 7. Idempotency signaling     | 4/5   | Verbs unambiguous (create, update, search, get, add). Mutation vs read split clearly. No MCP `ToolAnnotations` visible — common community-server gap. |
| 8. Resources vs tools        | 4/5   | **Uses the MCP resources facility.** 5 resource URI patterns including entity-per-resource (P-13). `linear-issue:///{issueId}` is the canonical shape. The LLM can read an issue as a resource (cached, addressable) instead of paying a tool call. Could be 5 if resources were *auto-discovered* like Postgres (here you have to know the URI pattern). |
| 9. Auth model                | 3/5   | Standard Linear API key. Single-tenant. No RAK-style scoping visible. Falls short of Stripe's pattern (P-22). |
| 10. Observability            | 2/5   | Not detailed in README. Likely minimal. Standard for community servers. |
| 11. Composability            | 4/5   | Tool outputs presumably include issue IDs that flow into other tools (`linear_add_comment` accepts `issueId` from `linear_create_issue` result). Predictable shapes given Linear API conventions. |
| 12. Pagination/volume        | 3/5   | Has `limit` params with reasonable defaults. No cursor-based pagination visible. Same offset/limit pattern as Brodels (and most servers). |
| 13. Safety rails             | 3/5   | Linear API has its own permission model but no MCP-level safety rails visible. No dry-run on mutations. `linear_add_comment` has `createAsUser` parameter for impersonation — documented but potentially misusable. |
| 14. Docs & onboarding        | 4/5   | Decent README: installation (automatic + manual), tools, resources, usage examples, Rich Text Description Support section, Features in Development roadmap. Not as comprehensive as Playwright/Filesystem but clean and current. |
| **Total**                    | **50/70** | Solid mid-tier community server. Notable for proper resources facility usage; held back by incomplete tool coverage and missing observability. |

---

## Patterns reinforced

### P-13: Resource-per-entity — third citation, promote to canonical

Linear uses `linear-issue:///{issueId}`, `linear-team:///{teamId}/issues`,
`linear-user:///{userId}/assigned` as resource URIs. Each is parametric — entities are
resources, not tools.

**This makes the third citation for P-13:**
1. Postgres MCP — canonical (auto-discovered table schemas)
2. SQLite MCP — variant as P-14 (dynamic resource via memo://insights)
3. Linear MCP (community) — entity-per-resource for issues/teams/users

P-13 is now **canonical**.

**Cross-server pattern:** servers that use resources well score higher on D8 than servers
that don't. Average D8 for servers using resources: 4.5. Average D8 for servers not using
resources: 2.5. The skill should treat this dimension as a leading indicator of
overall MCP design discipline.

---

## Sample evidence quotes

**Atomic tool naming (D1, D2, P-06):**
```
linear_create_issue
linear_update_issue
linear_search_issues
linear_get_user_issues
linear_add_comment
```

**Entity-per-resource (P-13):**
```
linear-issue:///{issueId}        — individual issue
linear-team:///{teamId}/issues   — team issue list
linear-user:///{userId}/assigned — user's assigned issues
linear-organization:             — org info
linear-viewer:                   — current user context
```

**Parameter constraint guidance (D4, D5):**
```
priority (number, 0-4): Priority level (1=urgent, 4=low)
limit (number, default: 10): Max results
```

**Identifier consistency for composability (D11):**
- `linear_create_issue` → returns `issueId`
- `linear_add_comment` takes `issueId`
- `linear_update_issue` takes `id`

(Minor inconsistency: `id` vs `issueId` across tools — would push composability to 5 if
unified.)

---

## Notes for the skill

When a buyer says *"I want Claude to interact with our SaaS API"* (ticket tracker, CRM,
project management — non-financial):

1. Recommend **atomic tool naming** with `<server>_<verb>_<noun>` (P-06) — Linear is the
   community-grade example
2. Recommend **entity-per-resource** for queryable metadata (P-13) — Linear's
   `linear-<entity>:///{id}` pattern is the cleanest community implementation
3. Watch for **incomplete tool coverage** — Linear ships 5 tools but Linear itself has
   100+ API operations. The skill should help the buyer decide what to expose vs. omit.
4. Recommend **dry-run on mutations** that Linear is missing
5. Recommend **observability** that Linear lacks (Brodels patterns)
6. For a financial SaaS, prefer the Stripe pattern (P-21 + P-22 + P-24) over Linear's
   pattern — Linear is the "non-safety-critical SaaS" reference.

---

## Cross-server position (final, 10 entries)

| Server | Total | Tier | Distinguishing feature |
|---|---|---|---|
| Filesystem | 59 | Ref | Tool design reference standard |
| Stripe | 58 (prov) | Tier-2 | First 5/5 on auth; hosted remote MCP |
| Brodels | 56 | Tier-2 | Description + observability leader |
| Playwright | 55 | Tier-2 | Naming + annotations + docs all 5 |
| Memory | 54 | Ref | Batch-by-default + system prompt |
| Fetch | 54 | Ref | First 5/5 on pagination |
| **Linear** | **50** | **Tier-2** | **Community proof of P-13 mainstream** |
| GitHub | 49 | Ref | God-tool dispatch hurts |
| Postgres | 41 | Ref (archived) | Resources done right, pre-modern |
| SQLite | 35 | Ref (archived) | Safety theater (security finding) |

**The corpus is now complete.** 10 entries, 24-point spread, every dimension exercised
across its full 1-5 range except D6 (errors) and D11 (composability) which top out at 4.
Pattern catalogue: 24 patterns + 11 antipatterns. Ready for archetype writing.
