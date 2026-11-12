# Admissions Counselling Slots: Original-Cohort Attribution Report

A MySQL 8 query that builds a clean, deduplicated report of booked admissions counselling sessions, attributing each booking to the cohort the student **originally** applied to and the entrance fee they **actually** paid.

## Problem Statement

An ed-tech admissions team books one-on-one counselling slots with prospective students. Reporting on these bookings looked simple, but the raw slot table gave misleading numbers for three reasons.

First, students move between cohorts after booking. They can be shifted manually (recorded in a cohort-shift history) or redirected automatically after a qualifier exam. The slot table only stores the *current* cohort, so bookings were credited to the cohort the student ended up in rather than the one that actually generated the lead.

Second, the entrance fee on a booking is often misleading. Students approved through the SOP (Statement of Purpose) route may carry a ₹0 entrance transaction on their new cohort, even though they paid earlier on a different cohort. Reading the amount directly undercounts revenue attributable to counselling.

Third, the data contains noise: test or low-intent categories, critical-risk leads, slots still awaiting a preferred-time confirmation, off-grid slot times, and duplicate rows created by one-to-many joins with transactions and qualifier records.

The result was inaccurate cohort-level funnel metrics, wrong counsellor workload views, and unreliable revenue attribution.

## Key Objectives

1. **Scope the right bookings**: only `BOOKED` slots in the `ADMISSIONS` category, within the reporting window, on standard :00 / :30 slot times.
2. **Resolve the original cohort** for every booking using a clear priority order: cohort-shift history first, then the SOP-flag exception (students granted directly into a cohort keep it), then qualifier-exam redirects, falling back to the current cohort.
3. **Compute a true booking amount** per student and cohort: use the paid amount when positive, and for ₹0 SOP-approved entrances, carry forward the student's most recent paid SOP-cohort entrance fee.
4. **Filter out noise**: exclude core/foundation cohorts, the `z` student category, `z` / `delta_critical` intent categories, and slots with a pending preferred-slot request.
5. **Enrich each row** with student profile, persona, counsellor assignment, track, student categories (with intent-model fallback) and lead tier.
6. **Guarantee one row per slot** with a deterministic deduplication step, ordered by most recent slot first.

## Approach

The query is built as a pipeline of CTEs:

| CTE | Purpose |
|---|---|
| `pids` | Profiles with an in-scope booked admissions slot |
| `shift_raw` / `shift_org` | Earliest recorded cohort shift per (profile, new cohort) |
| `redir_raw` / `redir_org` | Earliest qualifier redirect per (profile, destination cohort) |
| `sop_rejected` | SOP-flag approvals that keep the current cohort |
| `entrance_base` → `entrance_rows` | Forward-fill of the latest paid SOP-cohort fee |
| `booking_amt` | One booking amount per (profile, cohort) |
| `source_data` | Joins, cohort resolution, enrichment and filters |
| `ranked` | Deduplication to one row per slot |

## Requirements

- MySQL 8.0 or later (the query uses CTEs and window functions).
- JSON columns must contain valid JSON. If they are text columns that may hold malformed values, wrap extractions in `CASE WHEN JSON_VALID(col) THEN ... END`.

## Usage

Run `admissions_booked_slots.sql` against your database. Adjust the reporting start date (`DATE '2026-10-21'`) as needed; the end date is always `CURRENT_DATE`.




