# Test Plan: Daily Limit (`booking_limit`) — **Enforcement / Availability Filtering**

**Generated**: 2026-07-29
**Extends**: `generated-plans/2026-07-06-daily-limit/test-plan.md` (TC-01→TC-29, frontend rule-builder vs mock API) — that plan explicitly held enforcement out of scope. This plan adds it. New IDs continue from **TC-30**.
**Xray Test Plan**: ZPR-226789 "Scheduling Rules: Daily Limits" (currently holds 29 Tests, ZPR-227059→ZPR-227087)
**Sources**: AVAIL-420, AVAIL-421, **AVAIL-422** (BookingLimitRule evaluation), **AVAIL-424** (read-path integration), AVAIL-459, **AVAIL-484** (count-query construction §4.8), AVAIL-478, AVAIL-486, AVAIL-488, AVAIL-493, AVAIL-568 (Limit rule E2E testing), AVAIL-589, SQUAWK-6537/6538 (AppointmentBox `/by-dimension`), SQUAWK-6883 (cross-source dedup), SQUAWK-6884 (exclude Manual Intake), SQUAWK-6765 (EHR eligibility), local `daily-limit-notes.md` (PRD §4.1/§4.3)
**Testing Type(s)**: Integration + API (availability read path), cross-channel E2E
**Environment**: **Staging — real backend + real EHR sandbox** (real AppointmentBox / AppointmentList, real 2-way-sync integration). Per user decision 2026-07-29.
**Enforcement mode**: **ON** — AB enforcement flag flipped so exclusions actually filter slots. Per user decision 2026-07-29.
**Total New Test Cases**: 36 (TC-30 → TC-65)
**Priority Breakdown**: P0(20) P1(12) P2(4) P3(0)

---

## Scope

### In Scope
- **Count accrual by booking**: booking new and existing appointments until the configured daily limit is reached for a **specific date**, and observing availability disappear for that date.
- **Boundary at the limit**: `count = limit-1` (slots present) → `count = limit` (whole day blocked) → `count > limit` (still blocked, no error).
- **Whole-day exclusion semantics**: `SyntheticExclusion` with `StartDate = EndDate = date`, `TimeRanges = null` → the entire target date is blocked, adjacent dates untouched (AVAIL-422).
- **Patient type dimension**: rule `patient_type` = `New` / `Existing` / `All`; `All` sums both counts (AVAIL-422).
- **Provider integration shape**:
  - **Non-EHR-integrated provider** (Zocdoc-native calendar, no PMS sync).
  - **EHR-integrated provider, 2-way sync** — including the **double-count risk**: one physical appointment surfacing as both a Zocdoc-native record and an EHR-synced AppointmentList record.
  - Appointments booked **directly in the EHR** (never through Zocdoc) counting toward the limit.
- **Channels**: **Zo** (`phone_bot`), **Branded Directory** (`branded_directory`), and **Marketplace** (no channel param) — do the rule filters apply to availability in each.
- **Cross-channel global counting** (PRD §4.1.9): a booking made on one channel shrinks availability on the others.
- **Near-term vs long-term horizon**: both read paths — `GetFirstAvailability` (near term / next-available) and `GetAvailability` (date range) — at D+0/D+1/D+2, D+14, D+30, D+90, D+364, and at the count-query horizon edge (`now + 5 years`).
- **Count exclusions** (PRD §2 counting rules): cancellations, no-shows, reschedules, Manual Intake records must NOT count.
- **Timezone anchoring**: the calendar day in the provider's IANA local timezone, incl. the late-evening-crosses-UTC-midnight case and functional-duplicate zones (AVAIL-484).
- **Count-source filter**: only `appointment_sources = [appointment_list]` is cap-relevant (AVAIL-484, legal mandate).
- **Propagation latency**: time from booking → availability reflecting the new count (Valkey aggregate rebuild, AVAIL-486/488).
- **Fail-open on read path** when AppointmentBox/count store is unavailable.
- **Booking-time race condition** (PRD §4.3.8, open question).

### Out of Scope
- Everything already covered by the 2026-07-06 plan: rule-builder wizard UI, `buildPayload`, ReviewStep copy, integer BVA, mock-API behavior, page gating. **No enforcement case here re-tests rule creation** beyond using the UI/API as a fixture setup step.
- **Shadow / audit mode** assertions (explainability events without filtering) — per user decision, enforcement-ON only. Shadow-mode gate coverage remains with AVAIL-428 / AVAIL-493.
- **Provloc (per-location) daily limits** and hierarchical provloc-over-provider override — deferred phase (PRD §4.1.34/§4.1.35).
- **Non-daily periods** (daypart, day-of-week, week, month) — MVP is daily only.
- Backfill job (AVAIL-491), reconciliation endpoint (AVAIL-492), Kinesis/Firehose explainability export (AVAIL-429), latency SLA load testing (AVAIL-426).
- Insurance taxonomy depth (Carrier/Network/Program) beyond one smoke case — the MVP rule builder requires limit + patient type only.

---

## Test Fixtures (build these first — every case below references them)

| ID | Fixture | Detail |
|---|---|---|
| **PROV-A** | Non-EHR-integrated provider | Zocdoc-native calendar, no PMS/EHR sync. Bookable on Zo + Branded Directory + Marketplace. Open availability across D+0 → D+400. |
| **PROV-B** | EHR-integrated provider, **2-way sync** | Real EHR sandbox, bidirectional sync enabled, AppointmentList-enrolled + consented. Same availability span as PROV-A. |
| **PROV-C** | Control provider | Same practice as PROV-B, **no** daily-limit rule. Used to prove exclusions are provider-scoped. |
| **PROV-D** | Multi-location, multi-timezone provider | One location `America/Los_Angeles`, one `America/New_York`; a third location on a functional-duplicate zone (`US/Eastern`) per AVAIL-484. |
| **RULE-1** | Daily limit, `patient_type = New`, `max_count = 2` | On PROV-A and PROV-B (separate rules). Single visit reason VR-1. |
| **RULE-2** | Daily limit, `patient_type = All`, `max_count = 2` | On PROV-A. Used for the summing cases. |
| **RULE-3** | Daily limit, `patient_type = Existing`, `max_count = 3` | On PROV-B. |
| **RULE-4** | Multi-VR daily limit, `max_count = 3`, VRs {VR-1, VR-2, VR-3} | On PROV-A. "Max 3 cosmetic/day across several VRs" (PRD §4.1.10). |
| **DATE-N** | Near-term target date | D+2 (provider-local), fully open, no pre-existing appointments. |
| **DATE-F** | Far-term target date | D+14 (provider-local) — the horizon already manually tested. |
| **DATE-XF** | Extra-far target date | D+90 and D+364 — the "further out" ask. |

> **PHI**: all fixtures use synthetic staging patients. Do not copy appointment IDs, patient names, DOBs, or EHR payloads out of staging into tickets, Slack, or any external destination.

---

## Coverage Matrix

| Requirement / Concern | Source | Test Cases | Covered? |
|---|---|---|---|
| Booking new appointments accrues count until limit reached for a specific date | REQ:PRD-4.1.5; AVAIL-422 | TC-30, TC-31, TC-32 | Yes |
| Booking existing-patient appointments accrues count | REQ:PRD-4.1.5; AVAIL-422 | TC-33 | Yes |
| `PatientType.All` sums new + existing | AVAIL-422 | TC-34 | Yes |
| Patient-type-scoped rule leaves the other patient type bookable | AVAIL-422 | TC-35 | Yes |
| `count = limit` blocks the **whole day**, `TimeRanges = null` | AVAIL-422 | TC-36 | Yes |
| Adjacent dates unaffected; exclusion is provider-scoped | AVAIL-422 | TC-37 | Yes |
| Multi-VR rule combines counts across the VR set | AVAIL-422; PRD §4.1.10 | TC-38 | Yes |
| Non-EHR provider: Zocdoc-native bookings reach the count store as `appointment_list` | AVAIL-484; **Open Q1** | TC-39 | Yes (risk) |
| 2-way sync: one appointment counted **once**, not twice | AVAIL-484; SQUAWK-6883; **user concern** | TC-40, TC-41, TC-42 | Yes |
| EHR-native (booked in PMS) appointments count toward the limit | PRD §4.1.9; AVAIL-484 | TC-43 | Yes |
| Zo (`phone_bot`) availability respects the rule | PRD §4.3.11; AVAIL-424 | TC-44 | Yes |
| Branded Directory (`branded_directory`) availability respects the rule | PRD §4.3.11; AVAIL-424 | TC-45 | Yes |
| Marketplace (no channel param) — does the filter apply? | **Open Q2** (contradiction) | TC-46 | Yes (probe) |
| Counting is global across channels | PRD §4.1.9 | TC-47 | Yes |
| Near-term filtering: `GetFirstAvailability` path, D+0/D+1/D+2 | AVAIL-424; **user regression ask** | TC-48, TC-61 | Yes |
| Long-term filtering: `GetAvailability` path, D+14 | AVAIL-424; user manual test | TC-49, TC-62 | Yes |
| Further-out filtering: D+30 / D+90 / D+364 | **user ask ("go further out")** | TC-50 | Yes |
| Count-query horizon edge: `now + 5 years` and beyond | AVAIL-484 | TC-51 | Yes |
| `from = yesterday 00:00 UTC` lower edge / same-day D+0 | AVAIL-484 | TC-52 | Yes |
| Cancellation decrements the count / returns the day | PRD §2 counting rules | TC-53 | Yes |
| No-show does not count | AVAIL-484 status filter | TC-54 | Yes |
| Reschedule moves the count between dates | PRD §2 counting rules | TC-55 | Yes |
| Manual Intake records do not count | SQUAWK-6884; PRD §2 | TC-56 | Yes |
| Timezone anchoring: provider-local calendar day, late-evening UTC rollover | AVAIL-484; PRD §2 | TC-57 | Yes |
| Multi-timezone provider + functional-duplicate zone dedup | AVAIL-484 | TC-58 | Yes |
| Propagation latency booking → availability | AVAIL-486, AVAIL-488 | TC-59 | Yes |
| Fail-open when count store unavailable | AVAIL-422 ("empty counts → empty exclusions") | TC-60 | Yes |
| Booking-time race at `limit - 1` | PRD §4.3.8 (open) | TC-63 | Yes (probe) |
| Overbooked beyond limit stays blocked, no error | INFERRED | TC-64 | Yes |
| Insurance-scoped limit counts only the matching carrier | PRD §4.1.7 | TC-65 | Smoke only |
| Non-enrolled / no-rules provider skips evaluation cleanly | AVAIL-478, AVAIL-589 | TC-37 | Partial |

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation | Related Tests |
|---|---|---|---|---|
| **2-way-sync double count** — a Zocdoc booking is written to the EHR, syncs back, and lands in AppointmentList as a second record for the same physical appointment. Limit of 3 is hit after 2 real bookings. | **High** — silently removes real, bookable availability; provider loses patients and it looks like "Zocdoc broke my calendar" | **Medium-High** (user-raised; appointment-level dedup lives in AppointmentBox per SQUAWK-6883, *not* in AVAIL-484 — AVAIL-484's "functional dedup" is **timezone** dedup only, a commonly conflated distinction) | Explicit A/B comparison of AppointmentBox `/by-dimension` counts for PROV-A (non-EHR) vs PROV-B (2-way sync) after an identical booking sequence; inspect raw buckets, not just the filtered outcome | **TC-40, TC-41, TC-42** |
| **Non-EHR provider may never enforce** — the count query filters `appointment_sources = [appointment_list]`. If a Zocdoc-native provider has no AppointmentList feed, count is always 0 and the limit never fires. | **High** — rule appears saved and active in the UI but does nothing; false confidence | Medium (unverified; SQUAWK-6765 gates the UI to EHR-eligible practices, implying non-EHR may be intentionally unsupported) | TC-39 asserts the actual count-store contents for PROV-A before asserting any filtering. If count is 0, this is a **finding, not a test failure** — escalate to Open Q1 | **TC-39** |
| **Marketplace filters when the rollout plan says it shouldn't** — rules are wired into `AvailabilityControllerHelper`, which is channel-agnostic, but PRD §4.3.11 launches Zo + Branded Directory first, Marketplace later. | Medium — either premature Marketplace enforcement or an unimplemented channel gate | Medium (design contradiction, see Open Q2) | TC-46 probes and records actual behavior; expected result deliberately written as a decision point, not a pass/fail guess | **TC-46** |
| **Near-term vs long-term path divergence** — `GetAvailability` and `GetFirstAvailability` are separately wired (AVAIL-424). A fix to one may not cover the other. | High — near-term slots are the highest-converting availability; a leak there means patients book past the cap | Medium (this is the shape of the bug the user already fixed once) | Symmetric coverage: every horizon assertion runs through **both** read paths | **TC-48, TC-49, TC-50, TC-61, TC-62** |
| **Horizon ceiling** — count query runs to `now + 5 years`; availability beyond that is unfiltered. | Low-Medium — far-future booking is rare but unbounded | Low | TC-51 documents actual behavior at and past the edge | **TC-51** |
| **Timezone anchoring off-by-one-day** — a 9pm PT appointment is 04:00 UTC the next day; if counted against the UTC date the wrong calendar day gets blocked. | High — blocks a day that isn't full while leaving the full day open | Medium | TC-57 books deliberately at the UTC rollover boundary | **TC-57** |
| **Stale counts / propagation lag** — event-driven Valkey rebuild means availability may lag the booking. | Medium — over-booking window, or slots that stay hidden after a cancel | Medium | TC-59 measures and bounds the lag; TC-53 checks the decrement direction | **TC-59, TC-53** |
| **Fail-open hides enforcement failure** — AVAIL-422 returns empty exclusions when counts are unavailable, so a broken count store looks like "no limit configured". | Medium — silent non-enforcement in production | Medium | TC-60 verifies fail-open is graceful **and** observable (alarm/metric fires) | **TC-60** |
| **Booking-time race** (PRD §4.3.8, unresolved) — two patients hold slots at `limit - 1`. | Medium — cap exceeded by one | Medium (no revalidation confirmed) | TC-63 is a probe; result feeds Open Q3 | **TC-63** |

---

## Open Questions

1. **Does a non-EHR-integrated provider enforce at all?** `appointment_sources = [appointment_list]` is the only cap-relevant source (AVAIL-484, described as a legal mandate). SQUAWK-6765 gates the rule-builder UI to practices with an eligible EHR. If PROV-A has no AppointmentList feed its count is permanently 0 and RULE-1/2/4 are inert. **Confirm whether non-EHR providers are in scope for enforcement at all** — if not, TC-39 becomes a negative test (rule must not be creatable) and TC-30–TC-38 should run on PROV-B only.
2. **Marketplace channel gate.** Rules are applied inside `AvailabilityControllerHelper` (AVAIL-424), which serves all channels. PRD §4.3.11 + the Rollout Plan say Zo and Branded Directory launch first, Marketplace later. Is there a channel-level gate, is Marketplace enforcement intentional, or is it an unimplemented gap? TC-46 records observed behavior either way.
3. **Booking-time revalidation** (PRD §4.3.8, still open). Is availability re-evaluated at booking commit, or only at slot-presentation time? Determines whether TC-63 is a bug or a documented limitation.
4. **Rule-eval failure fallback** (PRD §4.3.9, still a `<<Fallback behavior>>` placeholder). AVAIL-422 implies fail-open. Confirm intended behavior and whether a metric/alarm exists (AVAIL-496 is To Do) — TC-60's observability assertion depends on this.
5. **Which appointment-level dedup is authoritative for 2-way sync**, and is it live in staging? SQUAWK-6883 ("Harden cross-source de-duplication and appointment correlation") is **To Do**. If dedup is not yet shipped, TC-40/TC-41 may legitimately fail — that is the finding, not a blocker.
6. **Near/long-term regression anchor.** The bug being guarded was described verbally ("filter near term as well as longer term"), with no ticket. TC-61/TC-62 are tagged `INFERRED`. Provide the ticket key to re-anchor them to the real root cause and tighten the assertions.
7. **Propagation SLA.** Is there a stated bound for booking → availability reflection? TC-59 currently asserts against a 60s working threshold that needs product/eng confirmation.
8. **Enforcement AB flag name and scoping unit** (AVAIL-420 / AVAIL-430). Needed to flip enforcement ON in staging; scope (practice? provider? location?) affects fixture setup.

---

## Assumptions

- **ASSUMPTION**: Staging has a working EHR sandbox with true bidirectional sync for PROV-B, and PROV-B's practice is AppointmentList-enrolled **and** consented. Without consent the counts never populate and every PROV-B case is blocked.
- **ASSUMPTION**: The enforcement AB flag can be flipped ON for the fixture practices in staging (Open Q8), and rules are read from the Availability-owned cache path in effect at test time (AVAIL-396/397).
- **ASSUMPTION**: "Achieving the daily limit" is asserted as **slots disappearing from availability reads**, not as a booking-time rejection — AVAIL-422/424 filter at presentation time. If booking-time rejection is also expected, that is additional scope (see Open Q3).
- **ASSUMPTION**: Whole-day blocking is intended at `count >= limit` (`TimeRanges = null`), so a partially-booked day flips entirely unavailable rather than exposing leftover slots. Called out because it is counter-intuitive to practices and will draw questions.
- **ASSUMPTION**: `D+0`/`D+2`/`D+14` are evaluated in the **provider's** local timezone, not the tester's.
- **ASSUMPTION**: Rule creation itself is a fixture step. Where the rule-builder UI is unavailable for a needed shape (e.g. multi-VR RULE-4), the rule is seeded via `POST /provider-preference-rules/v1/rules` against the real service.
- **ASSUMPTION**: TC-61/TC-62 guard the *symmetry* of near-term and long-term filtering rather than a specific known root cause (Open Q6).
- **ASSUMPTION**: Manual Intake exclusion is observable in staging (SQUAWK-6884 is Closed on the Calendar read path; whether the same exclusion applies to the `/by-dimension` count path needs confirmation during TC-56).

---

## Test Cases

### Pass 1: Specification Testing — Enforcement

#### Reaching the limit by booking (specific date)

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-30 | BookingLimitRule / read path | Count accrues below the limit — availability unchanged | With 1 of 2 new-patient appointments booked on the target date, that date stays fully bookable. | PROV-B + RULE-1 (`New`, max 2); DATE-N (D+2) empty; enforcement flag ON. | 1. Read availability for PROV-B, DATE-N, VR-1, patient type New — record the slot list.<br>2. Book 1 new-patient appointment on DATE-N via Zo.<br>3. Wait for count propagation (≤60s, TC-59).<br>4. Re-read availability for the same params. | Slot list for DATE-N after booking equals the baseline list **minus only the one slot just taken**. No `SyntheticExclusion` is emitted for DATE-N (`RemainingCapacity = 1`). Other DATE-N slots remain returned and bookable. | PROV-B, RULE-1, DATE-N | P0 | Spec — Accrual | AVAIL-422 |
| TC-31 | BookingLimitRule / read path | Limit reached by booking → whole target date removed | Booking the 2nd new-patient appointment reaches `max_count` and removes the entire date. | Continues from TC-30 (count = 1). | 1. Book a 2nd new-patient appointment on DATE-N via Zo.<br>2. Wait for propagation.<br>3. Re-read availability for PROV-B, DATE-N, VR-1, New.<br>4. Inspect the explainability event / exclusion metadata for DATE-N. | **Zero** slots returned for DATE-N. Exactly one `SyntheticExclusion` for DATE-N with `StartDate = EndDate = DATE-N`, `TimeRanges = null`, and `DailyLimitDecisionMetadata` = `Limit: 2`, `CurrentCount: 2`, `RemainingCapacity: 0`, `TargetingPatientType: New`, `TargetingVisitReasonIds: [VR-1]`, `Outcome: "blocked"`. | PROV-B, RULE-1, DATE-N | P0 | Spec — Limit reached | AVAIL-422 |
| TC-32 | BookingLimitRule | Limit reached on a far date, near dates untouched | The same accrual behaviour on D+14 confirms it is date-specific, not provider-wide. | PROV-B + RULE-1; DATE-F (D+14) empty; DATE-N empty. | 1. Book 2 new-patient appointments on DATE-F.<br>2. Wait for propagation.<br>3. Read availability for the range D+0 → D+21. | DATE-F returns zero slots; every other date in D+0 → D+21 (including D+13 and D+15) returns its full unfiltered slot list. Exactly one exclusion is emitted, and its `StartDate = EndDate = DATE-F`. | PROV-B, RULE-1, DATE-F | P0 | Spec — Limit reached | AVAIL-422 |
| TC-33 | BookingLimitRule | Existing-patient bookings accrue against an `Existing` rule | Existing-patient appointments count against a rule scoped to `Existing`. | PROV-B + RULE-3 (`Existing`, max 3); DATE-N empty; 3 synthetic patients with prior completed visits to PROV-B. | 1. Confirm each test patient resolves as **existing** for PROV-B.<br>2. Book existing-patient appointments on DATE-N one at a time, re-reading availability after each.<br>3. Inspect exclusion metadata after the 3rd. | After bookings 1 and 2 the date still returns slots (`RemainingCapacity` 2 then 1, no exclusion). After the 3rd, DATE-N returns zero slots for existing patients with `CurrentCount: 3`, `Limit: 3`, `TargetingPatientType: Existing`. | PROV-B, RULE-3, DATE-N | P0 | Spec — Patient type | AVAIL-422 |
| TC-34 | BookingLimitRule | `PatientType.All` sums new + existing | One new + one existing booking must reach a limit of 2 on an `All` rule. | PROV-A + RULE-2 (`All`, max 2); DATE-N empty. | 1. Book 1 **new**-patient appointment on DATE-N.<br>2. Book 1 **existing**-patient appointment on DATE-N.<br>3. Wait for propagation; read availability for DATE-N for both patient types. | DATE-N returns zero slots **for both new and existing** patients. Exclusion metadata shows `CurrentCount: 2`, `Limit: 2`, `TargetingPatientType: All` — i.e. the two patient-type buckets were summed, not evaluated separately. | PROV-A, RULE-2, DATE-N | P0 | Spec — Patient type | AVAIL-422 |
| TC-35 | BookingLimitRule | Patient-type-scoped rule leaves the other type bookable | A `New`-scoped limit at capacity must not block existing patients on the same date. | PROV-B + RULE-1 (`New`, max 2) and **no** existing-patient rule; DATE-N empty. | 1. Book 2 new-patient appointments on DATE-N.<br>2. Wait for propagation.<br>3. Read DATE-N availability as a **new** patient.<br>4. Read DATE-N availability as an **existing** patient.<br>5. Book an existing-patient appointment on DATE-N. | Step 3 returns zero slots. Step 4 returns the full remaining slot list for DATE-N. Step 5 succeeds and the appointment is created. The exclusion is scoped by `TargetingPatientType: New` and is not applied to the existing-patient read. | PROV-B, RULE-1, DATE-N | P0 | Spec — Patient type | AVAIL-422 |
| TC-36 | BookingLimitRule | Whole-day block, not just remaining-slot block | At capacity the entire calendar day is excluded, including slots at times unrelated to the booked ones. | PROV-B + RULE-1; DATE-N with wide open hours (e.g. 08:00–17:00 provider-local). | 1. Note DATE-N has ≥8 open slots.<br>2. Book the 2 new-patient appointments **both in the morning** (e.g. 08:00, 08:30).<br>3. Wait for propagation; read DATE-N availability. | Zero slots returned for the whole of DATE-N — the untouched afternoon slots (e.g. 14:00, 16:30) are **also** removed. The exclusion carries `TimeRanges = null` (whole day), not a time-bounded range. | PROV-B, RULE-1, DATE-N | P0 | Spec — Whole-day semantics | AVAIL-422 |
| TC-37 | BookingLimitRule | Exclusion is provider-scoped; unruled provider unaffected | A limit on one provider must not affect a colleague at the same practice, and a provider with no rules must evaluate cleanly. | PROV-B at capacity on DATE-N (from TC-31); PROV-C same practice, no rule. | 1. Read PROV-C availability for DATE-N.<br>2. Book an appointment with PROV-C on DATE-N.<br>3. Read a practice-wide / multi-provider availability response covering both providers.<br>4. Check logs for the non-enrolled evaluation path. | PROV-C returns its full DATE-N slot list and the booking succeeds. The multi-provider response omits DATE-N slots for PROV-B while returning them for PROV-C. PROV-C produces **no** exclusions and no per-provider count lookup is performed for it (enrollment gate, AVAIL-478/AVAIL-589); no errors or warnings logged. | PROV-B, PROV-C, DATE-N | P1 | Spec — Scoping | AVAIL-422, AVAIL-478 |
| TC-38 | BookingLimitRule | Multi-VR rule combines counts across the VR set | "Max 3 across VR-1/VR-2/VR-3" must count the set together, not per VR. | PROV-A + RULE-4 (max 3, VRs {VR-1, VR-2, VR-3}); DATE-N empty. | 1. Book 1 appointment for VR-1 on DATE-N.<br>2. Book 1 for VR-2 on DATE-N.<br>3. Read DATE-N availability for VR-3 — expect slots still present.<br>4. Book 1 for VR-3 on DATE-N.<br>5. Wait for propagation; read DATE-N for each of VR-1, VR-2, VR-3. | After step 2, VR-3 still returns slots (`CurrentCount: 2`). After step 4, **all three** visit reasons return zero slots for DATE-N, with a single exclusion carrying `CurrentCount: 3`, `Limit: 3`, `TargetingVisitReasonIds: [VR-1, VR-2, VR-3]` — counts summed across the set, not tracked per VR. | PROV-A, RULE-4, DATE-N | P1 | Spec — Multi-VR | AVAIL-422, PRD §4.1.10 |

#### Provider integration shape — non-EHR vs 2-way sync (double-count)

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-39 | AppointmentBox `/by-dimension` | Non-EHR provider: do Zocdoc-native bookings even reach the count store? | The count query filters to `appointment_sources = [appointment_list]`. Verify a non-EHR provider's native bookings are represented at all before asserting any filtering. | PROV-A (no EHR) + RULE-1; DATE-N empty. | 1. Query AppointmentBox `/by-dimension` for PROV-A / DATE-N — record the count (expect 0).<br>2. Book 1 new-patient appointment on DATE-N via Zo.<br>3. Re-query `/by-dimension` for PROV-A / DATE-N and inspect the returned buckets, including `appointment_source`.<br>4. Book the 2nd and re-read availability for DATE-N. | The step-3 response contains a bucket for DATE-N / patient_type `new` / VR-1 with `count = 1` and an `appointment_list` source; after step 4 the count is 2 and DATE-N returns zero slots. **If the count stays 0**, enforcement is inert for non-EHR providers — record as a finding against **Open Q1**, do not mark the case passed. | PROV-A, RULE-1, DATE-N | P0 | Spec — Non-EHR | AVAIL-484, Open Q1 |
| TC-40 | AppointmentBox dedup | **2-way sync: one Zocdoc booking counts once, not twice** | The core double-count concern. A Zocdoc booking written to the EHR and synced back must not produce two count units. | PROV-B (2-way sync, AppointmentList consented) + RULE-1 (max 2); DATE-N empty; sync healthy. | 1. Query `/by-dimension` for PROV-B / DATE-N — confirm count 0.<br>2. Book **exactly one** new-patient appointment on DATE-N via Zo.<br>3. Confirm in the EHR sandbox that the appointment was written to the EHR.<br>4. Wait for the inbound sync cycle to bring the EHR record back into AppointmentList.<br>5. Re-query `/by-dimension` for PROV-B / DATE-N and inspect the **raw buckets**.<br>6. Read DATE-N availability. | Total count for DATE-N / new / VR-1 is **exactly 1**, not 2 — either a single bucket with `count: 1`, or correlated records collapsed to one unit. DATE-N still returns slots with `RemainingCapacity: 1` and **no** exclusion. A count of 2 after one booking is the double-count defect: log it against SQUAWK-6883 and Open Q5. | PROV-B, RULE-1, DATE-N | **P0** | Spec — 2-way sync | **User concern**; AVAIL-484, SQUAWK-6883 |
| TC-41 | AppointmentBox dedup | 2-way sync: limit is reached at the **real** booking count | The practical consequence of TC-40 — the cap must fire at 2 real bookings, not 1. | PROV-B + RULE-1 (max 2); DATE-N empty; sync healthy. | 1. Book 1 new-patient appointment on DATE-N; wait for a full sync round-trip; read DATE-N availability.<br>2. Book a 2nd new-patient appointment on DATE-N; wait for sync; read DATE-N availability.<br>3. Compare the count trajectory to the same sequence on PROV-A (TC-39) side by side. | After **1** booking DATE-N still returns slots. Only after the **2nd** does DATE-N return zero slots, with `CurrentCount: 2`. The count trajectory for PROV-B (0→1→2) matches PROV-A's exactly — the EHR round-trip adds no extra units. If DATE-N blocks after one booking, that is the double-count defect surfacing as lost availability. | PROV-B vs PROV-A, RULE-1 | **P0** | Spec — 2-way sync | **User concern**; AVAIL-484 |
| TC-42 | AppointmentBox dedup | 2-way sync: EHR-side edit does not inflate the count | Editing a synced appointment in the EHR (time change, status touch) must update the existing count unit, not append one. | PROV-B + RULE-1 (max 2); one new-patient appointment already booked and synced on DATE-N (count = 1). | 1. In the EHR sandbox, change the appointment's time (same day, DATE-N) and save.<br>2. Wait for the sync cycle.<br>3. Re-query `/by-dimension` for PROV-B / DATE-N.<br>4. Repeat with a benign EHR-side field edit (e.g. note text). | Count for DATE-N remains **1** after both edits — no additional bucket or incremented count from the update, and DATE-N still returns slots with `RemainingCapacity: 1`. A count of 2+ means updates are being treated as inserts. | PROV-B, RULE-1, DATE-N | P1 | Spec — 2-way sync | AVAIL-484, SQUAWK-6883 |
| TC-43 | AppointmentBox / read path | EHR-native appointments count toward the limit | Appointments booked directly in the PMS (never via Zocdoc) must consume capacity — the limit is global across the provider's calendar. | PROV-B + RULE-1 (max 2); DATE-N empty. | 1. In the EHR sandbox, create 2 new-patient appointments on DATE-N for PROV-B, matching VR-1's canonical mapping.<br>2. Wait for the inbound sync to land them in AppointmentList.<br>3. Query `/by-dimension` for PROV-B / DATE-N.<br>4. Read DATE-N availability on Zo. | `/by-dimension` reports `count = 2` for DATE-N / new / VR-1 sourced from `appointment_list`, and DATE-N returns **zero** slots on Zo with `CurrentCount: 2`, `Outcome: "blocked"` — despite no booking ever passing through Zocdoc. | PROV-B, RULE-1, DATE-N | P0 | Spec — EHR-native | PRD §4.1.9, AVAIL-484 |

#### Channels

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-44 | Availability read path — Zo | Zo (`phone_bot`) availability respects the rule | Zo is a launch channel; its availability reads must be filtered. | PROV-B at capacity on DATE-N (count = limit); DATE-N+1 open. | 1. Request availability through the Zo path with channel param `phone_bot` for D+0 → D+7.<br>2. Attempt to select DATE-N through the Zo booking flow.<br>3. Confirm DATE-N+1 is still offered. | The `phone_bot` response contains **no** slots for DATE-N; DATE-N+1 slots are present. DATE-N cannot be selected/confirmed through the Zo flow. The exclusion appears in the explainability event with the request's `phone_bot` channel context. | PROV-B, DATE-N, `phone_bot` | P0 | Spec — Channel | PRD §4.3.11, AVAIL-424 |
| TC-45 | Availability read path — Branded Directory | Branded Directory (`branded_directory`) availability respects the rule | The second launch channel must filter identically to Zo. | Same as TC-44. | 1. Request availability with channel param `branded_directory` for D+0 → D+7.<br>2. Load the Branded Directory booking widget for PROV-B.<br>3. Compare the returned slot set to the Zo response from TC-44. | The `branded_directory` response contains no slots for DATE-N and the widget offers no DATE-N times. The filtered slot set is **identical** to the Zo response for the same date range — no channel-specific divergence in what the rule removes. | PROV-B, DATE-N, `branded_directory` | P0 | Spec — Channel | PRD §4.3.11, AVAIL-424 |
| TC-46 | Availability read path — Marketplace | Marketplace (no channel param) — probe whether the filter applies | Rules live in the channel-agnostic `AvailabilityControllerHelper`, but the rollout plan defers Marketplace. Record actual behavior. | Same as TC-44; Marketplace search/profile reachable for PROV-B. | 1. Request availability with **no** channel param (Marketplace) for D+0 → D+7.<br>2. Load PROV-B's Marketplace profile and the search-result availability card.<br>3. Attempt to book DATE-N via Marketplace.<br>4. Record which behavior occurred. | **Decision point, not a guess.** Record exactly one: **(a) Filtered** — DATE-N absent from Marketplace availability and unbookable, matching Zo/BD; enforcement is channel-agnostic and the PRD's staged rollout has no code-level gate → raise against Open Q2. **(b) Not filtered** — DATE-N slots present and bookable on Marketplace only; a channel gate exists and matches the rollout plan. Either way the count must still be consumed. Escalate (a) to product before enforcement ramp. | PROV-B, DATE-N, no channel param | P1 | Spec — Channel probe | Open Q2, AVAIL-424 |
| TC-47 | BookingLimitRule | Counting is global across channels | A booking on one channel must shrink availability on the others (PRD §4.1.9). | PROV-B + RULE-1 (max 2); DATE-N empty. | 1. Book 1 new-patient appointment on DATE-N via **Zo**.<br>2. Book the 2nd via **Branded Directory**.<br>3. Wait for propagation.<br>4. Read DATE-N availability via Zo, Branded Directory, and Marketplace. | DATE-N returns zero slots on **both** Zo and Branded Directory even though each channel only ever placed one booking — the count is a single provider-day total, not per-channel. `CurrentCount: 2` in a single exclusion. Marketplace behaves per the TC-46 finding. | PROV-B, RULE-1, DATE-N | P0 | Spec — Cross-channel | PRD §4.1.9 |

#### Horizon — near term vs long term

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-48 | `GetFirstAvailability` | Near-term filtering on the first-availability path | The next-available path is wired separately from the range path (AVAIL-424) and must filter D+0/D+1/D+2. | PROV-B + RULE-1 (max 2); D+0, D+1, D+2 all open. | 1. Call the `GetFirstAvailability` path for PROV-B — record the returned first-available date/time.<br>2. Fill D+0 to capacity (2 new-patient bookings); wait for propagation; call again.<br>3. Fill D+1 to capacity; call again.<br>4. Fill D+2 to capacity; call again. | Step 1 returns a D+0 slot. After step 2 the first-available result **skips D+0** and returns the earliest D+1 slot. After step 3 it returns D+2. After step 4 it returns D+3 or later. At no point does a capacity-reached date appear as the first-available result, and no "next available" card or search-result snippet shows a blocked date. | PROV-B, RULE-1, D+0/1/2 | **P0** | Spec — Near term | AVAIL-424, **user ask** |
| TC-49 | `GetAvailability` | Long-term filtering on the range path at D+14 | Re-confirm the horizon already manually validated, on the range read path. | PROV-B + RULE-1 (max 2); D+14 open. | 1. Fill D+14 to capacity (2 new-patient bookings); wait for propagation.<br>2. Request availability for the range D+7 → D+21.<br>3. Request a narrow range covering only D+13 → D+15. | Both requests return zero slots for D+14 while returning full slot lists for D+13 and D+15. Exactly one exclusion is emitted, `StartDate = EndDate = D+14`. The narrow-range request filters identically to the wide one — the result does not depend on the requested window size or on D+14's position within it. | PROV-B, RULE-1, D+14 | **P0** | Spec — Long term | AVAIL-424, **user ask** |
| TC-50 | `GetAvailability` / `GetFirstAvailability` | Further-out filtering: D+30, D+90, D+364 | Push well beyond the 2-week horizon already tested. | PROV-B + RULE-1 (max 2); D+30, D+90, D+364 open (extend availability templates as needed). | 1. For each of D+30, D+90, D+364: fill to capacity, wait for propagation, then request a range spanning that date **and** call the first-availability path with a start date just before it.<br>2. After all three, request a single wide range D+0 → D+400. | Each of D+30, D+90, D+364 returns zero slots on **both** read paths, with its own exclusion (`StartDate = EndDate` = that date). Neighbouring dates (D+29/31, D+89/91, D+363/365) return full slot lists. The wide D+0 → D+400 request blocks all three dates simultaneously and no others — filtering does not degrade with distance or with range width. | PROV-B, RULE-1, D+30/90/364 | **P1** | Spec — Long term | **User ask** |
| TC-51 | Count-query horizon | Behavior at and beyond the `now + 5 years` ceiling | AVAIL-484 caps the count query at ~now+5y; availability past that is unfiltered by construction. | PROV-B + RULE-1; availability opened at D+(5y − 2d) and D+(5y + 30d). | 1. Fill D+(5y − 2d) to capacity; wait for propagation; request that range.<br>2. Fill D+(5y + 30d) to capacity; wait; request that range.<br>3. Inspect the `/by-dimension` request's `to_appointment_time_utc`. | D+(5y − 2d) returns zero slots (inside the horizon, enforced). For D+(5y + 30d), record actual behavior: slots most likely **still returned** because the count query's `to_appointment_time_utc` ≈ now+5y excludes it — document the ceiling and confirm it is acceptable rather than treating it as a pass/fail. `to_appointment_time_utc` is ≈ now + 5 years, not unbounded. | PROV-B, RULE-1, 5-year edge | P2 | Spec — Horizon edge | AVAIL-484 |
| TC-52 | Count-query horizon | Lower edge: same-day (D+0) and the `yesterday 00:00 UTC` floor | The count window starts at yesterday 00:00 UTC; same-day enforcement must work and past days must not leak into today's count. | PROV-B + RULE-1 (max 2); D+0 open with slots later today; 2 appointments already existing on D−1. | 1. Confirm D−1 already holds 2 new-patient appointments.<br>2. Read D+0 availability — D−1's appointments must not count against D+0.<br>3. Fill D+0 to capacity with 2 bookings; wait for propagation.<br>4. Re-read D+0 availability, and read D−1 (past) availability. | Step 2 returns D+0's full remaining slot list (`CurrentCount: 0` for D+0 — yesterday's 2 appointments are counted against D−1 only, never pooled into D+0). Step 4 returns zero slots for D+0 with `CurrentCount: 2`. D−1 exposes no bookable slots regardless (past date) and produces no error. | PROV-B, RULE-1, D+0/D−1 | P1 | Spec — Horizon edge | AVAIL-484 |

### Pass 2: Breakage Hunting

#### Count exclusions — what must NOT count

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-53 | Count recompute | Cancellation frees capacity and returns the day | Cancelled appointments must not count; the day must become bookable again. | PROV-B at capacity on DATE-N (`CurrentCount: 2`, zero slots). | 1. Confirm DATE-N returns zero slots.<br>2. Cancel one of the two appointments (patient-cancel).<br>3. Wait for propagation.<br>4. Re-read DATE-N availability and `/by-dimension`.<br>5. Repeat with a **provider**-cancel on a re-filled day. | Count drops to 1, the DATE-N exclusion is no longer emitted, and DATE-N returns its remaining slot list — bookable again. Both `patient_cancelled` and `provider_cancelled` are excluded from the count per the AVAIL-484 status filter. No stale exclusion persists after the propagation window. | PROV-B, RULE-1, DATE-N | P0 | Breakage — Counting | AVAIL-484, PRD §2 |
| TC-54 | Count recompute | No-shows do not count | A no-show must not consume capacity. | PROV-B + RULE-1 (max 2); a past-or-today date with 2 appointments where one can be marked no-show. | 1. On the target date, mark one appointment `no_show` in the EHR/provider tool.<br>2. Wait for propagation.<br>3. Query `/by-dimension` and read availability for that date. | Count for the date is **1**, not 2; the `no_show` record is excluded by the status filter. The date returns slots again (if not otherwise in the past) with no exclusion emitted. | PROV-B, RULE-1 | P1 | Breakage — Counting | AVAIL-484 |
| TC-55 | Count recompute | Reschedule moves the count between dates | Rescheduling must decrement the origin date and increment the destination. | PROV-B + RULE-1 (max 2); DATE-N at capacity (2 appts); DATE-N+1 holds 1 appointment. | 1. Confirm DATE-N blocked, DATE-N+1 open (count 1).<br>2. Reschedule one DATE-N appointment to DATE-N+1.<br>3. Wait for propagation.<br>4. Read availability and `/by-dimension` for both dates. | DATE-N count = 1 → exclusion gone, slots returned. DATE-N+1 count = 2 → whole day blocked, exclusion emitted. The reschedule is counted **once at its destination only** — total across the two dates stays 3, with no phantom unit left on DATE-N. | PROV-B, RULE-1, DATE-N/N+1 | P1 | Breakage — Counting | PRD §2 |
| TC-56 | Count recompute | Manual Intake records do not count | Manual intake requests are not real appointments and must not consume capacity. | PROV-B + RULE-1 (max 2); DATE-N with 1 real appointment; ability to create a Manual Intake record on DATE-N. | 1. Note DATE-N count = 1, slots present.<br>2. Create 2 Manual Intake records on DATE-N.<br>3. Wait for propagation.<br>4. Query `/by-dimension` and read DATE-N availability. | Count for DATE-N remains **1** — Manual Intake records are excluded from the count path (SQUAWK-6884). DATE-N still returns slots with `RemainingCapacity: 1` and no exclusion. If the count rises to 3 and the day blocks, the exclusion is missing on the `/by-dimension` path — log against Open Q8/SQUAWK-6884. | PROV-B, RULE-1, DATE-N | P1 | Breakage — Counting | SQUAWK-6884, PRD §2 |

#### Timezone & multi-location

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-57 | Timezone grouping | Late-evening booking counts against the provider-local day, not the UTC day | A 21:00 PT appointment is 04:00 UTC the next day — it must count against the PT calendar day. | PROV-D, location in `America/Los_Angeles`, daily limit max 2, DATE-N open with a ≥21:00 local slot. | 1. Book 1 new-patient appointment at **21:00 America/Los_Angeles** on DATE-N (= 04:00 UTC on DATE-N+1).<br>2. Book a 2nd at 21:30 local the same DATE-N.<br>3. Wait for propagation.<br>4. Query `/by-dimension` and read availability for DATE-N **and** DATE-N+1. | Both bookings are counted against **DATE-N** (provider-local): DATE-N count = 2, whole day blocked. DATE-N+1 count = **0** and returns its full slot list — no capacity leaked forward across the UTC boundary. The `/by-dimension` request carries the provider's `iana_time_zone_id`. | PROV-D, `America/Los_Angeles` | P0 | Breakage — Timezone | AVAIL-484, PRD §2 |
| TC-58 | Timezone grouping + functional dedup | Multi-timezone provider: limit shared across locations; duplicate zones collapse | A provider working two timezones the same day shares one limit; functionally identical zones must not double-query or double-count. | PROV-D with locations in `America/Los_Angeles`, `America/New_York`, and `US/Eastern`; provider-level limit max 2; DATE-N open at all three. | 1. Book 1 appointment at the LA location and 1 at the NY location, both on DATE-N provider-local.<br>2. Wait for propagation.<br>3. Read DATE-N availability for **all three** locations.<br>4. Inspect the `/by-dimension` calls issued for PROV-D / DATE-N. | DATE-N is blocked at **all three** locations — the limit is shared across locations, not per-location (provloc is a later phase). Count = 2, not 3: `America/New_York` and `US/Eastern` are collapsed as functionally identical (equal summer **and** winter offsets), producing **one** call per distinct zone (2 calls, not 3) and no duplicate count units. | PROV-D, 3 locations | P1 | Breakage — Timezone | AVAIL-484 |

#### Resilience, timing, race

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-59 | Valkey aggregate rebuild | Propagation latency booking → availability | Bound the window in which availability is stale after a booking or cancellation. | PROV-B + RULE-1 (max 2); DATE-N with 1 appointment; stopwatch/log timestamps available. | 1. Book the appointment that reaches capacity on DATE-N; record T0.<br>2. Poll DATE-N availability every 5s until zero slots are returned; record T1.<br>3. Cancel one appointment; record T2; poll until slots return; record T3.<br>4. Repeat 3× for both directions. | Both `T1 − T0` (block) and `T3 − T2` (unblock) are ≤ **60s** on every repetition, and the transition is monotonic — availability never flickers between blocked and unblocked. Record actual medians and worst case; any value over 60s is a finding against **Open Q7** (no confirmed SLA). | PROV-B, RULE-1, DATE-N | P1 | Breakage — Timing | AVAIL-486, AVAIL-488, Open Q7 |
| TC-60 | Fail-open path | Count store unavailable → fail open, but observably | AVAIL-422 returns empty exclusions when counts are unavailable; that must not be silent. | PROV-B at capacity on DATE-N; ability to induce an AppointmentBox/count-store failure or timeout in staging. | 1. Confirm DATE-N is blocked.<br>2. Induce a count-source failure (block the `/by-dimension` dependency or force timeout).<br>3. Read DATE-N availability.<br>4. Check logs/metrics for the failure signal.<br>5. Restore the dependency and re-read. | Step 3 returns DATE-N slots (**fail open** — patients are never shown an error and availability is not lost) with **zero** exclusions emitted. A warning/metric records the count-source failure and distinguishes "counts unavailable" from "no rules configured" — the two must not be indistinguishable in telemetry. Step 5 restores the DATE-N block within the TC-59 window. | PROV-B, DATE-N, induced failure | P1 | Breakage — Resilience | AVAIL-422, Open Q4 |
| TC-63 | Booking-time revalidation | Concurrent bookings at `limit − 1` | PRD §4.3.8 is unresolved: two patients holding the last slot. | PROV-B + RULE-1 (max 2); DATE-N with exactly 1 appointment (`RemainingCapacity: 1`). | 1. Open two independent booking sessions, both loading DATE-N availability while capacity remains 1.<br>2. Have session A complete its booking.<br>3. Without reloading, have session B submit its booking for DATE-N.<br>4. Inspect the final appointment count on DATE-N. | **Probe — record actual behavior.** Preferred: session B is rejected at commit with a clear "no longer available" message and DATE-N ends with **2** appointments (revalidation exists). If DATE-N ends with **3**, the cap was exceeded by one — expected given no confirmed booking-time revalidation; record against **Open Q3** with the exact sequence. Either way session B must not receive a 500 or a silent success with no appointment. | PROV-B, RULE-1, DATE-N | P2 | Breakage — Race | PRD §4.3.8, Open Q3 |
| TC-64 | BookingLimitRule | Overbooked beyond the limit stays blocked, no error | `count > limit` (via EHR-side overbooking) must remain blocked without negative remaining-capacity artifacts. | PROV-B + RULE-1 (max 2); DATE-N with **4** appointments created EHR-side. | 1. Create 4 new-patient appointments on DATE-N in the EHR sandbox; wait for sync.<br>2. Query `/by-dimension` and read DATE-N availability.<br>3. Inspect the exclusion metadata. | DATE-N returns zero slots. Exclusion metadata reads `Limit: 2`, `CurrentCount: 4`, `RemainingCapacity: 0` — clamped at zero, **not** `-2`. No exception, no 500, no malformed explainability event. Cancelling down to 3 keeps the day blocked; cancelling to 1 returns slots. | PROV-B, RULE-1, DATE-N | P2 | Breakage — Boundary | INFERRED |

#### Insurance dimension (smoke)

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-65 | BookingLimitRule | Insurance-scoped limit counts only the matching carrier | PRD §4.1.7 allows insurance-scoped limits; verify the count is carrier-scoped, not global. | PROV-B; daily-limit rule scoped to carrier CARRIER-X, max 2; DATE-N open. | 1. Book 2 appointments on DATE-N with insurance **CARRIER-Y**; wait; read DATE-N availability.<br>2. Book 2 on DATE-N with **CARRIER-X**; wait; read DATE-N availability for CARRIER-X and for CARRIER-Y/self-pay. | After step 1 DATE-N still returns slots (`CurrentCount: 0` for the CARRIER-X rule — non-matching insurance does not accrue). After step 2 DATE-N is blocked for CARRIER-X patients while CARRIER-Y/self-pay patients still receive slots. | PROV-B, CARRIER-X/Y | P2 | Breakage — Insurance | PRD §4.1.7 |

### Pass 3: Historical / Regression

| ID | Component | Summary | Description | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-61 | `GetFirstAvailability` | **Regression: near-term filtering must not be skipped** | Guards the near-term half of the previously-fixed near/long-term filtering gap. Locks the near-term read path against re-regression. | PROV-B + RULE-1 (max 2); D+0, D+1, D+2 open. | 1. Fill D+1 to capacity.<br>2. Call the first-availability path with no explicit start date.<br>3. Call it again with a start date of D+1.<br>4. Load the search-result / profile "next available" surface for PROV-B. | D+1 never appears as a first-available result in any of the three surfaces — with no start date, with a start date landing exactly on the blocked day, or on the "next available" card. The result skips to D+2. Filtering applies to the first-availability path, not only the range path. | PROV-B, RULE-1, D+1 | **P0** | Regression — Near term | INFERRED (user-reported fix, Open Q6) |
| TC-62 | `GetAvailability` | **Regression: long-term filtering must not be skipped** | Guards the long-term half of the same gap, across several window shapes. | PROV-B + RULE-1 (max 2); D+14 open. | 1. Fill D+14 to capacity.<br>2. Request D+14 as a single-day range.<br>3. Request D+8 → D+21 (blocked date mid-window).<br>4. Request D+14 → D+28 (blocked date at window start).<br>5. Request D+1 → D+14 (blocked date at window end). | D+14 returns zero slots in **all four** window shapes — single-day, mid-window, first day, last day. No window shape or boundary position causes the exclusion to be dropped, and all other dates in each window return full slot lists. | PROV-B, RULE-1, D+14 | **P0** | Regression — Long term | INFERRED (user-reported fix, Open Q6) |

---

## Historical Context

- **Near/long-term filtering gap (user-reported, fixed; no ticket key supplied)**: filtering did not hold uniformly across near-term and longer-term availability. Manual verification reached ~2 weeks out. `GetAvailability` and `GetFirstAvailability` are wired separately in **AVAIL-424**, which is a plausible structural cause for a horizon- or path-dependent gap. Locked by **TC-48, TC-49, TC-50, TC-61, TC-62**. Re-anchor to the real ticket to tighten these (**Open Q6**).
- **AVAIL-478 (Closed)**: `BookingLimitProcessor` empty-rules lifecycle bug + `HasRulesAsync` enrollment gate. A provider with no rules must short-circuit before any per-provider count lookup. Touched by **TC-37**; see also AVAIL-589 (pre-gate on a cheap enrollment check, To Do).
- **AVAIL-484 (Closed)**: established the count-query contract these cases assert against — `appointment_list`-only source, cancelled/no-show excluded, `yesterday 00:00 UTC → now+5y` horizon, per-IANA-zone calls with summer+winter offset dedup. Note the naming trap: AVAIL-484's "functional dedup" is **timezone** dedup, **not** appointment-level dedup. Appointment-level cross-source dedup is **SQUAWK-6883**, still **To Do** — which is exactly why the 2-way-sync double-count concern is live.
- **SQUAWK-6884 (Closed)**: Manual Intake excluded from the Calendar read path. Whether the same exclusion holds on the `/by-dimension` count path is unconfirmed — **TC-56** checks it.
- **SQUAWK-6765 (Closed)**: Daily Limits only surface for practices with an eligible EHR — the basis for **Open Q1** (whether non-EHR providers can enforce at all).
- **AVAIL-422 (Closed)**: `AppointmentBox` empty counts → empty exclusions, i.e. deliberate fail-open on the read path. Exercised by **TC-60**.

---

## Exit Criteria

### Must Pass (blocking enforcement ramp)
- **TC-30, TC-31, TC-32, TC-33, TC-34, TC-35, TC-36** — the limit is actually reached by booking, and blocks the right date and patient type.
- **TC-39, TC-40, TC-41, TC-43** — count integrity for both provider shapes. **TC-40/TC-41 are hard blockers**: a confirmed double count means enforcement removes real availability and must not ramp.
- **TC-44, TC-45, TC-47** — Zo and Branded Directory both filter, and counting is global across channels.
- **TC-48, TC-49, TC-61, TC-62** — near-term and long-term filtering both hold, on both read paths.
- **TC-52, TC-53, TC-57** — same-day/floor boundary, cancellation frees capacity, timezone anchoring correct.

### Should Pass (non-blocking with sign-off)
- TC-37, TC-38, TC-42, TC-50, TC-54, TC-55, TC-56, TC-58, TC-59, TC-60.
- **TC-46** must be *executed and its outcome recorded* before ramp even though it has no pass/fail — Marketplace behavior must be a known decision, not a surprise.

### Recommended Before Full Rollout
- TC-51, TC-65, TC-63, TC-64 executed and results recorded.
- **Open Q1 (non-EHR enforcement), Q2 (Marketplace gate), and Q5 (dedup shipped?) resolved** — these three change what "correct" means for whole sections of this plan.
- Open Q3 (booking-time revalidation) and Q4 (fail-open fallback + alarm) have documented answers.

---

## Exploratory Testing Charters

| Charter | Mission | Time Box | Focus Areas |
|---|---|---|---|
| 2-way-sync count forensics | Drive appointments through every EHR-side mutation and watch raw `/by-dimension` buckets for extra units | 90 min | Create in EHR vs create on Zocdoc, time edit, VR/provider change, cancel-in-EHR vs cancel-on-Zocdoc, delete-then-recreate, sync replay/backfill, sync outage then catch-up |
| Horizon sweep | Probe filtering across the whole bookable span to find any horizon where it silently stops | 60 min | D+0 … D+400 at intervals, DST transition dates, month/year boundaries, leap day, the now+5y ceiling, `GetFirstAvailability` vs `GetAvailability` at each |
| Channel parity diff | Same provider, same date, same rule — diff the slot sets across all three channels | 60 min | `phone_bot` vs `branded_directory` vs no-param, logged-in vs anonymous, deep-link into a blocked date, cached/CDN responses, mobile vs desktop surfaces |
| Timezone abuse | Hunt off-by-one-day blocking around midnight and DST | 60 min | 23:45 and 00:15 provider-local bookings, spring-forward/fall-back dates, provider tz ≠ patient tz ≠ server tz, multi-location same-day, `US/Eastern` ≡ `America/New_York` |
| Capacity thrash | Rapidly cross the limit boundary in both directions and look for stuck or flickering availability | 45 min | Book/cancel/rebook loops, simultaneous cancel + book, reschedule chains across dates, stale exclusions after cancel, count drift after many cycles |
| Blast-radius check | Confirm one provider's limit never affects anyone else | 45 min | Same-practice colleagues, shared locations, supervisor/resource configs, practice-wide and search-level availability responses, multi-provider result cards |
