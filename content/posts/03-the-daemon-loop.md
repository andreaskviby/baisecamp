---
title: "The 10-minute poll, the 8-minute dispatch: scheduling an AI worker that doesn't loop on itself"
series: "AI ↔ Basecamp, part 3 of 5"
date: 2026-05-10
tags: ['basecamp', 'ai-agents', 'daemon', 'architecture']
---

Last post I covered authentication. This one is about the part that actually runs the work: the daemon. It's the most boring component of the system, and that's exactly why it's worth a whole post.

If you're going to put an LLM behind any kind of "watch this inbox and do the work" job, the failure modes are the same: it doesn't run when it should, it runs too often, it runs forever on one task, or it runs the same task twice. The daemon's job is to make all four of those impossible.

## The shape of the thing

Four background processes, all managed by macOS launchd:

```
~/Library/LaunchAgents/
├── com.openclaw.acme-daemon.plist         # KeepAlive, polls every 10 min
├── com.openclaw.acme-blocker-monitor.plist # StartInterval=90s
├── com.openclaw.acme-morning-sync.plist   # StartCalendarInterval Hour=7 Minute=0
└── (acme-daily-report-1600 — internal scheduler at 16:00)
```

The main worker is the first one. The other three are support roles I'll cover later in the series. This post is about the daemon — what each tick does, and why each guardrail exists.

## launchd, not cron

A small detour. macOS still ships with cron, and most tutorials reach for it. Don't.

launchd is harder to learn — the plist syntax is verbose — but it gives you three things cron doesn't: `KeepAlive` (restart if the process dies), `RunAtLoad` (start on boot or login), and explicit logging paths. A long-running daemon with a 10-minute internal loop is a much better fit for launchd than for a once-per-minute cron job that has to check whether the previous instance is still running.

If you're on Linux, the equivalent is systemd with `Restart=always` and a timer unit. Same idea.

## What each tick does

Every ten minutes, the daemon runs through this sequence:

1. **Refresh the Basecamp token** via the shared wrapper from [post 2](./02-shared-oauth-pattern.md). Idempotent and basically free.
2. **Fetch new file uploads and Basecamp documents** from the project's vaults. Anything new gets parsed to markdown and cached locally. PDFs through `pdftotext`. DOCX through `pandoc`. Images stay binary because the LLM has vision.
3. **Fetch all open todos in the project.**
4. **Filter to "assigned to Auto Andy AND not already in the state file."** Everything else is logged as "skipped" with a reason. Skipped-and-logged is a deliberate choice — when somebody asks "why didn't you do this todo?" I want to be able to point at a log line.
5. **For each eligible todo, up to `MAX_DISPATCHES_PER_TICK = 3`:**
   1. **Mark it as dispatched in the state file *before* spawning the agent.**
   2. **Spawn the agent process** with a structured prompt, a timeout, and a delivery channel.
   3. **Wait up to `DISPATCH_TIMEOUT_MS = 8 minutes`** before killing it.

That's the whole loop. The interesting decisions are in those constants.

## The three constants that prevent runaway behavior

### `POLL_INTERVAL_MS = 10 minutes`

Why ten and not one? Because the customer's expectation, set by the rest of Basecamp, is "this is a project management tool, not a chat app." Nobody assigns a todo and watches a clock for 30 seconds. Ten minutes is the longest delay where it still feels responsive, and it gives me 144 ticks per day — more than enough for any realistic Basecamp project's volume.

Polling that fast also keeps the API budget reasonable. 37signals' rate limits are generous but not infinite, and most ticks find nothing to do, so most ticks make exactly one cheap API call.

### `MAX_DISPATCHES_PER_TICK = 3`

This one is about blast radius. If the client files twenty todos in a batch — which has happened — I do not want the daemon to spawn twenty concurrent LLM dispatches. Three at a time keeps the cost predictable, keeps API throughput sane, and means a single bad prompt can damage at most three tasks before I notice. The remaining todos sit in the queue and get picked up next tick. From the customer's perspective the worst case is "the last seventeen took an hour to start instead of starting immediately," which is fine.

### `DISPATCH_TIMEOUT_MS = 8 minutes`

The hard kill. Combined with a `~5 tool calls` budget in the prompt itself, eight minutes is enough for any well-scoped todo and not enough for a stuck one to burn meaningful money. If an LLM gets confused and tries to read every file in the workspace looking for a thing that doesn't exist, the timer wins. The dispatch dies, the todo stays marked-dispatched (more on that below), and I see it the next morning in the daily report.

A reasonable question: shouldn't I retry stuck todos? No. If a todo timed out, the LLM was lost. Retrying with the same prompt gets the same result. The right move is for me to look at the todo, figure out what was wrong with it, and either reword it or split it. The timeout is doing a load-bearing job: it tells me which todos are too ambiguous.

## The state-file trick that prevents loops

This is the single most important pattern in the whole daemon. I want to be very explicit about it because it's the kind of thing that, if you get it wrong, you find out by losing $400 to OpenAI in an afternoon.

When the daemon decides to dispatch a todo, it does this:

1. Open `acme-todos-state.json`.
2. Add this todo's id with a `dispatched_at` timestamp.
3. Save the file.
4. *Then* spawn the agent.

The dispatch is recorded *before* the work begins. Not after.

Why? Because the alternative is a runaway loop. If you mark the todo as handled *after* the dispatch completes, here's what happens when the dispatch crashes mid-flight (and it will — networks fail, models 500, processes get OOM-killed, the Mac mini reboots for an OS update):

- Tick at T+0: daemon picks up todo. Spawns agent.
- T+3min: agent crashes. State file untouched.
- Tick at T+10min: daemon sees the same open todo, not yet recorded as dispatched. Spawns agent again.
- T+13min: agent crashes again. State still untouched.
- ...

This is a loop. It costs money and produces nothing. With a flaky integration in the agent, it can run for hours before you notice.

Recording the dispatch *before* spawning the agent breaks the loop. If the agent crashes, the todo is already marked dispatched. The next tick won't re-pick it. The downside is that a crashed dispatch produces no result — but that's the right failure mode. **A todo that didn't run is visible** (it sits as not-completed in Basecamp). **A todo that ran in a loop is expensive and invisible**.

I tell people about this trick and they say "but what if I want to retry?" Manual retry is easy: open the state file, delete the entry, save. The next tick re-dispatches. The point is that retry is a *deliberate human decision*, not an automatic behavior.

## What goes into the spawn command

When the daemon decides to dispatch, it runs this:

```bash
openclaw agent --agent acme \
  --message "<the dispatch prompt>" \
  --deliver --channel telegram \
  --reply-account stafegroup_acme_bot \
  --reply-to 8245748191 \
  --timeout 480
```

A few notes:

- The `--timeout 480` is seconds — 8 minutes. The shell-level timeout is a backstop in case the agent's own internal limits fail.
- `--deliver --channel telegram` means the agent's final message goes to me on Telegram. In an earlier version of this system I got pinged for every dispatch and it became unmanageable; now the prompt instructs the agent to reply `NO_REPLY` and the real status comes through a 16:00 daily summary. (Post 4 covers the philosophy of that suppression.)
- `--reply-to 8245748191` is my personal chat id. Hardcoded. There's exactly one human who gets pinged when something catastrophic happens.

## The dispatch prompt

The agent doesn't get a freeform "do this task" instruction. It gets a tight, structured prompt with hard limits baked in:

```
You have ONE Basecamp todo to handle. Do exactly what it asks. Nothing more.

**Todo** (id <ID>): "<title>"
**List**: <list name>
**URL**: <basecamp app_url>
**Attachments**: N file(s) — search memory for them via memory_search.

Required workflow — exactly these steps, no extras:
1. Read attached documents via memory_search.
2. Read the "Initial knowledge for AI Agents" doc via memory_search.
3. Do the work. Stay in scope.
4. Post a result comment using scripts/post-todo-comment.mjs.
5. Mark the todo complete using scripts/complete-todo.mjs.
6. Update the KB using scripts/update-kb.mjs.
7. Commit + push the KB file via /Users/andreaskviby/myopenclaw/sync.sh.
8. Send ONE short Telegram update (1-2 lines, English, final result only).

HARD LIMITS:
- No git clone unless the todo explicitly says so
- No package install
- No prod commands
- ~5 tool calls maximum
- ONE dispatch = ONE result. Do not loop.

If blocked: file a Things I miss todo, comment on the original "Blocked — see <link>", STOP. Do not mark complete.
```

Two ideas worth pulling out:

**"~5 tool calls maximum."** This is the most important line. Without a tool-call budget, agents will happily browse for an hour. Five is enough for the canonical flow (read attachments, read KB, do the work, comment, complete) and not enough for "let me read fifty files to be sure." If a task genuinely needs more, it needs to be split into multiple todos — and the agent has a script for that, which it uses proactively when it sees an overstuffed parent task.

**"ONE dispatch = ONE result. Do not loop."** Pair this with the dispatch-before-running state trick and you have belt and suspenders. The prompt tells the model not to loop. The state file ensures that even if the model ignores the prompt and crashes, the daemon won't re-dispatch.

## The state file is the source of truth

The whole system can be understood from a few JSON files. The most important is `acme-todos-state.json`. It looks roughly like:

```json
{
  "12345678": {
    "dispatched_at": "2026-05-21T09:14:33.000Z",
    "title": "Add OTP login screen",
    "status": "completed"
  },
  "12345679": {
    "dispatched_at": "2026-05-21T09:24:12.000Z",
    "title": "Update dependency versions",
    "status": "dispatched"
  }
}
```

When somebody pings me with "the agent didn't run my todo," my first move is to open this file. If the todo id isn't in there, the daemon never saw it (probably an assignment issue). If it's there with `status: dispatched` from an hour ago, the agent died silently (check the log). If it's there with `status: completed`, the work is done and the customer is looking at the wrong todo.

The state file is also the manual override surface. Want to force a re-dispatch? Delete the entry. Want to pause a runaway? Add the entry with a fake timestamp. The file is plain JSON; you can edit it with `vim` and the next tick will respect your changes.

## Operating the daemon

A few muscle-memory commands I use weekly:

```bash
# Confirm everything is loaded
launchctl list | grep acme

# Tail the main log
tail -f ~/.openclaw/workspace-acme/acme-daemon.log

# Force a restart after a config change
launchctl kickstart -k gui/$(id -u)/com.openclaw.acme-daemon

# Find out why a specific todo wasn't picked up
grep '<todo_id>' ~/.openclaw/workspace-acme/acme-daemon.log
```

The combination of "obvious state file" plus "obvious log file" plus "obvious restart command" means that 90% of debugging happens in a terminal in under sixty seconds. That's the goal. You should never need to read source code to diagnose a problem with a system like this.

## The constants, summarized

| Constant | Value | What it stops |
|---|---|---|
| `POLL_INTERVAL_MS` | 10 minutes | API hammering, fake urgency |
| `MAX_DISPATCHES_PER_TICK` | 3 | Cost blowouts on bulk-filed todos |
| `DISPATCH_TIMEOUT_MS` | 8 minutes | Stuck agents burning tokens |
| Tool-call budget (in prompt) | ~5 | Agents wandering off the task |
| Dispatch-before-running state | — | Runaway re-trigger loops |

Five guardrails. None of them are clever. All of them are essential. The reason this daemon has run for months without supervision isn't because the LLM is good; it's because the daemon assumes the LLM will fail in every possible way and is designed to fail safe when it does.

---

*Next in the series: what happens when the agent gets stuck mid-task and needs to ask the customer a question. The "Things I miss" pattern is the part of this system I'm proudest of, and I haven't seen it described anywhere else.*
