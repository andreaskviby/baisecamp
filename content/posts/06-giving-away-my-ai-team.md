---
title: "I'm giving away my 94-person AI team"
date: 2026-05-28
tags: ['ai-agents', 'open-source', 'prompts', 'team']
---

Over the past year I've built an AI product called [Ullabella](https://ullabella.ai). It's an executive assistant for Swedish small businesses — the kind of thing that reads your email, manages your calendar, and knows when to delegate work to specialists.

The thing that makes it useful isn't the code. It's the team.

## 94 colleagues, each with a job

Ullabella has 94 AI "colleagues." Each one is a markdown file containing a detailed persona, a set of capabilities, rules for when to delegate to other colleagues, and specific domain knowledge.

There's **Glenn**, the internationalization expert with 30 years of export experience. He knows the difference between setting up a subsidiary and working through an agent. He'll tell you that Germans expect longer sales cycles and that Business Sweden has an office in your target market.

There's **Rolf-Arne**, the travel assistant who has direct API access to Sweden's public transport systems. He doesn't guess at bus times — he queries ResRobot.

There's **Selma**, the video producer who knows which AI model to use for vertical Reels versus horizontal YouTube content.

There's **Patent-Per** for intellectual property, **Berit** for procurement, **Bo** for accounting, **Liv** for social media, **Karin** for weather (she talks to SMHI), **Bertil** for company registrations (he queries Bolagsverket).

The main agent, **Ullabella** herself, is an experienced executive secretary — warm, efficient, 60 years old, with decades of experience. She knows when to handle something herself and when to delegate to a specialist.

Each colleague file is 100-200 lines of carefully tuned prompt. Not vague instructions — specific personality, specific expertise, specific rules for collaboration.

## They're in Swedish. That's fine.

The entire team is written in Swedish. The personas are Swedish names. The cultural references are Swedish. The API integrations talk to Swedish government services.

But here's the thing: these files are prompts. They translate. You can run them through any LLM and get an English version in seconds. The *structure* — the way I've organized personas, capabilities, delegation rules, and domain knowledge — that transfers directly.

If you're building AI agents for a different market, you don't need my Swedish colleagues. You need the pattern. How do you make an AI specialist feel like a real expert? How do you make delegation work? How do you prevent the main agent from hallucinating about topics it should hand off?

The answers are in the files.

## Why I'm giving them away

I've been writing this blog series about connecting AI agents to Basecamp. The whole point is to share what I've learned. Keeping 94 persona files locked up while writing about architecture patterns feels hypocritical.

Also: I want to see what other people build. The colleague pattern works. I know it works because I use it every day and my customers use it every day. But I've only built it for one market, one language, one set of integrations. Someone else might take the same structure and build something I never thought of.

## What you get

A zip file containing 94 markdown files. Each file has:

- **YAML frontmatter**: name, category, profession, avatar, color, gender, age, keywords, system flags
- **Personality section**: who they are, how they talk, what makes them tick
- **Capabilities section**: what they can actually do, what tools they have access to
- **Rules section**: when to delegate, when to ask for help, what they should never do
- **Collaboration section**: which other colleagues they work with and how

The main agent (Ullabella) has extensive rules for delegating to specialists. The specialists have rules for when to escalate back. The whole thing is designed to feel like a real team, not a single AI pretending to know everything.

## What you don't get

You don't get the Ullabella application. You don't get the code that loads these files, routes conversations, handles tool calls, or manages the delegation system. You don't get the API integrations to Swedish services.

You get the prompts. The personas. The structure.

If you're building your own agent system, that's probably what you need anyway. The infrastructure is the easy part. The hard part is making agents that feel competent and know their limits. That's what's in these files.

## Download

**[colleagues.zip](/colleagues.zip)** (293 KB, 94 markdown files)

The files are MIT licensed. Do whatever you want with them. Translate them, adapt them, build a competing product, I don't care. If you build something cool, I'd love to hear about it.

---

*This is a side note from my [AI + Basecamp series](/tags/basecamp/). If you're interested in the architecture of connecting AI agents to project management tools, start with [part 1: Auto Andy and the inbox](/posts/01-auto-andy-the-inbox/).*
