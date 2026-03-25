# Suggested Test Cases — Provider Inbox & Provider Configuration

> **Date:** 2026-03-25
> **Source Document:** [Regression and Smoke Tests - Appointment Management](https://docs.google.com/document/d/1Ybw4Ze6_PAqrnKHKd-pEP825F4UptunrP09vLmjrI7s/edit)
> **Scope:** Business-critical test cases for regression and smoke testing

---

## Table of Contents

- [Provider Inbox](#provider-inbox)
  - [Smoke Tests — Core Inbox](#smoke-tests--core-inbox)
  - [Regression — Appointment Flyout Actions](#regression--appointment-flyout-actions)
  - [Regression — Intake Submissions](#regression--intake-submissions)
  - [Regression — Role-Based Access](#regression--role-based-access)
  - [Regression — Multi-Practice & Edge Cases](#regression--multi-practice--edge-cases)
- [Provider Configuration](#provider-configuration)
  - [Smoke Tests — Calendar Core](#smoke-tests--calendar-core)
  - [Regression — Availability Management](#regression--availability-management)
  - [Regression — Synced Availability](#regression--synced-availability)
  - [Regression — Timezone Handling](#regression--timezone-handling)
  - [Regression — Calendar Display & Settings](#regression--calendar-display--settings)
  - [Regression — Mobile Calendar](#regression--mobile-calendar)
  - [Regression — Insurance & Network Settings](#regression--insurance--network-settings)
- [Lead Time](#lead-time)
  - [Smoke Tests — Lead Time Critical Path](#smoke-tests--lead-time-critical-path)
  - [Regression — Dropdown Options & Urgent Care](#regression--dropdown-options--urgent-care)
  - [Regression — Three-Tier Configuration Hierarchy](#regression--three-tier-configuration-hierarchy)
  - [Regression — Virtual Locations](#regression--virtual-locations)
  - [Regression — Table UX & Pagination](#regression--table-ux--pagination)
  - [Regression — Unsaved Changes](#regression--unsaved-changes)
  - [Regression — Error Handling](#regression--error-handling)
  - [Regression — Visibility Rules](#regression--visibility-rules)
- [Business Hours](#business-hours)
  - [Smoke Tests — Business Hours Critical Path](#smoke-tests--business-hours-critical-path)
  - [Regression — Day Selection](#regression--day-selection)
  - [Regression — Time Range Validation](#regression--time-range-validation)
  - [Regression — Save, Reset & Change Detection](#regression--save-reset--change-detection)
  - [Regression — Integration with Lead Time](#regression--integration-with-lead-time)
  - [Regression — Patient-Facing Display](#regression--patient-facing-display)
  - [Regression — Navigation & Screen Flow](#regression--navigation--screen-flow)
- [Summary & Prioritization](#summary--prioritization)

---

## Provider Inbox

### Smoke Tests — Core Inbox

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PI-S-01 | Verify inbox loads with correct appointment counts per tab (All, New Bookings, Reschedules, Cancellations, Intake Submissions) | Core functionality — if tab counts are wrong, staff misses appointments |
| PI-S-02 | Verify patient name search filters the appointment list correctly (debounced) | Staff need to quickly find specific patients |
| PI-S-03 | Verify provider and location filter dropdowns persist across page refreshes (localStorage) | Filters resetting causes repeated manual re-selection, time wasted |
| PI-S-04 | Verify "Clear All Filters" button resets provider, location, and patient name filters | Staff stuck with stale filters may miss appointments |
| PI-S-05 | Verify pagination works correctly — navigating pages shows correct appointments | Missing page 2+ appointments = missed patients |
| PI-S-06 | Verify sorting by "Last Updated by Patient" and "Appointment Time" in ascending/descending | Staff rely on sorting to prioritize workload |
| PI-S-07 | Verify auto-refresh updates appointment list in the background without showing a loading indicator | Real-time awareness of new bookings without UI disruption |
| PI-S-08 | Verify marking an appointment as read (ReadByPractice status) updates the notification badge | Unread indicators help staff track which items need attention |
| PI-S-09 | Verify mobile inbox loads correctly with tab filters and appointment list | Growing mobile usage — broken mobile = lost productivity |
| PI-S-10 | Verify mobile filters modal opens, allows provider/location selection, and applies correctly | Mobile staff need filtering too |

### Regression — Appointment Flyout Actions

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PI-R-01 | Verify "Request Patient Call" action sends the request successfully from the flyout | Missed call requests = missed patient communication |
| PI-R-02 | Verify "Collect Intake" button opens the intake collection modal and sends intake reminder | Intake collection drives revenue and pre-visit preparation |
| PI-R-03 | Verify cancellation modal shows reason selection and completes the cancellation flow (including abatement scenarios) | Incorrect cancellation flow = compliance/billing issues |
| PI-R-04 | Verify modify appointment restricts location selection to only approved locations (`isApprovedLocation`) | Recent feature — unapproved locations could cause scheduling errors |
| PI-R-05 | Verify insurance eligibility information displays correctly in the appointment flyout accordion | Insurance verification is critical for billing |
| PI-R-06 | Verify "One-Click Insurance" add/remove/decline actions work from the flyout | Speeds up insurance management workflow |
| PI-R-07 | Verify booking activity section in the accordion shows correct history timeline | Staff need audit trail of appointment changes |
| PI-R-08 | Verify referral section displays correctly when referral data exists | Referral tracking is required for specialist visits |

### Regression — Intake Submissions

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PI-R-09 | Verify Manual Intake Request flyout opens, validates required fields (name, date/time, contact method), and submits | Manual intake covers non-Zocdoc patients — critical for practices using intake broadly |
| PI-R-10 | Verify "Get Intake by Zocdoc" button works from the manual intake flyout | Connects manual flow to Zocdoc intake system |
| PI-R-11 | Verify intake completion status displays correctly for appointments (via `/intake/v3/completion/`) | Staff need to know which patients completed intake |
| PI-R-12 | Verify "Send Intake Reminder" action triggers successfully | Reminders drive intake completion rates |

### Regression — Role-Based Access

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PI-R-13 | Verify Full Access user can access all inbox features (filters, flyout actions, intake) | Access control validation |
| PI-R-14 | Verify "Appointments + Practice Settings" role user can access inbox but not admin-only features | Least-privilege enforcement |
| PI-R-15 | Verify "Appointments Only" role user can view inbox but cannot modify practice settings | Prevents unauthorized configuration changes |

### Regression — Multi-Practice & Edge Cases

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PI-R-16 | Verify multi-practice user can switch between practices and see correct inbox data per practice | Providers managing multiple practices need isolation |
| PI-R-17 | Verify API appointments, marketplace appointments, and patient intake appointments all appear correctly with proper source indicators | Different booking sources must all be visible |
| PI-R-18 | Verify appointment flyout handles appointments in different timezones correctly (practice timezone display) | Timezone bugs cause wrong appointment times shown |
| PI-R-19 | Verify inbox tutorial/onboarding coachmarks display correctly for first-time users | First impressions matter for new practice staff |
| PI-R-20 | Verify the "New to Inbox" modal appears for new users and can be dismissed | Onboarding experience |

---

## Provider Configuration

### Smoke Tests — Calendar Core

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-S-01 | Verify calendar loads in week view with correct availability blocks, busy blocks, and appointments for the selected provider/location | Core functionality — broken calendar = no scheduling |
| PC-S-02 | Verify adding a one-time availability slot with start/end date, time, and location saves correctly and appears on calendar | Providers need to add availability to receive bookings |
| PC-S-03 | Verify adding a recurring (weekly) availability slot saves correctly and generates slots for each recurrence | Most availability is recurring — this is the primary workflow |
| PC-S-04 | Verify editing an existing availability (change time, location, days of week) saves and updates on calendar | Schedule changes are daily operations |
| PC-S-05 | Verify deleting an availability removes it from the calendar | Providers need to close slots when unavailable |
| PC-S-06 | Verify creating a busy block on an existing availability slot blocks that time period | Busy blocks prevent bookings during unavailable times |
| PC-S-07 | Verify switching between day view and week view shows correct data | Both views are used by different practice workflows |
| PC-S-08 | Verify provider and location filters show/hide the correct calendar entries | Multi-provider practices need to filter by provider |

### Regression — Availability Management

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-R-01 | Verify adding a bi-weekly recurring availability saves with correct recurrence pattern | Supports providers with alternating schedules |
| PC-R-02 | Verify editing a single instance of a recurring availability (override) changes only that instance | Staff need to modify one day without affecting the series |
| PC-R-03 | Verify deleting a single instance of a recurring availability removes only that instance | Prevents accidental deletion of entire recurring series |
| PC-R-04 | Verify overlapping availability detection — modal warns when new availability overlaps existing | Prevents double-booking and scheduling confusion |
| PC-R-05 | Verify adding virtual visit availability allows selection of virtual location and visit type | Telehealth is a growing booking channel |
| PC-R-06 | Verify in-person and virtual location dropdowns show correct options based on practice config | Wrong location options = scheduling errors |
| PC-R-07 | Verify availability cannot be edited when it's in the past (past-date validation) | Prevents invalid historical changes |
| PC-R-08 | Verify busy block edit and delete operations work correctly | Staff need to manage blocked time |
| PC-R-09 | Verify Out of Office block can be created and edited | Out of Office covers vacation/leave scenarios |

### Regression — Synced Availability

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-R-10 | Verify synced availability (from external scheduling system) displays as read-only on the calendar | Synced practices should not manually edit availability |
| PC-R-11 | Verify clicking synced availability opens the "Read from your scheduling software" modal with correct details (date, time, timezone, location) | Staff need to verify what the sync pulled in |
| PC-R-12 | Verify synced availability does NOT show edit/delete actions in the popover | Editing synced availability would create data conflicts |
| PC-R-13 | Verify providers with `isUsingSchedulingApiAvailability` cannot manually add availability | Prevents manual/synced availability conflicts |

### Regression — Timezone Handling

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-R-14 | Verify timezone filter changes the calendar display to the selected timezone | Multi-timezone practices need correct time display |
| PC-R-15 | Verify availability blocks crossing midnight are clipped/displayed correctly | Edge case that causes visual display bugs |
| PC-R-16 | Verify location timezone is shown in filter dropdowns and availability modals | Staff need to know which timezone they're scheduling in |

### Regression — Calendar Display & Settings

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-R-17 | Verify "Display on my calendar" toggle correctly shows/hides availability vs appointments | Staff use this to declutter their view |
| PC-R-18 | Verify appointment blocks on calendar show correct patient info, time, and status | Visual verification that bookings landed correctly |
| PC-R-19 | Verify calendar navigation (previous/next day/week, date picker) works correctly | Basic navigation — broken = unusable calendar |
| PC-R-20 | Verify slot increment settings (5/10/15/30/60 min) display correct grid intervals | Slot granularity affects booking accuracy |
| PC-R-21 | Verify US holidays are displayed/indicated on the calendar | Practices need holiday awareness for scheduling |

### Regression — Mobile Calendar

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-R-22 | Verify mobile calendar grid renders availability and appointments correctly | Recent feature — mobile calendar is new |
| PC-R-23 | Verify mobile provider filter modal opens and applies filters | Mobile filtering is a newly shipped feature |
| PC-R-24 | Verify mobile date picker modal navigates between dates | Core mobile navigation |

### Regression — Insurance & Network Settings

| # | Test Case | Business Impact |
|---|-----------|----------------|
| PC-R-25 | Verify IN-Network opt-in allows provider to appear on in-network search results | Directly impacts patient acquisition and search visibility |
| PC-R-26 | Verify adding insurance carriers/networks/plans through the multi-step form saves correctly | Insurance configuration affects which patients can book |
| PC-R-27 | Verify removing an insurance carrier shows confirmation modal and removes after confirmation | Accidental removal = patients can't book |
| PC-R-28 | Verify Out-of-Network settings modal allows configuration | OON listings affect patient search results |
| PC-R-29 | Verify Accepted Programs and Plan Types modals save correctly | Governs which insurance plans are accepted |
| PC-R-30 | Verify provider-location mapping editor displays current mappings and allows updates | Controls which providers serve which locations |
| PC-R-31 | Verify `isApprovedLocation` filtering works — only approved locations appear in relevant flows | Recent feature — affects appointment modification and scheduling accuracy |

---

## Lead Time

### Smoke Tests — Lead Time Critical Path

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-S-01 | Verify Lead Time settings page loads when feature flag `IntakeSettings_ShouldShowLeadTimeSettings` is ON, and is hidden when OFF | Feature-gated — if flag misconfigured, practices lose access to lead time management |
| LT-S-02 | Verify practice-level lead time can be changed via the main dropdown and saves successfully (toast: "Changes saved") | Practice default affects ALL providers/locations without overrides — incorrect save = wrong booking windows |
| LT-S-03 | Verify default lead time is "2 business hours" for regular providers | Default drives first-available slot calculation for all new practices |
| LT-S-04 | Verify "Manage locations" link navigates to the lead time breakdown view with Locations tab active | Primary entry point for per-location configuration |
| LT-S-05 | Verify changing a location-specific lead time override saves correctly and displays the new value in the table | Per-location overrides are the most common configuration — broken save = incorrect booking availability |

### Regression — Dropdown Options & Urgent Care

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-01 | Verify regular (non-urgent-care) providers see options: 30 min, 1 hour, 2 hours, 1 day, 2+ days | Wrong options shown = invalid lead time selection |
| LT-R-02 | Verify urgent care providers (specialties sp_560, sp_563, sp_561, sp_562, sp_513) see additional options: 1 minute, 15 minutes | Urgent care needs short lead times — missing options prevents same-day bookings |
| LT-R-03 | Verify urgent care provider default is "15 business minutes" (not 2 hours) | Recent bug fix — wrong default shown = confusion for urgent care practices |
| LT-R-04 | Verify urgent care providers can always select the "1 hour" lead time option | Recent change — 1-hour option was previously missing for some urgent care configs |

### Regression — Three-Tier Configuration Hierarchy

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-05 | Verify changing practice-level lead time cascades to all locations/providers WITHOUT custom overrides | Inheritance model — if cascade is broken, child entities show stale values |
| LT-R-06 | Verify locations with custom overrides are NOT affected when practice-level default is changed | Overrides must be respected — otherwise custom configs get silently overwritten |
| LT-R-07 | Verify "Reset to default" on a location removes the override and reverts to practice default | Staff need to undo per-location customization cleanly |
| LT-R-08 | Verify provider-level lead time override (internal users only) takes precedence over location and practice defaults | Three-tier priority: Provider > Location > Practice |
| LT-R-09 | Verify providers tab is visible ONLY for internal users and hidden for external users | Provider-level overrides are internal-only — exposing to external users causes confusion |
| LT-R-10 | Verify non-overridden providers display "Determined by practice/location" label | Inheritance indicator helps staff understand which level controls the value |

### Regression — Virtual Locations

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-11 | Verify all virtual locations are grouped into a single "All virtual locations" row in the Locations tab | Virtual locations share one lead time — if ungrouped, staff sees confusing duplicate rows |
| LT-R-12 | Verify changing lead time for "All virtual locations" updates ALL virtual location entities together | Partial update = some virtual slots have wrong availability |
| LT-R-13 | Verify "Reset to default" for virtual locations cascades the delete to all virtual location entities | Partial delete = inconsistent state across virtual locations |

### Regression — Table UX & Pagination

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-14 | Verify lead time table pagination stays on current page after saving changes (does not auto-reset to page 1) | Recent fix — resetting pagination mid-edit disrupts workflow |
| LT-R-15 | Verify search/filter in the lead time table filters locations or providers by name (case-insensitive) | Practices with many locations need search to find specific entries |
| LT-R-16 | Verify inline edit mode activates on row click and allows dropdown selection | Core edit interaction — if broken, no way to change per-entity lead times |
| LT-R-17 | Verify editing a row and clicking another row without saving does NOT lose changes (or warns appropriately) | Prevents accidental data loss during multi-row editing |

### Regression — Unsaved Changes

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-18 | Verify navigating away from lead time breakdown with unsaved changes triggers the browser confirmation dialog | Prevents accidental loss of configuration changes |
| LT-R-19 | Verify the unsaved changes count is accurate (reflects number of modified entities) | Incorrect count misleads staff about what's pending |
| LT-R-20 | Verify "Save & leave" in the unsaved changes modal persists all changes before navigating | Staff expects changes to be saved — silent discard = lost work |

### Regression — Error Handling

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-21 | Verify partial failure when saving multiple lead times shows error toast but persists successful saves | Batch update with one failure shouldn't roll back all changes |
| LT-R-22 | Verify FORBIDDEN_ACCESS error displays "You don't have permission..." toast message | Role-based access — incorrect error message confuses staff |
| LT-R-23 | Verify lead time loading error state shows retry option (LeadTimeLoadingError view) | Network failures should be recoverable without page refresh |

### Regression — Visibility Rules

| # | Test Case | Business Impact |
|---|-----------|----------------|
| LT-R-24 | Verify breakdown view is shown for external users ONLY when practice has 2+ non-virtual locations | Single-location practices don't need breakdown — showing it adds unnecessary complexity |
| LT-R-25 | Verify breakdown view is shown for internal users when practice has 1+ locations | Internal users need access for all practices |
| LT-R-26 | Verify default active tab is "Providers" for internal users with a single non-virtual location, otherwise "Locations" | Smart tab default reduces clicks for the most common use case |

---

## Business Hours

### Smoke Tests — Business Hours Critical Path

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-S-01 | Verify Business Hours settings page loads when feature flag `IntakeSettings_ShouldShowBusinessHoursSettings` is ON, and is hidden when OFF | Feature-gated — misconfigured flag = no access to business hours |
| BH-S-02 | Verify default business hours are Monday–Friday, 8:00 AM – 6:00 PM | Default must match expected industry standard — wrong default affects all booking calculations |
| BH-S-03 | Verify changing business hours days and time range saves successfully via "Save" button | Core save flow — if broken, lead time calculations use stale hours |
| BH-S-04 | Verify "Edit business hours" link on the Lead Time page navigates to the Business Hours screen | Primary entry point — broken navigation = feature inaccessible |

### Regression — Day Selection

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-R-01 | Verify selecting/deselecting individual days of the week (Sunday through Saturday) updates the configuration | Practices with weekend hours need Saturday/Sunday enabled |
| BH-R-02 | Verify at least one day must be selected — error shown: "You must add business hours for at least one day of the week" | Zero-day config = lead time can never be calculated = patients can't book |
| BH-R-03 | Verify deselecting all days and attempting to save shows validation error and Save button is disabled | Prevents saving invalid state |

### Regression — Time Range Validation

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-R-04 | Verify start time must be before end time — error shown: "Start time must occur before the end time" | Inverted range = incorrect lead time window calculation |
| BH-R-05 | Verify each day's time range is validated independently (one invalid day doesn't block saving valid days) | Per-day validation — staff shouldn't be blocked from saving valid entries |
| BH-R-06 | Verify invalid days are visually highlighted and tracked in the UI | Staff needs clear indication of which day has the error |
| BH-R-07 | Verify time selectors display in 12-hour format (e.g., "2:30 PM") | 24-hour format confuses US-based practice staff |

### Regression — Save, Reset & Change Detection

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-R-08 | Verify Save button is disabled when no changes have been made | Prevents unnecessary API calls and confusion |
| BH-R-09 | Verify Save button is disabled when there are validation errors (even if changes exist) | Prevents saving invalid configuration |
| BH-R-10 | Verify "Reset to default" reverts business hours to Monday–Friday 8 AM – 6 PM and calls delete mutation | Staff need a clean way to undo custom hours |
| BH-R-11 | Verify Reset button is only shown when business hours differ from default | Showing reset when already default is confusing |
| BH-R-12 | Verify after save, the Apollo cache is updated and Lead Time page reflects new business hours without page refresh | Stale cache = lead time page shows old hours = inconsistent UI |

### Regression — Integration with Lead Time

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-R-13 | Verify business hours are fetched together with lead times in a single GraphQL query (`GET_LEAD_TIMES_QUERY`) | Separate fetches could show mismatched data or cause race conditions |
| BH-R-14 | Verify lead time calculation respects business hours — e.g., 2-hour lead time at 5 PM Friday (Mon-Fri 8-6 hours) should push first available to 10 AM Monday | Core business logic — wrong calculation = patients see incorrect available slots |
| BH-R-15 | Verify changing business hours to include weekends (e.g., adding Saturday) immediately affects when lead time applies | Practice opening on weekends should reflect in next booking availability |

### Regression — Patient-Facing Display

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-R-16 | Verify patient profile shows "Open now" (green) when current time is within business hours | Patients use open/closed status to decide when to call or visit |
| BH-R-17 | Verify patient profile shows "Closed" (red) when current time is outside business hours | Incorrect status misleads patients |
| BH-R-18 | Verify patient profile shows "Open 24 hours" for locations with 00:00–24:00 configuration | Edge case — 24-hour locations must display correctly |
| BH-R-19 | Verify location business hours display correct day-by-day schedule (e.g., "Mon-Fri: 8:00am - 5:00pm, Sat: Closed") | Patient-visible — incorrect hours cause failed visits |

### Regression — Navigation & Screen Flow

| # | Test Case | Business Impact |
|---|-----------|----------------|
| BH-R-20 | Verify Back button from Business Hours screen returns to the Lead Time settings (not intake settings root) | Broken back navigation = disorienting UX |
| BH-R-21 | Verify Business Hours screen is accessible via the Settings navigation sidebar ("Working Hours" item at `/provider/settings/synchronizer`) | Secondary entry point must work |

---

## Summary & Prioritization

### Test Case Counts

| Module | Smoke | Regression | Total |
|--------|-------|------------|-------|
| Provider Inbox | 10 | 20 | **30** |
| Provider Configuration | 8 | 31 | **39** |
| Lead Time | 5 | 26 | **31** |
| Business Hours | 4 | 21 | **25** |
| **Grand Total** | **27** | **98** | **125** |

### Top Priority Test Cases (Highest Risk / Most Recent Changes)

| Priority | Test Case ID | Description | Reason |
|----------|-------------|-------------|--------|
| P0 | LT-R-14 | Pagination not resetting after save | Recent bug fix, high regression risk |
| P0 | LT-R-03, LT-R-04 | Urgent care defaults and options | Recent fixes, affects same-day booking |
| P0 | BH-R-14 | Lead time respects business hours boundary | Core integration point between both features |
| P0 | LT-R-11 to LT-R-13 | Virtual location grouping | Complex batch logic with partial failure risk |
| P0 | BH-R-02 | Business hours day validation (zero-day) | Zero-day config breaks all lead time calculations |
| P1 | PI-R-04 | Approved locations in modify appointment | Recent feature (`isApprovedLocation`), high risk |
| P1 | PC-R-10 to PC-R-13 | Synced availability (read-only) | EHR-integrated practices, data integrity |
| P1 | PI-R-09 to PI-R-12 | Manual intake flows | Revenue-impacting, untested |
| P1 | PC-R-22 to PC-R-24 | Mobile calendar | Recently shipped, no coverage |
| P1 | PC-R-04 | Overlapping availability detection | Prevents double-booking |
