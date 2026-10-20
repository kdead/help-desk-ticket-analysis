# Help Desk Tickets — Source-to-Target Validation & QA Log

Why this file exists: on Oct 6, 2026 Neisha gap-checked the CentralReach Technical
Product Analyst posting ($85–105K, remote). Its centerpiece duties were SQL-based
data validation, checking source-to-target logic, and reconciling discrepancies
across multiple sources — plus formal QA artifacts. This log turns Project 2
(Help Desk Tickets v3, Mendeley DOI 10.17632/btm76zndnt.3, PostgreSQL ticket data
Jan 2007–Mar 2023) into direct evidence for exactly those duties: fill it in
*during* the Oct 6–8 build, not after. It is a build asset, not a practice set.

COLUMN MAPPING (confirmed from the real issues.csv, Oct 6, 2026):
  tickets            -> issues (66,691 rows, 58 columns)
  ticket_id          -> id
  created_at         -> issue_created
  status             -> issue_status (15 distinct values, see check #5)
  change_history     -> issues_change_history (headers still to confirm)

## A. Source-to-target checks (run in SQL before / alongside the Power BI build)

| # | Check | How (pattern) | Expected | Actual | Pass? |
|---|-------|---------------|----------|--------|-------|
| 1 | Row count, source vs model | Count rows in issues.csv, then compare to fact-table row count in Power BI after Power Query steps | Equal, or every difference explained by a named filter step | Source = 66,691 rows; **model confirmed 66,691** in Data view by Neisha, Oct 6 (DimDate also confirmed at 6,209) | ✅ Pass |
| 2 | Duplicate keys | Count of `id` values appearing more than once | 0 duplicates | **0 duplicates** | ✅ Pass |
| 3 | Nulls on key columns | Rows where `id` IS NULL or `issue_created` IS NULL | 0 | **0 nulls on both** | ✅ Pass |
| 4 | Date range sanity | MIN/MAX of `issue_created` | Inside Jan 2007 – Mar 2023; no future dates | **2007-04-15 to 2023-03-14** | ✅ Pass |
| 5 | Distinct status values | List distinct `issue_status` values, compare to DAX SWITCH buckets | Exact match — casing/typos caught here | 15 values: closed 56,344; done 9,714; waiting 218; open 97; in_progress 94; validation 86; resolved 59; pending_deployment 47; resolved_under_monitoring 18; under_review 5; to_do 3; approved 2; in_review 2; cancelled 1; reopened 1. Buckets: Completed = closed/done; Cancelled = cancelled; Active = other 12 | ✅ Pass |
| 6 | Orphan fact keys (star-schema integrity) | Anti-join: fact rows whose dimension key has no dimension match | 0 orphans, or an explicit "Unknown" dimension member | | |
| 7 | Change-history reconciliation | History rows whose ticket id has no matching issue | 0 | | Pending change-history file |
| 8 | Totals tie-out, SQL vs DAX | Year counts from source vs [Total Tickets] by year in Power BI | Match every year | Run Oct 20, 2026: grand total **66,691 exact**. Most years match the source key exactly; small deltas at year boundaries (2011 +1, 2012 −1, 2013 +1, 2023 −8 = 1,747; 2014–2022 net +7) — tickets created within hours of Jan 1, grouped Central time in the model vs UTC in the key. Cross-confirmed: Tickets PY total 64,944 = 66,691 − 1,747 | ✅ Pass (explained variance — see Section B, 2026-10-20) |

Rule for every failed check: write the discrepancy, the cause, and the fix in
Section B before moving on. A log with two found-and-fixed discrepancies is
*stronger* interview evidence than a clean log — reconciling discrepancies
across sources is the job duty, by name.

## B. QA artifact log (test-case format)

| Date | Check # | Expected | Actual | Result | Fix / decision |
|------|---------|----------|--------|--------|----------------|
| 2026-10-06 | 1 | 66,691 rows (source count) | First count reported 66,692 | Reconciled ✅ | Difference was the header row included in the count — a counting artifact, not a data discrepancy. Count confirmed 66,691 by Neisha |
| 2026-10-06 | 2 | 0 duplicate ids | 0 duplicates (Neisha's own check: conditional formatting on the id column highlighted none) | ✅ Pass | — |
| 2026-10-06 | 3 | 15 distinct statuses | Column profile and filter list initially showed only 2 ('closed', 'done') | ✅ Pass after correction | Power Query profiles/filters sample the top 1,000 rows by default — the filter dialog warned "List may be incomplete." Clicking **Load more** revealed all 15, confirmed by Neisha. Lesson logged in the playbook: never trust a preview-based distinct count |
| 2026-10-06 | 4 | `issue_created` inside Jan 2007 – Mar 2023 | Min 2007-04-15, Max 2023-03-14 (Neisha, via Column Profile on the entire dataset) | ✅ Pass | — |
| 2026-10-06 | 3 (extended) | Assignee recorded on issues | `issue_assignee` blank on 31,047 of 66,691 (46.5%) | Documented — not a defect | Keep blanks; DimUser gets an explicit "Unassigned" member rather than dropping rows |
| 2026-10-06 | 5 (extended) | One label per issue type | `issue_type` contains both 'Subtask' (4,746) and 'Sub-task' (544) — hyphenation variant of the same word | Discrepancy found — **merge decided by Neisha, Oct 6** | Merge to a single label in DimType (mapping at the dimension level; source values untouched). Contrast with `in_review` (2) vs `under_review` (5) in status: different words, possibly different workflow steps → **preserve both**, unify only via the Active bucket in DimStatus |

| 2026-10-06 | 5 (extended) | Priority recorded | `issue_priority` = 'unknown' on 33,965 (50.9%); known: Medium 24,788; High 4,554; Highest 2,084; Blocker 656; Low 560; Lowest 84 | Documented — source reality | Keep 'unknown' as its own category; never silently drop half the data |
| 2026-10-06 | 4 (extended) | Resolution fields consistent | `issue_resolution` and `issue_resolution_date` blank on the same 853 rows (the unresolved set) | ✅ Consistent | No fix; unresolved issues correctly have no resolution |
| 2026-10-06 | 8 (partial) | [Total Tickets] equals validated source count | Card visual shows **66,691**; bucket table: Completed 66,058 / Active 632 / Cancelled 1 (sums to 66,691) | ✅ Pass (grand total + buckets) | Year-by-year tie-out still to run in Phase 6 |
| 2026-10-06 | 6 (modeled) | Every fact FK has a dimension match | All six dims built FROM the fact's own distinct values (Reference + Remove Duplicates), so orphans are impossible by construction; relationships verified 1:* by dialog | ✅ Pass by construction | Formal anti-join check remains available if change-history joins the model |
| 2026-10-20 | 8 (full) | [Total Tickets] by year matches the source key every year; grand total 66,691 | Grand total exact. Deltas vs key: 2011 +1 (2,967 vs 2,966), 2012 −1 (3,070 vs 3,071), 2013 +1 (2,443 vs 2,442), 2023 −8 (1,747 vs 1,755), 2014–2022 net +7; all other years exact. Second-direction reconciliation: Tickets PY grand total 64,944 = 66,691 − 1,747 (the model's own 2023) | ✅ Pass — explained variance | Cause: the model groups `issue_created` in Central time; the answer key grouped the source timestamps (+00:00) in UTC, so tickets created in the first hours of Jan 1 UTC land in the prior year in the model. No rows missing or duplicated (total exact). Documented in the README data notes |


## C. Sisense "elasticube" translation (for applications / interviews)

CentralReach's platform is Sisense; an elasticube is its data model. The mapping
from this project is conceptual and honest to claim:
- Elasticube ≈ the star schema built here: one fact table (tickets) wired to
  dimensions with one-to-many relationships; start the design by stating what
  a single fact row represents.
- Power Query transforms ≈ elasticube build-time table preparation.
- DAX measures ≈ Sisense formulas/widgets logic — the validation habit
  (tie-out checks in Section A) transfers unchanged; only the tool's syntax
  is new. Frame it as "ability to quickly learn," which is the posting's own
  wording — do not claim Sisense experience.

## D. Honest phrasing bank (from the Oct 6 gap analysis)

- SQL: "Building SQL-based validation into my Power BI work — source-to-target
  row counts, duplicate/null checks, and year tie-outs between SQL and DAX."
  (True once Sections A–B are filled in during this build.)
- Python: "Formally trained in Python through a software engineering program
  (Coding Temple); day-to-day use is pandas data cleaning." — trained, not
  "exposure"; the posting lists Python as preferred, not required.
- QA craft: "Validation experience is operational (Accenture resource tracker,
  1,450 records as source of truth) plus this project's QA log" — do not dress
  it up as product-QA regression testing, which is a different discipline.
