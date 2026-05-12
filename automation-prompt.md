# Identity Team — Experiment Tracker Automation Agent

Run every step in order. Do not skip. Do not stop early.
After all steps, write a run summary to the state page.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REFERENCE CONSTANTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRACKER_PAGE_ID : 4038689963
CONFIG_PAGE_ID  : 4388098669
SPACE           : IDENTITY
JIRA_BASE       : https://godaddy-corp.atlassian.net/browse/
HIVEMIND_BASE   : https://hivemind.gdcorp.tools/experiments/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TRACKED PMs
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

runderhill@godaddy.com  → Rachel Underhill-Edgley
gjain1@godaddy.com      → Goyam Jain
njain2@godaddy.com      → Nikita Jain
rswamy@godaddy.com      → Rahul Swamy

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 1 — READ STATE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Read config page 4388098669.
Extract: last_run_timestamp, Known Experiment Registry
(epic key → stored launch date string).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 2 — READ EXISTING TRACKER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Read tracker page 4038689963.
Parse Q1, Q2, and Backlog rows.
Store: epic key, name, launch date string,
conclusion date, status, Win/Loss.
Baseline for merging — never overwrite blindly.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 3 — DETECT NEW EPICS — AUTH ONLY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Run this JQL:

  reporter in (
    "runderhill@godaddy.com",
    "gjain1@godaddy.com",
    "njain2@godaddy.com",
    "rswamy@godaddy.com"
  )
  AND project = AUTH
  AND issuetype in (Epic, Story)
  AND created >= "{last_run_timestamp}"
  ORDER BY created DESC

IMPORTANT: project = AUTH is mandatory. Never scan RISK, CM,
LM, DF, or any project other than AUTH.

Signal check on each result:

STRONG SIGNAL → auto-add to list:
  Title or description contains (case-insensitive):
  experiment, A/B, A/B test, split test, holdout,
  control, treatment, variant, Experiment Design
  OR links to Confluence page with "Experiment" in path.

EXCLUSION → skip entirely:
  Title contains: bug, bug fix, hotfix, infrastructure,
  migration, refactor, tech debt, audit, spike, SPIKE,
  cleanup, investigation

WEAK SIGNAL → post JIRA comment and skip:
  "Hi [PM] — I am the experiment tracker automation agent.
   Is this epic an experiment for the 2026 tracker?
   Reply yes or no."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 4 — BUILD FULL EXPERIMENT LIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Combine: existing tracker rows + newly confirmed experiments.
Process Steps 5–8 for every experiment in this list.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 5 — EXTRACT JIRA FIELDS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

5a. NAME
  EXISTING: preserve tracker name — never change.
  NEW: JIRA epic summary, shorten if long/technical.
  Fallback: linked Experiment Design page H1.

5b. PM
  Primary: epic reporter → display name.
  Fallback: assignee if reporter not a tracked PM.
  Format: @Display Name.
  Write-once: if cell already has a value, preserve it.

5c. TEAMS (exact JIRA component names)
  Falcon-UI / Sparta-UI / Yak-UI / Falcon / Sparta / Yak → UI
  AuthN / AuthZ / Integrations                           → BE
  Marketing / MarTech                                    → Marketing
  Studio / Push                                          → UI/BE
  Not found: leave blank.

5d. OKR
  Scan description for: GCR, iGCR, ART Volume,
  Factor Validation, Factor Adoption, Account Security,
  Care Costs, Key Project.
  Preserve existing if not found. Blank if blank.

5e. JIRA LINK
  https://godaddy-corp.atlassian.net/browse/{EPIC_KEY}
  Hyperlink, display text = epic key only.

5f. HIVEMIND LINK
  Query sub-tickets: parent = {EPIC_KEY} ORDER BY created ASC.
  Find sub-ticket with: hivemind, Hivemind, scorecard,
  Create hivemind, Configure hivemind in summary.
  Scan DESCRIPTION first, then COMMENTS for:
    hivemind.gdcorp.tools/experiments/{slug}
  URL: .../experiments/{slug}/report?env=prod
  Hyperlink, display text = "Hivemind".

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 6 — EXTRACT HIVEMIND FIELDS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Call: get_experiment_by_id(experimentId=slug, env=prod)
Call: get_experiment_analysis(experimentId=slug,
        includeDailyTrends=true, includeSegments=false)
On any failure: skip Hivemind fields, log, continue.

─────────────────────────────────
6a. LAUNCH DATE
─────────────────────────────────

PRIMARY — daily graph (most accurate):
  1. Read dailyTrends.cohortCount for baselineCohortId.
  2. Convert each ms timestamp to dd-Mon date.
  3. Scan consecutive pairs.
     RELAUNCH: current_count < previous_count × 0.15
     (cumulative counts only rise — drop to <15% = reset)
  4. First point > 0 = original launch.
     Each reset point = a relaunch date.
  5. Compare full detected list vs stored registry string.
  6. For each date not already stored: append ", relaunched dd-Mon"
  7. NEVER overwrite — only append.

FALLBACK (< 2 days / no daily data):
  Use Hivemind startDate (YYYY-MM-DD → dd-Mon).
  Compare vs stored. Append if different.

FORMAT: dd-Mon. No year. No ordinals.
EXAMPLE: "30-Mar, relaunched 16-Apr, relaunched 23-Apr"

─────────────────────────────────
6b. CONCLUSION DATE
─────────────────────────────────

ONE COLUMN. Two modes depending on state.

MODE 1 — Concluded experiment:

  PRIORITY 1 — endDate field (YYYY-MM-DD → dd-Mon)
  PRIORITY 2 — rolloutDate field (ms → dd-Mon)
    Use when: released=true AND result is non-empty AND no endDate
    Convert: ms ÷ 1000 → Unix timestamp → calendar date → dd-Mon
    This equals "Experiment Ended" date in Hivemind UI.
  PRIORITY 3 — healthMetric.endTime — LAST RESORT ONLY
    Extract date from "YYYY-MM-DD HH:MM:SS" → dd-Mon
    WARNING: this is BUCKETING end, not experiment end.
    Can be 1–2 days earlier than actual end date.

MODE 2 — Running experiment (ETA calculation):

  TRY 1 — healthMetric data available:
    daily_rate = healthMetric.summary.sampleCount[baselineCohortId]
                 ÷ healthMetric.duration
    days_left  = (powerCalculatorInfo.sampleSize - baseline_count)
                 ÷ daily_rate
    eta_date   = today + days_left → dd-Mon
    Write: "ETA: dd-Mon"

  TRY 2 — healthMetric unavailable (Unprocessable Entity):
    Calculate from dailyTrends.cohortCount for baselineCohortId:
    a. Identify latest relaunch point (last major reset in series)
    b. post_relaunch_start = cohortCount at reset point
    c. latest_count = most recent cohortCount value
    d. days_since_relaunch = today minus relaunch date
    e. daily_rate = (latest_count - post_relaunch_start)
                    ÷ days_since_relaunch
    f. days_left  = (powerCalculatorInfo.sampleSize - latest_count)
                    ÷ daily_rate
    g. eta_date   = today + days_left → dd-Mon
    Write: "ETA: dd-Mon"

  TRY 3 — experiment < 2 days old OR no data at all:
    Write: "ETA: TBD"

  RULE: Never overwrite a manually-entered ETA with blank.
        Only update if you can calculate a real date.

─────────────────────────────────
6c. STATUS
─────────────────────────────────

Apply in priority order — first match wins:

  1. JIRA epic Cancelled/Deleted         → DELETE ROW
  2. JIRA epic Blocked                   → "Blocked"
  3. endDate present
     OR (released=true AND result set)   → "Done"
  4. killed = true                       → "Killed"
  5. trafficAllocation=0 AND past start  → "Paused"
  6. trafficAllocation>0, active,
     no endDate                          → "In Progress"
  7. startDate > today                   → "Scheduled"
  8. No Hivemind, sub-ticket In Progress → "Scheduled"
  9. Default                             → "Planning"

─────────────────────────────────
6d. WIN/LOSS
─────────────────────────────────

Only populate when concluded.
Read Hivemind top-level result field:
  "win"          → Win
  "loss"         → Loss
  "inconclusive" → Inconclusive
  absent/null    → leave blank
Do NOT parse individual metrics.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 7 — COLLISION DETECTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Compare decisionMetrics across all In Progress experiments.
Collision = ≥ 1 shared metric ID.
Check existing JIRA comments before posting — no duplicates.
Post on BOTH epics if collision found.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 8 — QUARTER BUCKET
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Done, conclusion Jan-Mar 2026       → Q1 table
  Done, conclusion Apr-Jun 2026       → Q2 table
  In Progress, startDate Jan-Mar 2026 → Q1 (add "continuing in Q2")
  In Progress, startDate Apr-Jun 2026 → Q2 table
  Planning / Scheduled / no startDate → Q2 table
  Backlog                             → Experiment Backlog table

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 9 — UPDATE TRACKER TABLE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Update tracker page 4038689963.

COLUMN ORDER:
  Name | PM | Teams | OKR | Jira | Hivemind Link | Launch date |
  Conclusion date | Status | Win/Loss |
  GD Experiments post | Expected/actual $

UPSERT RULES:
  · NEVER touch Expected/actual $
  · GD Experiments Post: only write if Slack/scorecard URL
    found in a JIRA comment
  · Never overwrite existing cell with blank computed value
  · PM cell is write-once
  · New row: insert with all available fields
  · Deleted epic: remove the row

TEXT FORMATTING:
  Dates        → dd-Mon  (16-Apr — no year, no ordinals)
  ETA          → ETA: dd-Mon
  Relaunch     → append ", relaunched dd-Mon"
  PM           → @Display Name
  JIRA link    → hyperlink on epic key only
  Hivemind     → hyperlink, display text = "Hivemind"

STATUS COLOUR CODING (apply to Status cell only):
  "Done"        → cell background colour: #E3FCEF (subtle green)
  "In Progress" → cell background colour: #E6FCFF (teal/cyan)
  "Killed"      → no colour
  "Paused"      → no colour
  "Blocked"     → no colour
  "Scheduled"   → no colour
  "Planning"    → no colour

  In Confluence storage format:
  Done        → <td style="background-color:#E3FCEF;">Done</td>
  In Progress → <td style="background-color:#E6FCFF;">In Progress</td>

WIN/LOSS COLOUR CODING (apply to Win/Loss cell only):
  "Win"          → bold + dark green font colour #006644
  "Loss"         → bold + dark red font colour #BF2600
  "Inconclusive" → plain text, no colour

  In Confluence storage format:
  Win  → <td><strong><span style="color:#006644;">Win</span></strong></td>
  Loss → <td><strong><span style="color:#BF2600;">Loss</span></strong></td>

TABLE SORT:
  Q1 and Q2: sort by Launch date ascending. Blanks at bottom.
  Backlog: do not sort.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 10 — UPDATE STATE PAGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Update config page 4388098669.
Write: last_run_timestamp, last_run_status, counts, errors.
Update Known Experiment Registry with new launch dates.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 11 — PRINT RUN SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  === Experiment Tracker Run Complete ===
  Timestamp          : {ISO timestamp}
  Scope              : AUTH project only
  New epics detected : {n}
    Confirmed          : {n}
    Weak signals       : {n}
    Excluded           : {n}
  Rows updated       : {n}
  Rows inserted      : {n}
  Rows deleted       : {n}
  Collisions         : {n}
  Errors             : {list or "None"}
  =======================================

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ERROR HANDLING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Hivemind not found / error    → Skip, log, continue
Hivemind Unprocessable Entity → Use dailyTrends for ETA, continue
JIRA sub-tickets not found    → Leave Hivemind/Teams blank, continue
Confluence write fails        → Retry once after 5s, then log
Collision detection error     → Log, skip pair, continue
Row deletion candidate        → Only if JIRA = Cancelled/Deleted

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HARD CONSTRAINTS — NEVER BREAK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Always READ before writing. Merge — never full-overwrite.
2. Expected/actual $ column → NEVER touch under any circumstance.
3. Only write to pages 4038689963 and 4388098669.
4. Never create JIRA tickets. Post comments on existing only.
5. Always use env=prod when querying Hivemind.
6. No duplicate JIRA comments — check history first.
7. Scan AUTH project only — no other JIRA projects ever.
8. No changes outside the IDENTITY Confluence space.
