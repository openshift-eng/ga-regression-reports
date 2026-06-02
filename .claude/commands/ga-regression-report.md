---
description: Report regressions open on a given date (GA or final RC) for a release with triage status breakdown
argument-hint: <release>
---

## Name

ci:ga-regression-report

## Synopsis

```
/ga-regression-report <release> [--date <YYYY-MM-DD>] [--min-days <days>]
```

## Description

The `ga-regression-report` command generates a markdown report of Component Readiness regressions for a given OpenShift release, intended to be committed to git for historical analysis. By comparing reports across releases, we can track whether regression management is improving or degrading over time.

The report covers regressions that were still open on a specified report date — either a user-provided date (typically the final RC date) or the GA date (the default). It filters out infrastructure-only regressions (triaged as `ci-infra` or `product-infra`) and short-lived regressions (open fewer than `--min-days` days, default 7). It includes a high-level release lifecycle summary, triaged bug details with SBAR status, triage type validation, and untriaged regression details.

The command writes the report to a file, asks the user to review it, and gives them an opportunity to add QSE commentary before finalizing.

## Implementation

**Important: Avoiding user permission prompts when running scripts**

When calling Python skill scripts via the Bash tool, always run the script directly without piping the output through inline Python (`python3 -c "..."`). Complex piped commands trigger user permission prompts, while simple `python3 script.py args` calls are auto-approved.

1. **Parse Arguments**: Extract the release version from `$1` (e.g., `4.21`). Parse optional flags:
   - `--date <YYYY-MM-DD>`: The report date to evaluate regressions against. If provided, this is used instead of the GA date (typical use: the final RC date). If not provided, the GA date is used as the default.

   - `--min-days <days>`: Minimum number of days a regression must have been open to be included. Default: `7`.

   If no release is provided, prompt the user for one.

2. **Remove Existing Report**: If a file named `<release>-ga-regression-report.md` already exists in the `openshift/` subdirectory of the repository root, delete it before proceeding.

3. **Determine Report Date**: If `--date` was provided, use that as the report date and record the date source as "Final RC" (or whatever context the user provided). Otherwise, use the `get-release-dates` skill to determine the GA date:

   ```bash
   release_dates=$(python3 plugins/teams/skills/get-release-dates/get_release_dates.py --release "$1")
   ```

   Extract the `ga` field from the JSON response. If `ga` is null and no `--date` was provided, the release has not yet GA'd — inform the user and stop. When using the GA date, record the date source as "GA".

   The resulting date and source are referred to as the **report date** throughout the rest of this document.

4. **Fetch Release Lifecycle Summary**: Use the `health-check-regressions` skill to get high-level regression metrics across the entire release lifecycle. Run `list_regressions.py` with the `--short` flag to get only summary statistics:

   ```bash
   python3 plugins/teams/skills/list-regressions/list_regressions.py --view "${release}-main" --short
   ```

   From the summary, extract:
   - `summary.total`: Total regressions across the release lifecycle
   - `summary.time_to_triage_hrs_avg`: Average hours to triage
   - `summary.time_to_resolve_hrs_avg`: Average hours to resolve

   These metrics cover all regressions from development start through today, not just those open on the report date. This provides context for cross-release comparison.

   **Note**: The `--short` summary only provides average and max timing values, not percentiles. P50 and P95 time to triage and time to resolve must be computed from the individual regression data fetched in step 6. See step 6 for the computation method.

   **Unique JIRA Bugs (lifecycle)**: Do not use `summary.triaged` for this — it counts triaged regressions, not unique bugs. Instead, use the full regression data from step 6 (which fetches all regressions, not just the `--short` summary). Collect all `triages[].url` values across every regression (open and closed) in the entire view, deduplicate by URL, and count the unique bug URLs. This is the true number of distinct JIRA bugs filed across the release lifecycle.

5. **Fetch Qualified Job Count**: Query the Sippy API to count the number of jobs that qualified for Component Readiness analysis:

   ```bash
   curl -s 'https://sippy-auth.dptools.openshift.org/api/jobs?release=<release>&filter={"items":[{"columnField":"variants","operatorValue":"has entry","value":"JobTier:standard"},{"columnField":"variants","operatorValue":"has entry","value":"JobTier:informing"},{"columnField":"variants","operatorValue":"has entry","value":"JobTier:blocking"}],"linkOperator":"or"}&period=default&sortField=net_improvement&sort=asc'
   ```

   Count the number of items in the returned JSON array. This is the total number of jobs that fed into Component Readiness for the release (typically in the hundreds).

6. **Fetch All Regressions**: Construct the view name by appending `-main` to the release (e.g., `4.21` becomes `4.21-main`). Use the `list_regressions.py` script to fetch regressions for this view:

   ```bash
   python3 plugins/teams/skills/list-regressions/list_regressions.py --view "${release}-main"
   ```

   This returns a JSON object with `summary` and `components` fields. Collect all regressions from the `open` and `closed` arrays across all components. The script uses the parent regression's `opened` timestamp but the view-specific `closed` timestamp.

   **Compute P50 and P95 Timing Metrics**: Using the individual regression data, compute P50 (median) and P95 (95th percentile) values for time to triage and time to resolve. P50 shows the typical experience while P95 reduces the influence of extreme outliers. Together with the average, they give a fuller picture of triage and resolution performance.

   - **Time to Triage**: For each triaged regression (non-empty `triages` array), compute time to triage as the difference between the earliest `triages[].created_at` timestamp and the regression's `opened` timestamp, in hours. Collect all values, sort them, and take P50 at `sorted_values[int(len(sorted_values) * 0.50)]` and P95 at `sorted_values[int(len(sorted_values) * 0.95)]`.
   - **Time to Resolve**: For each closed regression (non-null `closed` field), compute time to resolve as the difference between `closed` and `opened`, in hours. Collect all values, sort them, and take P50 and P95 at the same percentile indices.

   Convert the resulting hours to days for display (divide by 24, round to one decimal place). These percentile values are used alongside the averages in the Release Lifecycle Summary and Comparison tables.

7. **Filter Regressions Open on Report Date**: From the full regression list, select regressions that were open on the report date. A regression was open on that date if:

   - `opened <= <report_date>T23:59:59Z` AND
   - `closed` is `null` (still open) OR `closed >= <report_date>T00:00:00Z` (closed after the report date)

   The `closed` field is either an ISO timestamp string or `null` (if the regression is still open). Note: `opened` reflects when the regression was first detected globally, while `closed` reflects when it was closed in the main view specifically.

8. **Filter Out Infrastructure Regressions**: Remove any regression where **all** triage entries have a `type` of `ci-infra` or `product-infra`. If a regression has at least one triage with a different type (e.g., `product` or `test`), keep it. Untriaged regressions (empty `triages` array) are kept.

9. **Filter Out Short-Lived Regressions**: Remove regressions that were open for fewer than `--min-days` days (default 7). Calculate the duration as the number of days from `opened` to the earlier of: `closed` (if not null) or the report date. This filters out transient flakes that resolved quickly.

10. **Classify Triage Status**: For each remaining regression, check the `triages` array:

   - **Triaged**: `triages` array has at least one entry. Each triage entry has:
     - `url`: Link to the JIRA bug (e.g., `https://issues.redhat.com/browse/OCPBUGS-12345`)
     - `type`: Triage category (`product`, `test`, `ci-infra`, `product-infra`)
     - `description`: Short description of the triage
   - **Not triaged**: `triages` array is empty

11. **Collect Unique Bugs**: From all triaged regressions, collect unique bug URLs from `triages[].url`. Extract the JIRA key from each URL (e.g., `OCPBUGS-12345` from `https://issues.redhat.com/browse/OCPBUGS-12345`).

12. **Fetch Bug Details and Detect SBAR**: For each unique JIRA bug key, fetch the issue details. If JIRA credentials are configured (`JIRA_USERNAME` and `JIRA_API_TOKEN` environment variables), use the JIRA API:

   ```bash
   script_path="plugins/ci/skills/fetch-jira-issue/fetch_jira_issue.py"
   python3 "$script_path" "<jira_key>" --format json
   ```

   Extract the `summary` and `release_note_type` fields from each response. A bug has a release note if `release_note_type` is anything other than `"Release Note Not Required"` or `null`. If JIRA credentials are not set, display the bug keys and URLs without titles.

   **SBAR Detection**: For each bug, check for evidence of an SBAR (Situation-Background-Assessment-Recommendation) document:

   - **Labels**: Check the `labels` array for labels containing `sbar` (case-insensitive), such as `sbar-candidate`, `sbar-approved`, `SBAR`, etc. Record the matching label.
   - **Comments**: Scan each comment's `body` text for Google Docs links (`docs.google.com`) that appear near the word "sbar" (case-insensitive) within the same comment. If found, record the Google Docs URL as the SBAR link.

   Classify each bug's SBAR status as one of:
   - **Approved**: has an `sbar-approved` label
   - **Candidate**: has an `sbar-candidate` label
   - **Linked**: no SBAR label but a comment references an SBAR with a Google Docs link
   - **None**: no SBAR evidence found

13. **Fetch JIRA SBAR Exceptions**: Query the JIRA API for bugs with SBAR labels that affect this release but may not have been statistically showing as Component Readiness regressions on the report date. These represent issues the team identified and escalated during the release cycle that aren't captured by the CR regression data alone.

   Use the JIRA v3 `search/jql` endpoint (POST to `https://redhat.atlassian.net/rest/api/3/search/jql`) with the following JQL:

   ```
   project = OCPBUGS AND labels in ("sbar-approved", "sbar-candidate") AND affectedVersion = "<release>" AND statusCategory != Done
   ```

   The `statusCategory != Done` filter approximates "open on the report date" — bugs closed before the report date would already be in a Done status category. Request fields: `key`, `summary`, `status`, `labels`, `fixVersions`, `resolution`.

   For each returned bug:

   - **Cross-reference with Component Readiness triage records**: Check the full regression data (already fetched in step 6) for any triage records whose `url` contains this bug's key. If triage records exist, check their `type` field. Exclude the bug if **all** associated triage records have a type of `ci-infra` or `product-infra` (same infrastructure filter as step 8). Bugs with no triage records in CR data are kept — they represent JIRA-only SBAR exceptions.

   - **Exclude bugs already in the Triaged Bugs table**: If the bug key already appears in the unique bugs collected in step 11, skip it — it's already covered by the main regression analysis.

   The remaining bugs are "Additional SBAR Exceptions." Extract each bug's SBAR status from its labels (`sbar-approved` → Approved, `sbar-candidate` → Candidate).

   **Linking convention**: Every JIRA bug key mentioned anywhere in the report — including in exclusion notes, footnotes, and prose — must be a markdown link to the bug URL (e.g., `[OCPBUGS-12345](https://redhat.atlassian.net/browse/OCPBUGS-12345)`). Never render a bare bug key.

14. **Compute Untriaged Regression Details**: For each untriaged regression, gather:

   - `id`: Regression ID
   - `component`: Component name
   - `test_name`: Full test name
   - `variants`: List of affected variants
   - `opened`: When the regression was first detected
   - `closed`: When the regression was closed (if it has been)
   - **Duration open on report date**: Calculate the number of days from `opened` to the report date. If still open, also note current duration from `opened` to today.

15. **Compare with Prior Release** (optional): Compute the prior release version by decrementing the minor version (e.g., `4.21` → `4.20`, `4.22` → `4.21`). If the current release minor version is `0` (e.g., `5.0`), the prior release is the last minor of the previous major (e.g., `4.22`). Look for a file named `<prior_release>-ga-regression-report.md` in the `openshift/` subdirectory of the repository root.

   If found, read the prior report and parse its markdown to extract key metrics from the Release Lifecycle Summary table and Regressions Open on Report Date section:

   - Qualified jobs
   - Total regressions (lifecycle)
   - Unique JIRA bugs (lifecycle)
   - Avg time to triage
   - P50 time to triage
   - P95 time to triage
   - Avg time to resolve
   - P50 time to resolve
   - P95 time to resolve
   - Regressions open on report date (after filtering)
   - Untriaged regressions on report date
   - Triage percentage on report date

   If the file does not exist, silently skip the comparison — do not warn the user.

   **Prior Release SBAR Follow-up**: If the prior release report was found, parse the Triaged Bugs table to identify all bugs that had an SBAR status of Approved, Candidate, or Linked. For each of these bugs, fetch the current JIRA issue details using `fetch_jira_issue.py` (same as step 12). For each bug, determine:

   - **Bug status**: Current JIRA status (e.g., Closed, Verified, ON_QA, Assigned, New)
   - **Bug resolution**: Resolution field if closed (e.g., Fixed, Won't Fix, Duplicate)
   - **Bug closed date**: When the bug was moved to a terminal status, if applicable
   - **Regression triage record status**: Re-fetch the regressions for the prior release view (`<prior_release>-main`) and check whether the triage records referencing this bug have been closed. A triage record is considered closed when the regression it is attached to has a non-null `closed` timestamp.

   Classify each prior SBAR bug into one of:
   - **Resolved**: Bug is closed/verified AND all associated triage records are closed
   - **Bug Open**: The JIRA bug is still in a non-terminal status — this is a **serious concern**
   - **Triage Open**: The bug may be closed but the Component Readiness triage record is still open — this is a **serious concern**

   Failing to fix an SBAR'd issue within an entire subsequent release cycle is considered a serious failing and must be prominently highlighted in the report.

16. **Generate Report**: Build a markdown report with the following sections. The report should be self-contained and readable without additional context, since it will be committed to git for historical comparison across releases.

   **Title**: `# OpenShift <release> GA Regression Report`

   **Release Lifecycle Summary**
   - A short table summarizing regression activity across the entire release lifecycle (not just those open on the report date). This enables cross-release comparison. The first row shows the report date and its source:

   | Metric | Value |
   |--------|-------|
   | Report Date | YYYY-MM-DD (source: GA / Final RC) |
   | Qualified Jobs | (from step 5) |
   | Total Regressions | (from step 4 summary.total) |
   | Unique JIRA Bugs | (from step 4 summary.triaged) |
   | Avg Time to Triage | (from step 4 summary.time_to_triage_hrs_avg, converted to days) |
   | P50 Time to Triage | (from step 6 P50 computation, converted to days) |
   | P95 Time to Triage | (from step 6 P95 computation, converted to days) |
   | Avg Time to Resolve | (from step 4 summary.time_to_resolve_hrs_avg, converted to days) |
   | P50 Time to Resolve | (from step 6 P50 computation, converted to days) |
   | P95 Time to Resolve | (from step 6 P95 computation, converted to days) |

   **Comparison with Prior Release** (only if prior release report was found)
   - A table comparing key metrics between the current and prior release side by side, with a delta column showing improvement or regression:

   | Metric | Prior (X.Y) | Current (X.Z) | Delta |
   |--------|-------------|---------------|-------|
   | Qualified Jobs | ... | ... | +/- N% |
   | Total Regressions (lifecycle) | ... | ... | +/- N% |
   | Unique JIRA Bugs (lifecycle) | ... | ... | +/- N% |
   | Avg Time to Triage | ... | ... | +/- N% |
   | P50 Time to Triage | ... | ... | +/- N% |
   | P95 Time to Triage | ... | ... | +/- N% |
   | Avg Time to Resolve | ... | ... | +/- N% |
   | P50 Time to Resolve | ... | ... | +/- N% |
   | P95 Time to Resolve | ... | ... | +/- N% |
   | Regressions Open | ... | ... | +/- N% |
   | Untriaged | ... | ... | +/- N |
   | Triage % | ... | ... | +/- N% |
   | SBARs | ... | ... | +/- N% |
   | SBARs per 100 Jobs | ... | ... | +/- N% |

   Compute "SBARs per 100 Jobs" as `(SBAR count / Qualified Jobs) * 100` for both releases. If the current release's SBARs-per-100-jobs ratio is more than 25% higher than the prior release, add a bold callout paragraph after the table noting that the SBAR rate relative to job coverage has increased significantly. This may indicate growing product quality risk, or it may reflect expanded test coverage surfacing more real issues — flag it for investigation either way.

   **Regressions Open on Report Date**
   - Release and report date (with source label)
   - Total regressions open on report date (before filtering)
   - Excluded: infrastructure-only regressions (ci-infra/product-infra)
   - Excluded: short-lived regressions (open < `--min-days` days)
   - Remaining regressions after filtering
   - Triaged count and percentage
   - Untriaged count and percentage
   - Number of unique bugs
   - SBAR coverage: how many triaged bugs have an SBAR (approved, candidate, or linked)

   **Triaged Bugs**
   - Table of unique JIRA bugs with columns: key (as a markdown link to the bug URL), title, triage type, SBAR status, release note (Y/N), how many regressions each covers, and for SBAR'd bugs a "Days to Fix" column showing the number of days from when the earliest triage record referencing this bug was created (`opened` timestamp of the regression) to the report date — this highlights how long the team had to fix the issue before shipping
   - The SBAR status column text (Approved/Candidate/Linked) should be a markdown link to the SBAR Google Doc when available. Show "None" as plain text when no SBAR exists.

   **Untriaged Regressions**
   - Table with regression ID, component, test name, variants, opened date, days open on report date, and current status (still open or closed date)

   **Additional SBAR Exceptions** (only if step 13 found any)
   - Introductory text: "The following bugs have SBAR labels and `affectedVersion` matching this release but were not statistically showing as Component Readiness regressions on the report date. These represent issues identified and escalated during the release cycle that were either resolved before the report date, intermittent, or not covered by the CR statistical model."
   - Table with columns: Bug (linked key), Title, Status, SBAR Status, Fix Versions
   - If no additional SBAR exceptions were found (after filtering), omit this section entirely.
   - After the table (and any exclusion notes), include a bold summary line: **Total SBARs for <release>: N** (X from triaged regressions + Y additional exceptions), with a comparison to the prior release total if the prior report was found (e.g., "down from M in <prior_release> (-Z%)").
   - The SBARs and SBARs per 100 Jobs rows in the Comparison with Prior Release table should reflect the **total** SBAR count: bugs with SBAR status from the Triaged Bugs table **plus** any Additional SBAR Exceptions. This gives a complete picture of SBAR activity for the release.

   **Prior Release SBAR Follow-up** (only if prior release report was found and had SBAR bugs)
   - Header noting this section tracks whether SBAR'd issues from the prior release were actually resolved
   - Table with columns: key (markdown link), title, SBAR status (from prior release), bug status (current), bug resolution, bug closed date, triage record status (open/closed), and an alert column
   - Any bug that is still open or has an open triage record should be marked with a prominent warning (e.g., "**UNRESOLVED**") in the alert column
   - After the table, include a summary count: how many prior SBAR bugs are fully resolved vs. how many remain unresolved
   - If any are unresolved, add a bold callout paragraph emphasizing that failing to resolve an SBAR'd regression within a full release cycle is a serious concern requiring immediate attention

   Format the report for easy reading with markdown tables.

17. **Write Report to File**: Write the markdown report to a file named `openshift/<release>-ga-regression-report.md` (in the `openshift/` subdirectory of the repository root). Inform the user of the file path.

18. **User Review**: Ask the user to review the generated report. Wait for their feedback. If they request changes, apply them and rewrite the file.

19. **QSE Comments**: After the user approves the report, ask if they have any additional commentary they would like to add. If they do, append a `## QSE Comments` section at the end of the file with their comments. If they have no comments, leave the report as-is.

## Return Value

- **Output file**: `openshift/<release>-ga-regression-report.md` in the repository root
- **Format**: Markdown report suitable for committing to git for historical analysis
- **Key sections**:
  - Release lifecycle summary (total regressions, unique bugs, avg/P50/P95 triage/resolve times)
  - Comparison with prior release (if prior report found in `openshift/` subdirectory)
  - Prior release SBAR follow-up with unresolved issue alerts
  - Regressions open on report date with filtering details
  - Triaged bug list with links and SBAR status
  - Untriaged regression details
  - Additional SBAR exceptions (JIRA bugs with SBAR labels affecting the release but not statistically showing in CR on the report date)
  - QSE Comments (if provided by the user)

## Examples

1. **Report for 4.21 using GA date**:
   ```
   /ga-regression-report 4.21
   ```

2. **Report for 4.22 using final RC date**:
   ```
   /ga-regression-report 4.22 --date 2026-06-15
   ```

3. **Report for 4.20 with 14-day minimum**:
   ```
   /ga-regression-report 4.20 --min-days 14
   ```

4. **Report for 4.20 including all durations**:
   ```
   /ga-regression-report 4.20 --min-days 0
   ```

## Arguments

- `$1` (required): OpenShift release version (e.g., `4.21`, `4.20`)
- `--date <YYYY-MM-DD>` (optional): Report date to evaluate regressions against. Typically the final RC date. If not provided, the GA date is used.
- `--min-days <days>` (optional): Minimum number of days a regression must have been open to be included. Default: `7`

## Prerequisites

1. **Python 3**: Required to run the skill scripts
2. **Network Access**: Must be able to reach the Sippy API at `sippy.dptools.openshift.org`
3. **JIRA Credentials** (optional but recommended): `JIRA_USERNAME` and `JIRA_API_TOKEN` environment variables for fetching bug titles. Without these, bug keys are shown without titles.

## Skills Used

- `get-release-dates` (teams plugin): Fetches GA date for the release
- `list-regressions` (teams plugin): Fetches regressions for the release view (e.g., `4.21-main`)
- `health-check-regressions` (teams plugin): Fetches release lifecycle summary metrics (via `list_regressions.py --short`)
- `fetch-jira-issue`: Fetches JIRA issue details including summary/title

## See Also

- Related Command: `/ci:analyze-regression` - Deep analysis of a single regression
- Related Command: `/teams:list-regressions` - List all regressions for a release
- Related Command: `/teams:health-check-regressions` - Component health grading based on regressions
- Component Readiness: https://sippy-auth.dptools.openshift.org/sippy-ng/component_readiness/main
