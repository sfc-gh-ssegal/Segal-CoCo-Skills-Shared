---
name: notification-get-se-opp-violations
description: "Retrieve SE opp violations for SEs on the team and send Slack notifications. Checks 9 violation types on Capacity opps (TOTAL_ACV_C > 0, no Segment opps): No TW Date, TW After Close, TW Past NDY, No SE Comments, stale comments, invalid format, and placeholder TW Date. Messages grouped by opp so SE can fix all issues in one click. Triggers: SE violations, opp violations, TW violations, SE comment violations, notify SEs, violation report, check SE opps, opp health check, compliance check, pipeline violations."
---

# SE Opp Violations — Notification Pipeline

Detects pipeline hygiene violations for Steven Segal's 8-person SE team and delivers Slack DMs per SE with SFDC links (grouped by opp), plus a manager summary. Runs every Monday 7am CT.

---

## QUICK REFERENCE

```sql
-- Run in test mode (all messages go to steven.segal@snowflake.com)
CALL TEMP.SEGAL.RUN_SE_OPP_VIOLATIONS(TRUE);

-- Run live (messages go to each SE)
CALL TEMP.SEGAL.RUN_SE_OPP_VIOLATIONS(FALSE);

-- Just detect violations without sending messages
CALL TEMP.SEGAL.DETECT_SE_OPP_VIOLATIONS('MANUAL_RUN');

-- Enable/suspend the weekly task
ALTER TASK TEMP.SEGAL.SE_OPP_VIOLATIONS_TASK RESUME;
ALTER TASK TEMP.SEGAL.SE_OPP_VIOLATIONS_TASK SUSPEND;

-- Disable a single violation type without code changes
UPDATE TEMP.SEGAL.VIOLATION_CONFIG SET IS_ACTIVE = FALSE WHERE VIOLATION_KEY = 'V7_INVALID_FORMAT';

-- Check recent runs
SELECT RUN_ID, RUN_AT, COUNT(*) AS violations
FROM TEMP.SEGAL.SE_OPP_VIOLATIONS
GROUP BY 1, 2 ORDER BY 2 DESC LIMIT 10;
```

---

## ARCHITECTURE

### Design Principles

**Fat SP, thin agent.** All intelligence — which violations to flag, who to notify, how to format the message, what link to include — lives in SQL and Python stored procedures. The agent only does what SQL can't: make an HTTP call to Slack. This makes the pipeline deterministic, testable, and cheap to run.

**Extensible by config table.** Each violation type has a row in `VIOLATION_CONFIG`. Toggling `IS_ACTIVE = FALSE` disables a violation without touching code. Adding a new violation = one INSERT into VIOLATION_CONFIG + one UNION ALL block in the SP.

**Test mode.** `RUN_SE_OPP_VIOLATIONS(TRUE)` sends all SE messages to `steven.segal@snowflake.com` instead of the SE, with `[TEST — Notifications for: SE Name]` prepended so you can preview exactly what each SE will receive.

**One agent call per SE.** The orchestrator loops over SEs and calls the agent once per person. If one SE's Slack send fails, the rest still go through.

**Messages grouped by opp.** Each SE gets one Slack DM with opps as headers — all violations for a given opp are listed as sub-bullets so the SE can click the SFDC link and fix everything for that opp in one visit.

### Component Map

```
Snowflake Task (Monday 7am CT) — ACTIVE
  └── CALL RUN_SE_OPP_VIOLATIONS(FALSE)
        │
        ├─ 1. CALL DETECT_SE_OPP_VIOLATIONS(run_id)   [SQL SP]
        │       Queries FIVETRAN.SALESFORCE.*
        │       Runs all 9 active violation blocks
        │       Each block CROSS JOINs VIOLATION_CONFIG (IS_ACTIVE gate)
        │       INSERT → SE_OPP_VIOLATIONS table
        │
        ├─ 2. SELECT from SE_OPP_VIOLATIONS WHERE RUN_ID = run_id
        │       Groups by SE then by OPP, builds per-SE message with SFDC links
        │
        └─ 3. For each SE (+ manager summary):
                DATA_AGENT_RUN('TEMP.SEGAL.MESSAGING_AGENT', payload)
                  Agent: looks up Slack user by email → sends DM
```

### Data Flow

```
FIVETRAN.SALESFORCE.OPPORTUNITY  ──┐
FIVETRAN.SALESFORCE.USER         ──┼──► BASE_OPPS CTE
FIVETRAN.SALESFORCE.ACCOUNT      ──┘        (filters: Capacity, no Renewal,
                                             no Segment, TOTAL_ACV_C > 0)
                                          │
                                          ▼
                                   9 violation UNION ALL blocks
                                   × VIOLATION_CONFIG (IS_ACTIVE gate)
                                          │
                                          ▼
                                   SE_OPP_VIOLATIONS table (keyed by RUN_ID)
                                          │
                               Python SP: group by SE → group by OPP
                               Build SFDC hyperlinks for each opp
                                          │
                               DATA_AGENT_RUN → MESSAGING_AGENT
                                          │
                               natoma_-_slack MCP → Slack DM per SE
                               + summary DM to manager
```

---

## SNOWFLAKE OBJECTS

### Tables

| Table | Purpose |
|---|---|
| `TEMP.SEGAL.SE_OPP_VIOLATIONS` | Results: one row per violation per run. All runs retained (no truncation). |
| `TEMP.SEGAL.VIOLATION_CONFIG` | Controls which violations are active, their display name, and criticality (Red/Yellow). |

**SE_OPP_VIOLATIONS columns:**
`RUN_ID, RUN_AT, SE_NAME, SE_EMAIL, ACCOUNT_NAME, OPP_NAME, OPP_ID (18-char SFDC ID), CLOSE_DATE, TW_STATUS, TW_DATE, VIOLATION, CRITICALITY, SE_COMMENT_EXCERPT`

**VIOLATION_CONFIG columns:**
`VIOLATION_KEY (PK), VIOLATION_NAME, CRITICALITY ('Red' or 'Yellow'), IS_ACTIVE (default TRUE), DESCRIPTION, CREATED_AT`

### Stored Procedures

| SP | Language | Purpose |
|---|---|---|
| `DETECT_SE_OPP_VIOLATIONS(RUN_ID VARCHAR)` | SQL | Runs all 9 violation queries, INSERTs into SE_OPP_VIOLATIONS. Returns violation count (INTEGER). |
| `RUN_SE_OPP_VIOLATIONS(TEST_MODE BOOLEAN)` | Snowpark Python | Orchestrator. Calls DETECT, groups by opp, formats messages, calls MESSAGING_AGENT per SE. |

### Agent

| Object | Details |
|---|---|
| `TEMP.SEGAL.MESSAGING_AGENT` | Cortex Agent — Slack MCP only, no data tools. Reusable for any notification workflow. |
| MCP Server | `SNOWFLAKE_INTELLIGENCE.MCP.NOVA_SLACK_MCP` |
| System prompt | "Send messages exactly as provided. Do not modify content. Look up Slack user by email, send DM, return confirmation." |

**Calling the agent from Python:**
```python
import json
payload = json.dumps({
    "messages": [{
        "role": "user",
        "content": [{"type": "text", "text": f"Send a Slack DM to {email} with:\n\n{message}"}]
    }]
})
# MUST use params= binding — do NOT embed payload in f-string.
# Snowflake interprets \n in single-quoted SQL strings as real newlines,
# corrupting the JSON. Bind variables bypass SQL string parsing entirely.
session.sql(
    "SELECT SNOWFLAKE.CORTEX.DATA_AGENT_RUN('TEMP.SEGAL.MESSAGING_AGENT', ?) AS resp",
    params=[payload]
).collect()
```

### Task

| Task | Schedule | Status |
|---|---|---|
| `TEMP.SEGAL.SE_OPP_VIOLATIONS_TASK` | Every Monday 4:15pm CT (`CRON 15 16 * * 1 America/Chicago`) | **ACTIVE** — moved from 7am 2026-08-03 (OAuth warm at 4pm) |

---

## FILES

| File | Purpose |
|---|---|
| `apps/se-violations/setup/01_tables.sql` | Creates SE_OPP_VIOLATIONS and VIOLATION_CONFIG |
| `apps/se-violations/setup/02_violation_config.sql` | Seeds VIOLATION_CONFIG with all 9 violation rows |
| `apps/se-violations/pipeline/03_detect_violations.sql` | DETECT_SE_OPP_VIOLATIONS SQL SP |
| `apps/se-violations/pipeline/04_run_violations.sql` | RUN_SE_OPP_VIOLATIONS Python SP |
| `apps/se-violations/pipeline/05_task.sql` | Task DDL |

**Deploy any file:**
```bash
snow sql --connection snowhouse_ExtBrowser --role SALES_ENGINEER --warehouse SALES_STREAMLIT_WH -f <file.sql>
```

---

## VIOLATION TYPES

9 violations — V1–V7 are Red, V8 has Red (urgent) and Yellow tiers based on close date proximity.

| Key | Violation Name | Criticality | Close Date / TW Date Window | Condition |
|---|---|---|---|---|
| `V1_NO_TW_DATE` | No TW Date | Red | Close within 2 Qs | `tw_date IS NULL` |
| `V2_TW_AFTER_CLOSE` | TW Date After Close Date | Red | Close within 3 Qs | `tw_date > close_date AND tw_date != '2027-01-31'` |
| `V3_TW_PAST_NDY` | TW Date Past — TW = No Decision Yet | Red | Close within 3 Qs | `tw_date < CURRENT_DATE() AND tw_status = 'No Decision Yet'` |
| `V4_NO_SE_COMMENTS` | No SE Comments — TW = No Decision Yet | Red | Close within 2 Qs | `se_comments IS NULL OR TRIM(se_comments) = ''` AND `tw_status = 'No Decision Yet'` |
| `V5_STALE_2WK` | No Comment in Last 2 Weeks (TW Due <4 Weeks) | Red | TW Date within 4 weeks | `most_recent_comment_date < CURRENT_DATE - 2wk` (or NULL) |
| `V6_STALE_4WK` | No Comment in Last 4 Weeks (TW Due 4-12 Weeks) | Red | TW Date 4-12 weeks out | `most_recent_comment_date < CURRENT_DATE - 4wk` (or NULL) |
| `V7_INVALID_FORMAT` | Invalid SE Comment Format | Red | Close within 3 Qs | comment doesn't match valid format regex AND `most_recent_comment_date >= '2026-07-27'` ← grace period |
| `V8_PLACEHOLDER_TW_RED` | TW Date is Placeholder (2027-01-31) — Urgent | Red | Close within 2 months | `tw_date = '2027-01-31'` AND `close_date <= DATEADD('MONTH', 2, CURRENT_DATE())` |
| `V8_PLACEHOLDER_TW_YELLOW` | TW Date is Placeholder (2027-01-31) | Yellow | Close 2–4 months out | `tw_date = '2027-01-31'` AND close date between 2 and 4 months from today |

**Notes:**
- `2027-01-31` is a known placeholder TW Date used by multiple SEs. Excluded from V2 to avoid false positives. V8 flags it instead with appropriate urgency.
- V7 grace period: format violations are only flagged for comments written on or after `2026-07-27` (the date the standard was communicated to the team).
- Yellow violations (`V8_PLACEHOLDER_TW_YELLOW`) appear in SE messages but are not visually differentiated from Red yet. Future enhancement: `:yellow_circle:` prefix.

---

## HOW TO ADD A NEW VIOLATION

1. **Add config row:**
```sql
INSERT INTO TEMP.SEGAL.VIOLATION_CONFIG (VIOLATION_KEY, VIOLATION_NAME, CRITICALITY, IS_ACTIVE, DESCRIPTION)
VALUES ('V9_YOUR_KEY', 'Your Violation Name', 'Red', TRUE, 'What this checks');
```

2. **Add UNION ALL block** to `03_detect_violations.sql` following the template at the bottom of the SP:
```sql
UNION ALL
-- VN: [description]
SELECT :RUN_ID, SYSDATE(), se_name, se_email, account_name, opp_name, opp_id,
       close_date, tw_status, tw_date, vc.VIOLATION_NAME, vc.CRITICALITY,
       [LEFT(se_comments, 300) or NULL]
FROM BASE_OPPS[, FQ]
CROSS JOIN (SELECT * FROM TEMP.SEGAL.VIOLATION_CONFIG
            WHERE VIOLATION_KEY = 'V9_YOUR_KEY' AND IS_ACTIVE = TRUE) vc
WHERE [your conditions here]
```

3. **Redeploy:**
```bash
snow sql --connection snowhouse_ExtBrowser --role SALES_ENGINEER --warehouse SALES_STREAMLIT_WH \
  -f /Users/ssegal/Cortex_Workspace/apps/se-violations/pipeline/03_detect_violations.sql
```

---

## OPP SCOPE & FILTERS

**Included:**
- `AGREEMENT_TYPE_C LIKE 'Capacity%'` — all Capacity variants (Cap, Cap-AWS, Cap-Azure, Cap-GCP)
- `TYPE != 'Renewal'` — excludes TYPE=Renewal opps
- `NAME NOT ILIKE '%-Segment%'` — excludes future-year deal segments (Segment 2/3/4/5). These are TYPE='New Business' in SFDC but are effectively renewal slices of multi-year deals.
- `TOTAL_ACV_C > 0` — only opps with a known Total ACV. Same field used by `opp-review-generation` pipeline as TACV. NULL = not yet priced. Segment opps always have NULL TOTAL_ACV_C so this is belt-and-suspenders with the name filter.
- `IS_CLOSED = FALSE` and `IS_DELETED = FALSE`
- SE in team of 8 (via `LEAD_SALES_ENGINEER_C → FIVETRAN.SALESFORCE.USER.ID`)

**Excluded:** Technical Services, On Demand, Renewals, Segment 2/3/4/5 opps, unpriced opps (TOTAL_ACV_C = NULL or 0), closed opps.

---

## SE ROSTER

| SE Name | Email | Initials |
|---|---|---|
| Abhinav Bannerjee | abhinav.bannerjee@snowflake.com | AB |
| James Newsom | james.newsom@snowflake.com | JN |
| Julie Heckman | julie.heckman@snowflake.com | JH |
| Lisa Batteiger | lisa.batteiger@snowflake.com | LB |
| Michael Hughes | michael.hughes@snowflake.com | MH |
| Stephen Pace | stephen.pace@snowflake.com | SP |
| Tim Whitaker | tim.whitaker@snowflake.com | TW |
| Whitney Burke | whitney.burke@snowflake.com | WB |

SE name and email resolved via `JOIN FIVETRAN.SALESFORCE.USER u ON o.LEAD_SALES_ENGINEER_C = u.ID` — always returns `@snowflake.com` for this team.

---

## SFDC FIELD MAPPING

| Field | SFDC Column |
|---|---|
| TW status | `TECHNICAL_WIN_C` — values: `'Yes'`, `'No Decision Yet'`, NULL, `'Lost'` |
| TW Date | `TECHNICAL_WIN_DATE_C` (DATE) |
| SE Comments | `SE_COMMENTS_C` (TEXT — running log, newest entry prepended at top) |
| Agreement Type | `AGREEMENT_TYPE_C` |
| Opp Type | `TYPE` |
| Close Date | `CLOSE_DATE` |
| ACV (filter) | `TOTAL_ACV_C` (NUMERIC) — same field as TACV in opp-review-generation. Filter: `TOTAL_ACV_C > 0` |
| Opp ID (18-char) | `o.ID` — used for SFDC links |
| SE User ID | `LEAD_SALES_ENGINEER_C` → join `FIVETRAN.SALESFORCE.USER` |
| SE Email | `FIVETRAN.SALESFORCE.USER.EMAIL` |
| Account Name | `FIVETRAN.SALESFORCE.ACCOUNT.NAME` |

**SFDC link format:**
```
https://snowforce.lightning.force.com/lightning/r/Opportunity/{OPP_ID}/view
```

---

## FISCAL QUARTER LOGIC

Snowflake FY (Feb start): Q1 Feb-Apr, Q2 May-Jul, Q3 Aug-Oct, Q4 Nov-Jan.

Quarter boundaries computed dynamically from `CURRENT_DATE()` — no hardcoded dates.

| Term | Meaning |
|---|---|
| "2 Qs" | q0_start → q1_end (current Q + next Q) |
| "3 Qs" | q0_start → q2_end (current Q + next 2 Qs) |

V8 uses `DATEADD('MONTH', N, CURRENT_DATE())` instead of fiscal quarters for simpler month-based urgency thresholds.

---

## SE COMMENT FORMAT

Valid formats accepted by V7 (lenient — any of these pass):
```
M/D/YYYY [initials] TW: ...      ← slash date, bracketed initials
M/D/YYYY initials TW: ...        ← slash date, bare initials
YYYY-MM-DD [initials] TW: ...    ← ISO date, bracketed initials
YYYY-MM-DD initials TW: ...      ← ISO date, bare initials
```
TW may be on the same line or the next line. Case-insensitive. 2-digit years (e.g. `07/22/26`) accepted.

**Valid examples:**
- `7/10/2026[TW] TW: Yes` — Tim's format (initials = TW)
- `6/25/2026 [JMN] TW: Yes!` — James
- `2026-03-26 JH TW: text` — Julie, ISO date, bare initials
- `2026-06-18 [LB] TW: Yes` — Lisa, ISO date, bracket initials
- `07/22/26 [WB] TW: text` — Whitney, 2-digit year

**Invalid examples (V7 fires, if dated >= 2026-07-27):**
- `[06/18 AB] TW:` — date inside brackets
- `[mh 6/10/26].` — initials+date mixed inside brackets, no TW following
- `1/2/25 AW SE not requested` — no TW keyword

**Regex (Snowflake):**
```
^([0-9]{1,2}/[0-9]{1,2}/[0-9]+|[0-9]{4}-[0-9]{2}-[0-9]{2})[[:space:]]*(\[[A-Za-z]+\]|[A-Za-z]{2,4}).*TW.*
```
Flags: `is`. Snowflake `REGEXP_LIKE` is full-string match — trailing `.*` required.

> **SNOWFLAKE REGEX WARNING:** `\d` and `\s` are **NOT** supported by Snowflake's regex engine and silently return NULL / FALSE. Always use `[0-9]` instead of `\d`, and `[[:space:]]` instead of `\s`. This caused a bug where every opp appeared to have no recent comment (all stale violations fired) and every comment appeared invalid (all V7 fired). Fixed 2026-07-28.

**Most recent comment date extraction** (single unified regex, scoped to first 30 chars):
```sql
COALESCE(
    TRY_TO_DATE(REGEXP_SUBSTR(LEFT(SE_COMMENTS_C, 30), '([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})', 1, 1, 'e', 1), 'YYYY-MM-DD'),
    TRY_TO_DATE(REGEXP_SUBSTR(LEFT(SE_COMMENTS_C, 30), '([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})', 1, 1, 'e', 1), 'MM/DD/YYYY'),
    TRY_TO_DATE(REGEXP_SUBSTR(LEFT(SE_COMMENTS_C, 30), '([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})', 1, 1, 'e', 1), 'MM/DD/YY')
)
```
Uses `LEFT(SE_COMMENTS_C, 30)` to ensure we only extract the date from the **newest** (topmost) comment entry — not an older ISO-date entry buried further down. The `[/-]` character class matches both slash and dash separators in a single pass. TRY_TO_DATE COALESCE tries ISO first, then MM/DD/YYYY, then MM/DD/YY. NULL = no date found (treated as no recent comment).

> **BUG FIX (2026-08-11):** Previously, ISO format was tried against the *entire* SE_COMMENTS_C field first. When a newer slash-format entry sat at the top but an older ISO entry existed further down, the ISO regex matched the stale date — causing false V5/V6 stale violations. Fixed by scoping to LEFT(..., 30).

---

## SLACK MESSAGE FORMAT

### SE Message (per SE, grouped by opp)

```
[TEST — Notifications for: *Whitney Burke*]    ← test mode only

:red_circle: *Opp Violations — Week of 07/27/2026*

*<https://snowforce.../OPP_ID/view|Versova>* — Close: 08/28, TW: 07/30
  • No Comment in Last 2 Weeks (TW Due <4 Weeks)
  _Last comment: 07/22/26 [WB] TW: Need to trial an end to end pipeline..._

*<https://snowforce.../OPP_ID/view|SureScripts>* — Close: 11/10, TW: 01/31/2027
  • TW Date is Placeholder (2027-01-31)
```

Each opp is a clickable link to SFDC. All violations for that opp are bullets underneath — the SE can open one opp and fix everything.

### Manager Summary

```
:bar_chart: *SE Violations Summary — 07/27/2026*
Run: `8B00005933F0` | Mode: `LIVE` | Total: *31 violations* across *7 SEs*

*Julie Heckman* — 12 violations
  • No Comment in Last 4 Weeks: 11
  • No SE Comments: 1

*Michael Hughes* — 10 violations
  • Invalid SE Comment Format: 8
  • No SE Comments: 1
  • TW Date After Close Date: 1
...
```

Manager summary groups by SE → violation type (not by opp).

---

## KNOWN DATA ISSUES

- **`2027-01-31` placeholder TW Date:** Treated as "TBD" — excluded from V2, flagged separately by V8 with Red/Yellow urgency based on months to close.
- **Year typo in comments:** `7/9/20206` (Aimbridge/James) — skipped gracefully by date extraction.
- **Future date in comment header:** `2027-07-09 [LB]` (AlertMedia/Lisa) — likely typo for `2026-07-09`.
- **Ancient TW Dates still open:** EDP Renewables (TW=2021), Industrial Info Resources (TW=2022) — legitimate long-cycle or zombie deals.

---

## AD-HOC QUERY (no SPs)

```sql
WITH FQ_START AS (
    SELECT
        CASE
            WHEN MONTH(CURRENT_DATE()) = 1             THEN DATE_FROM_PARTS(YEAR(CURRENT_DATE()) - 1, 11, 1)
            WHEN MONTH(CURRENT_DATE()) BETWEEN 2 AND 4 THEN DATE_FROM_PARTS(YEAR(CURRENT_DATE()), 2, 1)
            WHEN MONTH(CURRENT_DATE()) BETWEEN 5 AND 7 THEN DATE_FROM_PARTS(YEAR(CURRENT_DATE()), 5, 1)
            WHEN MONTH(CURRENT_DATE()) BETWEEN 8 AND 10 THEN DATE_FROM_PARTS(YEAR(CURRENT_DATE()), 8, 1)
            ELSE DATE_FROM_PARTS(YEAR(CURRENT_DATE()), 11, 1)
        END AS q0_start
),
FQ AS (
    SELECT q0_start,
           DATEADD('DAY',-1,DATEADD('MONTH',3,q0_start))  AS q0_end,
           DATEADD('MONTH',3,q0_start)                     AS q1_start,
           DATEADD('DAY',-1,DATEADD('MONTH',6,q0_start))  AS q1_end,
           DATEADD('MONTH',6,q0_start)                     AS q2_start,
           DATEADD('DAY',-1,DATEADD('MONTH',9,q0_start))  AS q2_end
    FROM FQ_START
),
BASE_OPPS AS (
    SELECT u.NAME AS se_name, u.EMAIL AS se_email,
           a.NAME AS account_name, o.NAME AS opp_name, o.ID AS opp_id,
           o.CLOSE_DATE, o.TECHNICAL_WIN_C AS tw_status,
           o.TECHNICAL_WIN_DATE_C AS tw_date, o.SE_COMMENTS_C AS se_comments,
           COALESCE(
               TRY_TO_DATE(REGEXP_SUBSTR(o.SE_COMMENTS_C,'([0-9]{1,2}/[0-9]{1,2}/[0-9]{4})',1,1,'e',1),'MM/DD/YYYY'),
               TRY_TO_DATE(REGEXP_SUBSTR(o.SE_COMMENTS_C,'([0-9]{1,2}/[0-9]{1,2}/[0-9]{2})',1,1,'e',1),'MM/DD/YY')
           ) AS most_recent_comment_date,
           CASE
               WHEN o.SE_COMMENTS_C IS NULL OR TRIM(o.SE_COMMENTS_C) = '' THEN 'no_comment'
               WHEN REGEXP_LIKE(TRIM(o.SE_COMMENTS_C),
                   '^([0-9]{1,2}/[0-9]{1,2}/[0-9]+|[0-9]{4}-[0-9]{2}-[0-9]{2})[[:space:]]*(\\[[A-Za-z]+\\]|[A-Za-z]{2,4}).*TW.*','is')
                   THEN 'valid'
               ELSE 'invalid'
           END AS comment_format
    FROM FIVETRAN.SALESFORCE.OPPORTUNITY o
    JOIN FIVETRAN.SALESFORCE.USER    u ON o.LEAD_SALES_ENGINEER_C = u.ID
    JOIN FIVETRAN.SALESFORCE.ACCOUNT a ON o.ACCOUNT_ID = a.ID
    WHERE o.IS_DELETED=FALSE AND o.IS_CLOSED=FALSE
      AND o.AGREEMENT_TYPE_C LIKE 'Capacity%' AND o.TYPE != 'Renewal'
      AND o.NAME NOT ILIKE '%-Segment%'
      AND o.TOTAL_ACV_C > 0
      AND u.NAME IN ('Abhinav Bannerjee','James Newsom','Julie Heckman',
                     'Lisa Batteiger','Michael Hughes','Stephen Pace',
                     'Tim Whitaker','Whitney Burke')
)
SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date, violation, criticality, se_comment_excerpt
FROM (
    -- V1: No TW Date
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'No TW Date' AS violation, 'Red' AS criticality, NULL::VARCHAR AS se_comment_excerpt
    FROM BASE_OPPS, FQ WHERE tw_date IS NULL AND close_date BETWEEN q0_start AND q1_end
    UNION ALL
    -- V2: TW After Close
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'TW Date After Close Date', 'Red', NULL
    FROM BASE_OPPS, FQ WHERE tw_date > close_date AND tw_date!='2027-01-31' AND close_date BETWEEN q0_start AND q2_end
    UNION ALL
    -- V3: TW Past, NDY
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'TW Date Past — No Decision Yet', 'Red', NULL
    FROM BASE_OPPS, FQ WHERE tw_date < CURRENT_DATE() AND tw_status='No Decision Yet' AND close_date BETWEEN q0_start AND q2_end
    UNION ALL
    -- V4: No SE Comments
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'No SE Comments (TW=NDY)', 'Red', NULL
    FROM BASE_OPPS, FQ WHERE (se_comments IS NULL OR TRIM(se_comments)='') AND tw_status='No Decision Yet' AND close_date BETWEEN q0_start AND q1_end
    UNION ALL
    -- V5: Stale 2wk
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'No Comment in Last 2 Weeks (TW Due <4 Weeks)', 'Red', LEFT(se_comments,300)
    FROM BASE_OPPS WHERE tw_date>=CURRENT_DATE() AND tw_date<=DATEADD('WEEK',4,CURRENT_DATE())
      AND (most_recent_comment_date IS NULL OR most_recent_comment_date<DATEADD('WEEK',-2,CURRENT_DATE()))
    UNION ALL
    -- V6: Stale 4wk
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'No Comment in Last 4 Weeks (TW Due 4-12 Weeks)', 'Red', LEFT(se_comments,300)
    FROM BASE_OPPS WHERE tw_date>DATEADD('WEEK',4,CURRENT_DATE()) AND tw_date<=DATEADD('WEEK',12,CURRENT_DATE())
      AND (most_recent_comment_date IS NULL OR most_recent_comment_date<DATEADD('WEEK',-4,CURRENT_DATE()))
    UNION ALL
    -- V7: Invalid Format (grace period 2026-07-27)
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'Invalid Comment Format', 'Red', LEFT(se_comments,300)
    FROM BASE_OPPS, FQ WHERE comment_format='invalid'
      AND most_recent_comment_date >= '2026-07-27'
      AND close_date BETWEEN q0_start AND q2_end
    UNION ALL
    -- V8-Red: Placeholder TW Date, close < 2 months
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'TW Date is Placeholder (2027-01-31) — Urgent', 'Red', NULL
    FROM BASE_OPPS WHERE tw_date='2027-01-31' AND close_date<=DATEADD('MONTH',2,CURRENT_DATE())
    UNION ALL
    -- V8-Yellow: Placeholder TW Date, close 2-4 months
    SELECT se_name, account_name, opp_name, close_date, tw_status, tw_date,
           'TW Date is Placeholder (2027-01-31)', 'Yellow', NULL
    FROM BASE_OPPS WHERE tw_date='2027-01-31'
      AND close_date>DATEADD('MONTH',2,CURRENT_DATE())
      AND close_date<=DATEADD('MONTH',4,CURRENT_DATE())
) v
ORDER BY se_name, violation, close_date;
```
