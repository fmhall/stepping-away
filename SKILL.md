---
name: stepping-away
description: Activate autonomous work mode for a fixed duration when the user steps away. Use this whenever the user invokes /stepping-away with a minutes argument (e.g., "/stepping-away 5"), or says natural variants like "afk for 10", "stepping out for 5 — keep going", "be back in N min, keep working", "bio break, continue without me". Sets up a deadline-bounded autonomous loop using Monitor (heartbeat + stop signal) and ScheduleWakeup (self-paced continuous-work engine), so Claude keeps making forward progress on the in-flight task until the timer expires, then halts cleanly with a summary. Do NOT use this for indefinite background work, ralph-style infinite loops, or anything not anchored to a specific user step-away — this skill is bounded by wall-clock and tied to the user being temporarily AFK.
argument-hint: "[minutes: 1|3|5|10|15|30]"
retrieval:
  aliases:
    - stepping away
    - afk
    - bio break
    - brb
    - autonomous mode
    - keep working
  intents:
    - step away from machine
    - work autonomously while user is afk
    - continue task without user input
    - bounded autonomous loop
  entities:
    - "1"
    - "3"
    - "5"
    - "10"
    - "15"
    - "30"
    - "45"
    - "60"
---

# Stepping-Away Mode

The user is stepping away from their machine for a fixed number of minutes and wants you to keep making progress on the current task while they're gone, then stop cleanly when they're due back.

## Reading the argument

`$ARGUMENTS` is the minutes value. Parse the first integer found. If missing or unparseable, default to **5 minutes** and mention the default in your acknowledgement.

## How this works (mental model)

Two complementary mechanisms run in parallel:

| Mechanism | Role |
|---|---|
| `Monitor` (persistent) | The **deadline alarm** and a **heartbeat safety net**. Emits `TICK` every minute, `WRAP_UP` at T-30s, `COMPLETE` at T=0. |
| `ScheduleWakeup` chain | The **continuous-work engine**. Each turn ends with a wakeup scheduled for ~90s out, so you re-enter and keep going without waiting for user input. |

The Monitor gives you a hard stop (`COMPLETE`) and reorients you periodically (`TICK`). The wakeup chain is what keeps you actively working between Monitor events. If the chain ever drops a link, the next Monitor TICK will arrive within ~60s and re-engage you — that redundancy is intentional.

## Setup — do this once, immediately, then resume the in-flight task

### Step 1: Start the Monitor

Call the `Monitor` tool with:
- `description`: `"stepping-away timer (<MINUTES>m)"` — make it specific so each notification's source is obvious
- `persistent`: `true` — it must outlive the default 5-minute Monitor timeout for any window longer than that
- `command`: substitute `<MINUTES>` with the parsed value

```bash
deadline=$(($(date +%s) + <MINUTES>*60))
while true; do
  now=$(date +%s); remaining=$((deadline - now))
  if [ $remaining -le 0 ]; then
    echo "STEPPING_AWAY_COMPLETE — timer expired. Halt the autonomous loop, do NOT call ScheduleWakeup again, and write a short summary of what changed."
    exit 0
  elif [ $remaining -le 30 ]; then
    echo "STEPPING_AWAY_WRAPUP — ${remaining}s left. Finish only what is actively in flight; no new edits, no new files, no further ScheduleWakeup."
    sleep 15
  else
    echo "STEPPING_AWAY_TICK — ${remaining}s remaining; you are still in the autonomous window. Continue the in-progress task; if it's done, work on adjacent polish (tests, edge cases, docs, error handling) within the same scope."
    sleep 60
  fi
done
```

### Step 2: Kick off the wakeup chain

Call `ScheduleWakeup` immediately with:
- `delaySeconds: 90` — under the 5-minute Anthropic prompt-cache TTL, so wakeups stay cache-warm
- `prompt: <<autonomous-loop-dynamic>>` — the runtime sentinel that resumes autonomous-loop instructions
- `reason: "stepping-away autonomous tick — continuing in-progress task"`

This is the engine. Without it, the current turn ends and you go silent until the next Monitor TICK ~60s later.

### Step 3: Acknowledge briefly and resume the work

One short sentence to the user — something like *"Autonomous mode armed for 5m (Monitor + wakeup chain). Resuming the refactor."* — then go right back to whatever you were working on before they invoked the skill. Don't switch tasks. Don't ask clarifying questions.

## Rules during the autonomous window

You'll be operating without user input for several minutes. Bias toward sensible defaults and keep momentum.

1. **Stay scoped.** Continue the task that was in flight when `/stepping-away` was invoked. If it finishes mid-window, work on adjacent polish in the same area: tests, edge cases, docs, error handling, cleanup of what you just wrote. Never branch into unrelated work — the user can't redirect you right now, and unscoped exploration is the most common way an autonomous window goes off the rails.
2. **No clarifying questions.** If something is ambiguous, pick the more conservative choice and note your assumption inline (short code comment or a line in the final summary).
3. **Re-arm the engine after each work chunk.** At the natural end of each meaningful chunk of work, call `ScheduleWakeup` again with the same parameters as Step 2. The chain only stays alive if you keep linking it. **Stop calling it once you receive a `WRAP_UP` or `COMPLETE` event.**
4. **No destructive or externally-visible actions.** No `git push`, no production deploys, no sending messages (Slack/email/PR comments), no destructive deletes, no irreversible config changes. The user isn't here to confirm. Local edits, builds, tests, refactors, new files in the working tree — yes. Anything visible outside the local repo — no.
5. **Treat Monitor events as orientation, not instructions to reply.**
   - `TICK` → confirms you're still in window. Don't write a status update for every tick — they're not reading them live. Just keep working.
   - `WRAP_UP` → finish what's actively in flight; **stop scheduling new wakeups**; don't start new edits or open new files.
   - `COMPLETE` → halt. Write the summary (format below). The Monitor exits on its own.

## When the user comes back early

Any new user message arriving during the window means they're back. Stop autonomous mode immediately:

1. Call `TaskStop` on the stepping-away Monitor task — otherwise the heartbeat keeps emitting `TICK`s into your conversation.
2. Don't call `ScheduleWakeup` again.
3. Briefly note what was accomplished in the window (use the summary format below, condensed).
4. Then respond to whatever the user actually said.

## Final summary format

When `COMPLETE` fires (or the user returns), end with a tight summary — they'll read the diff for details:

- **Done:** 1–3 bullets of concrete changes.
- **In flight at deadline:** anything unfinished, or "none" if cleanly stopped.
- **Assumptions:** any non-obvious choices made without their input.
- **Next:** one-line suggestion for what to tackle next.

Keep it terse. The user wants to scan it in five seconds and dive into the diff.
