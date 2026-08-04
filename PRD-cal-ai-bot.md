# Cal AI Bot — Natural Language Bulk Availability Creation

**Status:** Draft
**Owner:** Mervis Mascarenhas
**Last updated:** 2026-08-04
**Reviewers:** _TBD_

---

## 1. Overview

Cal AI Bot is a conversational assistant that lets internal users (and eventually
providers/practice staff) create and manage provider availability using plain
English instead of navigating multi-step calendar forms.

Today, setting up availability for a set of providers across several locations
and date ranges is a repetitive, click-heavy task. A user must repeat the same
form flow once per provider, per location, per schedule pattern. This is slow,
error-prone, and does not scale during onboarding pushes, seasonal capacity
changes, or bulk backfills.

Cal AI Bot collapses that into a single sentence:

> "Add availability for Dr. A. Rivera at the Midtown office from Sept 25, 2026
> to Sept 30, 2026, 8:00 AM to 10:00 PM."

The bot parses the intent, validates every required field, asks follow-up
questions for anything missing or ambiguous, shows a preview of exactly what
will be created, and only writes after explicit confirmation. Every write is
reversible.

---

## 2. Goals & Non-Goals

### Goals

| # | Goal | Success signal |
|---|------|----------------|
| G1 | Reduce time to create bulk availability | ≥70% reduction in median time vs. the existing UI flow |
| G2 | Reduce input errors | ≥50% fewer corrective edits within 24h of creation |
| G3 | Support one-off and recurring patterns | Weekly and bi-weekly recurrence supported end-to-end |
| G4 | Never write unintended data | 100% of writes preceded by an explicit confirmation |
| G5 | Make mistakes cheap | Any committed change reversible within the undo window |

### Non-Goals (v1)

- Deleting or bulk-clearing existing availability (read + create only).
- Booking, rescheduling, or cancelling appointments.
- Managing provider profiles, services, or insurance.
- Voice input.
- Autonomous/scheduled availability creation without a human in the loop.
- Multi-language support (English only in v1).

---

## 3. Users & Use Cases

| Persona | Description | Primary need |
|---------|-------------|--------------|
| Internal Ops / Onboarding specialist | Sets up availability for newly onboarded providers | Bulk creation across many providers at once |
| Practice administrator | Manages schedules for a group practice | Fast recurring schedule setup, quick corrections |
| Individual provider | Manages their own calendar | Simple one-off additions ("I'm free next Tuesday evening") |
| Support agent | Fixes schedules on behalf of a practice | Confident preview + undo when acting for someone else |

### Representative use cases

- **UC1 — Single provider, single location, date range.** Onboarding a new provider.
- **UC2 — Multiple providers, one location, same pattern.** A practice adds Saturday hours for its whole team.
- **UC3 — One provider, multiple locations, different days.** Provider splits the week across two offices.
- **UC4 — Recurring weekly schedule with no end date.** Standing hours going forward.
- **UC5 — Bi-weekly rotation.** Provider works alternating Fridays.
- **UC6 — Correction mid-conversation.** User realizes the end date is wrong before confirming.
- **UC7 — Undo after commit.** User confirms, then spots a mistake and reverts.

---

## 4. Functional Requirements

### 4.1 Intent parsing

- **FR-1** Parse a free-text message into a structured `AvailabilityRequest`:
  providers, locations, date range, time range, recurrence, slot duration.
- **FR-2** Support relative dates ("next Monday", "starting tomorrow",
  "for the next 6 weeks") resolved against the user's timezone and the current date.
- **FR-3** Support multi-entity prompts ("Dr. A and Dr. B", "all providers at the Midtown office").
- **FR-4** Handle multiple distinct schedule blocks in a single message
  ("Mondays 9–12 at Midtown and Wednesdays 2–6 at Downtown").
- **FR-5** Return a normalized structured object that the user can inspect and edit.

### 4.2 Validation

Every field is validated before a preview is offered. Validation failures are
surfaced as conversational follow-ups, not error codes.

| Field | Required | Rules |
|-------|----------|-------|
| Provider(s) | Yes | Must resolve to exactly one active provider per name; ambiguity triggers disambiguation; user must have permission to edit that provider |
| Location(s) | Yes | Must resolve to a location the provider is actually associated with; ambiguity triggers disambiguation |
| Start date | Yes | Must be today or in the future (provider's local timezone) |
| End date | Conditional | Required for one-off ranges; optional for recurring (open-ended allowed). Must be ≥ start date. Range capped at a configurable max (proposed: 12 months) |
| Start time / End time | Yes | End must be after start; the resulting slots must be in the future |
| Recurrence | No (defaults to one-off) | One of: one-off, weekly, bi-weekly. Days-of-week required for recurring |
| Location type (in-person / virtual) | No | Inferred from the location; only asked if the location supports both |
| Slot duration | No | Defaults to the provider's configured default |
| Timezone | No | Defaults to the location's timezone; stated explicitly in every preview |

Additional validation rules:

- **FR-6** Detect and report overlaps with existing availability; ask whether to skip, merge, or proceed.
- **FR-7** Detect conflicts with existing booked appointments; **never** silently create availability that conflicts — surface and require a decision.
- **FR-8** Respect practice-level blackout dates and holidays; flag them in the preview.
- **FR-9** Enforce a maximum blast radius per request (proposed: 50 providers, 5,000 generated slots). Above the limit, require the user to split the request.
- **FR-10** Warn on suspicious-but-legal input (e.g. a 16-hour day, availability on a date the practice is normally closed) without blocking.

### 4.3 Conversational interaction

- **FR-11** The bot asks one focused follow-up at a time for missing required fields.
- **FR-12** The user can amend any field in natural language before confirming
  ("actually make it 9 AM", "drop Dr. B", "extend it through October").
- **FR-13** The bot maintains conversation state across turns; the user never re-states the whole request.
- **FR-14** The user can say "start over" to clear the draft.
- **FR-15** The bot supports "what did I just create?" and "show me Dr. A's availability next week" read-back queries.

### 4.4 Confirmation

- **FR-16** Before any write, render a **preview** showing: each provider, each
  location, resolved concrete dates, times with explicit timezone, recurrence
  description, total slot count, and any warnings.
- **FR-17** Require an unambiguous affirmative ("confirm", "yes", button click) to commit. Ambiguous replies re-prompt.
- **FR-18** Previews expire after a configurable idle window (proposed: 15 minutes) and must be re-validated before commit — dates and conflicts may have changed.
- **FR-19** Partial failure is reported per-item: which providers succeeded, which failed and why.

### 4.5 Undo

- **FR-20** Every committed request produces a single reversible **change set** with a stable ID.
- **FR-21** "Undo" reverts the most recent change set in the session; "undo change set #X" targets a specific one.
- **FR-22** Undo window: configurable (proposed: 24 hours), surfaced explicitly in the confirmation message.
- **FR-23** If any slot in a change set has since been booked, undo does **not** remove that slot. Report exactly what was and was not reverted.
- **FR-24** Undo is itself an audited action and cannot be undone (no redo in v1).

### 4.6 Auditability & permissions

- **FR-25** Log every request: raw prompt, parsed structure, validation results, preview, decision, commit outcome, actor, timestamp.
- **FR-26** Authorization is enforced server-side per provider/location, independent of what the model produced. The model never widens scope.
- **FR-27** All writes are attributed to the human actor, with the bot recorded as the mechanism.

---

## 5. Worked Examples

### Example 1 — Happy path, one-off range

> **User:** I want to add availability for Provider A at Location A from September 25, 2026 to September 30, 2026, 8:00 AM to 10:00 PM.

> **Bot:** Here's what I'll create — please confirm.
>
> | | |
> |---|---|
> | Provider | A. Rivera, MD |
> | Location | Midtown Office (in-person) |
> | Dates | Fri Sep 25 – Wed Sep 30, 2026 (6 days) |
> | Time | 8:00 AM – 10:00 PM (America/New_York) |
> | Recurrence | One-off (every day in range) |
> | Slot length | 30 min (provider default) |
> | Total slots | 168 |
>
> ⚠️ Sep 26–27 fall on a weekend. Include them?
>
> `[Confirm]` `[Weekdays only]` `[Edit]` `[Cancel]`

> **User:** Weekdays only.

> **Bot:** Updated — 4 days (Sep 25, 28, 29, 30), 112 slots. Confirm?

> **User:** Yes.

> **Bot:** ✅ Created 112 slots. Change set `CS-4821`. You can undo this until Aug 5, 2026 2:14 PM.

---

### Example 2 — Missing fields, progressive disclosure

> **User:** Add some availability for Dr. Chen next week.

> **Bot:** I can do that. Which location? Dr. Chen is associated with **Midtown Office** and **Downtown Clinic**.

> **User:** Downtown.

> **Bot:** Which days next week (Aug 10–16), and what hours?

> **User:** Mon Wed Fri, 9 to 5.

> **Bot:** [preview] Mon Aug 10, Wed Aug 12, Fri Aug 14 · 9:00 AM – 5:00 PM ET · Downtown Clinic · 48 slots. Confirm?

---

### Example 3 — Ambiguous provider

> **User:** Add Friday afternoons for Dr. Patel at Midtown.

> **Bot:** Two providers named Patel are at Midtown Office:
> 1. **N. Patel, MD** — Cardiology
> 2. **R. Patel, DO** — Family Medicine
>
> Which one? (Or say "both".)

---

### Example 4 — Bulk, multiple providers

> **User:** Set up Saturday hours 9am–1pm at the Brooklyn office for Dr. Rivera, Dr. Chen, and Dr. Osei, every Saturday through the end of the year.

> **Bot:** [preview]
> - 3 providers × 21 Saturdays (Aug 8 – Dec 26, 2026)
> - 9:00 AM – 1:00 PM ET, Brooklyn Office
> - Recurrence: weekly, Saturdays
> - **504 slots total**
>
> ⚠️ Dr. Osei already has availability on Sat Aug 8, 10:00 AM – 12:00 PM. Skip that date for Dr. Osei, or merge?
>
> `[Skip conflicts]` `[Merge]` `[Edit]` `[Cancel]`

---

### Example 5 — Bi-weekly recurrence

> **User:** Dr. Chen works every other Thursday at Downtown, 1pm to 6pm, starting Aug 13.

> **Bot:** [preview] Every other Thursday starting Thu Aug 13, 2026 → Aug 13, Aug 27, Sep 10, Sep 24… No end date (open-ended).
>
> I'll generate 6 months forward (through Feb 2027) and extend automatically. Confirm?

---

### Example 6 — Mid-conversation correction

> **User:** Actually make it 2pm, and add Dr. Rivera too.

> **Bot:** Updated: Dr. Chen + Dr. Rivera, every other Thursday, 2:00 PM – 6:00 PM. [new preview] Confirm?

---

### Example 7 — Validation failure, past time

> **User:** Add availability for Dr. Rivera yesterday from 9 to 5.

> **Bot:** I can't create availability in the past — yesterday was Aug 3, 2026. Did you mean **tomorrow (Aug 5)**, or a different date?

---

### Example 8 — Undo with a booked slot

> **User:** Undo that.

> **Bot:** Reverting change set `CS-4821`…
> - ✅ Removed 110 slots
> - ⚠️ Kept 2 slots — they now have booked appointments (Sep 28, 10:00 AM; Sep 29, 2:30 PM). Cancel those appointments through the normal flow if needed.

---

### Example 9 — Blast radius guardrail

> **User:** Add 9–5 availability for every provider in the network for the next year.

> **Bot:** That request covers 1,240 providers and ~2.4M slots — well over the 50-provider / 5,000-slot limit per request. Let's narrow it: which practice or location should we start with?

---

## 6. Additional Requirements Worth Adding

These are not in the original brief but are worth including — grouped by how strongly I'd recommend them.

### Strongly recommended for v1

1. **Conflict detection with existing appointments** (FR-7). Without this, the bot can create availability that double-books a provider. This is the single highest-risk gap in the current requirement list.
2. **Explicit timezone handling.** Availability without a stated timezone is the most common source of silent errors. Every preview must name the timezone.
3. **Blast radius limits** (FR-9). A misparsed "all providers" is a production incident. Cap it.
4. **Change-set model for undo.** Undo only works cleanly if the commit is grouped and identified. Design this into the data model from day one, not bolted on.
5. **Server-side authorization independent of the model.** The LLM proposes; the API authorizes. Never let parsed output determine scope.
6. **Full audit trail** (FR-25). Needed for support, debugging misparses, and compliance.
7. **Partial-failure semantics** (FR-19). Bulk operations fail partially; the UX must say exactly what landed.

### Recommended, could be v1.5

8. **Dry-run / "what would this do?" mode** without entering the commit flow.
9. **Templates & saved patterns.** "Apply Dr. Rivera's standard week to Dr. Chen."
10. **Copy an existing schedule.** "Same as last month" / "same as Dr. Rivera."
11. **Bulk input via file.** Paste or upload a CSV of providers; the prompt supplies the pattern.
12. **Holiday & blackout calendar awareness** (FR-8) with a practice-level holiday list.
13. **Break / lunch handling.** "9–5 with a noon-to-1 lunch."
14. **Capacity per slot.** Some slots allow more than one appointment.
15. **Confidence signaling.** When parse confidence is low, the bot says so and asks rather than guessing.
16. **Session summary.** "Here's everything you changed today" at the end of a session.

### Future / v2

17. **Availability removal and modification** via the same conversational surface (higher risk — needs its own confirmation design).
18. **Proactive suggestions.** "Dr. Chen has no availability after Sep 15 — want to extend?"
19. **Notifications** to affected providers when someone edits their calendar on their behalf.
20. **Multi-channel deployment** (Slack, embedded web widget, existing admin console).
21. **Approval workflow** — staff proposes, provider approves, before commit.
22. **Analytics dashboard** on bot usage, misparse rate, and time saved.
23. **Localization** and non-English prompts.

---

## 7. Suggested Document Sections to Add

To make this a complete, review-ready project document, add the following.
Sections already drafted above are marked ✅.

| # | Section | Why it matters | Status |
|---|---------|----------------|--------|
| 1 | Overview / Problem statement | Frames the "why" for reviewers outside the team | ✅ |
| 2 | Goals & Non-Goals | Prevents scope creep during review | ✅ |
| 3 | Users & personas | Grounds every requirement in a real person | ✅ |
| 4 | Functional requirements | The core contract | ✅ |
| 5 | Worked examples / conversation scripts | The fastest way to align reviewers on behavior | ✅ |
| 6 | **Success metrics & KPIs** | Define baseline + target: time-to-create, misparse rate, confirmation abandonment rate, undo rate, slots created per session | ⬜ |
| 7 | **User flows / journey diagrams** | Happy path, missing-field path, conflict path, undo path | ⬜ |
| 8 | **System architecture** | Chat UI → orchestration layer → LLM (Claude) → entity resolution service → validation engine → availability API → audit store | ⬜ |
| 9 | **Data model** | `AvailabilityRequest`, `ResolvedSchedule`, `ChangeSet`, `AuditEntry`, `ConversationState` — with the JSON schema the model must emit | ⬜ |
| 10 | **API contract** | Endpoints for preview, commit, undo, and query; request/response shapes; idempotency keys | ⬜ |
| 11 | **Prompt & tool design** | System prompt, tool/function definitions, structured-output schema, few-shot examples, how ambiguity is signaled | ⬜ |
| 12 | **Error handling & edge cases** | DST transitions, leap day, midnight-crossing shifts, provider deactivated mid-conversation, concurrent edits, timeouts | ⬜ |
| 13 | **Security, privacy & compliance** | AuthZ model, PHI boundary (this system handles provider scheduling, **not** patient data — state that explicitly and enforce it), data retention for prompts and logs, what is sent to the LLM provider | ⬜ |
| 14 | **Testing strategy** | Golden set of prompts → expected parses; regression suite for the parser; conversation-level integration tests; adversarial prompts; load test for bulk | ⬜ |
| 15 | **Evaluation plan for the LLM** | Parse accuracy per field, ambiguity-detection recall, hallucinated-entity rate, and the acceptance thresholds for shipping | ⬜ |
| 16 | **Rollout plan** | Internal dogfood → pilot practices → GA; feature flags; kill switch | ⬜ |
| 17 | **Risks & mitigations** | See §8 | ✅ (partial) |
| 18 | **Open questions** | See §9 | ✅ |
| 19 | **Milestones & timeline** | Phased delivery with dates and owners | ⬜ |
| 20 | **Dependencies** | Availability/scheduling API, provider directory, location service, auth service, LLM provider | ⬜ |
| 21 | **Cost model** | Tokens per conversation × expected volume; infra cost | ⬜ |
| 22 | **Appendix: glossary** | Slot, availability block, recurrence rule, change set, location type | ⬜ |

---

## 8. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Model misparses dates or entities | Wrong availability created | Mandatory preview with resolved concrete dates; never commit without confirmation |
| Model hallucinates a provider or location | Data written to the wrong entity | Entity resolution against the real directory only; the model may only select from returned candidates, never invent |
| Over-broad request ("all providers") | Mass incorrect writes | Blast radius cap (FR-9); explicit second confirmation above a threshold |
| Timezone / DST errors | Slots at the wrong wall-clock time | Store in UTC with an IANA timezone; render local time in every preview; explicit DST test cases |
| Availability conflicts with booked appointments | Double booking | Conflict check before preview (FR-7); block or require explicit decision |
| Undo blocked by bookings | User can't fully revert | Report partial revert clearly (FR-23); short undo window reduces exposure |
| LLM latency or outage | Bot unusable | Timeouts with graceful fallback to the existing manual UI; the manual path is never removed |
| Prompt injection via provider/location names | Scope escalation | Treat all retrieved data as untrusted content, never as instructions; authorize server-side |
| Users over-trust the bot and skim previews | Confirmed mistakes | Make previews scannable — counts, warnings, and diffs up top; require typed confirmation above the blast-radius threshold |

---

## 9. Open Questions

1. Who are the v1 users — internal ops only, or provider-facing from the start?
2. What is the correct undo window, and does it differ for one-off vs. recurring?
3. For open-ended recurrence, how far forward do we materialize slots, and what extends them?
4. Should the bot ever *modify* existing availability in v1, or strictly create?
5. Does the existing availability API support atomic bulk writes, or do we need a compensating-transaction layer for change sets?
6. What is the authoritative source for provider ↔ location associations?
7. What is the acceptable parse accuracy threshold for GA?
8. Which surface ships first — standalone web chat, embedded in the admin console, or Slack?
9. Do providers get notified when someone else edits their calendar?
10. What is the retention policy for raw prompts and conversation logs?

---

## 10. Appendix — Proposed Structured Output Schema

The shape the model should emit, for the validation layer to consume:

```json
{
  "intent": "create_availability",
  "confidence": 0.0,
  "blocks": [
    {
      "providers": [{ "raw": "Dr. Patel", "resolved_id": null, "candidates": [] }],
      "locations": [{ "raw": "Midtown", "resolved_id": null, "candidates": [] }],
      "start_date": "2026-09-25",
      "end_date": "2026-09-30",
      "start_time": "08:00",
      "end_time": "22:00",
      "timezone": "America/New_York",
      "recurrence": {
        "type": "one_off | weekly | biweekly",
        "days_of_week": ["MO", "WE", "FR"],
        "open_ended": false
      },
      "slot_duration_minutes": null,
      "location_type": null
    }
  ],
  "missing_required_fields": ["locations"],
  "ambiguities": [
    { "field": "providers", "reason": "multiple matches", "options": [] }
  ]
}
```

Rules for the model:
- Never invent a `resolved_id`. Resolution happens in the entity service.
- Emit `missing_required_fields` rather than guessing a default.
- Emit `ambiguities` rather than picking the first candidate.
