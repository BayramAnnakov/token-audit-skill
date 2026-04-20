# Additional optimizations — think beyond the canned detectors

The nine Python detectors catch the patterns that generalize. But every user has unique waste patterns. After presenting audit findings, **brainstorm user-specific optimizations** that the canned detectors can't see.

This file is a prompt, not an exhaustive list. Use it as a starting point, then think harder.

## How to brainstorm

Look at the audit output + what you know about the user + Claude Code token economics. Ask:

- What's unusual about this user's profile that the canned detectors don't capture?
- What does their model mix, session length distribution, tool usage, project mix tell you?
- What language, timing, or cadence patterns jump out?
- Are there patterns that only make sense if you know THIS user's setup?

Then propose 1-3 custom ideas in a "Other ideas worth considering" section at the end of the report.

## Seeds — patterns worth checking per user

These are not auto-detectors (they're too contextual). Consider each for every audit.

### 1. Non-English structural content

**Why:** BPE tokenizers are trained mostly on English. Russian, Chinese, Japanese, Arabic, and other scripts produce 2-3× more tokens per equivalent content. A 4k-token Russian CLAUDE.md is equivalent to a ~1.5k English one.

**Check:** does the user's CLAUDE.md, global CLAUDE.md, or frequently-read structural files contain non-Latin scripts? If yes, suggest:
- Keep structural content (CLAUDE.md, skill descriptions, system instructions) in English. These load on every turn.
- User messages can stay in their preferred language — that's conversational, not structural.
- Output can stay in the user's language — output is cheaper than input in most plans.

**Priority:** high for Russian/Chinese/Japanese/Arabic/Hindi users with large CLAUDE.md files.

### 2. Off-peak shifting for background jobs

**Why:** Thariq Shihipar (Anthropic): *"If you run token-intensive background jobs, shifting them to off-peak hours will stretch your session limits further."* Your 5-hour block starts at your first prompt. If a cron job fires at 9am when you're about to start interactive work, you've now shared the block with the automation.

**Check:** do the user's heavy automations (detected via `recurring_scripts` detector OR hook activity) fire during their working hours?

**Suggest:**
- Shift cron schedules to off-hours (e.g., 3-5am local)
- Run automations as separate authentication / separate `claude-code` invocations if they MUST run during work hours
- Consolidate multiple small cron jobs into one larger batch at off-peak time

**Priority:** medium for anyone with automation running in work hours; high for Max-plan users close to their weekly cap.

### 3. "Why is my sub dying on Monday" — runaway recurring scripts

**Why:** Kieran Klaassen tweeted about burning 91% of his sub by Monday because a cron ran every 5 minutes instead of the intended cadence. This is one of the most common hidden-waste patterns, and the canned `recurring_scripts` detector catches the obvious cases — but not all.

**Check beyond the detector:**
- Launchd agents at `~/Library/LaunchAgents/` and `/Library/LaunchAgents/`
- `crontab -l`
- GitHub Actions scheduled workflows (`.github/workflows/*.yml` with `schedule:` triggers)
- Background scripts in any tool that uses `claude -p` in a loop (e.g., outbound bots, monitoring scripts)
- The user's morning-autopilot equivalent — is it firing at the frequency they think?

**Suggest:** run `launchctl list | grep claude` or `crontab -l`, cross-reference cadence with audit findings.

**Priority:** always check when ccusage shows a jump ≥ 2× weekly average, or when `recurring_scripts` detector surfaces anything.

### 4. Ollama / local models for hook + scheduled work

**Why:** Background analysis that runs unattended (log summarization, daily digest generation, lead enrichment) often doesn't need frontier-model quality. An Ollama-served local model (Llama, Mistral, Qwen) can handle 90% of these jobs at zero token cost against the subscription.

**Check:** does the user have hooks or scheduled jobs using `claude -p` for tasks that don't need Opus?

**Suggest:** route low-stakes recurring work through a local model. Reserve Claude subscription for interactive and high-stakes work.

**Priority:** medium for developers comfortable with Ollama; low otherwise.

### 5. Prompt-caching strategy mismatch

**Why:** Claude's cache has a 5-minute default TTL (1-hour premium). If the user opens a session, works for 20 minutes, goes for coffee for 10 minutes, comes back — cache is dead, next prompt pays full cache-write. Happens ~several times a day for many users.

**Check:** audit the user's `cache_hit_ratio` per session (already in data). Ask about their workflow — do they take breaks mid-session?

**Suggest:** for sustained work, stay engaged or start a new session after breaks rather than resuming stale ones. Or upgrade to premium 1-hour cache if it's clearly worth it.

**Priority:** medium for users with cache-hit ratios in the 40-60% range (there's a lot of recoverable waste there).

### 6. Desktop / web client blindness

**Why:** the Claude.ai web and Claude Desktop don't show `/cost` or `/context` statuslines. Users who split time between Claude Code and the desktop/web may never see usage until they hit limits.

**Check:** does the user use multiple Claude surfaces?

**Suggest:** use Claude Code for any token-intensive work (because it has telemetry); use desktop/web for brief tasks.

**Priority:** low (informational).

### 7. Skill / plugin over-installation

**Why:** every installed skill's description sits in the `<system-reminder>` available-skills list every turn. 149 skills × 100 tokens/description = 14.9k tokens of per-turn tax. (Already surfaced by the `skills` detector for aggregate bloat, but worth checking individual skills that aren't invocation-justified.)

**Check beyond the detector:** for each skill that has fired < 5 times in the audit window, ask "do I actually need this installed, or can I install on-demand?"

**Suggest:** aggressive skill pruning. Keep the 10-20 you actually use; uninstall the rest. Re-install when needed.

**Priority:** medium-high for heavy plugin users (50+ installed).

### 8. Reading CLAUDE.md that isn't informing answers

**Why:** a large CLAUDE.md that Claude doesn't actually consult is pure tax. Happens when CLAUDE.md has stale content, is too structured/technical to reference naturally, or duplicates info already in the code.

**Check:** do user interactions show Claude repeatedly asking clarifying questions about things ostensibly covered in CLAUDE.md?

**Suggest:** make CLAUDE.md content match how Claude actually reasons. Action-oriented ("When X, do Y") beats declarative ("The system has feature Z"). Review every 2-3 months for staleness.

**Priority:** medium — hard to detect automatically, worth asking the user about.

## What to do with these ideas in the report

At the end of the standard audit report, add a section:

```markdown
## Other ideas worth considering

(Not auto-detected, but worth a look for your profile:)

- [Language efficiency] Your bayram-os/CLAUDE.md is in English — good. But your daily-log entries are in Russian, and `~/.claude/SOUL.md` is 60% Russian. Consider moving structural rules to English (every-turn load); keep personal voice content in Russian.
- [Off-peak shift] Your `empatika-outreach-agent` scheduled jobs fire at 8am when you're about to start interactive work. Shift them to 3-4am to stop sharing the 5-hour block.
- [Runaway script check] Worth verifying: `launchctl list | grep claude` and `crontab -l`. If anything is set to < hourly cadence, double-check it should be.
```

Keep this section SHORT (3-5 bullets). Prioritize what's likely to matter for THIS user based on their profile. Don't list every seed from this file — curate.

## When to consult external sources

If the user's waste pattern doesn't map to any detector or seed here, **consult** before making stuff up:

- `claude-code-guide` agent — for Claude Code-specific questions (hook API, settings options, skills system)
- https://code.claude.com/en/docs/claude-code/costs — Anthropic's current cost guide
- https://claude.com/blog/using-claude-code-session-management-and-1m-context — Thariq's guide (evergreen for session management)
- ccusage docs (https://ccusage.com/) — for questions about spend measurement

Never fabricate Claude Code features or hook APIs. If you're not sure, ask the user to run a quick diagnostic command, or defer to an external source.
