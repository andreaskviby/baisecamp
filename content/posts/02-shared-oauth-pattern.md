---
title: "One OAuth app for a whole fleet of bots: the shared-auth pattern"
series: "AI ↔ Basecamp, part 2 of 5"
date: 2026-05-05
tags: ['basecamp', 'ai-agents', 'oauth', 'architecture']
---

I run several AI agents that talk to Basecamp. There's Auto Andy on the Acme project. There's another agent on a different product. There's a third one I'm prototyping. Each does different work, in a different workspace, with a different personality.

There's exactly **one** OAuth integration registered at 37signals.

If you're starting down this path, this is the post I wish I'd had on day one. It saves you from a sprawling mess of "Basecamp Bot #4 (Andreas)" entries in your account settings, and it solves a refresh-token gotcha that I hit twice before I sat down and fixed it properly.

## Why not one OAuth app per agent

The naive design is: each agent registers its own integration, gets its own client id and secret, manages its own tokens. It's the textbook approach. And it's wrong for this use case, for three reasons:

1. **Maintenance scales linearly with agents.** Every new bot is a new app to register, a new pair of secrets to store, a new place where tokens can rot.
2. **The audit trail fragments.** Each bot writes under a different OAuth-app name in the Basecamp activity feed. From the client's perspective, half a dozen "apps" are leaving comments. It looks like an invasion.
3. **Rate limits are per-app.** 37signals rate-limits by OAuth app. Splitting your bots across apps means each one runs into the wall on its own. Sharing one app pools the budget.

The fix is to register one OAuth app in your name, store the encrypted token in one well-known place, and have every agent read from the same place.

## The directory layout

Everything auth-related lives in one folder, shared by every agent:

```
~/.openclaw/shared/basecamp-auth/
├── README.md         # The rules (below)
├── curl.sh           # Thin wrapper: refresh token, then curl <args>
└── refresh.mjs       # Reads the encrypted store, refreshes if expired, prints JSON
```

The encrypted token store itself is at `~/Library/Preferences/basecamp-cli-nodejs/config.json`. I let the existing 37signals CLI manage that file — no point reinventing keychain handling.

## The two entry points

**Shell** — for scripts:

```bash
~/.openclaw/shared/basecamp-auth/curl.sh /projects/47228140.json
~/.openclaw/shared/basecamp-auth/curl.sh /buckets/47228140/todolists/X/todos.json \
  -X POST -H 'Content-Type: application/json' -d '{"content":"foo"}'
```

`curl.sh` refreshes the token if it's near expiry, then forwards everything to `curl` with the right `Authorization` header. From the script's perspective, it's just curl.

**Node** — for in-process callers:

```js
const out = execFileSync(
  process.env.HOME + '/.openclaw/shared/basecamp-auth/refresh.mjs',
  [],
  { encoding: 'utf8' }
);
const { ok, accessToken, accountId } = JSON.parse(out);
```

`refresh.mjs` does the same job and prints JSON. Any Node script in any agent's workspace can shell out to it without pulling in a library.

The key insight: **every caller goes through the same two files**. No agent ever talks to the OAuth endpoint directly. No agent ever holds a hardcoded token. If I need to rotate credentials, I rotate them in one place.

## The house rules

I have these posted in the README in the auth folder, because future-me forgets and needs to be reminded:

1. **Never register a second OAuth app.** If you think you need one, you don't.
2. **Never hardcode an access token in code or env.** The token in the encrypted store is the only one. Hardcoded tokens age out and become incident reports.
3. **Set a per-project `User-Agent`.** 37signals enforces this — they want to know which of your integrations is making the call. `Acme (andreas@stafegroup.com)` is what mine looks like.
4. **On 401: refresh once, retry once.** Still 401? The refresh token is dead. Stop, page the human.
5. **Don't refresh in a tight loop.** The refresh token rotates on every call. If you re-refresh every 30 seconds because you didn't cache the result, you'll rotate yourself out of an account in an afternoon.

Rule 5 is the one that bit me. The first version of `curl.sh` called `refresh.mjs` on every invocation. Every refresh rotated the refresh token. Every rotation invalidated the previous one. When two scripts ran in parallel, one of them would get an old refresh token, fail, and the whole system would lock up until I went into my password manager and reset things manually. The fix is to cache the access token until it's near expiry, only refresh when needed, and serialize refreshes through a lockfile if you have concurrent callers.

## The war story: the `api_write` 400

Now the fun part. There's a parallel path I want to mention because it's a great lesson in framework abstractions.

I have a plugin installed at `~/.openclaw/extensions/openclaw-basecamp/`. It exposes ten convenience tools to the agent: `basecamp_create_todo`, `basecamp_complete_todo`, `basecamp_add_boost`, and so on. Nine of them work perfectly. One of them — `basecamp_api_write` — returns HTTP 400 every single time you call it with a non-empty body.

This is `basecamp_api_write`'s whole job. It's the generic write tool, the one you reach for when there isn't a dedicated convenience method. POST, PUT, PATCH, with a JSON body. And every body-bearing call comes back 400.

I spent half a day on this. I assumed it was Basecamp's API. I tested the same payload via `curl.sh` — worked perfectly. I tested via Postman — worked perfectly. I tested via the plugin — 400.

The bug turned out to be in the OpenClaw runtime, not the plugin and not Basecamp. The plugin's tool schema declared the body parameter as `Type.Unknown` (because the body can be any JSON shape — that's the whole point of a generic write tool). The runtime, in the middle of doing input validation, was stripping out fields it didn't recognize. `Type.Unknown` apparently triggered the strip. By the time the plugin received the call, the body was an empty object. Basecamp dutifully returned 400 because you can't POST an empty body to most of its endpoints.

The fix in the runtime is a one-liner: don't strip `Type.Unknown`, pass it through. The workaround until then is: **use `curl.sh` for any body-bearing write**. That's what every script in my agent's workspace does. The convenience tools that don't take arbitrary bodies (mark a todo complete, add a boost) work fine. The generic write tool is bypassed entirely.

Lessons:

- When a framework abstraction "works for nine cases and fails for the tenth," the bug is almost always in the abstraction's schema layer, not in the underlying API.
- Always keep a low-level escape hatch. `curl.sh` saved my week. If I'd built everything on top of the plugin, this bug would have blocked me for days. Because the daemon and every script use the raw wrapper, the plugin can be broken and the system still works.
- Good abstractions are reversible. You should be able to drop one layer and use the next layer down. If you can't, you've over-abstracted.

## What you actually need to build this for yourself

If you're setting up the equivalent for your own AI agents — Basecamp, GitHub, whatever:

1. **One OAuth app per service, registered in a real human's name.** The human is the accountable party. The bots act under their account.
2. **One encrypted token store on disk.** Use the platform's keychain if you can. macOS Keychain, Linux Secret Service, Windows DPAPI. If you can't, encrypt the file with a passphrase that's not in the repo.
3. **One refresh script that knows how to cache and serialize.** Cache the access token until expiry-minus-5-minutes. Lock around refreshes so concurrent calls don't fight.
4. **One thin curl wrapper.** Every other tool calls this. Don't let agents talk to the OAuth endpoint directly.
5. **Per-project User-Agent.** Even when sharing one app, distinguish callers in the User-Agent so the activity feed makes sense.
6. **A keep-the-escape-hatch rule.** No matter how good your higher-level tools get, the raw wrapper stays usable. The day a tool returns a mysterious 400, you'll be glad it's there.

That's the entire auth architecture. Six files, one OAuth registration, no per-bot maintenance. Add a new agent tomorrow and it inherits everything.

---

*Next in the series: the polling daemon itself — how the 10-minute tick, the 8-minute dispatch cap, and the one-line state file trick prevent the whole system from running away in a loop.*
