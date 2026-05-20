---
title: "Things I miss: how an AI agent asks the customer a question and waits"
series: "AI ↔ Basecamp, part 4 of 5"
date: 2026-05-14
tags: ['basecamp', 'ai-agents', 'blockers', 'architecture']
---

Most "AI does your tasks" demos cheat. They pick tasks the AI can complete in one shot, given the input it already has. The tasks are clean. The instructions are unambiguous. The agent runs, produces output, and the demo ends.

Real work isn't like that. Real work has half-specified todos. The customer writes "Add OTP login" and forgets to mention which phone-number library they want to use, which provider sends the SMS, what the OTP length should be, and whether the screen lives inside the existing auth navigator or in a new one. A good human teammate asks. A bad one guesses and gets it wrong.

My agent asks. This post is about how.

## The pattern, in one paragraph

When the agent hits a question it can't answer from the todo, the attachments, and the knowledge base, it does three things: it creates a new todo on a special todolist called **"Things I miss"** addressed to the customer with the specific question; it comments on the original todo with a link to the question and the word "Blocked"; and it stops. It does not mark the original complete. It does not guess. It does not retry.

Then a separate background process — the **blocker monitor** — wakes up every 90 seconds, checks whether any blocker todo has a new comment or has been completed, and if so, re-dispatches the agent with the original task plus the customer's response embedded in the prompt.

That's the whole pattern. It's not complicated. The reason I'm dedicating a whole post to it is that nobody else seems to be doing it, and I think it's the difference between a demo and a production system.

## Why most AI agents skip this

The honest answer: it's annoying. Blocking is a state. State means storage. Storage means a state file, a poll loop, a re-trigger path, a way to handle "the customer never answered," a way to handle "the customer answered with a question of their own." All of that is plumbing, and plumbing is unglamorous.

The dishonest answer, which is also true: many AI agent demos are explicitly built to avoid this case. They run on benchmarks where the input is sufficient. The day you put one in front of a real customer, the input stops being sufficient.

So you have three options:

1. **Guess.** The agent fills in the blanks and ships something. Sometimes right, often wrong. When it's wrong, the customer has to manually undo and explain.
2. **Fail.** The agent says "I don't have enough information" and gives up. The customer reads this, has to remember what they were trying to do, has to figure out what was missing, has to re-file the todo. Worst of all worlds.
3. **Ask.** The agent files a structured question, links it to the original, and pauses. The customer answers when they can. The agent resumes.

Option 3 is the only one that scales. And it requires a real implementation, not a clever prompt.

## The "Things I miss" todolist

Inside the project, alongside the working todolists (in my case "Flutter App"), there is one more list called "Things I miss." Its sole purpose is to hold blockers.

Some properties of this list that matter:

- **Customers see it natively.** It's a Basecamp todolist. No new tool, no separate app, no email, no "AI inbox." Same UI they use for everything else.
- **Items are assigned to humans, not to Auto Andy.** A blocker is a question for the customer. Assigning it to Auto Andy would create a circular reference where the agent is waiting on itself.
- **The blocker todo's title is the question.** "Which SMS provider should we use for OTP? Twilio, MessageBird, or a Swedish provider like 46elks?" Not "blocker on todo #12345" — the question itself. The customer should be able to answer from the title alone, in two seconds.
- **The blocker's description has context.** A link to the original todo, a paragraph summarizing what the agent was trying to do, and a clear list of what's needed. If the customer needs more, the link is right there.

When the agent files a blocker, it also comments on the original todo: "Blocked — see [link]." That comment is the breadcrumb that lets the customer trace from one to the other.

## The state file

When a blocker is created, it gets recorded in `acme-blockers-state.json`:

```json
{
  "98765432": {
    "todo_id": 98765432,
    "original_task_id": 12345678,
    "last_seen_comment_id": null,
    "created_at": "2026-05-21T09:14:33.000Z"
  }
}
```

The three fields that matter:

- `todo_id`: the blocker todo's own id, the one on "Things I miss."
- `original_task_id`: the todo the agent was working on when it got stuck.
- `last_seen_comment_id`: a cursor. Initially null. After every poll, updated to the most recent comment seen.

The original task id is the load-bearing field. Without it, when the customer answers the blocker, the monitor wouldn't know what work to resume. With it, the resume is trivial.

## The monitor loop

A second launchd process polls every 90 seconds:

```
~/Library/LaunchAgents/com.openclaw.acme-blocker-monitor.plist
StartInterval = 90
```

Why 90 seconds and not 10 minutes like the main daemon? Because the blocker monitor is a *conversation*. The customer is actively waiting. A 10-minute delay between "I answered" and "the agent resumed" would feel slow. 90 seconds feels like the agent is paying attention. Faster than that and I'd hit Basecamp's rate limit on this many idle pollers running in parallel.

Each cycle, for every active blocker, the monitor checks two things:

### 1. New comments since `last_seen_comment_id`

If the customer (or anyone else) added a comment to the blocker since the last cycle, the monitor:

- Captures the new comment text.
- Updates `last_seen_comment_id` to the new most-recent id.
- Re-dispatches the agent with a prompt that includes:
  - The *original* task's context (id, title, attachments).
  - The blocker's title (the question).
  - The new comment text (the answer).
  - An instruction to resume the original task with this new information.

The agent doesn't see this as "a new task." It sees it as "you got stuck on task X, the customer said Y, now finish X."

### 2. Completion state flipped to `completed`

A customer who's a little impatient — or who answered in the title bar by editing — sometimes just marks the blocker complete. The monitor treats completion as an implicit "I'm done answering, continue." It re-dispatches the agent with the same context but without a new comment.

The agent has to be robust to this. The prompt template says "if the comment field is empty, assume the customer answered by editing the blocker title or by completing it; re-read the blocker title and proceed."

## The handoff prompt

Here's the structure of the prompt the monitor builds when re-dispatching:

```
You previously got stuck on todo <original_task_id> ("<original title>")
because you needed information you didn't have.

You filed a "Things I miss" blocker (id <blocker_id>) with the question:

    "<blocker title>"

The customer has now responded:

    "<new comment text>"

(or: "The customer marked the blocker complete without comment;
re-read the blocker title and proceed.")

Resume the original task using this new information. Follow the
standard dispatch workflow (read KB, do work, comment, complete,
update KB). If you find you need still more information, you may
file *one more* blocker — but only one. If two consecutive blockers
fail to unstick you, comment on the original task explaining the
situation and stop.
```

The "only one more blocker" rule is a debouncer. It prevents an agent from incrementally extracting a full spec one question at a time, which would feel awful to the customer. If the first answer isn't enough, the agent gets one more shot, then it has to either ship something or fail loudly.

## Why this works

Three reasons.

**It uses Basecamp's existing primitives.** Todos, todolists, comments, assignments. The customer doesn't learn anything new. They don't install anything. They don't visit a new URL. The "AI is asking me a question" experience feels like "a teammate filed a todo on me," which is exactly the right mental model.

**It separates question from answer in storage.** The blocker is a discrete todo. It has its own id, its own URL, its own comments. When the customer answers and the blocker gets resolved, there's a clean audit trail: who asked what, when, and what was answered. Compare this to threading questions inside the original todo as comments, which produces an unreadable wall when there are multiple back-and-forths.

**Resuming is automatic but bounded.** The monitor handles the resume without human intervention, so the customer never has to remember to "re-trigger" anything. But the bound on consecutive blockers stops the agent from playing an indefinite game of twenty questions.

## What I'd change

Now that the pattern has been in production for a while, here's what I'd do differently if I were rebuilding it:

- **Persist the conversation, not just the cursor.** Right now the agent gets the latest comment as a single string. If there are three back-and-forth comments on the blocker, the agent only sees the latest. I should pass the full thread. It would help when the customer says "yes" and the question was "should we use option A or option B?" — without the full thread, the agent loses the context.
- **Auto-close stale blockers.** A blocker that's been open for 30 days probably isn't getting answered. The monitor should auto-close those, comment on the original ("Blocker timed out — please re-file when ready"), and remove them from the state file. Right now they just sit around.
- **Allow multiple parallel blockers per task.** Currently, if the agent needs to ask two unrelated questions about the same todo, it files them sequentially. It would be cleaner to file both at once, with the original task resuming only when both are answered. This is a fan-in pattern and it's a little more complex, but the customer experience would be better — they answer both questions in one sitting instead of being pinged twice.

None of these are blockers (heh) to the system working. They're polish items I'll get to when I'm not busy shipping the next thing.

## What this gets you, philosophically

The "Things I miss" pattern is the part of the system that turns the AI from a one-shot task runner into a *collaborator*. A collaborator is something that can say "I don't know — please tell me," without that being a failure. A one-shot task runner can't do that; the moment it stops, the customer has to do the rest.

I think this is the gap most AI agent products are going to have to cross in the next year. The agent that doesn't know how to ask a question is the agent the customer eventually stops trusting. The agent that asks well — with the right amount of context, in the right format, at the right cadence — becomes the agent the customer files everything to.

Auto Andy asks well. That's most of why it works.

---

*Next, the final post in the series: the three layers of memory that keep the agent smart across dispatches, how I edit the knowledge base directly in Basecamp's UI, and the git sync that means losing my Mac mini is a twenty-minute recovery, not a catastrophe.*
