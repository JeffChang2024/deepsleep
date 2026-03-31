# 🧠 DeepSleep

> Like humans need sleep for memory consolidation, AI agents need DeepSleep to persist context across sessions.

Two-phase daily memory persistence for [OpenClaw](https://github.com/openclaw/openclaw) AI agents.

## What It Does

AI agents wake up fresh every session — no memory of yesterday's conversations, decisions, or context. DeepSleep fixes this with a nightly pack + morning dispatch cycle:

1. **Pack (23:40)** — Scans all active group chats, extracts key decisions/lessons/progress, writes a daily summary
2. **Dispatch (00:10)** — Sends personalized morning briefs to each group, updates per-group memory snapshots
3. **Restore (on demand)** — When the agent receives a message in any group, it loads that group's memory snapshot before replying

The result: your agent remembers what happened yesterday, knows what's on the agenda today, and never asks you to repeat yourself.

## Features

- **Auto-discovery** — Finds all active sessions automatically, no manual config per group
- **Privacy-safe** — Each group only sees its own summary; DM content stays private
- **Idempotent** — Safe to re-run; duplicate detection for both pack and dispatch
- **Midnight-safe** — Locks target date at start, handles cross-midnight execution
- **3-day rolling snapshots** — Keeps recent context compact; open questions persist until resolved
- **Failure recovery** — If pack fails, dispatch does an emergency mini-pack so no day is lost
- **Schedule tracking** — Extracts future tasks with dedup keys, reminds when due
- **MEMORY.md guard rails** — Prevents LLM from accidentally deleting or restructuring long-term memory

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw) with cron support
- `tools.sessions.visibility` set to `all`
- At least one active group chat session

## Quick Start

```bash
# 1. Enable cross-session visibility
openclaw config set tools.sessions.visibility all
openclaw gateway restart

# 2. Create pack cron (23:40 local time)
openclaw cron add \
  --name "deepsleep-pack" \
  --cron "40 23 * * *" \
  --tz "Your/Timezone" \
  --session isolated \
  --message "Execute DeepSleep Phase 1. Read the deepsleep skill pack-instructions.md and follow it strictly." \
  --timeout-seconds 900 \
  --no-deliver

# 3. Create dispatch cron (00:10 local time)
openclaw cron add \
  --name "deepsleep-dispatch" \
  --cron "10 0 * * *" \
  --tz "Your/Timezone" \
  --session isolated \
  --message "Execute DeepSleep Phase 2. Read the deepsleep skill dispatch-instructions.md and follow it strictly." \
  --timeout-seconds 900 \
  --no-deliver

# 4. Initialize schedule
mkdir -p memory
echo "# Schedule\n\n| Key | Date | Source | Item | Priority | Status |\n|-----|------|--------|------|----------|--------|" > memory/schedule.md

# 5. Add Phase 3 restore to your AGENTS.md (see SKILL.md for template)
```

## File Structure

```
deepsleep/
├── SKILL.md                    # Full skill definition (OpenClaw reads this)
├── pack-instructions.md        # Phase 1 instructions (cron reads this)
├── dispatch-instructions.md    # Phase 2 instructions (cron reads this)
├── README.md                   # This file
├── .gitignore
└── references/
    └── design.md               # Architecture & design decisions
```

## Runtime Files (generated)

```
memory/
├── YYYY-MM-DD.md              # Daily summaries
├── schedule.md                # Task tracking with dedup keys
├── dispatch-lock.md           # Prevents duplicate sends
├── dispatch-log.md            # Send history (rolling 30 entries)
└── groups/
    └── <chat_id>.md           # Per-group 3-day rolling snapshots
```

## v2.1 Improvements

- 🔴 **Midnight race fix** — Date locked at script start, no drift across midnight
- 🔴 **Dispatch dedup** — Lock file prevents double-sending on retry/re-run
- 🟡 **DM privacy** — DM content never leaks to group broadcasts
- 🟡 **Schedule dedup** — Key column prevents duplicate task entries
- 🟡 **Rolling snapshots** — 3-day window keeps context fresh and compact
- 🟡 **Send logging** — Full audit trail of dispatch success/failure
- 🟡 **Memory guard rails** — LLM can append but never delete/restructure MEMORY.md
- 🟡 **Time perspective** — Morning briefs use reader's perspective ("yesterday/today" not "today/tomorrow")

## Inspirations

Built with insights from the community: agent-sleep (multi-level sleep), memory-reflect (filtering criteria), jarvis-memory-architecture (cron inbox), memory-curator (open questions).

## License

MIT
