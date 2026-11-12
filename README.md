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




-- =====================================================================
-- Admissions Counselling Slots: Original-Cohort Attribution Report
-- Dialect : MySQL 8.0+ (requires CTEs and window functions)
-- =====================================================================

WITH
/* ---- profiles with a booked admissions slot in the window ---- */
pids AS (
    SELECT DISTINCT
        s.profile_id
    FROM `msi`.`onwards_support_slots` AS s
    WHERE s.status = 'BOOKED'
      AND s.category = 'ADMISSIONS'
      AND s.slot_date >= DATE '2026-10-21'
      AND s.slot_date <= CURRENT_DATE
      AND EXTRACT(MINUTE FROM CAST(s.slot_time AS TIME)) IN (0, 30)
),

/* ---- move mechanism 1: cohort shift history ---- */
shift_raw AS (
    SELECT
        ss.profile_id,
        CAST(
            NULLIF(
                NULLIF(
                    JSON_UNQUOTE(JSON_EXTRACT(ss.meta, '$.cohortShiftHistory[0].oldCohortId')),
                    ''
                ),
                'null'
            ) AS SIGNED
        ) AS old_c,
        ss.cohort_id AS new_c,
        MIN(ss.id) AS sid
    FROM `msi`.`onwards_support_slots` AS ss
    JOIN pids
      ON pids.profile_id = ss.profile_id
    WHERE ss.category = 'ADMISSIONS'
      AND ss.slot_date >= DATE '2026-10-21'
      AND ss.slot_date <= CURRENT_DATE
      AND JSON_EXTRACT(ss.meta, '$.cohortShiftHistory') IS NOT NULL
    GROUP BY 1, 2, 3
),

shift_org AS (
    SELECT
        profile_id,
        new_c,
        old_c
    FROM (
        SELECT
            r.*,
            ROW_NUMBER() OVER (
                PARTITION BY r.profile_id, r.new_c
                ORDER BY r.sid ASC
            ) AS rn
        FROM shift_raw AS r
        WHERE r.old_c IS NOT NULL
    ) AS z
    WHERE rn = 1
),

/* ---- move mechanism 2: qualifier redirect ---- */
redir_raw AS (
    SELECT
        q.profile_id,
        q.cohort_id AS src_c,
        CAST(
            NULLIF(
                NULLIF(
                    JSON_UNQUOTE(JSON_EXTRACT(q.meta, '$.qualifierRedirectCohortId')),
                    ''
                ),
                'null'
            ) AS SIGNED
        ) AS dst_c,
        MIN(q.id) AS qid
    FROM `msi`.`onwards_qualifier_exam_user` AS q
    JOIN pids
      ON pids.profile_id = q.profile_id
    WHERE JSON_EXTRACT(q.meta, '$.qualifierRedirectCohortId') IS NOT NULL
    GROUP BY 1, 2, 3
),

redir_org AS (
    SELECT
        profile_id,
        dst_c,
        src_c
    FROM (
        SELECT
            r.*,
            ROW_NUMBER() OVER (
                PARTITION BY r.profile_id, r.dst_c
                ORDER BY r.qid ASC
            ) AS rn
        FROM redir_raw AS r
    ) AS z
    WHERE rn = 1
),

/* ---- SOP-rejected exception: granted into this cohort ---- */
sop_rejected AS (
    SELECT DISTINCT
        g.profile_id,
        g.cohort_id
    FROM `msi`.`onwards_transactions` AS g
    JOIN pids
      ON pids.profile_id = g.profile_id
    WHERE g.payment_type = 'ENTRANCE_FEE'
      AND g.status = 'PAID'
      AND JSON_UNQUOTE(JSON_EXTRACT(g.meta, '$.approvedFrom')) = 'sop-flag'
),

/* ---- booking amount, step 1: flag SOP-cohort payments ---- */
entrance_base AS (
    SELECT
        t.id,
        t.profile_id,
        t.cohort_id,
        t.amount,
        t.created_at,
        JSON_UNQUOTE(JSON_EXTRACT(t.meta, '$.approvedFrom')) AS approved_from,
        CASE
            WHEN t.amount > 0
             AND JSON_UNQUOTE(JSON_EXTRACT(ec.settings, '$.directCounsellingPostPayment')) = 'true'
            THEN t.amount
        END AS sop_amt
    FROM `msi`.`onwards_transactions` AS t
    JOIN pids
      ON pids.profile_id = t.profile_id
    JOIN `msi`.`onwards_cohorts` AS ec
      ON ec.id = t.cohort_id
    WHERE t.payment_type = 'ENTRANCE_FEE'
      AND t.status = 'PAID'
),

/* ---- step 2: forward-fill groups (replaces LAST_VALUE ... IGNORE NULLS) ---- */
entrance_grouped AS (
    SELECT
        b.*,
        COUNT(b.sop_amt) OVER (
            PARTITION BY b.profile_id
            ORDER BY b.created_at, b.id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS sop_grp
    FROM entrance_base AS b
),

/* ---- step 3: latest PAID SOP-cohort amount at or before each row ---- */
entrance_rows AS (
    SELECT
        g.*,
        MAX(g.sop_amt) OVER (
            PARTITION BY g.profile_id, g.sop_grp
        ) AS sop_amount_so_far
    FROM entrance_grouped AS g
),

/* ---- booking amount: one row per (profile, cohort) ---- */
booking_amt AS (
    SELECT
        profile_id,
        cohort_id,
        CASE
            WHEN amount > 0
                THEN amount
            WHEN approved_from IN ('sop-flag', 'sop-to-sop')
                THEN sop_amount_so_far
            ELSE amount
        END AS booking_amount
    FROM (
        SELECT
            e.*,
            ROW_NUMBER() OVER (
                PARTITION BY e.profile_id, e.cohort_id
                ORDER BY
                    CASE WHEN e.amount > 0 THEN 0 ELSE 1 END,
                    e.created_at,
                    e.id
            ) AS rn
        FROM entrance_rows AS e
    ) AS z
    WHERE rn = 1
),

source_data AS (
    SELECT
        s.status,

        /* Original cohort resolution: shift first, then SOP exception, then redirect */
        CASE
            WHEN so.old_c IS NOT NULL
                THEN so.old_c
            WHEN rj.profile_id IS NOT NULL
                THEN s.cohort_id
            ELSE COALESCE(ro.src_c, s.cohort_id)
        END AS cohort_id,

        s.outcome,
        s.slot_date,

        /* Preserve the HH:MM time format */
        SUBSTRING(CAST(s.slot_time AS CHAR), 1, 5) AS slot_time,

        s.category,
        s.id,
        s.meeting_link,
        s.meeting_type,
        s.profile_id,

        p.email,
        p.name,
        p.phone_number,

        oc.name AS cohort_name,

        JSON_UNQUOTE(JSON_EXTRACT(p.meta, '$.persona'))           AS persona,
        JSON_UNQUOTE(JSON_EXTRACT(s.meta, '$.accepterCounselor')) AS `accepterCounselor`,
        JSON_UNQUOTE(JSON_EXTRACT(s.meta, '$.newCounselor'))      AS `newCounselor`,

        otr.description AS track,

        COALESCE(
            REPLACE(
                NULLIF(JSON_UNQUOTE(JSON_EXTRACT(s.meta, '$.studentCategories')), 'null'),
                '","',
                '", "'
            ),
            JSON_UNQUOTE(JSON_EXTRACT(q.meta, '$.intentModel.result.category'))
        ) AS `studentCategories`,

        JSON_UNQUOTE(JSON_EXTRACT(q.meta, '$.intentModel.result.category')) AS intent_category,

        tr.amount AS entrance_amount,

        CASE
            WHEN CAST(tr.meta AS CHAR) LIKE '%?isDirectEnrolment=true%'
             AND tr.amount = 0
            THEN TRUE
            ELSE FALSE
        END AS new_entrance_flow,

        JSON_UNQUOTE(JSON_EXTRACT(q.meta, '$.counsellingTier')) AS `leadTier`,

        ba.booking_amount

    FROM `msi`.`onwards_support_slots` AS s

    LEFT JOIN `msi`.`onwards_profile` AS p
        ON s.profile_id = p.id

    LEFT JOIN `msi`.`onwards_cohorts` AS c
        ON s.cohort_id = c.id

    LEFT JOIN `msi`.`onwards_cohort_groups` AS cg
        ON s.cohort_group_id = cg.id

    LEFT JOIN `msi`.`onwards_transactions` AS tr
        ON s.profile_id = tr.profile_id
       AND s.cohort_id = tr.cohort_id
       AND tr.status = 'PAID'
       AND tr.payment_type = 'ENTRANCE_FEE'

    LEFT JOIN booking_amt AS ba
        ON ba.profile_id = s.profile_id
       AND ba.cohort_id = s.cohort_id

    LEFT JOIN `msi`.`onwards_qualifier_exam_user` AS q
        ON s.profile_id = q.profile_id
       AND s.cohort_id = q.cohort_id

    /* Origin resolution */
    LEFT JOIN sop_rejected AS rj
        ON rj.profile_id = s.profile_id
       AND rj.cohort_id = s.cohort_id

    LEFT JOIN shift_org AS so
        ON so.profile_id = s.profile_id
       AND so.new_c = s.cohort_id

    LEFT JOIN redir_org AS ro
        ON ro.profile_id = s.profile_id
       AND ro.dst_c = s.cohort_id

    LEFT JOIN `msi`.`onwards_cohorts` AS oc
        ON oc.id = CASE
            WHEN so.old_c IS NOT NULL
                THEN so.old_c
            WHEN rj.profile_id IS NOT NULL
                THEN s.cohort_id
            ELSE COALESCE(ro.src_c, s.cohort_id)
        END

    LEFT JOIN `msi`.`onwards_tracks` AS otr
        ON oc.track_id = otr.id

    WHERE s.status = 'BOOKED'
      AND s.category = 'ADMISSIONS'

      /* LIKE BINARY makes the match case-sensitive */
      AND c.name NOT LIKE BINARY '%core%'
      AND c.name NOT LIKE BINARY '%foundation%'

      AND s.slot_date >= DATE '2026-10-21'
      AND s.slot_date <= CURRENT_DATE
      AND EXTRACT(MINUTE FROM CAST(s.slot_time AS TIME)) IN (0, 30)

      /* Exclude the exact student category "z" */
      AND COALESCE(
              JSON_UNQUOTE(JSON_EXTRACT(s.meta, '$.studentCategories')),
              ''
          ) NOT LIKE '%"z"%'

      AND (
          JSON_UNQUOTE(JSON_EXTRACT(q.meta, '$.intentModel.result.category')) IS NULL
          OR JSON_UNQUOTE(JSON_EXTRACT(q.meta, '$.intentModel.result.category'))
             NOT IN ('z', 'delta_critical')
      )

      AND (
          JSON_UNQUOTE(JSON_EXTRACT(s.meta, '$.preferredSlotRequest.status')) IS NULL
          OR JSON_UNQUOTE(JSON_EXTRACT(s.meta, '$.preferredSlotRequest.status')) <> 'PENDING'
      )
),

ranked AS (
    SELECT
        source_data.*,
        ROW_NUMBER() OVER (
            PARTITION BY source_data.id
            ORDER BY
                source_data.slot_date DESC,
                source_data.slot_time DESC
        ) AS row_num
    FROM source_data
)

SELECT
    ranked.status,
    ranked.cohort_id,
    ranked.outcome,
    ranked.slot_date,
    ranked.slot_time,
    ranked.category,
    ranked.id,
    ranked.meeting_link,
    ranked.meeting_type,
    ranked.profile_id,
    ranked.email,
    ranked.name,
    ranked.phone_number,
    ranked.cohort_name,
    ranked.persona,
    ranked.`accepterCounselor`,
    ranked.`newCounselor`,
    ranked.track,
    ranked.`studentCategories`,
    ranked.intent_category,
    ranked.`leadTier`,
    ranked.booking_amount
FROM ranked
WHERE ranked.row_num = 1
ORDER BY
    ranked.slot_date DESC,
    ranked.slot_time DESC;
