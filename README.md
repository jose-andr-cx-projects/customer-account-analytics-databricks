# Customer Account Analytics — Databricks

## Purpose

Establish how customers experience City of Melbourne portal self-service — from account access through application start, progress, submission, outcomes and ongoing service management — using reliable, reusable analytics. This comprises:

1. A Databricks AI/BI dashboard measuring the portal customer experience across Digital Access, Customer Activity, Customer Friction and Journey Outcomes. The former Power BI model is a reference for comparable measures, not a mandatory implementation baseline.
2. A Genie Agent allowing authorised users to explore portal self-service and account management using the same validated business definitions, while distinguishing observations from unverified causes.
3. A Customer Account Management Datamart enhancement specification for the data-owning team, documenting validated CX reporting gaps and reusable, governed improvements.

This is a **fresh implementation**. The previous Genie Agent is historical evidence, not a configuration baseline.

## Data scope

**Implementation decision, 15 October 2026:** Build the dashboard now using available read-only views in `datahub_refined` and, where needed, `datahub_refined_source.salesforce`. Do not wait for Datamart enhancements, and do not simply transplant older Power BI metric definitions. Source reads and dashboard SQL datasets are permitted; changes to production sources are not.

The dashboard's four areas are **Digital Access, Customer Activity, Customer Friction and Journey Outcomes**. Customer Friction has three investigation signals: **Process Interruptions** (FIR, repeat FIR, delays and lingering applications); **Access Barriers** (account/login and identity support); and **Recovery Effort** (repeat contact, rework and unresolved issues). These are domains for investigation, not a composite friction score or proof of root cause.

## Service Account model naming convention

**Decision, 19 October 2026:** Use the prefix `sa_` for every newly defined analytical model component and dashboard SQL dataset. **SA means Service Account**, the City of Melbourne **digital self-service portal**, not a Salesforce account object, service account credential, or the CX organisational branch. CX remains the business perspective and measurement objective, not a dataset prefix.

Proposed logical model components (to be validated before implementation):

| Dataset | Grain | Role |
| --- | --- | --- |
| `sa_application` | One BusinessLicenseApplication (BLA) | Creation, current lifecycle state, submission and outcomes |
| `sa_case` | One Salesforce Case | Application processing, enquiries and linkage |
| `sa_status_event` | One status transition | Progression, FIR and loops |
| `sa_fir_episode` | One observed FIR entry/exit episode | Duration and censoring flags |
| `sa_account` | One verified account or portal-user key | Digital access and onboarding |
| `sa_service` | One validated service key | Consistent service filtering |
| `sa_reporting_period` | One calendar/reporting period | Financial-quarter comparisons |
| `sa_process_interruptions` | Period × service aggregate, subject to final model | FIR and repeat-FIR rates |
| `sa_quarterly_performance` | Reporting quarter × validated dimensions | Quarterly performance measures |
| `sa_trend_forecast` | Forecast period × measure × model version | Predictions and uncertainty, after backtesting |

Do not rename original Databricks operational views. The dashboard dataset `cx_process_interruptions` already exists and is a **legacy draft artefact**: retain it and its dependencies until a validated `sa_process_interruptions` replacement is created and visuals migrated. No Databricks rename or dataset migration has occurred from this documentation change.

## Implementation sequence

1. **Model and validate:** establish customer experience questions, event grains, eligibility, timestamps, source mappings and coverage using refined/source views.
2. **Build:** create dashboard SQL datasets and visualisations now; verify weighted measures, reporting dates and denominators. Keep provisional signals clearly identified.
3. **Expand:** implement additional access, activity, friction and outcome measures as their data definitions pass targeted tests.
4. **Assess Datamart enhancements:** once the CX dashboard model is demonstrated, specify reusable governed facts, joins, transformations, quality tests and migration needs for DataHub owners.
5. **Align Genie:** reuse validated metric definitions and test Genie's question-answer behaviour; do not assert deployment before it happens.

## Candidate subject areas — validate against Power BI

- Enabled portal accounts and adoption over time
- Statutory and non-statutory account mix
- New versus existing customer onboarding
- Portal applications and transaction breakdowns
- Eligible portal support per 100 transactions, only if source and denominator rules reconcile; never use the retired `vwsupport_enriched` view
- Broader account management exploration once the first dashboard is validated

Do not presume all areas are available from the Databricks schema or in scope for the first release. Exclude Genesys-origin measures from the initial replication unless separately approved.

## GA4 on-demand behavioural evidence

GA4 analytics may be supplied on demand to complement governed Databricks/Salesforce business records. No standing GA4 ingestion, permanent data join or additional integration is required for the initial release.

- **Databricks/Salesforce**: authoritative application, account, case and outcome events.
- **GA4**: website engagement, navigation, device mix and funnel events, only where configured and evidenced.
- **Synthesis**: compare compatible service, timeframe and journey-stage slices. Keep separate denominators, identities and collection limitations visible.
- **Genie**: query governed Databricks metrics; contextual GA4 analysis can be reviewed alongside its answers. Do not imply Genie has access to GA4 unless it has been deliberately and securely integrated.
- Do not infer abandonment, user identity, cause or transaction conversion from page views or sessions alone.

Use GA4 as diagnostic evidence when a specific self-service question warrants it, avoiding ongoing manual reporting overhead.

## Retired analytical source

`vwsupport_enriched` is decommissioned **from this project's analytical scope** and must not be used in the dashboard, Genie Agent, metric definitions or discovery candidate list. This is a logical exclusion, not approval to drop the physical Databricks view. Confirm dependencies and ownership before any physical decommissioning.

## Salesforce Permit vs BLA workflow — discovery requirement

The rebuilt analytics model must distinguish **Permit** and **BLA** (Business Licensing Application) workflow behaviour using the Salesforce service-configuration findings. Do not assume these are interchangeable solely because an item is recorded as an APPLY case or has permit-related wording.

During read-only discovery:

- Recover the approved Salesforce service-configuration classifications, service names/groups, record types and workflow transitions that identify Permit versus BLA processes.
- Verify how each process creates or links Salesforce cases, applications, permits and customer accounts; check whether a case represents a transaction, an enquiry or an intermediate workflow step.
- Test the existing `vwpermit` and `vwcase` grain, linking keys and coverage for each workflow. Previous Power BI work required a permit-to-account bridge through `vwcase` because `vwpermit` did not expose a direct account ID; revalidate this against the live schema.
- Check how duplicate or multi-stage cases could inflate application counts, support rates and account activity.
- Produce a documented and testable classification before changing any source view or adding Permit/BLA dashboard splits.
- Compare historical APPLY/ASK logic and BLA-suffix heuristics with actual configured workflow behaviour; heuristics are not authoritative classifications.

**Status:** Requirement captured; Salesforce configuration evidence, mapping and resulting KPI effects still need validation.

## Datamart enhancement deliverable

The dashboard is **being built from refined/source views now**. After the model is established, a formal subsequent deliverable is a **Customer Account Management Datamart enhancement specification** for data owners to assess what should move to the governed layer. Datamart changes do not block current dashboard implementation. This is a request artefact, not approval to change production data.

For each proposed enhancement capture: customer experience question and journey/friction signal; existing Datamart view, fields and grain; specific missing capability; read-only evidence from refined/source layers; required fact, event, field or relationship and business logic; event date and eligible population; relevant completeness/privacy and lineage caveats; proposed priority; owner and approval status; and aggregate reconciliation/acceptance tests. Prefer reusable event facts and canonical mappings over separate hardcoded KPI columns.

**First candidate CA-01, application status-transition history:** Existing `datahub_datamart.customer_account_management` metadata exposes current Case/application status and lifecycle date fields but has no confirmed event-level FIR history. A search of accessible `datahub_refined` metadata for named Case/status-history views returned none. The Salesforce source `datahub_refined_source.salesforce.vwcasehistory` exposes candidate status transitions and timestamps. Propose a governed, Case-linked transition fact with application/BLA mapping, transition timestamp, old/new status and suitable service classification, subject to confirming grain, access, ordering, coverage, status semantics and join cardinality. Potential downstream indicators are FIR incidence, repeat FIR, FIR status duration and status loops. **Status: enhancement candidate, not approved.** Source episode duration was calculated subsequently, but remains provisional due to open episodes, current-status mismatches and episode-only duration coverage.

DataHub/data-owner review determines implementation scope, governance and timetable. Dashboard datasets can be built directly on refined/source views during this implementation phase; record source, grain, logic and evidence status, and assess later Datamart migration after the model stabilises. Do not use `vwsupport_enriched`.

## Source hierarchy

- **Official organisational sources and Databricks:** authoritative for governed operational data and live schemas.
- **Existing Power BI implementation:** reference for the validated dashboard's implemented measures and presentation, subject to source-quality and metric-definition checks.
- **This repository:** project decisions, validated metric definitions, safe SQL/model logic, test cases, findings and implementation documentation.
- **Strategic OS:** reusable interpretation and architecture patterns; not raw organisational data.

Never commit raw customer data, credentials, secrets, query result exports, sensitive screenshots or controlled documents. Use synthetic examples and metadata only where permitted.

## Completion criteria for first milestone

- Every replicated metric names its Databricks source, grain, filters, time logic and caveats.
- Same-population, same-period Power BI vs Databricks results reconcile, or discrepancies are documented and approved.
- Dashboard measures use reviewed views or governed metric definitions rather than duplicated business rules.
- Validated Datamart gaps are recorded as actionable, testable enhancement requests, with interim lower-layer dependencies visible until replaced.
- Genie reproduces validated figures for an agreed set of questions and explains limitations.
- Data/table changes are reviewed for downstream impact before deployment.

## Current status

**Dashboard build underway, 15 October 2026.** In Databricks AI/BI dashboard `Portal Self-Service Experience | CX Analytics`, page `Customer Experience Overview`, dataset `cx_process_interruptions` was created and executed successfully. It groups by Case creation month and Salesforce service name and contains eligible Case counts, FIR 1+ Case counts, FIR 2+ Case counts and rates. Current dataset also has a `median_fir_days` column that is a median of Case-level medians: **do not use this field as an overall FIR episode-duration KPI**.

Independent source-level SQL reproduced **24,745 eligible Cases**, **6,286 FIR Cases**, **695 repeat FIR Cases**, **25.40% FIR incidence** and **2.81% repeat incidence**. The Case-level calculation is available as dashboard custom measure `FIR Incidence %` with expression `100.0 * SUM(fir_cases) / NULLIF(SUM(eligible_cases), 0)`. User confirmed measure creation succeeded; its final rendered Counter value was not separately captured.

Separately, FIR status history checks reported 7,141 entry transitions, 6,939 exit transitions and zero missing timestamps; sequencing reported zero consecutive entry/exit anomalies and zero timestamp ties, plus one orphan exit. Episode calculation returned **6,938 completed**, **203 no observed exit**, **171 currently still FIR**, **32 status mismatches** (27 Closed, 5 In Progress), median completed episode **5.86 calendar days** and 75th percentile **18.98 days**. This duration is *provisional*: calculated per completed episode, not customer response time; open episodes excluded.

**Next action:** return to the draft dashboard (which autosaves). Finish configuring and visually checking the FIR Incidence Counter, then build the remaining validated Process Interruptions visuals. Do not publish the dashboard until the initial page and metric presentation are reviewed. Capture additional Datamart requirements only after model development, with CA-01 as the first candidate. No production source changes or Genie deployment claimed.
