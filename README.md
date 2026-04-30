# OpenShift GA Regression Reports

oakjsdh
This repository stores a report of the component readiness "regressions" that were unresolved at GA or final RC date. 

Because component readiness finds more than just product regressions (build
system, cloud accounts, cloud outages, test bugs, rate limiting, etc), we limit
the noise of the day to day by focusing only on regressions that were open for
at least 7 days. This is a strong representation of the product/test issues
that we failed to fix in the release.

We also will filter out regressions tied to a triage of type ci-infra or product-infra. 

## Setup

Install [APM](https://microsoft.github.io/apm/) and then install the required skills:

```bash
apm install --target claude
```

## The Process

1. During buildup to final RC, we already have a good idea of what bugs are suitable for SBAR (sbar-candidate label) and which feel too risky. This is communicated to the relevant teams and release management as early as possible.
1. At final RC date, we begin the process of getting all SBARs written and approved.
  1. We will continue to push for SBARS even for infra/test problems, unless the issue falls outside our org. 
1. When an SBAR is approved, we will link it in the relevant bug and switch to sbar-approved label. (dgoodwin will manage this for now)
1. One week before GA, we will run the report against what was open on the final RC date, with the current state of the bugs. (now with sbar-approved) 
Report filters ci-infra and product-infra regressions, not relevant to customers or teams performance. Test issues remain considered. These types are human best guesses when the triage record is filed, but the report will attempt to analyze the jira comments vs the triage type and report any discrepancies. 
1. Report will be posted in a PR to this repository and must be approved by Nick Stielau to merge. 
1. Cincinnati graph data repo will gain a presubmit that will detect when a new .0 is going to the live channel, it will verify the report for that release has been committed and merged to this repo.

