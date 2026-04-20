# token-audit

> Find where your Claude Code tokens are leaking — and what to fix first.

A Claude Code skill that audits your local Claude Code usage, surfaces the highest-$-impact leaks, and tells you which one to fix this week. All analysis is **local** — transcripts never leave the machine.

## What it catches

| Leak | Typical impact |
|---|---|
| **Opus on simple turns** | Up to 5× overspend on short lookups |
| **Context rot** | Turns past 400k tokens waste dollars + quality |
| **CLAUDE.md bloat** | Every turn pays for every extra token; target <2k tokens |
| **Hook output re-injection** | Big SessionStart hooks re-feed on every session |
| **Skill description overhead** | 149 installed skills = 149 descriptions × every turn |
| **Bash anti-patterns** | `cat`/`head`/`find`/`grep` dump full output; native tools stream |
| **Tool-schema dumping** | With tool search OFF, all MCP schemas load every turn (~20k tok) |
| **Cache miss storms** | Mid-session CLAUDE.md edits kill cache |
| **Redundant file reads** | Same file Read 3+ times = rewind candidate |

Each detection comes with a **ballpark weekly $ saving** so you can prioritize. Heuristics are grounded in:
- Thariq Shihipar's official Claude Code session-management article (Anthropic, Apr 2026)
- Anthropic's [cost-management docs](https://code.claude.com/en/docs/claude-code/costs)
- Samarth Gupta's measured audits (Claudest plugin)
- ccusage's authoritative spend data

## Install

### Option A: as a Claude Code skill (recommended)

Clone into your skills directory:

```bash
git clone https://github.com/BayramAnnakov/token-audit-skill ~/.claude/skills/token-audit
```

Then invoke in Claude Code:
```
/token-audit
```

### Option B: run the CLI directly

```bash
git clone https://github.com/BayramAnnakov/token-audit-skill
cd token-audit-skill
python3 scripts/audit.py --days 7
```

Outputs a JSON report.

## Prerequisites

- **Python 3.11+** — for the analyzer
- **Node.js 20+** — for `ccusage` (fetched automatically via `npx`, no global install required)
- Read access to `~/.claude/projects/` and `~/.claude/settings.json` (yours)

If ccusage is unavailable, the audit proceeds without baseline spend totals; detectors still work.

## Example output (abridged)

```
Token Audit — 2026-04-19 (last 7 days)

Spend summary
  Total:    $47 on ccusage daily
  Models:   Opus 80% · Sonnet 19% · Haiku 1%
  Peak day: Wed Apr 17 ($12.10)

Top leaks (ranked by weekly savings)

🔴 Opus used on 5,505 simple turns — ~$2,200/wk savings
  • 5,505 Opus turns with <1k output tokens
  • Current cost on Opus: $2,750  →  Sonnet: $550
  • Fix: set default model to Sonnet; escalate to Opus only for planning

🔴 1,646 turns past the context-rot threshold — ~$765/wk savings
  Thariq (Anthropic): "Due to context rot, the model is at its
  least intelligent point when compacting." Compact PROACTIVELY.

🔴 CLAUDE.md bloat: 5 oversized files, worst ~8.9k tok — ~$282/wk savings
  • ~/GH/bayram-os/CLAUDE.md: 8.9k tok (35k bytes)
  • Target: <2k tokens (Anthropic's 200-line recommendation)

🟡 Skill descriptions total 8.9k tok (per-turn tax) — ~$168/wk savings
🟡 401 Bash calls that should use native tools — ~$3.21/wk savings
🟢 Redundant Read on 33 file paths — ~$0.49/wk savings

One fix this week: Set project default model to Sonnet.
Estimated total weekly savings: ~$3,418 (≈ 68× one Max20x subscription-week).
```

(Ballpark numbers. Cross-reference with `ccusage daily` for authoritative spend.)

## Privacy

- **All analysis is local.** No network calls except the optional `npx ccusage@latest` fetch.
- **Transcript content is parsed for sizes/counts/tool-names only.** Tool results, user messages, and code are never retained or transmitted.
- **Settings.json is read-only.** Nothing is written to your config without explicit confirmation (and the v1 report-only mode doesn't write anything).

## How accurate are the savings estimates?

They're **ballparks**, not invoices. Rankings are reliable; absolute dollar figures have two known sources of inflation:
1. List Anthropic pricing vs. your effective plan rate
2. Conservative cache-hit assumptions

Use the estimates to **prioritize**, not to predict your next invoice. For authoritative spend, run `ccusage daily`.

## Design principles

- **Local only.** Transcript content never leaves the machine.
- **Humble phrasing.** "Looks like" not "is." Every heuristic is an educated guess with a stated basis.
- **Ranked by savings.** Detectors compete for your attention by $ impact, not by category.
- **Authoritative where possible.** Cites Thariq Shihipar / Anthropic docs for every behavioral recommendation.
- **Conservative estimates.** Claim 40-70% fix recovery, not 100%.
- **Safe.** v1 is read-only. Fix application will be opt-in with timestamped backups.

## Roadmap

See the `## Roadmap` section in [SKILL.md](./SKILL.md).

Short version: more detectors (extended-thinking runaway, agent-team multiplier, missing `.claudeignore`, plan-mode underuse, stale-session resume), cross-assistant support (Codex, Aider, Cursor), auto-weekly digest, and an opt-in fix-apply mode.

## Credits

- [Thariq Shihipar](https://x.com/trq212) (Anthropic Claude Code team) — authoritative session-management guidance
- [Samarth Gupta](https://medium.com/@samarthgupta1911) — measured waste categories via the Claudest plugin
- [@ryoppippi / ccusage](https://github.com/ryoppippi/ccusage) — the baseline $ reporting tool this skill sits on top of
- Bayram Annakov — AI Personal OS course, where this skill was born

## License

MIT
