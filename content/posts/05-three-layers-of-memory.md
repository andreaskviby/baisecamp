---
title: "Three layers of memory: how the agent stays smart across dispatches"
series: "AI ↔ Basecamp, part 5 of 5"
date: 2026-05-19
tags: ['basecamp', 'ai-agents', 'memory', 'architecture']
---

This is the last post in the series. The first four covered the moving parts: the inbox identity, the shared OAuth, the daemon, the blocker monitor. This one is about the thing that turns the system from "a script that runs LLM calls" into something that gets better the longer it runs.

It's about memory.

An LLM has no memory between calls. Every dispatch in my system is a fresh model invocation with a fresh context window. If I don't bring memory in deliberately, every dispatch starts from zero — same mistakes, same questions, same wasted tool calls. The whole game is figuring out *what* memory to inject, *where* it lives, and *how* it stays fresh without me babysitting it.

My agent uses three layers. Each one solves a problem the others don't.

## Layer 1: the canonical doc, edited in Basecamp

The first layer is a single Basecamp document titled **"Initial knowledge for AI Agents."** It lives in a KB sub-vault in the project. I edit it like I'd edit any other Basecamp doc — in the Basecamp UI, in my browser, in my own time.

This is the layer that surprises people the most when I describe the system. They assume the AI's knowledge base must live in a database, or a vector store, or at least a markdown file in a git repo. Mine lives in Basecamp.

Why?

**Because the people who need to update it use Basecamp.** When the client realizes that "we always use Twilio for SMS" should be a permanent fact and not something the agent re-asks, they don't need to file a pull request or learn about a knowledge management tool. They open the doc, type a sentence, save. Next morning the agent knows.

**Because it's auditable.** Basecamp has built-in version history on documents. If a bad edit confuses the agent, I roll it back through the Basecamp UI. No git, no merges, no diff tools.

**Because it eliminates the "edit happens, system doesn't notice" failure mode.** I'll get to the sync mechanism in a moment, but the short version: every morning, a 07:00 cron re-fetches the doc, re-parses it, and re-indexes it. There's no way for the doc to be stale by more than a day. And if I need it sooner, there's a one-liner I run manually.

What goes in this doc? Stable, opinionated, codifiable knowledge:

- "We always use Twilio for SMS."
- "The app's package name is `com.acme.app`."
- "Don't add new top-level Flutter dependencies without filing a 'Things I miss' first."
- "When asked to add a screen, use the existing `AuthNavigator`, don't create a new navigator."
- "Builds go to the 'Builds' vault, not as a comment on the todo."

This is the source of truth that overrides anything else. If the doc says one thing and the agent's other internal config says another, the doc wins. That's the contract.

## Layer 2: the parsed file cache

The second layer is the local cache of every file and document attached to the project. Whenever the client uploads a PDF, a docx, a screenshot, or attaches a Basecamp document to a todo, the daemon fetches it the next time it polls. Then:

- PDFs are run through `pdftotext` and saved as markdown.
- DOCX files are run through `pandoc` and saved as markdown.
- Images stay as-is. The LLM has vision; nothing's gained by OCR.

Each file lives in `memory/basecamp-files/<attachment_id>/` with the original plus the parsed markdown side by side. The state file `acme-files-state.json` tracks which files have already been processed so we don't re-fetch on every tick.

What this gets you: when the agent reads a todo that says "implement the changes from the attached spec," the spec is already on local disk in a format the LLM can search through. The agent doesn't have to download anything, doesn't have to fight Basecamp's authenticated file URLs, doesn't waste a tool call on retrieval. It just runs `memory_search` and gets the relevant chunk.

The "edits to existing files get caught" question matters here. The morning sync (the same 07:00 job that refreshes the canonical doc) also runs `fetch-basecamp-files.mjs --refresh-all`, which re-parses everything. If the client edits the spec doc, by the next morning the cache reflects the edit. If I need it immediately, I run that command manually.

## Layer 3: the agent's own learnings

The third layer is the agent's running notebook. Every successful dispatch ends with a call to `update-kb.mjs`, which writes one file:

```
memory/kb/2026-05-21-todo-12345678.md
```

Inside is a structured entry: what the todo was, what the agent did, what it learned, what tripped it up, and any "if you see this pattern again, do this" notes. The agent writes this itself, with a template, at the end of each dispatch.

This is where you get compound improvement over time. The first time the agent does an OTP login screen, it has to figure out a bunch of things: where the auth navigator lives, which validator style the project uses, where the error toasts are defined, how the existing tests are structured. By the end of the dispatch, those answers are in a KB file. The *next* time anyone asks for any auth-related work, the agent searches the KB, finds the file, and skips the discovery phase.

Two things are essential for this layer to work:

**The agent writes the KB itself.** I don't curate it. The agent has a template, a budget of a few sentences per section, and instructions to write down "what would I want to know if I picked this todo up cold three months from now." Self-curation is the only way it scales — if I had to write KB entries for every completed todo, I'd never write any.

**The KB is indexed for semantic search, not keyword search.** I use `openclaw memory index --agent acme` to push the KB into an embedding store via Gemini's embeddings. The agent searches with `memory_search`. The reason this matters: nobody phrases the second occurrence of a problem the same way they phrased the first. "Add OTP login" and "we need a phone-number based sign-in flow" describe the same work, and keyword search would miss the match.

## How the three layers compose

When a dispatch starts, the prompt tells the agent to do two reads before doing any work:

1. Read attached documents via `memory_search`. (Layer 2.)
2. Read the canonical "Initial knowledge for AI Agents" doc via `memory_search`. (Layer 1.)

Layer 3 isn't explicitly mentioned in the read step, but it's in the same embedding index — when the agent searches for anything related to the current task, KB entries surface alongside parsed files and the canonical doc. The agent doesn't need to remember to look at it; it's just there.

The hierarchy when these layers conflict:

- Layer 1 (canonical doc) overrides everything. If the doc says "use Twilio," and an old KB entry from before Twilio was decided says "use MessageBird," Twilio wins.
- Layer 2 (parsed files) is task-specific. A spec attached to *this todo* takes priority over generic project knowledge for *this dispatch only*.
- Layer 3 (agent KB) is a hint, not a rule. The agent treats KB entries as "things that worked last time" rather than as authoritative. If the canonical doc has changed since the KB entry was written, the doc wins.

I don't enforce this hierarchy programmatically — I just describe it in the canonical doc itself. The doc says "you override SOUL.md, MEMORY.md, and KB entries on conflict." The agent reads that and behaves accordingly. So far it's worked.

## The morning sync

The thing that keeps all three layers fresh is a single shell script that runs at 07:00 every day:

```
~/Library/LaunchAgents/com.openclaw.acme-morning-sync.plist
StartCalendarInterval Hour=7 Minute=0
```

The script does three things:

```bash
# 1. Re-parse every file and document in the project's vaults
fetch-basecamp-files.mjs --refresh-all

# 2. Rebuild the embedding index from all three layers
openclaw memory index --agent acme

# 3. Send a heartbeat to my Telegram so I know it ran
send-heartbeat.sh "morning-sync OK"
```

That's it. The whole "is the agent's knowledge fresh?" question reduces to "did the morning sync run?" If yes, the agent knows everything I told it through the canonical doc, every file the customer uploaded, and every lesson the agent learned by 23:59 yesterday.

The heartbeat is the lazy version of monitoring. If I don't get the Telegram message at 07:00 sharp, something is wrong with the daemon host. Two missed heartbeats and I check the Mac mini.

## Disaster recovery, courtesy of git

Now the part that lets me sleep at night: every successful dispatch ends with a git commit.

After the agent updates the KB (layer 3), the dispatch runs:

```bash
bash /Users/andreaskviby/myopenclaw/sync.sh \
  --message "feat(acme): complete todo <id> — <short title>"
```

`sync.sh` rsyncs the workspace and global scripts into a separate repo (`myopenclaw`), strips out tokens and secrets, then commits and pushes to GitHub.

So at any moment, the GitHub repo contains:

- The full agent workspace (every script, every config, every state file shape).
- The KB folder (every learning the agent has ever written).
- The OpenClaw config template (with secrets stripped).

What's *not* in the repo:

- The encrypted Basecamp token. That stays in the macOS keychain.
- Telegram bot tokens. Those stay in `openclaw.json` on the local machine.
- Anything else with a credential in it.

What this gets me: **if my Mac mini dies tomorrow, recovery is twenty minutes.** Clone the repo onto a new machine. Re-authenticate the Basecamp OAuth (one browser tab). Restore the Telegram tokens (one config edit). Reload the LaunchAgents. The system is back, with all of its memory intact.

The Flutter app code itself lives in a different repo, `acme-org/acme-app`, where the agent works inside `~/Code/acme/flutter_app/`. Each code todo creates a branch and a pull request. The client reviews and merges. The agent never force-pushes, never rewrites history, never touches other agents' PRs. The split between "agent infrastructure repo" (my personal `myopenclaw`) and "product code repo" (the org's `acme-org/acme-app`) matches the split between Auto Andy's brain and Auto Andy's work product. Two different things, two different audit trails.

## What I'd add if I were starting fresh

A few items I haven't built and probably should:

- **A "forget this" command.** Sometimes a KB entry encodes a lesson that's no longer true. Right now I delete the file manually and re-index. A small `kb-forget --pattern "twilio"` command that finds matches, shows them, and lets me confirm-deletion would be cleaner.
- **Periodic KB compaction.** After 500 todos, the KB has 500 files, many of them on overlapping topics. A monthly job that asks an LLM to merge near-duplicate entries into a single canonical one would keep the corpus tidy. I haven't built this and the search quality is starting to degrade slightly because of it.
- **A KB "promotion" path.** When a KB entry encodes a fact that should actually be permanent (like "we use Twilio"), it should get promoted into the canonical Basecamp doc. Right now this happens by me reading the KB and editing the doc by hand. It could be a quarterly review todo the agent files on itself.

## The summary, for the whole series

Five posts, one system. Here's the whole thing in one paragraph:

The client files a Basecamp todo and assigns it to Auto Andy. Within ten minutes, a daemon picks it up, records the dispatch in a state file before spawning the agent, and runs the LLM with an eight-minute timeout and a five-tool-call budget. The agent reads the todo's attachments, the canonical knowledge doc, and any prior KB entries (all three indexed in one embedding store), does the work, comments on the todo, completes it, writes a new KB entry, and commits everything to a GitHub repo. If the agent gets stuck, it files a "Things I miss" blocker addressed to the customer, comments "Blocked" on the original, and stops — a separate 90-second monitor wakes it back up when the customer answers. Every morning at 07:00, all three memory layers refresh. Every afternoon at 16:00, a consolidated daily report goes to me on Telegram. Most days, I don't have to look at it.

That's the whole machine.

Thanks for reading the series. If you build something similar — or you've been doing this for a while and you've solved problems I haven't — I'd love to hear about it. The interesting unanswered questions in this space, in my opinion, are: how do you handle multiple agents collaborating on the same project without stepping on each other; how do you decide which decisions get promoted from KB into the canonical doc; and how does this all change when the LLM gets cheap enough that the ten-minute poll becomes a thirty-second poll. Those are my next three posts, probably.

— Andreas
