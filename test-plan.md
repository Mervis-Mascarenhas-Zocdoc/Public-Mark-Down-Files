# Test Plan: Visit Reason Channel Exclusion — Branded Directory only (SQUAWK-6485)

**Generated**: 2026-07-26
**Sources**: SQUAWK-6485 (description + comments through 2026-07-23), frontend-monorepo PR #22610 (merged 07-23), PR #22478 (merged 07-22), PR #22704 (open), wasabi PR #1489 (merged 07-10), wasabi PR #1500 (merged 07-21), branded e2e spec `profile-channel-vr-exclusion.spec.ts`
**Testing Type**: Desktop Web (Branded Directory / whitelabel `/wl/{brand}/...`)
**Test practice**: `pt_mce4PNR9lEW2owTqkO0rng` (whitelisted by wasabi #1500)
**Feature flags**: Out of scope per QA decision — dev has flipped them ON and left them on (comment 2026-07-23). Assume filtering is active for your session.
**Total Test Cases**: 18
**Priority Breakdown**: P0(5) P1(9) P2(4)

---

## Scope

### In Scope
- The **two** pages the branded-scheduling-web-app actually has: **Profile** and **Search**.
- Branded Profile inline visit-reason dropdown (`procedure-select`) — the surface PR #22610 fixed.
- Branded Profile + Search timesgrid **"Book an appointment"** modal — the surface PR #22478 threaded channel into for all consumers including branded.
- Channel isolation: `branded_directory` vs `marketplace` vs `all` exclusion keys.
- SSR vs client-side navigation render paths (PR #22610 touched both `fetchData.ts` and `withProvider.js`).
- Provider-state variants (virtual care / churned / preview) — PR #22610 deliberately decoupled `providerChannel` from `shouldFetchAvailability`.

### Out of Scope
- Feature-flag on/off and tracking-ID whitelisting (per QA decision).
- Marketplace channel (`www.zocdoc.com`) — already tested; only appears here as a cross-channel isolation assertion.
- Provider/admin-side experience.
- Mobile web, GQL/API-layer direct testing, Lambda load tests (SQUAWK-6784 / -6823).

---

## Pre-Test Setup (blocking)

1. Get from dev (Mawunyo Akabua) for practice `pt_mce4PNR9lEW2owTqkO0rng`:
   - the **branded site slug** → your base URL `/wl/{brand}/...`
   - the **provider slug(s)** that have exclusions
   - per provider: which VR(s) should drop, which should remain, and **which channel key** each exclusion uses (`branded_directory`, `marketplace`, or `all`)
   - (there is a read-only discovery script for this)
2. Confirm the practice is on the server-side **`AllowedPracticeIds`** env allowlist for the environment you're testing. PR #22478 noted *no* branded practice was on it at the time — if it still isn't, nothing will filter and every case below is a false pass.
3. Confirm with dev whether the exclusion rules for this practice were **created or re-saved after 2026-07-10** (wasabi #1489). Rules written before that fix used the wrong Redis key `brandeddirectory` instead of `branded_directory` and fail open; that PR explicitly did **not** backfill. This is the single most likely cause of a "nothing is filtered" result on BD.
4. Note which flag id is live (`channel_vr_exclusion_marketplace` today; PR #22704 renames it to `channel_vr_exclusion_branded_directory` and is still open). You are not testing the flag, but if branded stops filtering mid-run, a #22704 deploy is the first thing to check.

---

## Coverage Matrix

| Change | Source | Test Cases |
|--------|--------|------------|
| Branded profile root `provider()` query sends `providerChannel: BrandedDirectory` | PR #22610 | TC-01, TC-02, TC-07, TC-08 |
| `providerChannel` computed independently of `shouldFetchAvailability` | PR #22610 | TC-09, TC-10 |
| Timesgrid AvailabilityModal passes channel for branded consumers | PR #22478 | TC-03, TC-04, TC-05 |
| Branded search flow book modal (bug filed 2026-07-22) | Comment 1504537 #2 | TC-04, TC-05 |
| Branded profile VR dropdown (bug filed 2026-07-22) | Comment 1504537 #1 | TC-01 |
| `branded_directory` wire-key correctness / channel isolation | wasabi #1489 | TC-11, TC-12, TC-13, TC-14 |
| Zero-visible-VR provider handled gracefully on branded | PR #22704 body | TC-17 |
| Practice allowlist scoping | QA Notes | TC-15 |
| Projection propagation after rule change | wasabi #1489 note, Comment 1475568 | TC-16 |

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation | Tests |
|------|--------|------------|------------|-------|
| Exclusion rule still keyed `brandeddirectory` (no backfill) → BD fails open silently | High | **High** | Setup step 3; re-save rule before testing | TC-16 |
| Branded practice not on `AllowedPracticeIds` → all cases false-pass | High | Med | Setup step 2 | TC-15 |
| Shared timesgrid modal sends the *marketplace* channel from a branded page → marketplace-only exclusions leak onto BD | High | Med | Cross-channel matrix | TC-12, TC-13 |
| SSR path filters but client-nav path doesn't (or vice versa) — two separate code paths in #22610 | Med | Med | Test hard load + in-app nav separately | TC-07, TC-08 |
| Two dropdowns on the same page use different queries (dev flagged this as code-smelly in #22478) → inline dropdown and modal disagree | Med | Med | Compare both on one page load | TC-03 |
| Virtual-care / churned / preview provider skips filtering because availability isn't fetched | Med | Low | #22610 decoupled these — verify | TC-09, TC-10 |

---

## Test Cases

### Pass 1 — Core branded surfaces

| ID | Component | Summary | Preconditions | Steps | Expected Result | Priority | Source |
|----|-----------|---------|---------------|-------|-----------------|----------|--------|
| TC-01 | BD Profile — inline dropdown | Excluded VR absent from branded profile `procedure-select` | Provider in `pt_mce4PNR9lEW2owTqkO0rng` with a `branded_directory` or `all` exclusion | 1. Open `/wl/{brand}/doctor/{provider-slug}` (hard load)<br>2. Open the visit-reason dropdown | The excluded VR from the discovery script is **not** listed in the dropdown. Every non-excluded VR for that provider **is** listed. (This is the bug filed 2026-07-22 #1 — confirm PR #22610 fixed it) | P0 | PR:22610 / BUG:comment-1504537 |
| TC-02 | BD Profile — inline dropdown | Non-excluded VRs remain fully selectable and bookable | Same as TC-01 | 1. Open branded profile<br>2. Select a **non-excluded** VR from the dropdown | The VR is selectable, availability/timeslots render for it, and the page does not error or show "no appointments" where the marketplace shows availability for the same VR | P0 | PR:22610 |
| TC-03 | BD Profile — timesgrid modal | Profile inline dropdown and timesgrid modal agree | Same as TC-01 | 1. Open branded profile, note the exact VR list in the inline dropdown<br>2. Click a timeslot in the timesgrid to open the "Book an appointment" modal<br>3. Open the VR dropdown inside the modal | The modal's VR list is **identical** to the inline dropdown's list — excluded VR absent in both. No VR present in one and missing from the other | P0 | PR:22478 |
| TC-04 | BD Search — book modal | Excluded VR absent from branded search-flow book modal | Same as TC-01; provider surfaces in branded search results | 1. Open `/wl/{brand}/search` and search so the provider appears<br>2. Click the provider's timesgrid / a timeslot to open the "Book an appointment" modal<br>3. Open the VR dropdown | Excluded VR is **not** listed in the modal dropdown; non-excluded VRs are. (This is the bug filed 2026-07-22 #2 — confirm fixed) | P0 | PR:22478 / BUG:comment-1504537 |
| TC-05 | BD Search — search-by-VR | Searching branded directory *for* an excluded VR | Same as TC-01 | 1. On `/wl/{brand}/search`, search/filter by the **excluded** visit reason<br>2. Observe whether the provider appears in results<br>3. If it appears, open its book modal | Record actual behavior: either the provider is absent from results for that VR, or it appears but the excluded VR is not offered in the modal. It must **not** be possible to reach a bookable state for the excluded VR. Flag any inconsistency between search index and read-time filter to dev | P0 | INFERRED |
| TC-06 | BD Profile — booking completion | End-to-end booking on a remaining VR still works | Same as TC-01 | 1. From the branded profile, select a non-excluded VR<br>2. Pick a timeslot and proceed through the booking flow as far as the environment allows | Booking proceeds normally with the correct VR carried through each step; no VR-mismatch or validation error introduced by the channel filter | P1 | INFERRED |

### Pass 2 — Render paths, provider states, channel isolation

| ID | Component | Summary | Preconditions | Steps | Expected Result | Priority | Source |
|----|-----------|---------|---------------|-------|-----------------|----------|--------|
| TC-07 | SSR path (`fetchData.ts`) | Filtering applied on server-rendered first paint | Same as TC-01 | 1. Open the branded profile URL directly in a **fresh tab / hard reload** (no client-side nav)<br>2. Immediately open the VR dropdown; also View Source / disable JS and inspect the initial payload | Excluded VR is absent from the **server-rendered** markup/initial data — it never flashes into the dropdown before disappearing | P1 | PR:22610 |
| TC-08 | Client path (`withProvider.js`) | Filtering applied on client-side navigation | Same as TC-01 | 1. Land on `/wl/{brand}/search`<br>2. Navigate **in-app** (click through, no reload) to the provider's profile<br>3. Open the VR dropdown<br>4. Then browser Back and Forward again | Excluded VR absent on the client-navigated render and after Back/Forward. No difference vs. TC-07's hard-load result | P1 | PR:22610 |
| TC-09 | Virtual-care / churned provider | Filtering applies when availability is not fetched | A provider on the whitelisted practice that is virtual-care-only or churned (availability not fetched) and has an exclusion — ask dev to identify one | 1. Open that provider's branded profile<br>2. Open the VR dropdown | Excluded VR is still filtered out even though availability isn't fetched (#22610 computes `providerChannel` independently of `shouldFetchAvailability`). Page renders without error | P1 | PR:22610 |
| TC-10 | Preview provider | Filtering applies on preview/unpublished profile | A preview-mode provider on the whitelisted practice with an exclusion | 1. Open the provider's branded profile in preview mode | Excluded VR filtered out; preview page renders normally | P2 | PR:22610 |
| TC-11 | Channel key — `branded_directory` | BD-only exclusion drops on BD | Provider with an exclusion keyed **`branded_directory`** only | 1. Open branded profile + search book modal | VR is absent on **both** branded surfaces | P1 | wasabi:1489 |
| TC-12 | Channel isolation | BD-only exclusion does **not** affect marketplace | Same provider as TC-11 | 1. Open the same provider on `www.zocdoc.com` (profile + timesgrid modal) | The `branded_directory`-only excluded VR is **still present** on marketplace — the exclusion does not leak across channels | P1 | wasabi:1489 |
| TC-13 | Channel isolation (reverse) | Marketplace-only exclusion does **not** apply on BD | Provider with an exclusion keyed **`marketplace`** only | 1. Open that provider on the branded profile<br>2. Open the branded search book modal | The marketplace-only excluded VR is **still present** on both branded surfaces. (Highest-value case: the shared timesgrid modal from #22478 serves both channels — if it hardcodes/defaults to `marketplace` from a branded page, this test fails) | P1 | PR:22478 |
| TC-14 | `all` channel key | `all`-keyed exclusion drops on BD | Provider with an exclusion keyed **`all`** (the branded e2e fixture uses this shape) | 1. Open branded profile + search book modal | VR absent on branded; and absent on marketplace too (an `all` key covers every channel) | P1 | e2e-spec |

### Pass 3 — Degradation, scoping, propagation

| ID | Component | Summary | Preconditions | Steps | Expected Result | Priority | Source |
|----|-----------|---------|---------------|-------|-----------------|----------|--------|
| TC-15 | Allowlist scoping | Non-whitelisted practice on the same branded site is untouched | A provider on the **same** `/wl/{brand}` site whose practice is **not** in `AllowedPracticeIds` | 1. Open that provider's branded profile and search book modal | Full VR list shown, unchanged from pre-change behavior. Confirms filtering is scoped to the allowlisted practice, not the whole branded site | P1 | AC:SQUAWK-6485 |
| TC-16 | Projection / backfill | Rule change propagates with the correct `branded_directory` key | Dev able to add/remove a `branded_directory` exclusion for a provider on the whitelisted practice | 1. Have dev add a new BD exclusion<br>2. After the propagation window, reload the branded profile and search modal<br>3. Have dev remove it<br>4. Reload again | Newly excluded VR disappears from both branded surfaces after propagation; after removal it **reappears**. No stale list. Confirms the corrected wire key is written (wasabi #1489 shipped no backfill) | P2 | wasabi:1489 |
| TC-17 | Degradation — all VRs excluded | Provider with zero visible VRs on BD | Provider on the whitelisted practice with **all** VRs excluded for `branded_directory` (dev-configured) | 1. Open the branded profile<br>2. Open the branded search results for that provider and its book modal | Branded profile degrades gracefully — a defined empty state or booking cleanly unavailable, **no** "Practice not found"-style error, no blank page, no unhandled JS/500. (PR #22704 states branded handles this case gracefully while marketplace throws — verify that claim) | P2 | PR:22704 |
| TC-18 | Cross-surface consistency sweep | No branded surface leaks an excluded VR | Same as TC-01 | 1. Reach the provider's VR list via every branded path: search results → timesgrid → book modal; search → profile → inline dropdown; profile → timesgrid modal; direct profile deep-link; direct book deep-link if one exists; then Back/Forward through each | Excluded VR appears on **none** of these surfaces. Note the exact path of any leak — the marketplace round found four separate leaking entry points, so treat unlisted branded entry points as likely gaps | P2 | BUG:comment-1504492 |

---

## Open Questions (resolve with dev before/while testing)

1. **Was the exclusion rule for `pt_mce4PNR9lEW2owTqkO0rng` written or re-saved after 2026-07-10?** wasabi #1489 fixed the `brandeddirectory` → `branded_directory` key but shipped no backfill. If not re-saved, BD will fail open and everything reads as "not working."
2. **Is a branded practice actually on `AllowedPracticeIds`** in your test environment? PR #22478 said none were.
3. **Does branded search filter at index time or read time?** (TC-05) — determines whether the provider should still surface when searching *for* an excluded VR.
4. **Does the branded app have any booking surface beyond Profile and Search** (e.g. a direct `/book/` deep link)? If yes, add it to TC-18.
5. Backfill question from comment 1475568 is still open on the ticket.

## Assumptions

- ASSUMPTION: Flags are ON for branded in the test environment (dev comment 2026-07-23) — not re-verified per QA decision.
- ASSUMPTION: Branded profile URL shape is `/wl/{brand}/doctor/{provider-slug}` and search is `/wl/{brand}/search`, per the branded e2e spec.
- ASSUMPTION: The discovery script output is the source of truth for expected dropped/remaining VRs and their channel keys.

---

## Exit Criteria

- **Blocking**: TC-01 through TC-05 pass (both previously-filed BD bugs confirmed fixed), plus TC-13 (no marketplace exclusion leaking onto BD).
- **Should pass**: TC-06 through TC-12, TC-14, TC-15.
- **Before wider ramp**: TC-16, TC-17, TC-18 executed and Open Questions 1–3 answered.

## Exploratory Charter

| Charter | Mission | Time Box | Focus |
|---------|---------|----------|-------|
| Branded VR exclusion leak hunt | Reach the provider's visit-reason list through every path available on the branded site and hunt for one that still offers the excluded VR | 45 min | Search → timesgrid → modal, profile inline dropdown vs modal (different queries per #22478), SSR vs client nav, Back/Forward, deep links, second provider on the same practice |
| Cross-channel bleed | Same provider, same VR, side-by-side branded vs marketplace tabs for each exclusion key (`marketplace`, `branded_directory`, `all`) | 30 min | Verify each key affects exactly the channels it should — no more, no less |
