# stepping-away

<img width="2172" height="167" alt="stepping-away-highres-13x1-fullwidth" src="https://github.com/user-attachments/assets/d7dbad57-b338-4ec2-bd96-18322650c3c9" />

<br/>

A [Claude Code](https://claude.com/claude-code) skill that lets you step away from your machine and have Claude keep working on whatever it was doing — bounded by a wall-clock deadline, with a clean stop and summary when time's up.

```
/stepping-away 5
```

→ Claude continues the in-progress task autonomously for 5 minutes, then halts and summarizes what it did.

## How it works

Two complementary mechanisms run in parallel:

| Mechanism | Role |
|---|---|
| `Monitor` (persistent) | Heartbeat + hard deadline. Emits `TICK` every 60s, `WRAP_UP` at T-30s, `COMPLETE` at T=0. |
| `ScheduleWakeup` chain | Self-paced continuous-work engine. Each turn ends with a wakeup ~90s out so Claude re-enters and keeps going. |

The Monitor is the deadline alarm and a safety net — if the wakeup chain ever drops a link, the next `TICK` re-engages Claude within ~60s. Together they keep momentum without going off the rails.

## Guardrails (built into the skill)

While the timer is running, Claude is instructed to:

- **Stay scoped** — continue the in-flight task only; if it finishes, work on adjacent polish (tests, edge cases, docs, error handling) within the same area. No unrelated work.
- **No clarifying questions** — pick the conservative choice and note assumptions inline.
- **No destructive or externally-visible actions** — no `git push`, no production deploys, no sending external messages, no destructive deletes. Local edits, builds, tests, refactors only.
- **Stop cleanly** — on `WRAP_UP`, finish what's in flight; on `COMPLETE`, halt and write a tight summary.

If you come back early, just type something — Claude stops the Monitor and responds.

## Install

```bash
npx skills add fmhall/stepping-away
```

Then in any Claude Code session, type `/stepping-away 5` and step away.

## Differences from a Ralph loop

- **Bounded by wall-clock**, not iteration count.
- **Self-paced** between Monitor events — Claude judges its own cadence rather than running on a fixed cycle.
- **Structured wind-down** (`TICK` → `WRAP_UP` → `COMPLETE`) instead of a hard cutoff.
- **Tied to a user step-away**, not a generic background loop.

## License

MIT
