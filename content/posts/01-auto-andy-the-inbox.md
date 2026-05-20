---
title: "Auto Andy: why my Basecamp assistant is a separate person from the AI that does the work"
series: "AI ↔ Basecamp, part 1 of 5"
date: 2026-05-01
tags: ['basecamp', 'ai-agents', 'architecture']
---

I run an AI agent that picks up Basecamp todos, does the work, posts the result back to the todo, and marks it complete. Customers file tasks the same way they always did. They just have a new teammate to assign them to.

That teammate is called Auto Andy.

Here's the thing most people get wrong when they wire an LLM into a project management tool: they make the LLM the assignee. They give it a name, a profile picture, a "personality," and they treat it as if the model and the worker were the same entity.

They're not. And keeping them separate is the single most important design decision I've made on this system.

## The split

In my Basecamp account there is a person called **Auto Andy**. He has a Basecamp user id (`51469602`), an email (`akviby@hey.com`), and he can be assigned todos like any other teammate. He is, for all practical purposes, an inbox.

Somewhere else entirely — on a Mac mini in my office, running under launchd — there is a daemon. Every ten minutes it asks Basecamp: "What's been assigned to Auto Andy?" When it finds something new, it spawns an LLM dispatch to handle it. That dispatch is the *worker*. The model running it might be Opus, it might be GPT, it might be Sonnet on a cheap day. The customer never sees the model. They see Auto Andy.

The mental model is:

- **Auto Andy** = the Basecamp identity. The assignee stamp. The inbox.
- **The agent** = the LLM process that does the actual work. Ephemeral, swappable, model-agnostic.

These are two different things and they live in two different places, and once you internalize that, a lot of design problems get easier.

## Why this matters

**Assignment becomes the trigger.** The customer doesn't @-mention a bot. They don't paste a special prefix. They don't go to a separate UI. They assign a todo to Auto Andy the same way they'd assign it to me. If they want me to do it instead, they assign it to me. If they want both of us to look at it, they assign it to both. The mental load on the customer is zero — it's just Basecamp.

**Reassigning is a kill switch.** This is the one I love. If a dispatch is mid-flight and the customer realizes they don't want the AI to handle it, they reassign the todo to a human. The next time the daemon ticks, the filter ("assigned to Auto Andy AND not already dispatched") excludes it. The agent stops touching it. No special "cancel" button, no admin panel, no "are you sure?" modal. Just Basecamp's existing UI.

**The agent has no Basecamp identity.** Everything the agent writes — comments, completions, new todos — appears under my name, because the OAuth token belongs to me. This is honest. The agent isn't pretending to be a person. It's doing work as me, on my behalf, and the audit trail is clean: every write is traceable to a human owner.

**Model swaps are invisible.** When I switched the default model chain last month, no customer noticed. Auto Andy didn't change. His name didn't change. His behavior didn't change. The thing behind him got cheaper and faster. That's the whole point of the separation — the *interface* (a Basecamp person) is stable, while the *implementation* (which LLM, in what process, on what machine) is free to evolve.

## The implementation pattern, generalized

If you're building an AI worker for any project management tool — Basecamp, Asana, Linear, Jira, ClickUp — the same split applies:

1. **Create a user account** in the tool that represents the "inbox." Not a bot account, not a webhook integration — an actual person record with an actual user id.
2. **Make assignment the trigger.** Don't invent a new gesture. Use the gesture the tool already has.
3. **Run the worker somewhere else.** A daemon, a serverless function, a queue consumer. The worker reads from the inbox, does the work, writes back as the OAuth owner.
4. **Make the worker stateless.** Each task is one dispatch. The model can be anything. The framework can change. The inbox stays.

I've seen teams build elaborate Slack-bot personalities, custom dashboards, dedicated AI command centers. All of that adds surface area. The Auto Andy pattern adds zero.

## What this *doesn't* solve

I want to be honest about the limits. The split gets you a clean trigger, a clean kill switch, and a clean separation of concerns. It does not get you:

- **Conversations.** If the agent needs to ask the customer something, the assignment model isn't enough. (I solve this with a thing called "Things I miss" — a separate todolist where the agent files blockers back at the customer. That's post 4 in this series.)
- **Identity per agent.** I run multiple agents on the same Basecamp account. Auto Andy is just one of them. Each one needs its own inbox-person and its own scope rule. That's a maintenance question, not a design one.
- **Permission boundaries.** The OAuth token grants access to every project on the account. The fact that the agent only *should* touch certain projects is enforced by config, not by the auth layer. (More on that in the final post.)

## The minimum mental model

The client files a todo in "Flutter App" and assigns it to Auto Andy. Ten minutes later, an LLM I'll never see picks it up, does the work, comments on the todo, completes it, and goes back to sleep. From the client's side it looks like they assigned work to a coworker and the coworker did it.

That's the whole illusion. And the trick to making the illusion stable is to never let the LLM be the coworker. The coworker is Auto Andy. The LLM is just whatever's behind the curtain this week.

---

*Next in the series: how I authenticate the whole fleet through one shared OAuth integration — and the war story of the one Basecamp API tool that kept returning 400 for reasons that took two days to diagnose.*
