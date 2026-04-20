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

### Option A: Claude Code plugin (recommended)

In Claude Code:
```
/plugin marketplace add BayramAnnakov/token-audit-skill
/plugin install token-audit@token-audit-skill
```

Then:
```
/token-audit
```

### Option B: drop-in skill (no plugin system)

```bash
git clone https://github.com/BayramAnnakov/token-audit-skill
cp -r token-audit-skill/skills/token-audit ~/.claude/skills/
```

Invoke in Claude Code with `/token-audit`.

### Option C: run the CLI directly

```bash
git clone https://github.com/BayramAnnakov/token-audit-skill
cd token-audit-skill
python3 skills/token-audit/scripts/audit.py --days 7
```

Outputs a JSON report.

## Prerequisites

- **Python 3.11+** — for the analyzer
- **Node.js 20+** — for `ccusage` (fetched automatically via `npx`, no global install required)
- Read access to `~/.claude/projects/` and `~/.claude/settings.json` (yours)

If ccusage is unavailable, the audit proceeds without baseline spend totals; detectors still work.

## Example output (abridged)

```
Token Audit — last 7 days

Spend (ccusage)
  Total:    $N at API list pricing (reference only — you're on a flat subscription)
  Models:   Opus 80% · Sonnet 19% · Haiku 1%

Top bottlenecks (where to look first)

🔴 Single session: <project>/<session-id>  →  ~27% of this week's waste
  • 1,500+ turns past the 400k context-rot threshold, peak ~800k
  • 2,000+ Opus turns with <1k output (Sonnet would suffice)
  • FIX: compact proactively every ~2h with a scope hint

🔴 Project: <project>  →  ~48% of this week's waste
  • Add `"model": "claude-sonnet-4"` to this project's .claude/settings.json
  • Escalate to Opus only for planning / hard reasoning

🔴 File: <project>/CLAUDE.md  →  19% of this week's waste
  • ~9k tokens, paid on every turn. Target: ≤2k (Anthropic's 200-line recommendation)
  • Move command recipes and playbooks into @-referenced files

Top category leaks

🟡 Skill descriptions total ~9k tok (per-turn tax)
🟡 Hundreds of Bash calls using cat/head/find/grep where native tools exist
🟢 Same file Read 3+ times in several sessions (rewind candidate)

One fix this week: add Sonnet default to the top project's settings.json.
```

Numbers in the example are illustrative. Your report uses your real data from `~/.claude/projects/` and `ccusage`.

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
