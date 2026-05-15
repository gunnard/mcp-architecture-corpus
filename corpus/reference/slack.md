# Slack MCP (archived Anthropic reference)

**Source:** https://github.com/modelcontextprotocol/servers-archived/tree/main/src/slack
**Status flag:** **archived** — moved to `servers-archived` repo, no longer maintained by the MCP team. Still influential as an early reference.
**Vendor:** Anthropic (formerly part of `modelcontextprotocol/servers`).
**Archetype fit:** **02 — Writable System** (chat workspace messaging). Some read-mostly aspects (channel history, user profiles) overlap with archetype 01.
**Status:** Source verified from the archived repo's `README.md` (full tool list with input/output schemas). Live invocation NOT performed.
**Total score: 45/70** — bottom of the production-ready band, third-lowest in the corpus. Useful as a baseline for "what an early Anthropic reference looked like before the modern toolkit (annotations, resources, OAuth)."

---

## What it is

8-tool Slack workspace integration. Bot-token authentication via Slack's standard OAuth scope model, narrow read+write surface (no admin operations), stdio transport via `npx -y @modelcontextprotocol/server-slack`. Configurable per-server channel whitelist via `SLACK_CHANNEL_IDS` env var.

Notable for being **the first clean cursor-pagination example in the corpus** — fills a long-standing gap flagged in `_patterns.md` ("Still need a clean cursor-pagination example for structured paginated lists").

## Tool surface (8 tools)

Read-mostly:

- `slack_list_channels` — `limit` (default 100, max 200) + `cursor` pagination
- `slack_get_channel_history` — `channel_id`, `limit` (default 10)
- `slack_get_thread_replies` — `channel_id`, `thread_ts`
- `slack_get_users` — `cursor` + `limit` pagination
- `slack_get_user_profile` — `user_id`

Write (narrow):

- `slack_post_message` — `channel_id`, `text`
- `slack_reply_to_thread` — `channel_id`, `thread_ts`, `text`
- `slack_add_reaction` — `channel_id`, `timestamp`, `reaction`

**Conspicuously absent:** no channel creation/archiving, no message edit/delete, no user invite, no DM tools, no file uploads, no admin operations. The surface is deliberately narrow — agents can read public channels and post/react, nothing else.

## Why this entry matters

Three reasons:

1. **Last Tier-1 "must" entry scored.** The working-list flagged Slack as required for the comparison baseline against modern community implementations.
2. **First cursor-pagination example.** `slack_list_channels` and `slack_get_users` both expose `cursor` + `limit`, which is the canonical pattern for paginated structured lists. The corpus previously only had P-17 (text chunking via `start_index` + `max_length` from Fetch). This unblocks promotion to a numbered pattern when a second example surfaces (likely Linear or any modern SaaS MCP).
3. **Generational baseline.** Slack predates `ToolAnnotations`, `structuredContent`, MCP `Resources`, OAuth-based remote MCPs, and per-tool scope gating. Scoring it allows the skill to show the buyer "here's where MCP was, here's where it's gone, here's what's expected of new servers in 2025+."

---

## Scoring

| Dimension | Score | Confidence | Evidence |
|---|---|---|---|
| 1. Tool granularity | 5/5 | High | 8 atomic, intent-mapped, verb_object tools. No god-tools, no method-enum dispatchers. Reference-quality. |
| 2. Tool naming | 4/5 | High | All `slack_*` prefixed (P-06). Verb_object after prefix. -1 because the prefix is redundant *within* the Slack server (already namespaced by server name) — the prefix only buys you something at compose-time when a host runs multiple MCP servers. Modern servers like Sentry use `verb_object` directly without the namespace prefix. |
| 3. Description quality | 3/5 | High | README "Required inputs / Optional inputs / Returns" structure is clear, but per-tool descriptions don't explain when-to-use vs. siblings. No examples block, no hints, no cross-tool routing guidance. Compared to Sentry's `<examples>` + `<hints>` format (5/5), this is functional but bare. |
| 4. Parameter design | 4/5 | High | Sensible defaults (`limit: 100`, `max: 200`), required vs optional clearly marked, cursor pagination on list ops. -1 because no `min`/`max` on `limit` in the schema (relies on Slack API to enforce). |
| 5. Schema discipline | 4/5 | Provisional | TypeScript codebase, JSON Schema for tool inputs. Need source check to verify enum usage on emoji names, etc. Projecting 4. |
| 6. Error surface | 3/5 | Provisional | Not discussed in README. Likely propagates raw Slack API errors. No evidence of error wrapping or remediation hints. Projecting 3. |
| 7. ToolAnnotations | 2/5 | High | **Predates the ToolAnnotations standard.** No `readOnlyHint` / `destructiveHint` / `openWorldHint` on any tool. The write tools (`slack_post_message`, `slack_reply_to_thread`, `slack_add_reaction`) need `destructiveHint: true` to be safe in modern agent workflows. |
| 8. Resources vs tools | 2/5 | High | No MCP `Resources` exposed. Channel histories, user profiles, message threads are all natural resource candidates (stable URIs, read-mostly content). Tools-only design. -3 vs. Postgres/SQLite. |
| 9. Auth model | 3/5 | High | Bot token (xoxb-) via env var. No OAuth (despite Slack having OAuth!), no remote MCP, no per-tool scope gating in the MCP definition (scopes are configured at the Slack-app level, not surfaced to the LLM). Workable but well below Stripe/Sentry/Cloudflare's 5/5. |
| 10. Observability | 2/5 | Provisional | No evidence of telemetry instrumentation. Archived state suggests no plans to add any. Projecting 2. |
| 11. Composability | 4/5 | High | `channel_id` from `list_channels` flows to `post_message`, `get_channel_history`, `add_reaction`. `thread_ts` flows from `get_channel_history` to `get_thread_replies` and `reply_to_thread`. ID-driven composition works clean. -1 because no resource-URI composition (everything is plain string IDs). |
| 12. Pagination / volume | 4/5 | High | **First clean cursor-pagination example in corpus.** `cursor` + `limit` (default 100, max 200) on `slack_list_channels` and `slack_get_users` — exactly the cursor-token pattern from the Slack Web API. -1 because the response shape isn't documented in the README; the LLM has to discover whether a `next_cursor` was returned by trial and error. A 5/5 here would explicitly tell the LLM "if `response_metadata.next_cursor` is non-empty, more pages exist." |
| 13. Safety rails | 3/5 | High | `SLACK_CHANNEL_IDS` env var for channel whitelist (interesting per-server scoping, predates P-12 server policy modes). Bot scopes minimal (no destructive perms in defaults). BUT: no ToolAnnotations means agent has no in-band signal that `slack_post_message` is destructive, no policy modes, no human-in-the-loop framework. |
| 14. Docs & onboarding | 3/5 | High | README covers setup adequately (Slack App creation, scopes, env vars). Has Claude Desktop AND VS Code integration examples. -2 because no troubleshooting depth, no observability/log guidance, no migration notes ("here's what you should do instead now that this is archived"). |
| **Total** | **45/70** | — | Sits below GitHub (49). 4 dimensions provisional; worst case 41 if all drop a point, best case 49 if all land at 4. |

---

## Patterns confirmed (existing patterns this entry strengthens)

- **P-06 (Namespacing convention)** — `slack_*` prefix on all 8 tools. Second clear example after the modular reference servers. Worth flagging as "good when host runs multiple MCP servers, redundant within a single server."

## Patterns this entry contributes evidence toward (not yet promoted)

- **Cursor pagination for structured lists** — the working-list pending item. Slack provides ONE clean example (`cursor` + `limit` on list ops). Not yet a pattern (need a 2nd example), but unblocks promotion when Linear or another modern SaaS MCP confirms.

## Antipatterns observed

None outright. Closest is **the absence of `ToolAnnotations`** on write tools — this would be AP-09 territory ("destructive operations not annotated") if AP-09 existed. It's a generational miss, not a design failure.

## What modern servers do differently

For the skill: when a buyer compares "old Slack MCP" to "new Sentry MCP" the deltas are:

| Concern | Slack (2024-era reference) | Sentry (2025-era community) |
|---|---|---|
| Auth | Bot token via env | OAuth + per-tool `requiredScopes` + skill bundles |
| Annotations | None | `{ readOnlyHint, destructiveHint, openWorldHint }` per tool |
| Resources | None | `get_sentry_resource` polymorphic resolver (P-26) |
| Description format | Plain prose | `<examples>` + `<hints>` blocks (5/5 quality) |
| Observability | None | `import { getActiveSpan, setTag } from "@sentry/core"` |
| Policy modes | `SLACK_CHANNEL_IDS` env whitelist | `agentOnly: true`, `experimentalMode` |
| Self-discovery | None | `search_docs` + `get_doc` as tools |

This delta IS the upgrade path the skill should describe to buyers writing new MCP servers in 2025+.

---

## Notes for the skill

When a buyer says *"I want to do what the Slack MCP does for our team chat platform"* (Discord, Teams, Mattermost, Zulip, Rocket.Chat):

1. **Start from cursor pagination as the baseline** — Slack got this right. Use `cursor` + `limit` with default 100, max 200.
2. **Upgrade to OAuth, not bot tokens** — Slack's pattern was acceptable in 2024 but is below modern (P-21).
3. **Add `ToolAnnotations` to every write tool** — `slack_post_message` should have `destructiveHint: true`. Slack's miss is the canonical generational gap.
4. **Expose channel history as resources, not just tool calls** — channel history has stable URIs and is read-mostly; that's a textbook MCP `Resource` (P-13).
5. **Replicate `SLACK_CHANNEL_IDS` as a server policy mode (P-12)** — env-var whitelisting for tenant scoping is a great safety feature; promote it to the policy-mode pattern.
6. **Don't include destructive admin tools.** Slack's narrow surface (no archive, no edit, no delete) is correct — destructive admin operations should stay in the dashboard, not the LLM.

---

## Cross-server position (12 entries with Slack added; archived flag separate)

| Server | Total | Note |
|---|---|---|
| Sentry | 65 | Highest in corpus. P-26, P-27 origin. |
| Filesystem | 59 | Reference standard for read-only data sources. |
| Stripe | 58 prov. | First 5/5 on auth (D9). |
| Cloudflare | 57 | P-29 origin (Code Mode). Case study in deliberate AP-03 violation. |
| Brodels | 56 | Description + observability leader. |
| Playwright | 55 | Naming + annotations + docs all 5/5. |
| Memory | 54 | Batch-by-default, system prompt. |
| Fetch | 54 | One-tool reference; first 5/5 on text pagination. |
| Linear | 50 | Modern SaaS API. |
| GitHub | 49 | God-tool dispatch hurts. |
| **Slack** | **45** | **First cursor-pagination example. Generational baseline (no annotations, no resources, no OAuth).** |
| Postgres | 41 | Pre-modern, resources done right. |
| SQLite | 35 | Safety theater — counter-example. |

**Score distribution:** 65, 59, 58, 57, 56, 55, 54, 54, 50, 49, 45, 41, 35 — 30-point range, no gaps wider than 5 points, healthy discrimination.
