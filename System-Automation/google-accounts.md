# Google Accounts

**Status:** Planning
**Last updated:** 2026-04-25

> Living reference for Usman's Google accounts, plan comparisons, and open decisions about account strategy.

---

## Critical Assumptions

_Last reviewed: 2026-04-25_

| Assumption | Value | If this changes... |
|------------|-------|--------------------|
| "Take notes for me" is unavailable on personal Google plans | True as of April 2026 — confirmed via Google support thread | Re-evaluate personal AI Pro as the primary plan. At €20/month (or ~€11 effective with $10 dev credit) and 5 TB storage, it would be significantly cheaper and better than Business Standard (€15/month, 2 TB, no dev credit). Migration would then be worth the hassle. |

See: https://support.google.com/meet/thread/423663755/take-notes-no-longer-available-for-google-meet-with-google-one-pro-5tb-plan?hl=en

---

## Accounts

| Account | Type | Email shown | Notes |
|---------|------|-------------|-------|
| usmanramay@gmail.com | Personal Google account | usmanramay@gmail.com | Primary personal account |
| experimentoflife@gmail.com | Google Workspace (linked) | usman@experimentoflife.org | experimentoflife.org domain via Workspace |

---

## Decisions and Reasoning

### 1. Personal plan vs. Workspace — which is the right type of account?

**Decision: Workspace is required. Personal plans are ruled out.**

- "Take notes for me" (AI meeting notes) is **not available on any personal Google plan** — confirmed via Google's own documentation (see References).
- Google One Premium and Google AI Pro have identical Google Meet features — neither includes meeting notes or transcription.
- Since meeting notes are a hard requirement, personal plans are not a viable option regardless of price.
- **Workspace Business Standard (€15/month) is the only current plan that covers everything needed**: longer Meet calls, calendar booking pages, meeting notes, transcription, and recording.

### 2. The only remaining question: which account should hold the Workspace subscription?

Currently linked to experimentoflife@gmail.com. Two options:

- **Keep it on experimentoflife@gmail.com** — no action needed, status quo
- **Move it to usmanramay@gmail.com** — apparently unlocks **$300 in Claude AI credits**; worth investigating what conditions apply and whether the offer is still active

No other meaningful differences between the two accounts for Workspace purposes.

### 3. What if I want to upgrade to better AI in the future?

- Business Standard + AI expanded access add-on = €15 + €14 = **€29/month** — expensive, and still no developer API credit
- There is no personal plan equivalent that also includes meeting notes
- The upgrade path is constrained to Workspace tiers — personal AI Pro is off the table unless meeting notes become available on personal plans in the future

---

## Plan Comparison

### Personal Google Plans

_Note: All personal plans have identical Google Meet features. None include meeting notes ("Take notes for me") or transcription. This rules out personal plans entirely for this use case._

| Plan | Price | Storage | AI / Gemini | Longer Meet calls | Calendar booking | Meet notes/transcription | Dev API credit | Ruled out? |
|------|-------|---------|-------------|-------------------|------------------|--------------------------|----------------|------------|
| Google AI Plus | €8/month | 200 GB | Similar to Business Standard (assumed) | No | No | No | No | Yes — no longer Meet, no notes |
| Google One Premium | €10/month | 2 TB | Good tier | Yes | Yes | **No** | No | Yes — no meeting notes |
| Google AI Pro | €20/month | 5 TB | Full, highest tier | Yes | Yes | **No** | $10/month | Yes — no meeting notes |

See: https://one.google.com/about/#compare-plans and https://support.google.com/meet/answer/10459644?hl=en

### Google Workspace Plans

| Plan | Price | Storage | AI / Gemini | Longer Meet calls | Calendar booking | Meet notes/transcription/recording | Dev API credit | Ruled out? |
|------|-------|---------|-------------|-------------------|------------------|------------------------------------|----------------|------------|
| Business Starter | €7/month | 30 GB | Basic | Yes | No | No | No | Yes — no notes/recording, no booking |
| Business Standard (current) | €15/month | 2 TB | Better than personal base | Yes | Yes | Yes | No | No — current plan |
| Business Standard + AI expanded access | €29/month (€15 + €14) | 2 TB | Roughly equivalent to AI Pro | Yes | Yes | Yes | No | Effectively yes — very expensive |

---

## Current Status

- On Workspace Business Standard (€15/month) — staying on this plan; it is the only option that meets all requirements
- Personal plans ruled out: meeting notes not available on any personal Google plan
- Only open question: whether to move the Workspace subscription from experimentoflife@gmail.com to usmanramay@gmail.com for $300 Claude credits

---

## Open Tasks and Ideas

- [ ] Verify the $300 Claude credits offer for moving Workspace to usmanramay@gmail.com — confirm it's still active and what conditions apply
- Idea: Monitor whether Google restores meeting notes on personal plans — if they do, personal AI Pro becomes the better option (see Critical Assumptions)
- Idea: If EoL ever adds team members, Workspace becomes even more clearly the right choice

---

## References

| Resource | URL | What it shows |
|----------|-----|---------------|
| Google Meet features by plan | https://support.google.com/meet/answer/10459644?hl=en | Which Meet features (recording, notes, transcription, etc.) are available per plan |
| "Take notes for me" availability | https://support.google.com/docs/answer/13952129?sjid=5146736346178396468-NA#zippy=%2Cgoogle-meet | Confirms meeting notes are not available on personal AI Pro |
| Google One personal plans comparison | https://one.google.com/about/#compare-plans | Full feature breakdown for all personal Google plans |
| Meeting notes removed from personal plans (community confirmation) | https://support.google.com/meet/thread/423663755/take-notes-no-longer-available-for-google-meet-with-google-one-pro-5tb-plan?hl=en | Confirms "Take notes for me" is not available on personal plans as of April 2026 |

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-25 | Added Critical Assumption + confirmed note-taking removed from personal plans | Google support thread confirms "Take notes for me" was removed from personal plans; added as a tracked assumption — if restored, personal AI Pro becomes the better option |
| 2026-04-25 | Major reversal — personal plans ruled out entirely | Discovered that meeting notes ("Take notes for me") are not available on any personal Google plan; Workspace is the only option |
| 2026-04-25 | Finalized plan comparison and decisions | Confirmed Business Standard = €15; AI expanded access add-on = €14 (total €29); Business Starter ruled out (no notes/recording); previously assumed personal plans were viable |
| 2026-04-25 | File created | Captured account structure and plan comparison after research session |
