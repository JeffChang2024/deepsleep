---
name: deepsleep
description: Two-phase daily memory persistence for AI agents. Nightly pack at 23:40 plus morning dispatch at 00:10. Auto-discovers sessions, filters by importance, tracks open questions, and delivers per-group morning briefs.
---

# DeepSleep

Two-phase daily memory persistence for AI agents.

Like humans need sleep for memory consolidation, AI agents need DeepSleep to persist context across sessions.

## When to Activate

Activate when user mentions daily summary, memory persistence, sleep cycle, cross-session memory, morning brief, or nightly pack.

## Phases

### Phase 1: Deep Sleep Pack (23:40)

1. **Auto-discover sessions** — Use `sessions_list(kinds=['group', 'main'], activeMinutes=1440)` to find all active groups AND direct messages from the past 24 hours. New groups are automatically included.
2. **Pull conversation history** — For each active session, use `sessions_history(sessionKey=<key>, limit=100)`.
3. **Filter and summarize** — Apply filtering criteria to generate concise summaries:
   - Keep: Decisions, Lessons, Preferences, Relationships, Milestones
   - Skip: Transient (heartbeats, weather), Already captured in MEMORY.md
4. **Schedule future items** — Extract future-dated reminders and write to `memory/schedule.md` with trigger dates.
5. **Write daily file (idempotent)** — Write the `## Daily Summary (DeepSleep)` section to `memory/YYYY-MM-DD.md`. Before writing, check if a `## Daily Summary (DeepSleep)` header already exists for today — if so, replace it instead of appending a duplicate. This ensures retries and re-runs produce the same result.
6. **Update long-term memory (privacy-safe)** — Merge-update `MEMORY.md` (update in place, don't append duplicates; remove outdated info). **Important:** Do NOT copy private MEMORY.md content into the daily summary file. The daily file may be broadcast to groups in Phase 2 — only include information that originated from those groups' own conversations.

### Phase 2: Morning Dispatch (00:10)

1. **Read yesterday's summary** — Load `memory/YYYY-MM-DD.md` from the previous day.
2. **Send per-group briefs** — For each group with content, send a personalized morning recap via `message(action='send', target='chat:<id>')`. Only include information from that specific group's summary — never cross-leak content between groups or from MEMORY.md.
3. **Include reminders** — Attach any schedule items due today.
4. **Track open questions** — Include relevant Open Questions for continuity.

## Daily Summary Template

**⚠️ The section header must be exactly `## DeepSleep Daily Summary` — both pack and dispatch use this as the anchor for idempotent writes and existence checks. Do not vary this.**

```markdown
## DeepSleep Daily Summary

> Auto-discovered N active groups. schedule.md: [items due / none].

### [Group Name] <!-- chat:oc_abc123 -->
- Concise summary of key discussions and decisions

### [Another Group] <!-- chat:oc_def456 -->
- Summary

### Direct Messages
- (DM content if any)

### Open Questions
- Unresolved questions, tracked across days

### Today (Next Day)
- Actionable next steps (from the reader's perspective at 00:10, these are "today's" tasks)

### Todo
- [ ] Immediate action items
```

**Rules:**
1. Each group `###` header MUST include `<!-- chat:oc_xxx -->` HTML comment with chat_id.
2. Pack must self-check: every group section has a parseable chat_id; if missing, flag error.
3. Dispatch parses chat_ids from these annotations; fallback mapping table is last resort only.

## Schedule File Format

File: `memory/schedule.md`

```markdown
| Date | Source | Item | Status |
|------|--------|------|--------|
| YYYY-MM-DD | Group/DM | Description | pending/done |
```

## Setup

### 1. Create cron jobs

```bash
# Phase 1: Pack (needs ~60s per active session for history pull + summarization)
openclaw cron add \
  --name "deepsleep-pack" \
  --cron "40 23 * * *" \
  --tz "Your/Timezone" \
  --session isolated \
  --message "Execute DeepSleep Phase 1. Read the deepsleep skill pack-instructions.md and follow it strictly." \
  --timeout-seconds 900 \
  --no-deliver

# Phase 2: Dispatch (needs time to send messages + write snapshots)
openclaw cron add \
  --name "deepsleep-dispatch" \
  --cron "10 0 * * *" \
  --tz "Your/Timezone" \
  --session isolated \
  --message "Execute DeepSleep Phase 2. Read the deepsleep skill dispatch-instructions.md and follow it strictly." \
  --timeout-seconds 900 \
  --no-deliver
```

**⚠️ Must use `--session isolated` (not `--session main`)!** The `timeoutSeconds` field only works with isolated `agentTurn` jobs. Main session `systemEvent` jobs ignore the timeout setting and use the hardcoded heartbeat timeout (~120s), which is far too short. Use `--no-deliver` since the dispatch phase sends messages directly via the message tool.

**Timeout guidance:** Phase 1 needs ~20s per active session (history pull + LLM summarization). For 6 sessions, real-world pack takes ~135s. We recommend 900s (15 min) as a safe default to handle growth.

### 2. Enable cross-session visibility

```bash
openclaw config set tools.sessions.visibility all
openclaw gateway restart
```

### 3. Initialize schedule

Create `memory/schedule.md` with the table header above.

## Requirements

- OpenClaw with `tools.sessions.visibility` set to `all`
- Cron jobs using `agentTurn` mode (`--session isolated`) with `--timeout-seconds 900`
- `--no-deliver` on both cron jobs (dispatch sends messages directly via message tool)

## Phase 3: Session Memory Restore (on demand)

The most critical piece — when the agent receives a message in a group session:

1. Check if `memory/groups/<chat_id>.md` exists
2. If yes, read it to restore context about recent discussions, open questions, and todos
3. If the file is missing or older than 48 hours, fall back to reading `memory/YYYY-MM-DD.md` (today + yesterday) for that group's section

Without this step, Phases 1-2 only help the human (morning brief), but the agent itself still wakes up with no memory.

### AGENTS.md Configuration

Add this standing order to your `AGENTS.md`:

```markdown
### 🔄 Standing Order: Group Memory Restore (DeepSleep Phase 3)
**Trigger:** Before replying in any group chat session
**Steps:**
1. Check if `memory/groups/<current_chat_id>.md` exists
2. If yes → read it (< 2KB, fast load)
3. If no → read `memory/YYYY-MM-DD.md` (today + yesterday), find that group's section
4. Check `memory/schedule.md` for today's due items
5. Reply with restored context

**When required:**
- Your context has no recent conversation for this group
- Session just started or was reset
- You have no memory of what was discussed yesterday
```

### File structure
```
memory/groups/
├── oc_abc123.md    # Group A: recent 3-day summary + open questions
├── oc_def456.md    # Group B: recent 3-day summary + open questions
└── ...
```

Phase 2 generates/updates these files each morning. They are compact (< 2KB each) and designed for fast loading.

## Privacy Notes

- Phase 1 writes a daily summary that Phase 2 broadcasts to groups. Never include private MEMORY.md content in the daily summary.
- Each group only receives its own summary in the morning dispatch — no cross-group content leakage.
- MEMORY.md is updated separately and stays in the main session context only.
- Per-group memory snapshots only contain that group's own conversation summaries.

## Known Gotchas

1. **Cron timezone**: When using `--tz Asia/Shanghai`, the cron expression is in LOCAL time, not UTC. `40 23 * * *` means 23:40 Shanghai time. Do NOT convert to UTC yourself.
2. **Dispatch must write snapshots**: The dispatch phase MUST use the `write` tool to create/overwrite `memory/groups/<chat_id>.md` files. Without this, Phase 3 has nothing to load.
3. **Must use isolated agentTurn, NOT main systemEvent**: `timeoutSeconds` only works with `agentTurn` (isolated) jobs. Main session `systemEvent` jobs use a hardcoded ~120s heartbeat timeout that CANNOT be overridden. We learned this the hard way — our first two runs were killed at exactly 120s.
4. **Timeout must be generous**: Set `--timeout-seconds 900` (15 min). Real-world pack with 6 sessions takes ~135s. Without enough timeout, the cron gets killed mid-execution and produces no output.
5. **Parallel tool calls**: When pulling history from multiple sessions, batch `sessions_history` calls in parallel (same tool call block). Same for writing multiple snapshot files. This cuts execution time significantly.
6. **Chat ID mapping**: Keep a mapping of group names → chat_ids in the dispatch instructions file or TOOLS.md. The dispatch agent needs this to send messages and write snapshot files.
7. **Single schedule source**: Use `memory/schedule.md` (Markdown table) as the ONLY schedule/todo tracking file. Do not create separate JSON schedule files — they will diverge and cause confusion.
8. **Header must be exact**: The daily summary section title must be exactly `## DeepSleep Daily Summary` — pack, dispatch, and failure recovery all use this string as the detection anchor. Any variation (Chinese translation, extra words) will cause dispatch to think pack failed and trigger unnecessary emergency recovery.
9. **Chat_id annotation format**: Must be `<!-- chat:oc_xxx -->` (with space after `chat:`). Pack must self-check that every group section has this annotation. Missing annotation = that group gets no morning brief and no snapshot update.
10. **Time perspective shift**: Pack runs at 23:40 (still "today"), but dispatch sends at 00:10 (already "tomorrow"). The morning brief must use the **reader's perspective**: summaries describe "what was done **yesterday**", action items describe "what to do **today**". Never write "today we did X / tomorrow do Y" — that's the pack-time perspective and feels wrong to the reader.
11. **Midnight race condition**: Pack starts at 23:40 and may cross midnight. Lock the target date at script start (23:xx→today, 00:xx→yesterday) and never re-fetch current time in later steps. All filenames and headers use this locked `PACK_DATE`.
12. **Dispatch deduplication**: Use `memory/dispatch-lock.md` (contains one line: the dispatched date). Check before sending; write after all sends complete. If lock date matches target date, skip entirely. Lock must be written AFTER sends (not before) so a crash mid-send allows retry.
13. **Schedule ItemKey dedup**: `memory/schedule.md` has a `Key` column as unique identifier. Pack must check existing keys before writing; update status if key exists, only append if new.
14. **Snapshot 3-day rolling merge**: Keep last 3 days of progress entries. Open Questions and incomplete todos survive beyond 3 days. Completed todos older than 3 days are pruned.
15. **MEMORY.md guard rails**: Pack may only append/update — never delete existing sections, rewrite About/Channels/Lessons, or restructure the file. Humans own the structure; agents own incremental updates.
16. **DM privacy boundary**: Daily summary is broadcast material. DM content stays in MEMORY.md only. Open Questions and Today sections must not contain DM-originated items. Write "N DMs processed" summary, not details.

## Verification Checklist

After setup, run this to verify everything works:

```bash
# 1. Manually trigger pack
openclaw cron run <pack-job-id>

# 2. Wait for completion (~2-3 min), then check:
openclaw cron runs --id <pack-job-id> --limit 1
# Expected: status=ok, durationMs > 120000 (proves timeout works)

# 3. Check output files:
cat memory/YYYY-MM-DD.md | grep "DeepSleep Daily Summary"
# Expected: section exists with group summaries + chat_id annotations

# 4. Manually trigger dispatch
openclaw cron run <dispatch-job-id>

# 5. Check dispatch output:
ls -la memory/groups/
# Expected: one .md file per active group, updated timestamp = now

# 6. Check Feishu groups received morning briefs
```

If pack completes in exactly 120s → timeout is NOT working (you're using systemEvent instead of agentTurn). Recreate with `--session isolated`.

## Failure Recovery

If Phase 1 (pack) fails or times out, Phase 2 (dispatch) has a built-in fallback: it detects the missing daily summary and performs an emergency mini-pack — pulling recent history from all active sessions, generating a condensed summary, then proceeding with normal dispatch. This ensures **no day's memory is ever lost**, even if a cron job fails.

## Inspirations

Built with insights from the community: agent-sleep (multi-level sleep), memory-reflect (filtering criteria), jarvis-memory-architecture (cron inbox), memory-curator (open questions).
