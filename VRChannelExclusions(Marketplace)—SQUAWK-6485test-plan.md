# Test Plan: Visit Reason Channel Exclusions (Marketplace) — SQUAWK-6485

**Generated**: 2026-07-21
**Source**: SQUAWK-6485 (QA Notes comment by Mawunyo Akabua)
**Testing Type**: Desktop Web
**Mode**: Concise (variations consolidated into data tables)
**Feature Flags**: `channel_vr_exclusion_filtering` (backend, keyed on tracking ID) + `channel_vr_exclusion_marketplace` (frontend, keyed on tracking ID). Producer/projection flags already ON globally.
**Total Test Cases**: 12
**Priority Breakdown**: P0(4) P1(5) P2(3)

---

## Scope

### In Scope
- Marketplace-channel visit-reason (VR) exclusion on Desktop Web: provider profile, practice page, booking flow.
- Flag on/off behavior and tracking-ID whitelisting.
- Regression: providers/practices outside the allowlist, other channels, provider-side.

### Out of Scope
- Affiliate ScheduleWidget / Branded Directory embed (must always show full list — covered only as a regression guard, TC-09).
- Provider/admin-side experience (no roles other than anonymous/patient).
- Mobile Web, GQL/API-layer direct testing, backend Lambda/pipeline load tests (SQUAWK-6784 / -6823).

---

## Pre-Test Setup (do this first)
1. Get the **read-only discovery script** from the assignee (Mawunyo Akabua) and run it against test practice **`pt_cUdrQ_-fkEGyvWJ024id8h`** to identify: (a) which providers have exclusions, (b) exactly which VR(s) should drop, (c) which VR(s) should remain.
2. Whitelist your **tracking ID** on BOTH `channel_vr_exclusion_filtering` and `channel_vr_exclusion_marketplace`; set both to **"Running"** (audience % can stay 0).
3. Confirm your browser session carries the whitelisted tracking ID (fresh session / same cookie).
4. Note: additional practices are NOT testable without a code change to `AllowedPracticeIds` (redeploy).

---

## Coverage Matrix

| Requirement | Source | Test Cases | Covered? |
|-------------|--------|------------|----------|
| Excluded VRs absent on profile / practice / booking (marketplace) | QA Notes | TC-01, TC-02, TC-03 | Yes |
| Non-excluded VRs remain | QA Notes | TC-01–03 | Yes |
| Timeslot grid respects exclusion (known bug) | Comment 3 | TC-04 | Yes |
| Non-whitelisted user unaffected | QA Notes edge cases | TC-05 | Yes |
| Frontend flag OFF restores full list | QA Notes edge cases | TC-06 | Yes |
| Flag-combo: FE off + filtering on → no drop | QA Notes edge cases | TC-07 | Yes |
| Bad/blank tracking ID → no filtering | QA Notes edge cases | TC-08 | Yes |
| Other channel (ScheduleWidget) shows full list | QA Notes related | TC-09 | Yes |
| Provider outside test practice → full list | QA Notes edge cases | TC-10 | Yes |
| All/most VRs excluded → graceful degradation | Breakage | TC-11 | Yes |
| Projection propagation of new exclusion | Comment 1 / wasabi | TC-12 | Partial |

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation | Related Tests |
|------|--------|------------|------------|---------------|
| Timeslot grid leaks excluded VRs (known open bug) | High | High | Verify fix landed before sign-off | TC-04 |
| Booking deep-link bypasses filter | High | Med | Test `/book/` entry directly | TC-03 |
| Exclusion dict projection lag → stale list | Med | Med | Re-check after propagation delay | TC-12 |
| Provider side accidentally filtered | High | Low | Confirm `"all"` channel bypass | (out of scope UI; flag to dev) |

---

## Open Questions
- Backfill behavior (Comment 1): do exclusions configured before the projection flag was enabled propagate correctly? Confirm with dev.
- Empty-picker UX when all VRs excluded — is there a defined empty state, or is such a config invalid? (TC-11 documents observed behavior.)

## Assumptions
- ASSUMPTION: The discovery script output is the source of truth for expected dropped/remaining VRs.
- ASSUMPTION: Producer + projection flags remain ON globally during testing.

---

## Test Cases

### Pass 1: Specification Testing

| ID | Component | Summary | Preconditions | Steps | Expected Result | Priority | Source |
|----|-----------|---------|---------------|-------|-----------------|----------|--------|
| TC-01 | Provider profile | Excluded VR hidden, others remain (profile) | Setup done; whitelisted session; provider with known exclusion | 1. Open `/{specialty}/{provider-slug}`<br>2. Open the visit-reason picker | Excluded VR(s) from discovery script are absent; all non-excluded VRs present in the picker | P0 | AC:SQUAWK-6485 |
| TC-02 | Practice page | Excluded VR hidden, others remain (practice) | Same as TC-01 | 1. Open `/practice/{practice-slug}`<br>2. Locate the same provider<br>3. Open the VR picker | Same exclusion applies for that provider; other providers on the practice page filtered independently per their own config | P0 | AC:SQUAWK-6485 |
| TC-03 | Booking flow | Excluded VR hidden in booking + deep-link | Same as TC-01 | 1. Enter `/book/...` for the provider (a) via profile CTA, (b) via a direct deep-link URL<br>2. View VR selection | Excluded VR(s) absent in both entry paths; non-excluded VRs bookable | P0 | AC:SQUAWK-6485 |
| TC-04 | Timeslot grid | Timesgrid respects exclusion (regression of known bug) | Same as TC-01 | 1. On profile/practice, click into the timeslot/timesgrid view for the provider | Excluded VR(s) do NOT reappear in the timesgrid; matches picker. (Was failing per Comment 3 — confirm fix) | P0 | BUG:SQUAWK-6485 |
| TC-05 | Gating | Non-whitelisted user unaffected | Session WITHOUT whitelisted tracking ID | 1. Open same provider profile/practice/booking | Full VR list shows (no filtering) on all pages | P1 | AC:SQUAWK-6485 |
| TC-06 | Flag control | Frontend flag OFF restores full list | Whitelisted session; set `channel_vr_exclusion_marketplace` OFF | 1. Reload provider profile/practice/booking | Full VR list restored on all pages | P1 | AC:SQUAWK-6485 |
| TC-09 | Channel isolation | ScheduleWidget embed shows full list | Affiliate ScheduleWidget embed for same provider | 1. Load the affiliate embed | Full VR list shown; exclusion NOT applied (channel ≠ marketplace) | P1 | AC:SQUAWK-6485 |
| TC-10 | Allowlist scope | Provider outside test practice → full list | Whitelisted session; provider NOT in `pt_cUdrQ_-fkEGyvWJ024id8h` | 1. Open that provider's profile/booking | Full VR list; no filtering (practice not in `AllowedPracticeIds`) | P1 | AC:SQUAWK-6485 |

### Pass 2: Breakage Hunting

| ID | Component | Summary | Preconditions | Steps | Expected Result | Priority | Source |
|----|-----------|---------|---------------|-------|-----------------|----------|--------|
| TC-07 | Flag combo | FE off + filtering on → nothing drops | Whitelisted; `channel_vr_exclusion_marketplace` OFF, `channel_vr_exclusion_filtering` ON | 1. Open provider profile/practice/booking | Full VR list; no VRs dropped (frontend gate wins) | P1 | AC:SQUAWK-6485 |
| TC-08 | Gating | Blank/wrong tracking ID → no filtering | Session with blank or non-whitelisted tracking ID | 1. Open provider pages | No filtering; full VR list; no errors | P2 | AC:SQUAWK-6485 |
| TC-11 | Degradation | All/most VRs excluded → graceful state | Configure (or find via script) a provider with all VRs excluded | 1. Open profile/practice/booking | Picker shows a defined empty state OR booking blocked cleanly — no blank/broken page, no JS error | P2 | INFERRED |

### Pass 3: Historical / Regression

| ID | Component | Summary | Preconditions | Steps | Expected Result | Priority | Source |
|----|-----------|---------|---------------|-------|-----------------|----------|--------|
| TC-12 | Projection | New exclusion propagates (wasabi) | Ability to configure a new exclusion (dev-assisted) | 1. Configure a new VR exclusion<br>2. Reload provider pages after propagation delay | Newly excluded VR disappears from marketplace pages within expected propagation window; no stale list | P2 | INFERRED |

---

## Exit Criteria
- **Blocking:** TC-01–04 (all P0) pass — including the timesgrid fix (TC-04).
- **Should pass:** TC-05, 06, 07, 09, 10 (gating + channel isolation) pass.
- **Before wider rollout:** TC-08, 11, 12 reviewed; backfill open question resolved with dev.

## Exploratory Charter
| Charter | Mission | Time Box | Focus |
|---------|---------|----------|-------|
| Marketplace VR exclusion | Probe every path a patient can reach a provider's VR list (search → profile → practice → book → timesgrid, plus direct deep-links and back-button navigation) for a whitelisted session, hunting for any surface that leaks an excluded VR | 45 min | Deep-links, cached pages, back/forward nav, timesgrid, practice-page multi-provider |
