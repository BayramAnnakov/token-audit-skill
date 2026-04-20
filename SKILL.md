---
name: token-audit
description: "Audit Claude Code token usage and find where tokens are leaking. Reads ~/.claude/projects/*.jsonl transcripts locally, inventories settings/hooks/skills/MCPs, runs ccusage for baseline spend, then flags the highest-$-impact leaks (hook bloat, CLAUDE.md bloat, Opus on simple turns, context rot, skill description overhead, Bash anti-patterns, cache miss storms, redundant file reads) with ballpark weekly savings for each. All analysis is local — transcripts never leave the machine. Use when the user asks to audit their Claude Code usage, find where tokens are leaking, review token spend, optimize Claude Code, or understand why they're hitting weekly limits. Triggers: 'token audit', 'why am I hitting limits', 'optimize my Claude Code', 'token leak', 'audit my usage', 'where are my tokens going'."
---

# Token Audit

Find where Claude Code tokens are leaking — and what to fix first.

## When to use this skill

- User asks to audit their token usage or Claude Code costs
- User hitting weekly plan limits (Pro / Max 5x / Max 20x)
- User wonders why a task burned more tokens than expected
- Before/after installing lots of MCPs, plugins, or skills (drift check)
- As a periodic self-review (weekly or bi-weekly)

## What it does

Runs a local-only audit with three inputs:
1. **ccusage** (external CLI, 13k★ open source) — baseline $ totals per day/model/session
2. **JSONL transcripts** at `~/.claude/projects/**/*.jsonl` — per-turn analysis for patterns ccusage doesn't see
3. **Config inventory** — `~/.claude/settings.json`, plugins, skills, MCP servers

Then produces a ranked report of leaks with ballpark weekly $ savings for each.

## How to run it

```bash
python SKILL_DIR/scripts/audit.py --days 7
```

Outputs JSON to stdout. You (Claude) then synthesize a narrative report in the user's language — default English, switch to match the user's current conversation language.

### Language handling

- Default: English
- If the user's recent messages are in another language, write the report in that language
- Technical terms (ccusage, `/compact`, `/rewind`, MCP, CLAUDE.md) stay untranslated

### Report structure (adapt tone, keep sections)

1. **Spend summary** — total, by model, by project, trend vs prior week (from ccusage)
2. **Top leaks ranked by weekly savings** — each with:
   - Severity badge (🔴 critical / 🟡 warning / 🟢 suggestion)
   - Evidence (3-5 bullets with numbers)
   - Estimated weekly cost + savings
   - Concrete fix action
   - Thariq citation where applicable
3. **One fix to apply this week** — single highest-leverage action
4. **Trend / context** — plan-fee share, savings as % of subscription week

## Leak detectors (v1)

| Detector | What it catches | Source |
|---|---|---|
| `tool_schema` | Full MCP tool schemas loaded every turn (~20k tok) — only fires if `ENABLE_TOOL_SEARCH` is OFF | Samarth Gupta audit |
| `hook_bloat` | Session-start / PreCompact hooks re-injecting large output into every session | Novel |
| `claude_md_bloat` | CLAUDE.md > 2k tokens, paid on every turn | Anthropic cost doc (200-line target) |
| `model_selection` | Opus used on short/simple turns where Sonnet would suffice | Novel |
| `context` | Turns past 400k-token context (context rot zone) | Thariq Shihipar, Anthropic |
| `skill_descriptions` | Total skill-description budget per turn (flags fat individual skills + bloated totals) | Novel |
| `bash_antipatterns` | `cat`/`head`/`tail`/`find`/`grep` via Bash instead of native Read/Glob/Grep | Samarth Gupta audit |
| `cache` | Sessions with <50% cache hit ratio (churn-driven cache misses) | Novel |
| `file_reads` | Same file Read 3+ times in one session (rewind candidate) | Thariq Shihipar, Anthropic |

## Prerequisites

- **Node.js 20+** for ccusage (auto-fetched via `npx` — no install needed)
- **Python 3.11+** for the analyzer
- Read access to `~/.claude/projects/` and `~/.claude/settings.json` (already yours)

If ccusage is unavailable, the audit proceeds without baseline $ totals — detectors still work from JSONLs alone.

## Estimated savings — how to frame them

The `$ saved per week` numbers are **ballparks**, not invoices. They assume:
- List Anthropic pricing (not your actual plan's effective rate)
- Conservative fix recovery (usually 40-70% of measured waste)
- Dominant-model heuristic for costing (takes your most-used model)

For authoritative spend, cross-reference with `ccusage daily`. Our job is to rank fixes by impact and point you at the biggest one, not bill you.

## What the skill does NOT do

- Does not edit your CLAUDE.md, settings.json, hooks, or skills automatically
- Does not send transcripts, settings, or any content over the network
- Does not authenticate to Anthropic or any external service
- Does not analyze other coding assistants (Codex, Cursor, Aider) in v1 — see roadmap

## Recommended invocation pattern

1. Run `audit.py --days 7`
2. Read the JSON output
3. Present findings in the user's language as a tight report (see "Report structure" above)
4. If the user says "fix it" or "apply X" for a specific fix, guide them through it manually — do NOT edit their settings without explicit confirmation
5. Re-run after 1-2 weeks to measure delta

## Roadmap (v2+, not shipped)

- **Cross-assistant support**: Codex (`AGENTS.md`, reasoning_effort), Aider (`.aider.conf.yml`, map-tokens), Cursor (Auto vs API pool routing)
- **Auto-weekly cron digest** (to Telegram or email)
- **More detectors**: extended-thinking budget runaway, agent-team 7x multiplier, `.claudeignore` absence, plan-mode underuse, stale-session resume, pasted-blob vs `@file` mentions
- **Apply-fix mode**: safe settings.json edits with timestamped backup (currently report-only)
- **Trend tracking**: store results over time, show week-over-week deltas

## Authoritative references

Baked into the analysis:
- Thariq Shihipar (Anthropic Claude Code team), *Using Claude Code: Session Management & 1M Context* — https://claude.com/blog/using-claude-code-session-management-and-1m-context
- Anthropic cost management doc — https://code.claude.com/en/docs/claude-code/costs
- Samarth Gupta, *anthropic isn't the only reason you're hitting claude code limits* — https://medium.com/@samarthgupta1911
- ccusage by @ryoppippi — https://github.com/ryoppippi/ccusage (13k★)

## Privacy & safety

- All analysis is **local**. No network calls except the optional `npx ccusage@latest` fetch.
- Transcript content (tool result payloads, file contents, user messages) is parsed for size/counts/tool-names only — never retained, never transmitted.
- Settings.json is read-only. Nothing is written.
- Detectors summarize patterns; specific content (e.g., customer names, code) is not extracted into the report.
