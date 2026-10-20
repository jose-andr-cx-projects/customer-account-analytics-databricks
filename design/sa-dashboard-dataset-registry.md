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

## Journey efficiency Q1 FY27 vs Q4 FY26 — observed results (20 October 2026)

Results of a user-executed read-only query reproducing the existing `sa_journey_efficiency` logic: BLA CDH history first observed transition into Issued; joined to `vwpermit` (`category = 'Permit'`); Case status history FIR transitions between application creation and observed Issued timestamp. Grouped by **Issued event quarter**, not creation quarter. All eligible Permit applications, **not limited to customer-created**. Creation timestamp remains provisional because the source SQL derives `MIN(CreatedDate)` on rows identifying Issued transitions; do not certify end-to-end duration before its semantics are verified.

| Quarter | FIR group | Issued applications | Median days creation to Issued | P75 days |
| --- | --- | ---: | ---: | ---: |
| Q4 FY26 | No FIR observed | 4,155 | 7.6 | 20.0 |
| Q4 FY26 | One FIR | 677 | 12.5 | 25.7 |
| Q4 FY26 | Repeat FIR | 91 | 21.7 | 39.5 |
| Q1 FY27 | No FIR observed | 4,166 | 7.1 | 21.7 |
| Q1 FY27 | One FIR | 614 | 13.5 | 25.3 |
| Q1 FY27 | Repeat FIR | 98 | 22.6 | 42.6 |

Derived from grouped results: Issued applications Q4 4,923 and Q1 4,878 (-0.9%); observed any FIR 768/4,923 = 15.6% in Q4 and 712/4,878 = 14.6% in Q1. In Q1 the repeat-FIR median is approximately 3.2 times the no-FIR median (22.6 vs 7.1 days). This is an **observed association**, not evidence of FIR causation. Medians/P75 values cannot be combined to derive the overall median; query application-level journey rows to obtain overall quarter medians.

Reporting guardrails: issued-quarter outcomes must not be described as outcomes for the same Q1 creation cohort, and permit-only/all-creator scope must be kept separate from customer-created BLA draft/start measures. The Issued-duration result excludes unfinished and non-Issued journeys. Compare similar service mixes and source coverage before interpreting small QoQ changes.

## Validated service-name mapping — portal enablement and applications (20 October 2026)

**Status: Validated for the tested datasets and populations; not a universally governed Salesforce service identifier.** Evidence: user-executed Databricks read-only reconciliation and uniqueness queries. Do not substitute Product2 ID for a service key.

### Join contract

1. Application: `datahub_datamart.customer_account_management.vwpermit.application_id` joins to BLA source `datahub_refined_source.salesforce.vwbusinesslicenseapplication.Name` where creation/employee flags are needed.
2. Case bridge: `vwpermit.case_id = datahub_datamart.customer_account_management.vwcase.case_id`.
3. Service catalogue: `UPPER(TRIM(vwcase.service_name)) = UPPER(TRIM(datahub_datamart.customer_account_management.vwservice_enablement.service_name))`.
4. Portal availability for an application: require application `CreatedDate >= first_portal_enable_date` for matched service. This establishes *service availability*, not proof of portal channel initiation.
5. Catalogue `service_group` supplies reporting groups (PARKING, BUSINESS, CONSTRUCTION, COMMUNITY); do not join on Salesforce Case `service_group`, which uses different strings (for example PARKING PERMIT versus PARKING, BUSINESS PERMIT versus BUSINESS, CONSTRUCTION PERMIT versus CONSTRUCTION, SPORTS PERMIT versus COMMUNITY, STREET TRADING versus BUSINESS).
6. `vwpermit.permit_type` contains Product2 IDs beginning `01t`; a direct join to the composite catalogue `service_key` had **0/33** distinct permit-type keys matched and is invalid.

### Observed checks

- The catalogue normalised `service_name` uniqueness test (multiple `service_key` or `service_group` values per normalised service name) returned **zero rows**. Validates one-to-one classification by normalised name **in the catalogue snapshot tested**, not eternal stability.
- Q1 FY27 customer-created BLA applications (`Created_By_Employee__c = false`, BLA CreatedDate 1 Jul–30 Sep 2026): **6,986** distinct applications; **6,474** linked to a Case and matched a catalogue service by name; **0** Case-linked unmatched; **6,474** matched service's recorded enable date at or before application creation; **512** with no Case link. Of those unlinked, **509** current Draft, **2** current Submitted with `Submitted__c = false`, **1** Withdrawn with that flag false. Unlinked does not mean no `vwpermit` application row.
- A broader Issued-status mapping diagnostic (`vwpermit.category = 'Permit'`, BLA CDH rows with Status='Issued' and `_effective_from` from 1 Apr to 30 Sep 2026) found **30,470** distinct applications, with **30,470 service matches** and **zero** missing Case, missing Case record or unmatched service.
- **Critical scope boundary:** the 30,470 diagnostic counts any application with qualifying Issued-status history rows, **not** only the first observed transition to Issued used in `sa_journey_efficiency`. The latter previously yielded **4,923 Q4 FY26 and 4,878 Q1 FY27** eligible issued journeys. Mapping must be retested/reconciled against that exact narrower journey set before calling its coverage 100%.

### Reuse and safeguards

- Normalise names with `UPPER(TRIM(...))`; use catalogue `service_group` as a classification *after* the unique-name match.
- Keep join diagnostics (missing Case, missing service match, multiple service names per Case, multiple application rows) when extending to new cohorts. The successful tests do not establish universal temporal uniqueness or immutable service names.
- When reporting 'applications lodged through self-service', employee-created flag and enabled service availability are **proxies**; neither proves a submission event/channel. Use historical submission semantics if the headline requires lodgement.
- Do not modify original Datamart/source views. Reuse this mapping in analytical SQL as justified; document any scope, date or case-join exclusions.

## Issued journey duration by mapped service group — Q1 FY27 comparison (20 October 2026)

**Status:** User-run read-only Databricks aggregate reconciled to existing `sa_journey_efficiency` totals. Reporting period is **first observed Issued transition quarter**. Population: `vwpermit.category = 'Permit'` across all creators/channels; this is not the customer-created-only cohort.

Uses the existing BLA CDH history first-Issued-transition method and the validated service-name mapping `vwpermit.case_id → vwcase.case_id → UPPER(TRIM(vwcase.service_name)) = UPPER(TRIM(vwservice_enablement.service_name))` to obtain catalogue `service_group`. The existing source-date and join-cardinality caveats remain; results are not certified universal end-to-end service durations.

| Service group | Q4 Issued | Q1 Issued | Q4 median days | Q1 median days | Q4 P75 days | Q1 P75 days |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| BUSINESS | 385 | 466 | 13.6 | 10.5 | 32.8 | 34.6 |
| CONSTRUCTION | 1,418 | 1,408 | 9.3 | 8.8 | 16.5 | 17.5 |
| PARKING | 3,120 | 3,004 | 7.2 | 6.7 | 22.2 | 25.5 |
| **Total count only** | **4,923** | **4,878** | — | — | — | — |

Both quarter totals reconcile exactly with the existing FIR-group duration comparison (4,923 Q4; 4,878 Q1). Do not calculate an overall median by averaging grouped medians.

**Interpretation:** Median creation-to-Issued duration decreased for each service group (Parking -0.5 day; Business -3.1 days; Construction -0.5 day), while group P75 duration increased (Parking +3.3 days; Business +1.8 days; Construction +1.0 day). The distribution tail warrants investigation; neither change proves a causal performance improvement/deterioration. Mix within groups, status history and process complexity remain possible confounders. The separate FIR comparison is across all services and is not an adjusted within-service effect.

**Next bounded step:** if needed for Q1 results, split FIR frequency *within* the mapped service groups using unchanged application-level journey definitions. Do not rework the base model unless a demonstrated gap blocks reporting.

## Service group × FIR frequency — Issued journeys Q1 FY27 vs Q4 FY26 (20 October 2026)

**Status:** User-executed read-only grouped result, reconciled with the existing `sa_journey_efficiency` Issued-quarter counts. Definition: BLA first observed Issued transition, `vwpermit.category = 'Permit'`, Case history FIR entry count between creation and Issued, using validated Case `service_name` to portal catalogue `service_group` mapping. All creators included; creation-time semantics remain provisional.

| Quarter | Service | FIR group | Issued | Median days | P75 days |
| --- | --- | --- | ---: | ---: | ---: |
| Q4 FY26 | BUSINESS | No FIR observed | 309 | 13.6 | 31.4 |
| Q4 FY26 | BUSINESS | One FIR | 73 | 13.4 | 38.4 |
| Q4 FY26 | BUSINESS | Repeat FIR | 3 | 14.5 | 74.6 |
| Q1 FY27 | BUSINESS | No FIR observed | 398 | 9.4 | 33.3 |
| Q1 FY27 | BUSINESS | One FIR | 64 | 13.5 | 36.3 |
| Q1 FY27 | BUSINESS | Repeat FIR | 4 | 61.4 | 118.7 |
| Q4 FY26 | CONSTRUCTION | No FIR observed | 932 | 7.6 | 12.5 |
| Q4 FY26 | CONSTRUCTION | One FIR | 403 | 13.7 | 27.6 |
| Q4 FY26 | CONSTRUCTION | Repeat FIR | 83 | 22.5 | 39.5 |
| Q1 FY27 | CONSTRUCTION | No FIR observed | 945 | 7.5 | 13.6 |
| Q1 FY27 | CONSTRUCTION | One FIR | 369 | 14.4 | 26.5 |
| Q1 FY27 | CONSTRUCTION | Repeat FIR | 94 | 22.5 | 36.4 |
| Q4 FY26 | PARKING | No FIR observed | 2,914 | 7.0 | 22.9 |
| Q4 FY26 | PARKING | One FIR | 201 | 9.3 | 17.2 |
| Q4 FY26 | PARKING | Repeat FIR | 5 | 21.7 | 25.7 |
| Q1 FY27 | PARKING | No FIR observed | 2,823 | 6.5 | 26.1 |
| Q1 FY27 | PARKING | One FIR | 181 | 10.4 | 21.2 |
| Q1 FY27 | PARKING | Repeat FIR | 0 | — | — |

**Reconciliation:** Q4 BUSINESS 385 + CONSTRUCTION 1,418 + PARKING 3,120 = 4,923; Q1 BUSINESS 466 + CONSTRUCTION 1,408 + PARKING 3,004 = 4,878. No observed Q1 Parking repeat-FIR row: zero in this Issued cohort, not evidence of zero recurrence among unfinished applications.

**Interpretation:** Construction shows a substantial graded association (Q1 medians 7.5, 14.4 and 22.5 days for 0, 1, 2+ FIR). Other drivers affect the duration tail: Parking without FIR had Q1 P75 26.1 days, while Business without FIR had 33.3. Business repeat-FIR Q1 n=4 and Q4 n=3 are too small to generalise. Any-FIR share among *Issued* applications changed: Parking 206/3,120 (6.6%) to 181/3,004 (6.0%); Construction 486/1,418 (34.3%) to 463/1,408 (32.9%); Business 76/385 (19.7%) to 68/466 (14.6%). Do not conflate with FIR incidence among all started applications or attribute causality to FIR. Median and P75 measure completed Issued journeys only; provisional application-created timestamp.
