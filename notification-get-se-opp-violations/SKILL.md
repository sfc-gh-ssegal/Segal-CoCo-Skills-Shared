---
name: notification-get-se-opp-violations
description: "Retrieve SE opp violations for SEs on the team and send Slack notifications. Checks 9 violation types on Capacity opps (TOTAL_ACV_USD > 0, no Segment opps): No TW Date, TW After Close, TW Past NDY, No SE Comments, stale comments, invalid format, and placeholder TW Date. Messages grouped by opp so SE can fix all issues in one click. Triggers: SE violations, opp violations, TW violations, SE comment violations, notify SEs, violation report, check SE opps, opp health check, compliance check, pipeline violations."
---

# SE Opp Violations — Notification Pipeline

Detects pipeline hygiene violations for Steven Segal's 8-person SE team and delivers Slack DMs per SE with SFDC links (grouped by opp), plus a manager summary. Runs every Monday 4:15pm CT.

---

## QUICK REFERENCE

```sql
-- Run in test mode (all messages go to steven.segal@snowflake.com)
CALL TEMP.SEGAL.RUN_SE_OPP_VIOLATIONS(TRUE);

-- Run live (messages go to each SE)
CALL TEMP.SEGAL.RUN_SE_OPP_VIOLATIONS(FALSE, NULL);

-- Test a single SE — send their message to Steven only
CALL TEMP.SEGAL.RUN_SE_OPP_VIOLATIONS(TRUE, 'Whitney');  -- substring match on SE name

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

**Fat SP, no agent.** All intelligence — which violations to flag, who to notify, how to format the message, what link to include — lives in SQL and Python stored procedures. Slack delivery uses `IT.IT_UDFS.ENG_SLACK` (Snowhouse bot) directly via SQL UDF call, with Slack IDs resolved from `IT.IT_EMPLOYEE_ACCOUNT.DIM_EMPLOYEE`. No OAuth dependency — works reliably in unattended tasks.

**Extensible by config table.** Each violation type has a row in `VIOLATION_CONFIG`. Toggling `IS_ACTIVE = FALSE` disables a violation without touching code. Adding a new violation = one INSERT into VIOLATION_CONFIG + one UNION ALL block in the SP.

**Test mode.** `RUN_SE_OPP_VIOLATIONS(TRUE)` sends all SE messages to `steven.segal@snowflake.com` instead of the SE, with `[TEST — Notifications for: SE Name]` prepended so you can preview exactly what each SE will receive.

**One ENG_SLACK call per SE.** The orchestrator loops over SEs and calls `IT.IT_UDFS.ENG_SLACK` once per person. If one SE's Slack send fails, the rest still go through.

**Messages grouped by opp.** Each SE gets one Slack DM with opps as headers — all violations for a given opp are listed as sub-bullets so the SE can click the SFDC link and fix everything for that opp in one visit.

### Component Map

```
Snowflake Task (Monday 7am CT) — ACTIVE
  └── CALL RUN_SE_OPP_VIOLATIONS(FALSE)
        │
        ├─ 1. CALL DETECT_SE_OPP_VIOLATIONS(run_id)   [SQL SP]
        │       Queries SNOW_CERTIFIED.SALESFORCE_OPPORTUNITY.DD_SALESFORCE_OPPORTUNITY
        │       Runs all 9 active violation blocks
        │       Each block CROSS JOINs VIOLATION_CONFIG (IS_ACTIVE gate)
        │       INSERT → SE_OPP_VIOLATIONS table
        │
        ├─ 2. SELECT from SE_OPP_VIOLATIONS WHERE RUN_ID = run_id
        │       Groups by SE then by OPP, builds per-SE message with SFDC links
        │
        └─ 3. Bulk-resolve emails → Slack IDs via IT.IT_EMPLOYEE_ACCOUNT.DIM_EMPLOYEE
               For each SE (+ manager summary):
                IT.IT_UDFS.ENG_SLACK(slack_id, message) → Snowhouse bot → Slack DM
```

### Data Flow

```
SNOW_CERTIFIED.SALESFORCE_OPPORTUNITY
  .DD_SALESFORCE_OPPORTUNITY     ──────► BASE_OPPS CTE
                                         (filters: Capacity, no Renewal,
                                          no Segment, TOTAL_ACV_USD > 0)
                                         SE name/email + account name
                                         denormalized on opp view — no JOINs
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
                               Bulk email→Slack ID from DIM_EMPLOYEE
                                          │
                               IT.IT_UDFS.ENG_SLACK(slack_id, msg) per SE
                               + summary DM to manager
```

**SE-facing rules reference (GDoc):** https://docs.google.com/document/d/1PTEgnLVRFuy3PL--fc4ejMe3XE4fPSvETBx4pkyWM0M/edit?usp=sharing
This link is appended to every SE Slack message as "Pipeline Hygiene Rules Reference."

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
| `RUN_SE_OPP_VIOLATIONS(TEST_MODE BOOLEAN)` | Snowpark Python | Orchestrator. Calls DETECT, bulk-resolves Slack IDs from DIM_EMPLOYEE, formats messages, sends via ENG_SLACK per SE. |

### Slack Delivery

**No agent.** Slack DMs are sent via the Snowhouse `IT.IT_UDFS.ENG_SLACK` UDF — a bot-token-based SQL UDF that works reliably in unattended tasks with no OAuth expiry.

**Why not NOVA_SLACK_MCP / MESSAGING_AGENT:** `NOVA_SLACK_MCP` requires user OAuth that expires. When the token expires, `DATA_AGENT_RUN` silently returns a refusal text instead of raising an exception. Switched to ENG_SLACK on 2026-08-20.

| Object | Details |
|---|---|
| `IT.IT_UDFS.ENG_SLACK(channel VARCHAR, msg VARCHAR)` | Snowhouse bot UDF. `channel` = Slack user ID (not email). Returns VARIANT with `ok: true` on success. `SALES_ENGINEER` has USAGE. |
| `IT.IT_EMPLOYEE_ACCOUNT.DIM_EMPLOYEE` | Daily snapshot. Columns: `EMAIL`, `SLACK_ID`. Deduplicate: `QUALIFY ROW_NUMBER() OVER (PARTITION BY EMAIL ORDER BY DS DESC) = 1`. |

**Sending from Python:**
```python
# Bulk-resolve all recipient emails at start of run
email_list = ", ".join(f"'{e}'" for e in emails)
rows = session.sql(f"""
    SELECT EMAIL, SLACK_ID FROM IT.IT_EMPLOYEE_ACCOUNT.DIM_EMPLOYEE
    WHERE EMAIL IN ({email_list})
    QUALIFY ROW_NUMBER() OVER (PARTITION BY EMAIL ORDER BY DS DESC) = 1
""").collect()
slack_id_map = {r["EMAIL"]: r["SLACK_ID"] for r in rows if r["SLACK_ID"]}

# Send one DM — note: Snowpark Row uses subscript, not .get()
import json
slack_id = slack_id_map[recipient_email]
result = session.sql(
    "SELECT TO_JSON(IT.IT_UDFS.ENG_SLACK(?, ?)) AS resp",
    params=[slack_id, message]
).collect()
resp = json.loads(result[0]["RESP"])
ok = isinstance(resp, list) and resp[0].get("ok") is True
```
**Note:** Messages arrive via the "SnowHouse" bot app in Slack, visible under Activity/Apps — not in regular DMs.

### Task

| Task | Schedule | Status |
|---|---|---|
| `TEMP.SEGAL.SE_OPP_VIOLATIONS_TASK` | Every Monday 4:15pm CT (`CRON 15 16 * * 1 America/Chicago`) | **ACTIVE** |

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
- Yellow violations (`V8_PLACEHOLDER_TW_YELLOW`) show `🟡 •` prefix in SE messages. When *all* an SE's violations are yellow, the message header uses `:large_yellow_circle:` instead of `:red_circle:`.

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
- `SALESFORCE_OPPORTUNITY_AGREEMENT_TYPE LIKE 'Capacity%'` — all Capacity variants (Cap, Cap-AWS, Cap-Azure, Cap-GCP)
- `SALESFORCE_OPPORTUNITY_TYPE != 'Renewal'` — excludes TYPE=Renewal opps
- `SALESFORCE_OPPORTUNITY_NAME NOT ILIKE '%-Segment%'` — excludes future-year deal segments (Segment 2/3/4/5). These are TYPE='New Business' in SFDC but are effectively renewal slices of multi-year deals.
- `SALESFORCE_OPPORTUNITY_TOTAL_ACV_USD > 0` — only opps with a known Total ACV. Same field used by `opp-review-generation` pipeline as TACV. NULL = not yet priced. Segment opps always have NULL TOTAL_ACV_USD so this is belt-and-suspenders with the name filter.
- `IS_SALESFORCE_OPPORTUNITY_CLOSED = FALSE` and `IS_SALESFORCE_OPPORTUNITY_DELETED = FALSE`
- SE in team of 8 (via `SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_NAME IN (...)` — no USER join needed)

**Excluded:** Technical Services, On Demand, Renewals, Segment 2/3/4/5 opps, unpriced opps (TOTAL_ACV_USD = NULL or 0), closed opps.

---

## SE ROSTER

| SE Name | Email | Initials |
|---|---|---|
| Deborah Awe | deborah.awe@snowflake.com | DA |
| James Newsom | james.newsom@snowflake.com | JN |
| Julie Heckman | julie.heckman@snowflake.com | JH |
| Lisa Batteiger | lisa.batteiger@snowflake.com | LB |
| Michael Hughes | michael.hughes@snowflake.com | MH |
| Stephen Pace | stephen.pace@snowflake.com | SP |
| Tim Whitaker | tim.whitaker@snowflake.com | TW |
| Whitney Burke | whitney.burke@snowflake.com | WB |

SE name and email resolved via `SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_NAME` / `_EMAIL` columns denormalized on the certified opp view — no USER join needed. Always returns `@snowflake.com` for this team.

---

## SFDC FIELD MAPPING

| Field | SNOW_CERTIFIED Column |
|---|---|
| TW status | `SALESFORCE_OPPORTUNITY_TECHNICAL_WIN_STATUS` — values: `'Yes'`, `'No Decision Yet'`, NULL, `'Lost'` |
| TW Date | `SALESFORCE_OPPORTUNITY_TECHNICAL_WIN_AT` (DATE) |
| SE Comments | `SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS` (TEXT — running log, newest entry prepended at top) |
| Agreement Type | `SALESFORCE_OPPORTUNITY_AGREEMENT_TYPE` |
| Opp Type | `SALESFORCE_OPPORTUNITY_TYPE` |
| Close Date | `SALESFORCE_OPPORTUNITY_CLOSED_AT` |
| ACV (filter) | `SALESFORCE_OPPORTUNITY_TOTAL_ACV_USD` (NUMERIC) — same field as TACV in opp-review-generation. Filter: `> 0` |
| Opp ID (18-char) | `SALESFORCE_OPPORTUNITY_ID` — used for SFDC links |
| SE Name | `SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_NAME` (denormalized — no USER join) |
| SE Email | `SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_EMAIL` (denormalized — no USER join) |
| Account Name | `SALESFORCE_ACCOUNT_NAME` (denormalized — no ACCOUNT join) |

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
    TRY_TO_DATE(REGEXP_SUBSTR(LEFT(SE_COMMENTS_C, 30), '([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})', 1, 1, 'e', 1), 'MM/DD/YY'),
    TRY_TO_DATE(REGEXP_SUBSTR(LEFT(SE_COMMENTS_C, 30), '([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})', 1, 1, 'e', 1), 'MM/DD/YYYY')
)
```
Uses `LEFT(SE_COMMENTS_C, 30)` to ensure we only extract the date from the **newest** (topmost) comment entry — not an older ISO-date entry buried further down. The `[/-]` character class matches both slash and dash separators in a single pass. TRY_TO_DATE COALESCE tries ISO first, then `MM/DD/YY`, then `MM/DD/YYYY`. NULL = no date found (treated as no recent comment).

> **BUG FIX (2026-08-20):** `MM/DD/YYYY` was previously tried before `MM/DD/YY`. Snowflake's `TRY_TO_DATE('08/18/26', 'MM/DD/YYYY')` silently returns `0026-08-18` (year 26 AD) instead of NULL. This made 2-digit-year comments (e.g. `08/18/26`) appear ancient, falsely triggering V5/V6 stale violations. Fixed by putting `MM/DD/YY` first.

> **BUG FIX (2026-08-11):** Previously, ISO format was tried against the *entire* SE_COMMENTS_C field first. When a newer slash-format entry sat at the top but an older ISO entry existed further down, the ISO regex matched the stale date — causing false V5/V6 stale violations. Fixed by scoping to LEFT(..., 30).

---

## SLACK MESSAGE FORMAT

### SE Message (per SE, grouped by opp)

```
[TEST — Notifications for: *Whitney Burke*]    ← test mode only

:red_circle: *Opp Violations — Week of 08/20/2026*

*<https://snowforce.../OPP_ID/view|Versova>* — Close: 08/28, TW: 07/30
  • No Comment in Last 2 Weeks (TW Due <4 Weeks)
  _Last comment: 08/18/26 [WB] TW: ..._

*<https://snowforce.../OPP_ID/view|SureScripts>* — Close: 11/10, TW: 01/31/2027
  🟡 • TW Date is Placeholder (2027-01-31)    ← yellow dot for Yellow violations
```

Header uses `:large_yellow_circle:` instead of `:red_circle:` when **all** violations for the SE are Yellow.

Each opp is a clickable link to SFDC. All violations for that opp are bullets underneath — the SE can open one opp and fix everything.

### Manager Summary

```
:bar_chart: *SE Violations Summary — 08/20/2026*
Run: `8B00005933F0` | Mode: `LIVE` | Total: *55 violations* across *8 SEs*

✅ *Julie Heckman* — 12 violations
  • No Comment in Last 4 Weeks: 11
  • No SE Comments: 1

✅ *Lisa Batteiger* — 3 violations        ← all-yellow SE: 🟡 prefix only
  🟡 TW Date is Placeholder (2027-01-31): 3

✅ *Stephen Pace* — 5 violations          ← mixed: plain bullet for red, 🟡 for yellow
  • No TW Date: 2
  • No Comment in Last 2 Weeks: 1
  🟡 TW Date is Placeholder (2027-01-31): 2
...
```

Manager summary groups by SE → violation type. When yellows exist, red violations use plain `•` (no emoji) and yellow violations use `🟡` prefix. All-red SEs are unchanged.

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
    SELECT o.SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_NAME AS se_name,
           o.SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_EMAIL AS se_email,
           o.SALESFORCE_ACCOUNT_NAME AS account_name,
           o.SALESFORCE_OPPORTUNITY_NAME AS opp_name,
           o.SALESFORCE_OPPORTUNITY_ID AS opp_id,
           o.SALESFORCE_OPPORTUNITY_CLOSED_AT AS close_date,
           o.SALESFORCE_OPPORTUNITY_TECHNICAL_WIN_STATUS AS tw_status,
           o.SALESFORCE_OPPORTUNITY_TECHNICAL_WIN_AT AS tw_date,
           o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS AS se_comments,
           COALESCE(
               TRY_TO_DATE(REGEXP_SUBSTR(LEFT(o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS, 30),'([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})',1,1,'e',1),'YYYY-MM-DD'),
               TRY_TO_DATE(REGEXP_SUBSTR(LEFT(o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS, 30),'([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})',1,1,'e',1),'MM/DD/YY'),
               TRY_TO_DATE(REGEXP_SUBSTR(LEFT(o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS, 30),'([0-9]{1,4}[/-][0-9]{1,2}[/-][0-9]{2,4})',1,1,'e',1),'MM/DD/YYYY')
           ) AS most_recent_comment_date,
           CASE
               WHEN o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS IS NULL OR TRIM(o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS) = '' THEN 'no_comment'
               WHEN REGEXP_LIKE(TRIM(o.SALESFORCE_OPPORTUNITY_SALES_ENGINEER_COMMENTS),
                   '^([0-9]{1,2}/[0-9]{1,2}/[0-9]+|[0-9]{4}-[0-9]{2}-[0-9]{2})[[:space:]]*(\\[[A-Za-z]+\\]|[A-Za-z]{2,4}).*TW.*','is')
                   THEN 'valid'
               ELSE 'invalid'
           END AS comment_format
    FROM SNOW_CERTIFIED.SALESFORCE_OPPORTUNITY.DD_SALESFORCE_OPPORTUNITY o
    WHERE o.IS_SALESFORCE_OPPORTUNITY_DELETED=FALSE AND o.IS_SALESFORCE_OPPORTUNITY_CLOSED=FALSE
      AND o.SALESFORCE_OPPORTUNITY_AGREEMENT_TYPE LIKE 'Capacity%' AND o.SALESFORCE_OPPORTUNITY_TYPE != 'Renewal'
      AND o.SALESFORCE_OPPORTUNITY_NAME NOT ILIKE '%-Segment%'
      AND o.SALESFORCE_OPPORTUNITY_TOTAL_ACV_USD > 0
      AND o.SALESFORCE_OPPORTUNITY_LEAD_SOLUTION_ENGINEER_NAME IN ('Deborah Awe','James Newsom','Julie Heckman',
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
