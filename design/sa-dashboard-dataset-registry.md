# Service Account dashboard — dataset registry

Status: Working inventory captured 19 October 2026. Not a certification of SQL implementations.

## Source of truth

Databricks AI/BI dashboard SQL is authoritative for actual dataset definitions, fields and execution. This repository records implementation intent, confirmed observations, decisions and caveats. Follow [project README](../README.md), [dashboard visual standard](dashboard-visual-standard.md), and [business validation and data-gap log](../discovery/sa-business-validation-and-data-gaps.md). Do not create replacement measures solely because an existing dataset has not yet been inventoried here.

## Observed dashboard inventory

All ten dataset names below were visible in the Databricks **Data Model > Datasets** panel in a user-provided screenshot on 19 October 2026. Grain and metric definitions are recorded only when supported by separate evidence.

| Dataset | Known use / intended domain | Evidence and remaining detail |
| --- | --- | --- |
| `sa_process_interruptions` | Process interruption / FIR reporting | Named in project README; current SQL, fields and relationship to prior `cx_process_interruptions` require runtime inspection |
| `sa_crm_account` | CRM account population | Logical grain one CRM Account ID described in README; verify current SQL implementation |
| `sa_account_monthly` | CRM account creation, portal enablement and their cumulative populations | SQL supplied in conversation; one row per month × `account_record_type` in its final grouped output; fields recorded below |
| `sa_application` | Application lifecycle and outcomes | Logical grain one BusinessLicenseApplication described in README; verify current SQL implementation |
| `sa_application_monthly` | Monthly application activity | Referenced in project discovery log; current SQL and fields not yet captured |
| `sa_draft_friction_monthly` | Draft-related customer friction | Name observed; business definition, source and fields not yet captured |
| `sa_friction_monthly` | Monthly friction trends | Name observed; source, grain, fields and measures not yet captured |
| `sa_outcomes_monthly` | Monthly application outcomes | Name observed; source, grain, fields and measures not yet captured |
| `sa_service` | Service classification/filtering | Logical service-key component described in README; verify current SQL implementation |
| `sa_journey_efficiency` | Journey duration / efficiency | Name observed; source, grain, fields and measures not yet captured |

## Confirmed account-monthly outputs

The user supplied the `sa_account_monthly` SQL during dashboard construction. It derives event months from `datahub_datamart.customer_account_management.vwaccount`:

- Dimensions: `month_start`, `shifted_quarter_start`, `financial_year_end`, `financial_quarter`, `financial_period`, `account_record_type`.
- Monthly flows: `new_crm_accounts`, `new_portal_enabled`.
- Cumulative counts: `cumulative_crm_accounts`, `cumulative_portal_enabled` (windowed within account type).
- Precalculated ratio: `digital_adoption_pct` = 100 × cumulative portal-enabled / cumulative CRM accounts, by month and account type, with zero-denominator protection.

**Interpretation and aggregation safeguards:** The dataset reflects account records, not distinct people. The numerator uses `customer_portal = true` and a non-null first enablement date, and the denominator uses accounts with a creation date. Historical enablement rates are provisional because the flag is current-state based; existing organisation-account inconsistencies are documented in the gap log. Do **not** sum cumulative populations across months or average account-type percentages for an overall KPI. Aggregate compatible numerator and denominator populations for a single reporting month before deriving the whole-population share; this is visualization-grain handling, not permission to introduce redundant measures.

## Implemented visuals observed during current session

| Visual | Dataset and field | State |
| --- | --- | --- |
| New CRM accounts by month | `sa_account_monthly`, `SUM(new_crm_accounts)` by `month_start` | Built; half-width smooth line with hidden markers |
| New portal-enabled accounts by month | `sa_account_monthly`, `SUM(new_portal_enabled)` by `month_start` | Built; half-width smooth line with hidden markers |
| Portal-enabled share of CRM accounts | Candidate `sa_account_monthly` ratio; whole-population KPI | Planned; not claimed complete; must use one reporting month with correct cross-account-type aggregation |

The two trend visuals are configured to ignore one another's cross-filtering. The approved theme is at [service-account-dashboard-theme.json](service-account-dashboard-theme.json). A lightened line observed during editing was attributed to Databricks selection highlighting rather than a change to the theme.

## Remaining lightweight inventory work

When each existing dataset is next used, capture its SQL-derived **source, grain, date anchor, key measure fields, filters and caveats** here and map it to a dashboard visual. No wholesale revalidation and no new SQL by default. Check the existing dataset first; only propose additional logic for a demonstrated gap.


## Draft friction dataset — confirmed SQL review (20 October 2026)

**Runtime dataset:** `sa_draft_friction_monthly` (user-supplied SQL, not independently executed in this session).

- **Source and join:** `datahub_datamart.customer_account_management.vwpermit p` joined to `datahub_refined_source.salesforce.vwbusinesslicenseapplication b` on `p.application_id = b.Name`.
- **Population:** `p.application_status = 'Draft'`, valid `b.CreatedDate`, creation date on or before `CURRENT_DATE()`. This dataset is **not filtered** to customer-created records and must not be assumed to represent solely digital self-service customers.
- **Grain:** `cohort_month × category × permit_type`; cohort anchor is BLA `CreatedDate`.
- **Measures:** `current_drafts = COUNT(DISTINCT application_id)`; `drafts_older_than_30_days = COUNT_IF(draft_age_days > 30)`; `drafts_older_than_90_days = COUNT_IF(draft_age_days > 90)`; `median_draft_age_days = ROUND(PERCENTILE_APPROX(draft_age_days, 0.5),1)`.
- **Date semantics:** `draft_age_days = DATEDIFF(CURRENT_DATE(), CAST(b.CreatedDate AS DATE))` is elapsed **age of a currently Draft application**, *not* verified continuous time in Draft. It changes on each execution and cannot reconstruct a past quarter-end snapshot. Monthly grouping is creation cohort, not historical Draft count in that month.
- **Aggregation caution:** `COUNT_IF` assumes one joined row per application; verify join grain before treating these as distinct application counts. Recompute overall age medians from application-level rows; do not average monthly medians.
- **Q1 slide:** creation-cohort drafts can supplement activity, but current-state observations must have an explicit as-of date, must distinguish customer-created from all-application populations, and cannot be called abandonment.
- **Duration connection:** Application start = BLA `CreatedDate`; current unfinished age = creation to observation date; completed Issued duration lives in `sa_journey_efficiency` and uses first observed Issued transition. Submission event date remains unconfirmed. The two time measures describe different populations and must not be combined into a single duration statistic.

**Next validation (contained):** inspect data availability for a fixed observation date / historical snapshot and confirm application-level draft grain only if needed for the Q1 slide. No model SQL modifications have been made.

## BLA history quarter-end snapshots — observed test (20 October 2026)

Source: `datahub_refined_source.salesforce.vwbusinesslicenseapplication_cdh_history`, user-run read-only query. Historical version selection used `CreatedDate < cutoff`, `_effective_from < cutoff`, and (`_effective_to >= cutoff` or null); reporting cutoffs were 2026-07-01 and 2026-10-01 at 00:00:00. Use a half-open interval in later refinements (`_effective_to > cutoff`) after verifying version boundary semantics.

| Quarter end | Applications with valid history row | Multiple active versions | Status Draft | Submitted flag true |
| --- | ---: | ---: | ---: | ---: |
| 30 June 2026 | 121,013 | 0 | 28,492 | 85,592 |
| 30 September 2026 | 132,890 | 0 | 31,003 | 94,555 |

These are **all-application stock snapshots**, not customer-created quarterly cohorts; do not present changes as new quarterly draft or submission events. Version overlap was absent for matched application IDs, but **coverage against all eligible applications is not yet validated**. The submitted flag reflects the stored flag at cutoff, not a confirmed submission event date. Historical state reconstruction should use fixed cutoffs instead of the existing `sa_draft_friction_monthly` `CURRENT_DATE()` age for quarter-end comparisons.

Next contained test: filter to customer-created applications **created within each quarter**, select their active BLA history version at that quarter's cutoff, then reconcile counts against the previously observed Q4 6,774 and Q1 6,986 application-created populations. Check states and missing history before calculating quarter-end draft shares or age.

## Customer-created cohort quarter-end reconciliation (20 October 2026)

Read-only Databricks results supplied by project lead. Cohorts: BLA `CreatedDate` within Q4 FY26 (Apr–Jun 2026) and Q1 FY27 (Jul–Sep 2026), `Created_By_Employee__c = false`, joined to `vwpermit` by BLA Name/application_id. Historical version selected at each quarter's end via BLA Id and effective intervals.

| Measure | Q4 FY26 | Q1 FY27 |
| --- | ---: | ---: |
| Customer-created applications | 6,774 | 6,986 |
| Matched to historical version | 6,670 | 6,874 |
| Missing history at cutoff | 104 | 112 |
| Status Draft at quarter-end | 2,514 | 2,575 |
| Submitted flag true at quarter-end | 3,919 | 4,053 |

Both created counts reconcile exactly to earlier quarter-level application activity. Historical coverage is about 98.4% of each cohort, not complete. With **all starts** as denominator, draft share was 37.11% in Q4 and 36.86% in Q1 (approximately -0.25 percentage points); submitted flag share was 57.85% and 58.02% respectively (approximately +0.17 percentage points). These are quarter-end status proportions, not historical submission-event conversion rates. Missing-history records remain in denominators; do not silently classify them as not Draft or not Submitted. Flag is not a validated submission event. Draft and Submitted figures should not be summed as an exhaustive state classification. The data is not restricted through an independently validated portal-origin channel signal.

Next: investigate missing history and, if needed for the slide, age of Q1 drafts using *fixed* 2026-10-01 cutoff; connect to the existing `sa_journey_efficiency` Issued-duration measure without conflating all-applications and customer-created populations.
