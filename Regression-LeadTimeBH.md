# Test Plan: Lead Time & Business Hours — Daily and Weekly Regression Test Sets

**Generated**: 2026-07-30
**Sources**: User-defined scope (no Jira/Confluence/Figma/PR provided). Domain reasoning + stated fixtures.
**Testing Type(s)**: Desktop Web (Intake Settings, PFS) + Mobile Web spot checks
**Surfaces**: Intake Settings page (configuration) → PFS availability (verification)
**Execution**: Manual regression, Jira/Xray test sets
**Feature Flag**: N/A (confirm whether business-hours-aware lead time accrual sits behind a flag)
**Total Test Cases**: 40 — Daily set: 14, Weekly set: 26
**Priority Breakdown**: P0(16) P1(17) P2(7)

---

## Scope

### In Scope
- Configuration of **hours of operation** and **minimum lead time** on the Intake Settings page (save, persist, validate).
- **Availability rendering on PFS** as a consequence of that configuration: which timeslots appear, which are suppressed, day tabs, empty states.
- **Business-hours-aware lead time counting** — lead time accruing only during open hours, across closed periods.
- **Timezone and DST correctness** — provider-local vs patient-local rendering, spring-forward / fall-back, day-boundary rollover.
- Propagation latency between an Intake Settings save and PFS reflecting it.

### Out of Scope
- Completing a booking end-to-end past slot selection (payment, insurance, intake questions) — only the lead-time-boundary book attempt (TC-D14) is covered.
- Partner/syndication surfaces consuming the same availability.
- Backend availability-service contract testing at the API layer.
- Search results page ranking/filtering — PFS only.
- Provider-side calendar/blocked-time management outside the Intake Settings hours fields.

---

## Assumptions

Every one of these should be confirmed before the sets are imported. Each is marked `ASSUMPTION` where it drives an expected result.

| # | Assumption | Affects |
|---|-----------|---------|
| A1 | "Lead time" = minimum booking notice; a slot is bookable only if `slot_start >= now + lead_time`. | All lead time cases |
| A2 | Business-hours-aware accrual means the lead time clock advances **only during configured open hours**; time while closed does not count down. | TC-D06, TC-W07–W10 |
| A3 | PFS displays timeslots in the **provider location's** timezone, with an explicit timezone label. | TC-D08, TC-D09, TC-W14–W16 |
| A4 | A slot is only offered if the **full appointment duration** fits before the closing time. | TC-D11, TC-W05 |
| A5 | Intake Settings supports multiple open/close blocks per day (i.e., a midday closure). | TC-D12, TC-W04 |
| A6 | Lead time is configured per provider/location, and may differ for in-person vs video and new vs returning patients. | TC-W19, TC-W20 |
| A7 | There is a documented propagation SLA between save and PFS reflection (cache TTL). Substitute the real number for `{SLA}`. | TC-D13, TC-W25 |

---

## Standard Fixtures

Reuse these across both sets so cases stay short and results are comparable run over run.

| Fixture | Definition |
|---------|-----------|
| **PROV-A** | Single location, TZ `America/New_York`. Mon–Fri 09:00–17:00, midday closure 12:00–13:00, Sat/Sun closed. Appointment duration 30 min. Min lead time 2 hours. Business-hours-aware accrual ON. |
| **PROV-B** | Single location, TZ `America/Los_Angeles`. Mon–Sun 08:00–20:00. Duration 20 min. Min lead time 4 hours. |
| **PROV-C** | Single location, TZ `America/Phoenix` (no DST) — the DST control provider. Mon–Fri 09:00–17:00. Min lead time 1 hour. |
| **PROV-D** | Two locations, different timezones (`America/New_York`, `America/Chicago`) and different hours. |
| **PATIENT-PT** | Browser/device set to `America/Los_Angeles`. |
| **PATIENT-ET** | Browser/device set to `America/New_York`. |

---

# DAILY REGRESSION TEST SET

**Test Set name**: `Regression — Daily — Lead Time & Business Hours`
**Target runtime**: ~45–60 min manual
**Selection rationale**: every case here either blocks booking outright when broken, or is a silent-failure mode (wrong times shown, slots wrongly hidden) that costs patient-facing volume within hours of a bad deploy. All are P0/P1 and all use a single provider fixture to keep setup cheap.

| ID | Component | Summary | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|----|-----------|---------|---------------|-------|-----------------|-----------|----------|----------|--------|
| TC-D01 | Intake Settings | Hours of operation save and persist | Logged in with access to PROV-A Intake Settings | 1. Open Intake Settings for PROV-A<br>2. Set Mon–Fri 09:00–17:00, Sat/Sun closed<br>3. Save<br>4. Hard-reload the page | Save confirmation appears; after reload each weekday row shows `09:00 AM – 05:00 PM` and Sat/Sun show the closed state. No field reverts to a default. | PROV-A | P0 | Happy Path | INFERRED |
| TC-D02 | Intake Settings | Minimum lead time saves and persists | PROV-A loaded | 1. Set minimum lead time to `2 hours`<br>2. Save<br>3. Reload | Field reads `2 hours` after reload; no rounding to a different unit and no reset to the account default. | lead time = 2h | P0 | Happy Path | INFERRED |
| TC-D03 | PFS | Slots render only inside configured business hours | PROV-A configured per fixture; current time Tue 08:00 ET | 1. Open PFS for PROV-A as PATIENT-ET<br>2. Inspect Wednesday's slot list | Earliest Wednesday slot is `9:00 AM`, latest is `4:30 PM`. No slot before 9:00 AM or at/after 5:00 PM is offered. | PROV-A, Wed | P0 | Happy Path | ASSUMPTION A4 |
| TC-D04 | PFS | Closed day shows no availability | PROV-A, Sat/Sun closed | 1. Open PFS for PROV-A<br>2. Navigate to the upcoming Saturday and Sunday | Both days render zero timeslots and show the no-availability state for that day; the day is not silently skipped without indication. | PROV-A, Sat + Sun | P0 | Negative | INFERRED |
| TC-D05 | PFS | Lead time boundary — slot exactly at `now + lead time` is offered | PROV-A, lead time 2h; current time Tue 10:00 ET | 1. Open PFS<br>2. Inspect Tuesday's slots | The `12:00 PM` boundary is unavailable due to the midday closure, so the first offered Tuesday slot is `1:00 PM`. Repeat with the midday closure removed: the `12:00 PM` slot IS offered (inclusive boundary). | now=Tue 10:00 ET | P0 | BVA | ASSUMPTION A1 |
| TC-D06 | PFS | Business-hours-aware accrual — after-hours lead time rolls to next open day | PROV-A, accrual ON; current time Fri 16:30 ET (30 min before close) | 1. Open PFS for PROV-A<br>2. Identify the first bookable slot | 30 min of the 2h lead time accrues before Friday 17:00; the remaining 90 min accrues from Monday 09:00. First bookable slot is **Monday 10:30 AM**. No Saturday, Sunday, or Friday-evening slot is offered. | now=Fri 16:30 ET | P0 | Business Rule | ASSUMPTION A2 |
| TC-D07 | PFS | Slots before `now + lead time` are suppressed | PROV-A; current time Tue 10:00 ET | 1. Open PFS<br>2. Inspect the Tuesday slot list for anything earlier than the boundary | No Tuesday slot at or before `11:30 AM` is offered. Past-time slots (09:00, 09:30, 10:00) are absent, not greyed-but-clickable. | now=Tue 10:00 ET | P0 | Negative | ASSUMPTION A1 |
| TC-D08 | PFS | Times render in provider-location timezone with a visible label | PROV-A (ET); browser set to PATIENT-PT | 1. Set browser TZ to `America/Los_Angeles`<br>2. Open PFS for PROV-A | Slots still read `9:00 AM – 4:30 PM` and a timezone indicator identifies them as provider-local Eastern time. Times are NOT shifted to `6:00 AM – 1:30 PM`. | PROV-A + PATIENT-PT | P0 | Date/Time | ASSUMPTION A3 |
| TC-D09 | PFS | Cross-timezone provider renders its own hours | PROV-B (PT); browser set to PATIENT-ET | 1. Set browser TZ to `America/New_York`<br>2. Open PFS for PROV-B | Slots read `8:00 AM – 7:40 PM` Pacific with the Pacific timezone label. The window is not clipped or shifted by the Eastern browser. | PROV-B + PATIENT-ET | P0 | Date/Time | ASSUMPTION A3 |
| TC-D10 | PFS | Zero lead time — immediate booking | PROV-A with lead time set to `0`; current time Tue 10:05 ET | 1. Set lead time to 0 on Intake Settings, save<br>2. Open PFS after propagation | The next future slot (`10:30 AM`) is offered. No already-started slot (`10:00 AM`) is offered. | lead time = 0 | P1 | BVA | ASSUMPTION A1 |
| TC-D11 | PFS | Last slot of the day fits the full appointment duration | PROV-A, 30-min duration, closes 17:00 | 1. Open PFS<br>2. Inspect the final slot of any open weekday | Final slot is `4:30 PM`. No `4:45 PM` or `5:00 PM` slot is offered — a slot that would end after close is suppressed. | PROV-A | P1 | BVA | ASSUMPTION A4 |
| TC-D12 | PFS | Midday closure produces a gap, not a merged block | PROV-A with 12:00–13:00 closure | 1. Open PFS<br>2. Inspect a fully-open future weekday | Slots run 9:00–11:30 AM, then resume at 1:00 PM. No slot exists at 12:00 PM or 12:30 PM. | PROV-A, day+3 | P1 | Business Rule | ASSUMPTION A5 |
| TC-D13 | Intake Settings → PFS | Config change propagates to PFS | PROV-A open in Intake Settings; PFS open in a second tab | 1. Change closing time from 17:00 to 15:00, save<br>2. Wait `{SLA}`<br>3. Reload PFS | Within `{SLA}`, the last offered slot on future weekdays becomes `2:30 PM`. No 3:00–4:30 PM slot remains after the propagation window. | close 17:00→15:00 | P0 | Data Integrity | ASSUMPTION A7 |
| TC-D14 | PFS | Boundary slot is actually bookable, not just displayed | PROV-A; a slot at exactly `now + lead time` is visible on PFS | 1. Select the earliest offered slot<br>2. Proceed to the point of appointment confirmation | The slot is accepted and the flow advances. No "this time is no longer available" or lead-time-violation error is raised for a slot the page just offered. | earliest offered slot | P0 | Data Integrity | INFERRED |

---

# WEEKLY REGRESSION TEST SET

**Test Set name**: `Regression — Weekly — Lead Time & Business Hours`
**Target runtime**: ~3–4 hours manual
**Selection rationale**: permutations, DST, holidays, multi-location, and validation matrices. These break less often and are expensive to set up (clock manipulation, multiple fixtures), but each is a real production failure mode. Run the daily set first — if it fails, don't burn time here.

### Business-Hours-Aware Accrual (deep)

| ID | Component | Summary | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|----|-----------|---------|---------------|-------|-----------------|-----------|----------|----------|--------|
| TC-W01 | PFS | Accrual across a full closed weekend | PROV-A, accrual ON, lead time 2h; current time Sat 12:00 ET | 1. Set clock to Saturday noon<br>2. Open PFS | No accrual occurs over the weekend. First bookable slot is **Monday 11:00 AM** (2h after Monday's 09:00 open). | now=Sat 12:00 ET | P0 | Business Rule | ASSUMPTION A2 |
| TC-W02 | PFS | Accrual spans the midday closure | PROV-A, lead time 2h; current time Tue 11:00 ET | 1. Set clock to Tue 11:00<br>2. Open PFS | 1h accrues 11:00–12:00; the closure does not count; the remaining 1h accrues from 13:00. First bookable slot is **Tuesday 2:00 PM**. | now=Tue 11:00 ET | P0 | Business Rule | ASSUMPTION A2 |
| TC-W03 | PFS | Lead time longer than a single business day | PROV-A, lead time set to `12 hours`; current time Mon 10:00 ET | 1. Set lead time to 12h, save<br>2. Open PFS after propagation | Accrual: 7h Mon (10:00–17:00 less the 1h closure = 6h), continuing Tue. First bookable slot lands **Wednesday morning** per the accrual arithmetic — record the exact slot and assert it does not fall outside business hours or on a closed day. | lead time = 12h | P1 | Business Rule | ASSUMPTION A2 |
| TC-W04 | PFS | Accrual with multiple closures in one day | PROV-A extended to 09:00–12:00, 13:00–15:00, 16:00–18:00 | 1. Configure three blocks, save<br>2. Set clock to 11:30, open PFS with lead time 2h | Accrual: 30 min (11:30–12:00) + 90 min from 13:00. First bookable slot is **2:30 PM**. No slot at 12:00–12:30 or 15:00–15:30. | 3 blocks | P1 | Business Rule | ASSUMPTION A5 |
| TC-W05 | PFS | Boundary lands mid-closure | PROV-A, lead time 2h; current time Tue 10:30 ET | 1. Set clock to Tue 10:30, open PFS | Raw boundary would be 12:30 PM, inside the closure. First offered slot is **1:00 PM** — the boundary is pushed forward to the next open slot, not dropped to the following day, and not offered at 12:30. | now=Tue 10:30 ET | P0 | BVA | ASSUMPTION A2 |
| TC-W06 | PFS | Accrual toggle OFF behaves as wall-clock | PROV-A with business-hours-aware accrual disabled; current time Fri 16:30 ET | 1. Disable accrual, save<br>2. Open PFS | With wall-clock counting, the boundary is Friday 18:30 — outside hours — so the first slot is **Monday 9:00 AM** (the first in-hours slot at or after the boundary), NOT Monday 10:30 AM. Contrast against TC-D06. | accrual OFF | P1 | Feature Toggle | ASSUMPTION A2 |
| TC-W07 | PFS | Accrual when the provider is closed all of the next day | PROV-A with Wednesday additionally closed; current time Tue 16:45 ET | 1. Close Wednesday, save<br>2. Set clock to Tue 16:45, open PFS | 15 min accrues Tuesday; Wednesday contributes nothing; remaining 1h45 accrues from Thursday 09:00. First bookable slot is **Thursday 10:45 AM**. No Wednesday slot appears. | Wed closed | P1 | Business Rule | ASSUMPTION A2 |

### Timezone & DST

| ID | Component | Summary | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|----|-----------|---------|---------------|-------|-----------------|-----------|----------|----------|--------|
| TC-W08 | PFS | Spring-forward — the missing hour is not offered | PROV-A; clock set to the US DST spring-forward date, hours temporarily 01:00–06:00 | 1. Set hours to 01:00–06:00 on the spring-forward Sunday<br>2. Open PFS for that date | No slot is offered between 2:00 AM and 2:59 AM (the hour that does not exist). Slots run 1:00, 1:30, then 3:00 onward. No duplicate or shifted-by-one-hour slots. | spring-forward Sun | P0 | Date/Time | ASSUMPTION A3 |
| TC-W09 | PFS | Fall-back — the repeated hour is not double-listed | PROV-A; clock set to the US DST fall-back date, hours 00:00–06:00 | 1. Configure hours, open PFS for that date | The 1:00–1:59 AM hour appears exactly once. There are no two indistinguishable `1:00 AM` slots, and total slot count for the day matches the configured window. | fall-back Sun | P0 | Date/Time | ASSUMPTION A3 |
| TC-W10 | PFS | Lead time boundary crossing the spring-forward transition | PROV-A, lead time 4h; current time set to the night before spring-forward, 23:00 ET | 1. Set clock to 23:00 the night before spring-forward<br>2. Open PFS | The boundary accounts for the lost hour: the first bookable slot is 4 real elapsed hours later in local wall-clock terms (i.e., 4:00 AM local, not 3:00 AM). Assert against a computed expected value, not the UI's own arithmetic. | spring-forward eve | P0 | Date/Time | ASSUMPTION A1+A3 |
| TC-W11 | PFS | Lead time boundary crossing the fall-back transition | PROV-A, lead time 4h; clock set to 23:00 the night before fall-back | 1. Set clock, open PFS | The boundary accounts for the extra hour: first bookable slot is 4 real elapsed hours later (2:00 AM local after the repeat), and no slot before the true boundary is offered. | fall-back eve | P0 | Date/Time | ASSUMPTION A1+A3 |
| TC-W12 | PFS | No-DST provider is unaffected by the transition | PROV-C (`America/Phoenix`) | 1. Set clock to each DST transition date<br>2. Open PFS for PROV-C | Hours render `9:00 AM – 4:30 PM` unchanged on both transition dates. No slot shifts by an hour relative to a non-transition weekday. | PROV-C | P1 | Date/Time | ASSUMPTION A3 |
| TC-W13 | PFS | Patient in a DST-observing TZ viewing a non-DST provider | PROV-C; PATIENT-ET, clock on a DST transition date | 1. Set browser to ET on the transition date<br>2. Open PFS for PROV-C | Provider-local Phoenix times render unchanged with the correct timezone label; the patient's own transition does not shift the displayed slots. | PROV-C + PATIENT-ET | P1 | Date/Time | ASSUMPTION A3 |
| TC-W14 | PFS | Day-boundary rollover — lead time pushes the first slot to tomorrow | PROV-B (08:00–20:00 PT), lead time 4h; current time 18:30 PT | 1. Set clock to 18:30 PT<br>2. Open PFS for PROV-B | First bookable slot is on the **next calendar day**; "Today" shows no availability and the next-day tab is the first with slots. The next-day date label matches the provider-local date, not the patient-local date. | now=18:30 PT | P1 | Date/Time | ASSUMPTION A1 |
| TC-W15 | PFS | Patient near midnight in a TZ ahead of the provider | PROV-B (PT); PATIENT-ET at 00:30 ET (= 21:30 PT previous day) | 1. Set browser to ET, clock to 00:30<br>2. Open PFS for PROV-B | The "Today" tab reflects the **provider's** current date. Slot dates and day labels are self-consistent — the page does not show a day tab whose slots belong to a different date. | midnight cross-TZ | P1 | Date/Time | ASSUMPTION A3 |
| TC-W16 | Intake Settings | Location timezone change updates PFS rendering | PROV-A | 1. Change the location timezone from `America/New_York` to `America/Chicago`, save<br>2. Wait `{SLA}`, reload PFS | PFS slots read `9:00 AM – 4:30 PM` **Central** with the Central label. The configured 09:00–17:00 hours are reinterpreted in the new zone; they are not converted to `8:00 AM – 3:30 PM`. Confirm which behaviour is intended before asserting. | TZ change | P1 | Date/Time | OPEN QUESTION |

### Configuration, Validation & Propagation

| ID | Component | Summary | Preconditions | Steps | Expected Result | Test Data | Priority | Category | Source |
|----|-----------|---------|---------------|-------|-----------------|-----------|----------|----------|--------|
| TC-W17 | Intake Settings | Invalid hours are rejected | PROV-A | 1. Set Monday close time earlier than open (17:00 open, 09:00 close)<br>2. Attempt save | Save is blocked; an inline validation message identifies the Monday row; no partial write occurs (reload shows the prior valid values). | close < open | P1 | Input Validation | INFERRED |
| TC-W18 | Intake Settings | Overlapping blocks are rejected or merged deterministically | PROV-A | 1. Add blocks 09:00–13:00 and 12:00–17:00 on the same day<br>2. Save | Either save is blocked with an overlap message, or the blocks merge to a single 09:00–17:00 range — whichever the product specifies. Assert the same outcome on PFS: no duplicated 12:00–13:00 slots. | overlapping blocks | P1 | Input Validation | OPEN QUESTION |
| TC-W19 | Intake Settings | Lead time field boundary values | PROV-A | 1. Attempt to save lead time of `0`, the documented maximum, maximum+1, a negative value, and a non-numeric value | 0 and the maximum save successfully. Maximum+1, negative, and non-numeric are rejected with a field-level message and no write. Record the actual max in the test data column once confirmed. | 0 / max / max+1 / -1 / "abc" | P1 | BVA | INFERRED |
| TC-W20 | Intake Settings → PFS | Different lead time for in-person vs video | PROV-A with in-person lead time 2h, video lead time 30 min | 1. Configure both, save<br>2. On PFS, toggle between in-person and video | Each visit type shows its own boundary: the video list starts 30 min out, the in-person list 2h out. Switching the toggle re-computes the list rather than reusing the previous type's slots. | 2h vs 30m | P1 | Business Rule | ASSUMPTION A6 |
| TC-W21 | Intake Settings → PFS | Different lead time for new vs returning patients | PROV-A with new-patient lead time 24h, returning 2h | 1. Configure both, save<br>2. On PFS, switch the patient-type selection | New-patient availability starts ~24h out (business-hours-adjusted); returning starts 2h out. The two lists differ and each independently respects business hours. | 24h vs 2h | P1 | Business Rule | ASSUMPTION A6 |
| TC-W22 | Intake Settings → PFS | Holiday / one-off closure suppresses that date | PROV-A | 1. Add a one-off closure for a specific future weekday, save<br>2. Open PFS for that date | That date shows zero slots and the no-availability state. Adjacent dates are unaffected. Lead time accrual skips the closed date (see TC-W07 pattern). | one-off closure | P1 | Business Rule | INFERRED |
| TC-W23 | Intake Settings → PFS | Multi-location — each location keeps its own hours and timezone | PROV-D (2 locations, ET and CT, different hours) | 1. Configure distinct hours per location, save<br>2. On PFS, switch between locations | Each location renders its own hours in its own timezone with the correct label. Switching locations fully re-computes the slot list; no slot from the previous location persists. | PROV-D | P1 | Data Integrity | INFERRED |
| TC-W24 | PFS | Availability horizon end | PROV-A | 1. Navigate PFS forward to the last bookable date, then one beyond | The final in-horizon date renders slots normally; the date beyond the horizon shows the no-availability/end-of-horizon state rather than an error or an infinite scroll. | horizon edge | P2 | Edge Case | INFERRED |
| TC-W25 | Intake Settings → PFS | Propagation under rapid successive edits | PROV-A | 1. Change closing time three times in quick succession (17:00 → 15:00 → 16:00), saving each<br>2. Wait `{SLA}`, reload PFS | PFS settles on the **last** saved value (16:00 close, last slot 3:30 PM). No intermediate value (15:00) persists, and no stale 17:00 slots remain. | 3 rapid saves | P2 | Data Integrity | ASSUMPTION A7 |
| TC-W26 | PFS (Mobile Web) | Mobile viewport parity for hours and lead time | PROV-A; mobile browser | 1. Open PFS for PROV-A on a mobile viewport<br>2. Compare the offered slot list against the desktop run of TC-D03 and TC-D05 | The mobile slot list matches desktop exactly — same first slot, same last slot, same closure gap, same timezone label. No slot is truncated by the mobile layout. | mobile 390×844 | P2 | Visual/Layout | INFERRED |

---

## Coverage Matrix

| Behaviour | Daily coverage | Weekly coverage |
|-----------|----------------|-----------------|
| Hours config saves and persists | TC-D01, TC-D02 | TC-W17, TC-W18, TC-W19 |
| Slots confined to business hours | TC-D03, TC-D04, TC-D11, TC-D12 | TC-W04, TC-W22, TC-W24 |
| Lead time boundary enforcement | TC-D05, TC-D07, TC-D10, TC-D14 | TC-W05, TC-W19 |
| Business-hours-aware accrual | TC-D06 | TC-W01–W07 |
| Timezone rendering | TC-D08, TC-D09 | TC-W12, TC-W13, TC-W15, TC-W16, TC-W23 |
| DST transitions | *(none — weekly only)* | TC-W08–W11 |
| Day-boundary rollover | *(partial via TC-D06)* | TC-W14, TC-W15 |
| Config propagation | TC-D13 | TC-W25 |
| Visit-type / patient-type variants | *(none)* | TC-W20, TC-W21 |
| Mobile parity | *(none)* | TC-W26 |

DST is deliberately weekly-only: it requires clock manipulation and is only genuinely at risk twice a year — but it is P0 within the weekly set, and worth promoting to daily for the two weeks surrounding each transition.

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation | Related Tests |
|------|--------|------------|------------|---------------|
| Lead time computed in UTC but compared against provider-local wall clock | High — whole days of availability wrongly hidden or wrongly offered | Med | Cross-TZ and DST cases run against a computed expected value, never the UI's own arithmetic | TC-D08, TC-W10, TC-W11 |
| Business-hours accrual silently falls back to wall-clock counting | High — patients offered slots the practice can't honour | Med | Contrast pair TC-D06 (ON) vs TC-W06 (OFF) makes a silent fallback visible | TC-D06, TC-W06 |
| PFS renders slots the booking service later rejects | High — patient-visible failure at the last step | Med | Boundary slot is booked, not just observed | TC-D14 |
| Cache serves stale hours after a config change | Med — provider believes a change took effect | High | Propagation cases with an explicit `{SLA}` and a rapid-edit case | TC-D13, TC-W25 |
| Appointment duration ignored at the closing boundary | Med — appointments overrun close | Med | Last-slot-fits assertions | TC-D11, TC-W05 |
| Clock manipulation in the test environment is unreliable | Med — cases silently pass without exercising the boundary | High | Record the observed environment time in every time-sensitive case before asserting | All time-sensitive cases |

---

## Open Questions

1. **TC-W16** — when a location's timezone changes, are the stored hours reinterpreted in the new zone (09:00 stays 09:00) or converted (09:00 ET → 08:00 CT)? The expected result depends entirely on this.
2. **TC-W18** — are overlapping hour blocks rejected or merged? Both are defensible; the test needs the product's answer.
3. **TC-W19** — what is the documented maximum lead time value?
4. **A7 / `{SLA}`** — what is the actual propagation SLA between an Intake Settings save and PFS reflecting it? Every propagation case needs this number.
5. Is business-hours-aware accrual behind a feature flag? If so, both flag states need coverage in the daily set, not just weekly (TC-W06).
6. Can the test environment reliably set the system clock to arbitrary dates? If not, TC-W08–W11 need a different strategy (seeded fixture dates) and their steps must be rewritten.

---

## Exit Criteria

**Daily set — blocking**
- All 14 cases pass. Any failure in TC-D01–D09 or TC-D13–D14 blocks the deploy: these are the cases where breakage means patients see wrong times or cannot book.
- TC-D10, TC-D11, TC-D12 failures require a filed ticket and an explicit sign-off to proceed.

**Weekly set — blocking with sign-off**
- All P0 cases (TC-W01, W02, W05, W08–W11) must pass.
- P1 failures need a filed ticket before the next release train.
- P2 failures (TC-W24, W25, W26) are tracked, non-blocking.

**Around DST transitions**: promote TC-W08–W11 into the daily set for the week before and the week after each US transition.

---

## Exploratory Testing Charters

| Charter | Mission | Time Box | Focus Areas |
|---------|---------|----------|-------------|
| Clock drift hunt | Sit on a PFS page across a lead-time boundary without reloading and watch what happens to the slot list | 30 min | Does a slot silently become unbookable? Is the list refreshed client-side or frozen at page load? |
| Hours-config fuzzing | Push the Intake Settings hours fields into unusual shapes — 00:00–00:00, single-minute windows, all days closed, 24h open | 45 min | Validation gaps, PFS rendering of degenerate configs |
| Timezone matrix sweep | Cycle a patient browser through 6 timezones (including UTC, IST, and a half-hour-offset zone) against one provider | 45 min | Half-hour-offset zones, timezone label correctness, off-by-one-day errors |
| Provider-perspective sanity | As a practice, configure realistic hours and confirm the resulting PFS availability matches what a front-desk person would expect | 60 min | Semantic correctness rather than mechanical correctness — the thing unit tests never catch |

---

## Xray Import Notes

Two test sets to create:
- `Regression — Daily — Lead Time & Business Hours` → TC-D01 … TC-D14
- `Regression — Weekly — Lead Time & Business Hours` → TC-W01 … TC-W26

Component for all cases: split between `Intake Settings` and `PFS Availability` per the Component column. Label all cases `leadtime` and `business-hours`; add `dst` to TC-W08–W13 so they can be pulled into the daily set on demand around transitions.
