# Hermes Team Brain

A shared team memory in Slack, built on [Hermes Agent](https://github.com/NousResearch/hermes-agent)
(MIT, Nous Research) with [Mem0](https://mem0.ai) as the memory provider.

**This repo is configuration, not application code.** Hermes Agent already
implements the Slack bot, the memory-provider layer, session handling, and emoji
triggers. What lives here is the config that shapes it into a team brain, plus
preflight checks and the operational docs your team needs.

---

## Table of contents

- [What this actually is](#what-this-actually-is)
- [Architecture](#architecture)
- [Setup](#setup)
- [Configuration reference](#configuration-reference)
- [Daily use](#daily-use)
- [Operations](#operations)
- [Design decisions and known limits](#design-decisions-and-known-limits)
- [Troubleshooting](#troubleshooting)
- [Verification status](#verification-status)

---



## Repo layout



There is no application code. Hermes Agent supplies the bot; this repo supplies
configuration, guardrails, and docs. The Python here is tooling — preflight
validation and manifest patching — which is why `pyproject.toml` declares no
runtime `dependencies`. **Hermes is a prerequisite, not a dependency:** nothing
imports it, it ships its own venv, and installing a second copy into this
project would create two writers against one `$HERMES_HOME`.

---



## What this actually is

People hear "second brain" and expect perfect recall — *"what did Priya say
about the vendor contract in March?"* Mem0 cannot answer that. It extracts and
consolidates **facts**, lossily and by design. If that's the only thing behind
the bot, the first verbatim question fails and the team stops trusting it.

So there are two tiers, and Hermes gives you both:


| Tier       | Backed by                                                                       | Answers                                                     |
| ---------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Recall** | Hermes' built-in session search (SQLite + FTS5, no summarization or truncation) | "what exactly was said, when, by whom"                      |
| **Memory** | Mem0                                                                            | "what do we believe, what did we decide, what do we prefer" |


Set expectations at rollout accordingly. Conflating the two is the most common
way this kind of project disappoints.

## Architecture

```
Slack workspace
      │
      │  Socket Mode (WebSocket — no public URL, works behind a firewall)
      ▼
Hermes gateway  ──────────────────────────────────────────────┐
      │                                                        │
      ├─ allowed_channels        WHERE the bot may respond      │
      ├─ SLACK_ALLOWED_USERS     WHO may trigger it             │
      ├─ reaction_triggers       🧠 / 📌 capture path           │
      │                                                        │
      ▼                                                        ▼
 Memory provider layer                              Built-in memory
      │                                             (MEMORY.md / USER.md,
      │  auto: prefetch before turn                  always active)
      │        sync after response                        +
      ▼        extract on session end               Session search (FTS5)
   Mem0
   user_id: <your-team-id>
   agent_id: hermes
```

Only **one** external memory provider runs at a time. The built-in memory layer
runs alongside it and is never replaced.

---



## The trust boundary

> Read this section before you invite the bot anywhere that matters.



### Mem0's scope model has no channel dimension

The entire scope config is `user_id`, `agent_id`, and `rerank`. No channel
field, no author field. `mem0_search` accepts no channel filter. The four tools
are `mem0_search`, `mem0_add`, `mem0_update`, `mem0_delete`.

Meanwhile the provider layer **prefetches relevant memories before each turn**
and **syncs conversation turns after each response**, automatically.

**One Hermes instance is one flat memory pool.** Anything the bot sees in any
allowed channel can surface in any other allowed channel. No configuration
changes this, because the plugin has no channel concept to configure.

As a workaround I have created separate profiles to isolate each individual hermes by way of profiles (see figjam [Figma Diagram](https://www.figma.com/board/F4YoBpwMgrA7gSENRKJY4A/Brain?node-id=0-1)). This is also a way to contain the context between different brain agents within one instance (see figjam [Figma Hermes Instance and Profiles Diagram](https://www.figma.com/board/pbq9Hw23SG636c06DXLdHT/Hermes?node-id=0-1)). Launching each instance and all profiles, if any to contain each agent within its own sandbox helps keep the agents' from straying out of its workspace to meddle in of=ther files/directories. Each agent acts as a buffer between users and agents in messaging platforms--searching,retrieving, editing, and deleting memories from the mem0 container. 

### The user allowlist is not a containment boundary

`SLACK_ALLOWED_USERS` controls **who can trigger** the bot. It does **not** keep
non-allowlisted people's words out of memory. At least three paths carry their
content in:

1. **Emoji capture.** The reaction arrives threaded under the reacted-to
  message *so the agent sees what was reacted to*, and the reactor becomes the
   message's user. An allowlisted person reacting 🧠 to anyone's message pulls
   that content into memory. This is the capture mechanism, not an edge case.
2. **Thread sessions.** On Slack, sessions are keyed by **thread, not by user** —
  multiple people in a thread share one conversation.
3. **Explicit requests.** The bot holds `channels:history`. "@hermes summarize
  this thread" does what it says.



### What that leaves


| Control                                 | What it actually does                                                             |
| --------------------------------------- | --------------------------------------------------------------------------------- |
| `allowed_channels`                      | **Real boundary.** Messages outside the list are dropped before any other gating. |
| Which channels you `/invite` the bot to | **Real boundary.** Slack's own model.                                             |
| `SLACK_ALLOWED_USERS`                   | Who can trigger. Convenience filter, not containment.                             |
| `channel_prompts`                       | Tone and behavior. A prompt is a suggestion, not a control.                       |
| `group_sessions_per_user`               | Session grouping. Does not partition memory.                                      |


**Never point two gateway processes at the same Hermes home.** Both write
memory automatically and each loads the other's writes into its system prompt at
session start, so two writers on one home compound each other's state until it
stops being anything you configured. Profiles exist to prevent this; agents that
need shared memory should use an external memory provider — which is exactly
what Mem0 is doing here.

**Need tiers?** Run separate Hermes **profiles**. Config-file providers like
Mem0 store `mem0.json` per `$HERMES_HOME`, so each profile gets its own
`user_id` and its own pool. Don't try to make one instance span trust
boundaries.

### Open issue to check before you rely on the allowlist

GitHub issue **#23778** — *"Gateway auth bypass — unauthorized user messages
processed despite 'Unauthorized' log"* — reports that the auth check logs the
violation but does not block; the agent processes the message and responds.
Filed against Telegram in May 2026, so the Slack adapter may differ.

**Check whether it's closed before treating the user allowlist as a security
boundary.** Also confirm `GATEWAY_ALLOW_ALL_USERS` is unset — it authorizes
everyone on every platform.

`scripts/preflight.py` checks for both.

---



## Setup



### Prerequisites

- Hermes Agent installed (`hermes --version`) — a prerequisite CLI, installed globally and separately
- [uv](https://docs.astral.sh/uv/) — manages the script environment (optional)
- Python 3.12 (pinned in `.python-version`; uv fetches it if you don't have it)
- A Slack workspace where you can create apps
- A Mem0 API key ([app.mem0.ai](https://app.mem0.ai)) — free tier is fine to start

No uv? `python3 -m pip install -r requirements-dev.txt` works too — the scripts  
fall back to system `python3` when uv isn't on PATH.



## Configuration reference

Everything is commented inline in `config/config.yaml`. The settings that most
affect behavior:


| Setting                      | Default                             | Why it matters here                                                                                                                                                                                           |
| ---------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `allowed_channels`           | none (unrestricted)                 | Your only real containment boundary. Never leave empty in production.                                                                                                                                         |
| `SLACK_ALLOWED_USERS`        | none (**denies all**)               | Every teammate needs their Member ID added by hand.                                                                                                                                                           |
| `require_mention`            | `true`                              | Keep it. Without it the bot ingests far more.                                                                                                                                                                 |
| `strict_mention`             | `false`                             | Set `true` to stop thread auto-engagement, where the bot remembers past mentions and follows up unprompted.                                                                                                   |
| `ignore_other_user_mentions` | `false`                             | Set `true` so the bot stays quiet when a message opens by @mentioning a human.                                                                                                                                |
| `reaction_triggers`          | absent                              | Must be a **list** (`[brain, pushpin]`). `true` only routes reactions on the bot's *own* messages.                                                                                                            |
| `group_sessions_per_user`    | `true`                              | Slack keys sessions by thread regardless, so this has less effect here than on other platforms.                                                                                                               |
| `rerank` (mem0.json)         | `false` — **template ships** `true` | Platform mode only. Improves relevance at the cost of a retrieval call. Retrieval is the binding quota here (see [Watching cost](#watching-cost)), so set it back to `false` if you start crowding your tier. |


---



## Daily use


| Action                     | Result                                                              |
| -------------------------- | ------------------------------------------------------------------- |
| `@hermes <question>`       | Answers from team memory, with permalinks                           |
| React 🧠 or 📌             | Captures that message as a durable fact                             |
| `@hermes remember <fact>`  | Explicit save                                                       |
| `/help`, `/status`, `/new` | Slash commands (replies are ephemeral)                              |
| `!help` in a thread        | Slack blocks slash commands in threads; `!` is the alternate prefix |


In channels the bot only responds when @mentioned. Once it's active in a
thread, follow-ups don't need a mention. In DMs it always responds.

---



## Operations



### Retiring a stale fact

Team memory rots in one specific way: **decisions get reversed.** "We're using
Postgres" is true in March and wrong in June.

```
@hermes that's outdated — we moved off Postgres in June
```

The channel prompt instructs the agent to call `mem0_delete` or `mem0_update`.
Verify it actually did; there's no automatic supersession in Mem0 OSS mode, and
the Platform's background consolidation pass isn't instant.



### Watching cost

#### Mem0

Mem0 Platform meters two quotas monthly: **add requests** and **retrieval
requests**, roughly 10k/1k (Hobby), 50k/5k (Starter), 500k/50k (Pro).

That 1:10 ratio assumes a write-heavy application. A Slack brain is the
opposite — reads vastly outnumber writes, and Hermes prefetches on *every turn*.
Thirty people asking five questions a day is ~3,300 retrievals/month, past Hobby
and crowding Starter. The $79 Growth tier was retired in July 2026, so the next
rung is $249/mo.

#### Token Usage

Base models should be chosen with much consideratin of the agent's task =. For the brain, we will be using claude sonnet 4.6. This task is not highly complex thus lower cost models would be sensible. I've hard coded context window compaction at 50% usage to mitigate context rot and token spend.

#### Runaway Token Spend

Agents can be wasteful token spenders and to help identify any token usage abuse by users, hermes keeps logs in `~/.hermes/sessions/sessions.json` (if no profiles) or `~/.hermes/profiles/<profile_name>/sessions/sessions.json` under "last_prompt_tokens", "total_tokens", and other metrics. This lists user logs in channels and sessions by user name, date, token spend, channel, thread id, etc.

**Retrieval is your binding constraint, not storage.** Watch it for two weeks
before committing to a tier or to Platform-vs-OSS.

---



## Design decisions and known limits



### Capture is passive, not emoji-only

The provider layer syncs every turn regardless of `reaction_triggers`. Emoji
triggers **add** an explicit capture path; they don't gate the automatic one.
If emoji-only capture is a hard requirement, that's a custom memory provider
(see Hermes' developer guide), not a config setting.

### You cannot combine Mem0 with Holographic

Memory providers are provider plugins — only one active at a time.

Even if you could, it wouldn't do what you'd want. Holographic's `contradict`
action queries its own SQLite table. If Mem0 holds your facts, Holographic's
store is empty and finds nothing. Conflict detection only works over a
*complete* store.

**If contradiction detection matters more than hands-off extraction**, consider
switching providers entirely:


| Provider        | Storage                | Cost        | Unique strength                                                                         |
| --------------- | ---------------------- | ----------- | --------------------------------------------------------------------------------------- |
| **Mem0**        | Cloud / self-hosted    | Free / paid | Server-side LLM extraction, automatic dedup                                             |
| **Holographic** | Local SQLite           | Free        | `contradict` action, trust scoring (+0.05 helpful / −0.10 unhelpful), zero dependencies |
| **Hindsight**   | Cloud / local Postgres | Free / paid | Knowledge graph, entity resolution, `hindsight_reflect` cross-memory synthesis          |


Middle path: repackage Holographic's `fact_store` / `fact_feedback` as a
**general** plugin (those can be enabled in any combination) while Mem0 stays
the provider. You'd need dual writes and you'd lose the provider lifecycle.
Hermes' June 2026 policy is that such plugins ship as standalone repos in
`~/.hermes/plugins/`, not in-tree.

### Mem0 OSS lost features that matter here

In the v3 pipeline, several capabilities became Platform-only:

- **Dream** — background consolidation that supersedes outdated facts and merges
duplicates. Close to purpose-built for the staleness problem above.
- **Temporal Reasoning** — boosts memories whose dates match the query timeframe.
- **Graph Memory** — removed from OSS entirely; the Neo4j/Memgraph/Kuzu
integration was dropped.
- **Memory Decay**, `feedback()`, webhooks, memory export, batch ops, `app_id`.

OSS wins on: any metadata key filterable with full operators, your own vector
DB / LLM / embedder, no usage-based billing.

**If self-hosting on pgvector:** verify patch status for the CVSS 8.1 issue
disclosed 2026-04-17 (affected PGVector, Azure MySQL, Neptune Analytics).

### Dates are stored in text, not metadata

The Mem0 plugin exposes no timestamp metadata you can filter on, so the channel
prompt instructs the agent to prefix each stored fact with `YYYY-MM-DD`. It's a
convention, not a schema — the agent can forget. Spot-check periodically.

---



## Troubleshooting


| Problem                                            | Cause                                                                                                                               |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Bot works in DMs but not channels                  | Missing `message.channels` / `message.groups` events. Add, **reinstall**, `/invite`.                                                |
| Bot ignores everyone                               | `SLACK_ALLOWED_USERS` empty → gateway denies all by default.                                                                        |
| "Sending messages to this app has been turned off" | Messages Tab not enabled under **App Home**.                                                                                        |
| Emoji does nothing                                 | `reaction_triggers` set to `true` instead of a list; or missing `reactions:read` + `reaction_added` event; or you didn't reinstall. |
| `missing_scope`                                    | Add the scope, then **reinstall**.                                                                                                  |
| Changed config, nothing happened                   | `hermes gateway restart`. Scope/event changes also need an app reinstall.                                                           |
| Bot answers from general knowledge                 | Channel prompt not applied — check the channel ID key matches exactly.                                                              |


Hermes' own troubleshooting table is more complete:
[https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack#troubleshooting](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack#troubleshooting)

---



## Verification status

Honest accounting of what's confirmed versus inferred, so you know where to test
rather than trust.

**Confirmed from official docs:**

- Mem0 plugin scope fields, tool list, connection modes
- Provider layer auto-prefetch and auto-sync behavior
- One external provider at a time
- Reaction trigger semantics (list vs `true`; agent sees the reacted-to message)
- `allowed_channels` drops messages before other gating; 1:1 DMs exempt
- Gateway denies all when the user allowlist is empty
- Slack sessions keyed by thread
- Mem0 Platform-vs-OSS feature split and quota structure

**Reported but status unknown — verify yourself:**

- Issue #23778 auth bypass (Telegram; Slack adapter may differ)
- Mem0 pgvector CVE patch status

**Not verified — test in your workspace:**

- Exactly how much surrounding thread context enters a turn
- Whether the agent reliably follows the date-prefix convention
- Your real retrieval volume against Mem0 quotas

---



## Links

- Hermes Slack setup: [https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack)
- Memory providers: [https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)
- Hermes GitHub: [https://github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- Mem0 docs: [https://docs.mem0.ai](https://docs.mem0.ai)
- Nous Discord: [https://discord.gg/NousResearch](https://discord.gg/NousResearch)



## License

Configuration in this repo: use freely. Hermes Agent is MIT (Nous Research).
Mem0 core is Apache 2.0.
