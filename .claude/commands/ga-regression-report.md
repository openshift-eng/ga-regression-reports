---
description: Report regressions open on GA date for a release with triage status breakdown
argument-hint: <release>
---

## Name

ci:ga-regression-report

## Synopsis

```
/ga-regression-report <release> [--min-days <days>]
```

## Description

The `ga-regression-report` command generates a markdown report of Component Readiness regressions for a given OpenShift release, intended to be committed to git for historical analysis. By comparing reports across releases, we can track whether regression management is improving or degrading over time.

The report covers regressions that were still open on the GA date, filtering out infrastructure-only regressions (triaged as `ci-infra` or `product-infra`) and short-lived regressions (open fewer than `--min-days` days, default 7). It includes a high-level release lifecycle summary, triaged bug details with SBAR status, triage type validation, and untriaged regression details.

The command writes the report to a file, asks the user to review it, and gives them an opportunity to add QSE commentary before finalizing.

## Implementation

**Important: Avoiding user permission prompts when running scripts**

When calling Python skill scripts via the Bash tool, always run the script directly without piping the output through inline Python (`python3 -c "..."`). Complex piped commands trigger user permission prompts, while simple `python3 script.py args` calls are auto-approved.

1. **Parse Arguments**: Extract the release version from `$1` (e.g., `4.21`). Parse optional `--min-days <days>` flag (default: `7`). This controls the minimum number of days a regression must have been open to be included in the report.

   If no release is provided, prompt the user for one.

2. **Remove Existing Report**: If a file named `<release>-ga-regression-report.md` already exists in the `openshift/` subdirectory of the repository root, delete it before proceeding.

3. **Fetch GA Date**: Use the `get-release-dates` skill to determine the GA date for the release:

   ```bash
   release_dates=$(python3 plugins/teams/skills/get-release-dates/get_release_dates.py --release "$1")
   ```

   Extract the `ga` field from the JSON response. If `ga` is null, the release has not yet GA'd — inform the user and stop.

4. **Fetch Release Lifecycle Summary**: Use the `health-check-regressions` skill to get high-level regression metrics across the entire release lifecycle. Run `list_regressions.py` with the `--short` flag to get only summary statistics:

   ```bash
   python3 plugins/teams/skills/list-regressions/list_regressions.py --view "${release}-main" --short
   ```

   From the summary, extract:
   - `summary.total`: Total regressions across the release lifecycle
   - `summary.triaged`: Number triaged to JIRA bugs
   - `summary.time_to_triage_hrs_avg`: Average hours to triage
   - `summary.time_to_resolve_hrs_avg`: Average hours to resolve

   These metrics cover all regressions from development start through the report date, not just those open at GA. This provides context for cross-release comparison.

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

7. **Filter Regressions Open on GA Date**: From the full regression list, select regressions that were open on the GA date. A regression was open on GA if:

   - `opened <= <ga_date>T23:59:59Z` AND
   - `closed` is `null` (still open) OR `closed >= <ga_date>T00:00:00Z` (closed after GA)

   The `closed` field is either an ISO timestamp string or `null` (if the regression is still open). Note: `opened` reflects when the regression was first detected globally, while `closed` reflects when it was closed in the main view specifically.

8. **Filter Out Infrastructure Regressions**: Remove any regression where **all** triage entries have a `type` of `ci-infra` or `product-infra`. If a regression has at least one triage with a different type (e.g., `product` or `test`), keep it. Untriaged regressions (empty `triages` array) are kept.

9. **Filter Out Short-Lived Regressions**: Remove regressions that were open for fewer than `--min-days` days (default 7). Calculate the duration as the number of days from `opened` to the earlier of: `closed` (if not null) or the GA date. This filters out transient flakes that resolved quickly.

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

13. **Compute Untriaged Regression Details**: For each untriaged regression, gather:

   - `id`: Regression ID
   - `component`: Component name
   - `test_name`: Full test name
   - `variants`: List of affected variants
   - `opened`: When the regression was first detected
   - `closed`: When the regression was closed (if it has been)
   - **Duration open on GA**: Calculate the number of days from `opened` to the GA date. If still open, also note current duration from `opened` to today.

14. **Compare with Prior Release** (optional): Compute the prior release version by decrementing the minor version (e.g., `4.21` → `4.20`, `4.22` → `4.21`). If the current release minor version is `0` (e.g., `5.0`), the prior release is the last minor of the previous major (e.g., `4.22`). Look for a file named `<prior_release>-ga-regression-report.md` in the `openshift/` subdirectory of the repository root.

   If found, read the prior report and parse its markdown to extract key metrics from the Release Lifecycle Summary table and Regressions Open at GA section:

   - Qualified jobs
   - Total regressions (lifecycle)
   - Unique JIRA bugs (lifecycle)
   - Avg time to triage
   - Avg time to resolve
   - Regressions open at GA (after filtering)
   - Triage percentage at GA

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

15. **Generate Report**: Build a markdown report with the following sections. The report should be self-contained and readable without additional context, since it will be committed to git for historical comparison across releases.

   **Title**: `# OpenShift <release> GA Regression Report`

   **Release Lifecycle Summary**
   - A short table summarizing regression activity across the entire release lifecycle (not just GA-open regressions). This enables cross-release comparison:

   | Metric | Value |
   |--------|-------|
   | Qualified Jobs | (from step 5) |
   | Total Regressions | (from step 4 summary.total) |
   | Unique JIRA Bugs | (from step 4 summary.triaged) |
   | Avg Time to Triage | (from step 4 summary.time_to_triage_hrs_avg, converted to days) |
   | Avg Time to Resolve | (from step 4 summary.time_to_resolve_hrs_avg, converted to days) |

   **Comparison with Prior Release** (only if prior release report was found)
   - A table comparing key metrics between the current and prior release side by side, with a delta column showing improvement or regression:

   | Metric | Prior (X.Y) | Current (X.Z) | Delta |
   |--------|-------------|---------------|-------|
   | Qualified Jobs | ... | ... | +/- N% |
   | Total Regressions (lifecycle) | ... | ... | +/- N% |
   | Unique JIRA Bugs (lifecycle) | ... | ... | +/- N% |
   | Avg Time to Triage | ... | ... | +/- N% |
   | Avg Time to Resolve | ... | ... | +/- N% |
   | Regressions Open at GA | ... | ... | +/- N% |
   | Triage % at GA | ... | ... | +/- N% |
   | SBARs at GA | ... | ... | +/- N% |
   | SBARs per 100 Jobs | ... | ... | +/- N% |

   Compute "SBARs per 100 Jobs" as `(SBAR count / Qualified Jobs) * 100` for both releases. If the current release's SBARs-per-100-jobs ratio is more than 25% higher than the prior release, add a bold callout paragraph after the table noting that the SBAR rate relative to job coverage has increased significantly. This may indicate growing product quality risk, or it may reflect expanded test coverage surfacing more real issues — flag it for investigation either way.

   **Regressions Open at GA**
   - Release and GA date
   - Total regressions open on GA (before filtering)
   - Excluded: infrastructure-only regressions (ci-infra/product-infra)
   - Excluded: short-lived regressions (open < `--min-days` days)
   - Remaining regressions after filtering
   - Triaged count and percentage
   - Untriaged count and percentage
   - Number of unique bugs
   - SBAR coverage: how many triaged bugs have an SBAR (approved, candidate, or linked)

   **Triaged Bugs**
   - Table of unique JIRA bugs with columns: key (as a markdown link to the bug URL), title, triage type, SBAR status, release note (Y/N), and how many regressions each covers
   - The SBAR status column text (Approved/Candidate/Linked) should be a markdown link to the SBAR Google Doc when available. Show "None" as plain text when no SBAR exists.

   **Untriaged Regressions**
   - Table with regression ID, component, test name, variants, opened date, days open at GA, and current status (still open or closed date)

   **Prior Release SBAR Follow-up** (only if prior release report was found and had SBAR bugs)
   - Header noting this section tracks whether SBAR'd issues from the prior release were actually resolved
   - Table with columns: key (markdown link), title, SBAR status (from prior release), bug status (current), bug resolution, bug closed date, triage record status (open/closed), and an alert column
   - Any bug that is still open or has an open triage record should be marked with a prominent warning (e.g., "**UNRESOLVED**") in the alert column
   - After the table, include a summary count: how many prior SBAR bugs are fully resolved vs. how many remain unresolved
   - If any are unresolved, add a bold callout paragraph emphasizing that failing to resolve an SBAR'd regression within a full release cycle is a serious concern requiring immediate attention

   Format the report for easy reading with markdown tables.

16. **Write Report to File**: Write the markdown report to a file named `openshift/<release>-ga-regression-report.md` (in the `openshift/` subdirectory of the repository root). Inform the user of the file path.

17. **User Review**: Ask the user to review the generated report. Wait for their feedback. If they request changes, apply them and rewrite the file.

18. **QSE Comments**: After the user approves the report, ask if they have any additional commentary they would like to add. If they do, append a `## QSE Comments` section at the end of the file with their comments. If they have no comments, leave the report as-is.

## Return Value

- **Output file**: `openshift/<release>-ga-regression-report.md` in the repository root
- **Format**: Markdown report suitable for committing to git for historical analysis
- **Key sections**:
  - Release lifecycle summary (total regressions, unique bugs, avg triage/resolve times)
  - Comparison with prior release (if prior report found in `openshift/` subdirectory)
  - Prior release SBAR follow-up with unresolved issue alerts
  - Regressions open at GA with filtering details
  - Triaged bug list with links and SBAR status
  - Untriaged regression details
  - QSE Comments (if provided by the user)

## Examples

1. **Report for 4.21**:
   ```
   /ga-regression-report 4.21
   ```

2. **Report for 4.20 with 14-day minimum**:
   ```
   /ga-regression-report 4.20 --min-days 14
   ```

3. **Report for 4.20 including all durations**:
   ```
   /ga-regression-report 4.20 --min-days 0
   ```

## Arguments

- `$1` (required): OpenShift release version (e.g., `4.21`, `4.20`)
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
