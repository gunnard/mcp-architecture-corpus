# Tier-3 Counter-Examples: Shell-Wrapper MCPs

**Tier-3 status:** Per `_working-list.md`, Tier-3 entries are not scored against the full
14-dimension rubric. They exist to **anchor antipatterns with real, citable code** so the
skill's negative recommendations have evidence beyond a single offender.

This file captures three points on the AP-03 (god-tool) spectrum, all with the SAME tool
shape (one tool, unbounded `cmd`/`code` input) but DIFFERENT mitigation stacks. The
contrast is the lesson — see `_antipatterns.md` AP-03 for the comparison table.

---

## 1. `patrickomatik/mcp-bash` — unmitigated AP-03

**Source:** https://github.com/patrickomatik/mcp-bash
**Tools:** `set_cwd(path)`, `execute_bash(cmd)`
**Sandbox:** None. Runs in the user's shell with user privileges.
**Documented mitigations:** None. README explicitly delegates: *"Consider: running in a
container or restricted environment, adding command validation or allowlists, limiting
filesystem access with appropriate user permissions."*

**Author's own framing (verbatim from README):**

> *"This project allows you to execute bash commands via an MCP server interface, with all
> the potential security risks that entails!"*

> *"There's nothing stopping the LLM from running dangerous commands like `rm -rf /`. For
> personal use, the usefulness may outweigh the risks, but take care when deploying."*

**Why we cite it:** This is the **honest unmitigated** version of AP-03. The author is
not pretending the server is safe — they explicitly tell you it is not. That makes it the
ideal corpus reference for "what AP-03 looks like when nothing is done about it." It is
NOT a counter-example for the skill to recommend against shipping; it is the **before**
picture in a before/after sequence.

**Use this entry to teach:** the antipattern shape itself, and the question "what is the
minimum mitigation stack required to make this deployable?"

---

## 2. `MladenSU/cli-mcp-server` — mitigated AP-03 (same shape, deployable)

**Source:** https://github.com/MladenSU/cli-mcp-server
**Tools:** `run_command(...)`, `show_security_rules`
**Sandbox:** None at the runtime level — but defense-in-depth at the input level.
**Documented mitigations** (all from `Configuration` table in README):

| Variable | Default | What it prevents |
|---|---|---|
| `ALLOWED_DIR` | (required, no default) | Path-traversal attacks; commands cannot run outside this base directory |
| `ALLOWED_COMMANDS` | `ls,cat,pwd` | Arbitrary binary execution; locked to read-only commands by default |
| `ALLOWED_FLAGS` | `-l,-a,--help` | Flag-injection attacks; `--exec`-style flags blocked unless explicitly listed |
| `ALLOW_SHELL_OPERATORS` | `false` | Shell operator injection — `;`, `&&`, `\|\|`, `\|`, `>`, `<` cannot chain or redirect |
| `MAX_COMMAND_LENGTH` | `1024` | Prevents buffer-style attacks and runaway argument injection |
| `COMMAND_TIMEOUT` | `30s` | Caps blast radius of any single mistake |

**Why we cite it:** Same tool shape as mcp-bash, but the allowlist + shell-operator block
flip it from "lethal in deployment" to "deployable with thought." This is the **mitigated**
version of AP-03 — useful when teaching buyers that AP-03 is not always wrong, just usually
expensive to do safely.

**Use this entry to teach:** the minimum mitigation stack for shell-wrapper MCPs. If a
buyer is going to ship a CLI exposure, this is roughly the floor.

---

## 3. Cloudflare MCP — sanctioned AP-03 (sandboxed runtime, P-29)

**Source:** https://github.com/cloudflare/mcp
**Scoring entry:** `corpus/community/cloudflare.md` (full rubric, **57/70**).
**Tools:** `search(code)`, `execute(code)`
**Sandbox:** **V8 isolates via Cloudflare's Dynamic Worker Loader API** — real isolation
boundary, no filesystem, scoped API binding only.

**Why this is in Tier-3 cross-references** (not just Tier-2): because it scores in full
elsewhere, but its design *shares the tool shape* of the two above and demonstrates the
upper end of the mitigation spectrum.

**Why we cite it:** Cloudflare is the case study in "AP-03 is the right answer when the
upstream API has 2,594 endpoints and the alternative blows the context window." See P-29
(Code Mode Tool) in `_patterns.md` for the full pattern definition.

**Use this entry to teach:** when AP-03 is **promoted to a pattern** (P-29) — i.e., when
upstream scale forces the design AND the platform provides a real sandbox AND the auth
inheritance is robust.

---

## Cross-reference summary

The same surface area (one tool, unbounded code/command input) sits in three different
places on the safety spectrum. **The corpus's job is to teach the spectrum, not to
condemn the shape.**

| Server | Sandbox | Allowlist | Auth scope | Verdict |
|---|---|---|---|---|
| mcp-bash | none | none | user shell | **Unmitigated AP-03** — lethal in deployment, honest about it |
| cli-mcp-server | none | yes (commands+flags+operators+path) | filesystem perms | **Mitigated AP-03** — deployable with thought |
| Cloudflare MCP | V8 isolate | n/a (scoped binding) | OAuth + Cloudflare token scopes | **Sanctioned P-29** — legitimate exception |

When a buyer says *"can I just give Claude bash access?"* the right answer is **never the
mcp-bash design**, often the cli-mcp-server design, and rarely the P-29 design (only when
upstream scale forces it).

---

## Why no full rubric scoring for these?

Per `_working-list.md` Tier-3 process: *"Don't score these in full — just capture the
specific antipattern they illustrate."* These entries earn their place by anchoring AP-03
with three real, citable code points across the mitigation spectrum. Scoring them on all
14 dimensions would dilute the rubric's discrimination — most dimensions are uninteresting
for a 2-tool shell wrapper and would cluster around 2/5 to 3/5 in ways that don't teach.
