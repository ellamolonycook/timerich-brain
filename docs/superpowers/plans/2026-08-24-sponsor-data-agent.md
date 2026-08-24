# Sponsor Data-Ops Agent — Architecture & Guardrails Proposal

Status: DESIGN ONLY — awaiting Chris's go-ahead before any code.
Author: Gideon. Based on the manual pipeline run for NYC / Austin / London / LA, 23–24 Aug 2026.

Goal: an agent that keeps the sponsor-lead pipeline full, clean, and cheap — and that never sends, spends, or launches anything without Ella's explicit yes.

## What the agent replaces

The manual loop we ran this week, which is now proven end to end:

1. Run a saved SuperSearch per city in Instantly (titles + industries + headcount + one-lead-per-company).
2. Export raw rows (Fully Enriched Profile, 0.5 credits/row).
3. Classify every row with the Claude filter from the Sponsor ICP spec (section 7) → A / B / C / Reject.
4. Email-enrich only A/B survivors (Work Email, 1.5 credits/row).
5. Verify emails (ZeroBounce) before any campaign import.
6. Stage campaigns per tier; Ella reviews the list and the copy; only then launch.

Every stage of the agent maps 1:1 to a step above. Nothing new is invented; the agent is the loop with a scheduler, a ledger, and a conscience.

## Architecture

Same shape as the rest of timerich-brain: a Cloudflare Worker, cron-triggered, with pure testable stages. State in D1 (this workload earns persistence — it is exactly "the daily confirms, not the sliders").

```
            cron (daily)
                │
   ┌────────────▼────────────┐
   │ 1 INGEST                │  Instantly SuperSearch API, per-city saved searches
   └────────────┬────────────┘
   ┌────────────▼────────────┐
   │ 2 CLEAN                 │  dedupe (cross-city, cross-campaign), normalize,
   └────────────┬────────────┘  drop rows w/o domain, title-exclusion pass
   ┌────────────▼────────────┐
   │ 3 CLASSIFY              │  Claude + spec §7 prompt, batched; verdict/tier/score
   └────────────┬────────────┘  logged with reasons — order defensible out loud
   ┌────────────▼────────────┐
   │ 4 VERIFY (gated)        │  A/B rows only → Work Email → ZeroBounce
   └────────────┬────────────┘  spend requires budget headroom (see guardrails)
   ┌────────────▼────────────┐
   │ 5 STAGE (gated)         │  writes to campaign as DRAFT leads only;
   └────────────┬────────────┘  launch is a human action, never the agent's
   ┌────────────▼────────────┐
   │ 6 REPORT                │  weekly check-in to Ella + real-time alerts
   └─────────────────────────┘
```

### Data model (D1)

- `leads` — one row per person: source city, company, domain, title, tier, score, reason, email, verification status, campaign status. The dedupe registry lives here: a company key (domain) is unique per event cycle, so Vercel pitched from NYC can never also be pitched from Austin.
- `spend_ledger` — every credit-consuming call, appended before the call is made: provider (Instantly / ZeroBounce), operation, rows, estimated cost, actual cost, remaining balance. The ledger is what makes budget caps enforceable rather than aspirational.
- `decisions` — every Ella approval or rejection, with what she saw and when. The agent's memory of what a human already ruled on, so it never re-asks and never un-decides.
- `kill_list` — the spec's hard disqualifiers plus companies Ella has vetoed by hand (e.g. Passes, Builder.ai, Fanvue). Append-only from the human side; the classifier cannot override it.

### Classification

The spec §7 prompt, verbatim, as the system prompt of a batch call — the spec is written to be run, so run it unmodified. Two additions learned from this week:

- A pre-classifier rules pass (title excludes, kill list, crypto/agency/consumer patterns) rejects the obvious ~40% before any tokens are spent. Rules are cheap and auditable; the model handles only the judgment calls.
- "Do not invent headcount" is enforced structurally: fields the enrichment did not return are passed as absent, never defaulted. This week's exports came back with every enrichment field "Skipped" — the pipeline must score conservatively in that state, not hallucinate around it.

## Guardrails

These are the point of the doc. Ordered by blast radius.

**G1 — The agent never sends email and never launches a campaign.** Stage 5 writes draft leads into a draft campaign. The launch button belongs to a human, forever. This is not a config default; the agent has no code path to launch.

**G2 — Tier A lists require Ella's eyes before staging.** The exact flow from this week, formalized: agent posts the proposed A list (company, contact, title, one-line reason) → Ella replies approve / remove-these / hold → only approved rows proceed. Her removals go to `kill_list` or `decisions` so they are remembered. B lists batch into the weekly check-in; C and Reject never reach her unless she asks.

**G3 — Hard budget caps, checked against the ledger before every paid call.**
- Per-run cap (suggest: 400 credits) and per-week cap (suggest: 1,500) — both configurable by Ella only.
- Email enrichment only ever runs on classified A/B rows. Raw-list enrichment is capped per city and requires headroom.
- Below 20% monthly balance on any provider: paid operations pause, Ella is alerted, and nothing resumes until she acknowledges.

**G4 — Subscription watch.** Track renewal dates and balances for Instantly, ZeroBounce, Sales Nav trial (cancel-by date!), and anything added later. Alert at 7 days and 48 hours before renewal with a one-line keep/cancel recommendation based on the ledger's actual usage. The agent recommends; Ella decides. The agent never holds payment credentials.

**G5 — Weekly check-in (Ella's explicit ask).** Every Monday, one message:
- New initiatives? (new cities, new event dates, changed ICP — free-text answer updates config)
- Last week: rows ingested, classified, tier counts, credits spent vs cap.
- Queued and waiting on her: Tier A approvals, budget confirmations, renewal calls.
- Anything anomalous, in one line each.
No reply = nothing gated proceeds. Silence pauses the pipeline; it never green-lights it.

**G6 — Data-quality tripwires.** Auto-pause the affected stage and alert when: bounce rate on any campaign exceeds 3%; a batch's classification distribution shifts sharply from history (suggests a broken search or a changed ICP); enrichment returns >50% empty fields (this week's "Skipped" incident — the agent must notice, not shrug); duplicate rate spikes (search overlap).

**G7 — Immutable exclusions.** The spec's kill list (§1) compiles to rules, not prompt text. A prompt injection in a company's scraped description cannot argue its way past a regex. Any change to exclusion rules is a human commit, reviewed like code.

**G8 — Full audit trail.** Every row can answer: where did you come from, who scored you and why, what did you cost, who approved you, which campaign hold you. Same principle as the ranked lists in build-next: the order has to be defensible out loud.

## Weekly cadence (steady state)

- Daily cron: ingest new saved-search results → clean → classify. Zero credits except ingest.
- On A/B accumulation: request enrichment budget if gated, enrich, verify, stage as draft.
- Monday: check-in to Ella. Her replies update config and release gates.
- Continuous: balance, renewal, and tripwire monitoring.

## Deliberately not in this plan

- Auto-launch or auto-send — permanently out, not deferred.
- The agent writing or editing email copy — Ella's voice, Ella's words.
- Auto-adding new cities or changing the ICP — proposals only, via check-in.
- Payment handling of any kind.
- LinkedIn automation — one banned account is lesson enough; the agent works the Instantly database only.

## Open questions for Chris

1. Instantly API coverage: confirm SuperSearch execution, enrichment triggering, and lead/campaign writes are all available on the current plan (the UI supports them; API parity needs a check).
2. ZeroBounce position: verify-after-enrich (belt and braces, current assumption) or replace Instantly's built-in verification entirely?
3. Where does Ella's check-in live — Slack DM, WhatsApp, or the brain's own UI?
4. Claude API budget for classification (~1,300 rows/week at current pace is small, but it should sit in the same ledger).
5. Repo placement: worker under timerich-brain like build-next, or its own service?
