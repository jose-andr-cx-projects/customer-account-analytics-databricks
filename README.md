# Customer Account Analytics — Databricks

## Purpose

Establish how customers experience City of Melbourne portal self-service — from account access through application start, progress, submission, outcomes and ongoing service management — using reliable, reusable analytics. This comprises:

1. A Databricks AI/BI dashboard initially reproducing the validated, Databricks-origin portion of the existing Service Accounts Power BI dashboard, then extending to measurable stages of the portal self-service journey.
2. A Genie Agent allowing authorised users to explore portal self-service and account management using the same validated business definitions, while distinguishing observations from unverified causes.
3. A Customer Account Management Datamart enhancement specification for the data-owning team, documenting validated CX reporting gaps and reusable, governed improvements.

This is a **fresh implementation**. The previous Genie Agent is historical evidence, not a configuration baseline.

## Data scope

Initial catalog and schema: `datahub_datamart.customer_account_management`.

Prefer the highest suitable reusable layer, beginning with `datahub_datamart.customer_account_management`, then relevant `datahub_refined` views, then authorised source views such as `datahub_refined_source.salesforce`. Source suitability depends on event meaning, grain, completeness and quality, not layer name alone. Inspect existing tables and views before deciding which to reuse or improve. Source-system records and governed data remain in Databricks and their authorised systems of record.

## Initial delivery sequence

1. **Baseline:** inventory Power BI measures, filters, DAX, Power Query lineage and relationships; identify Databricks-origin measures only.
2. **Audit:** inspect schema objects for purpose, grain, duplicates, joins, coverage, freshness, reliability and performance.
3. **Specify gaps:** for each required CX indicator, record Datamart coverage, verified deficiencies, source-layer evidence, a proposed governed enhancement, owner decision and acceptance tests. Propose changes to data owners; do not modify shared sources as part of discovery.
4. **Implement:** use suitable governed Datamart objects where available; only use clearly labelled, read-only interim SQL datasets when essential, and track their intended replacement by Datamart enhancements. Reconcile comparable Power BI baselines without carrying over semantically invalid definitions.
5. **Query:** configure a Genie Agent against validated data/metrics, with instructions and tested example questions.
6. **Validate:** run numerical parity, question-answer, permission, caveat and regression checks before use. Submit enhancement specifications for data-owner review and rerun tests after governed changes are released.

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

The dashboard is intended to use the Customer Account Management Datamart as its trusted reporting layer. A second formal project output is a **Customer Account Management Datamart enhancement specification** for data owners. This is a request and validation artefact, not permission to change production data.

For each proposed enhancement capture: customer experience question and journey/friction signal; existing Datamart view, fields and grain; specific missing capability; read-only evidence from refined/source layers; required fact, event, field or relationship and business logic; event date and eligible population; relevant completeness/privacy and lineage caveats; proposed priority; owner and approval status; and aggregate reconciliation/acceptance tests. Prefer reusable event facts and canonical mappings over separate hardcoded KPI columns.

**First candidate CA-01, application status-transition history:** Existing `datahub_datamart.customer_account_management` metadata exposes current Case/application status and lifecycle date fields but has no confirmed event-level FIR history. A search of accessible `datahub_refined` metadata for named Case/status-history views returned none. The Salesforce source `datahub_refined_source.salesforce.vwcasehistory` exposes candidate status transitions and timestamps. Propose a governed, Case-linked transition fact with application/BLA mapping, transition timestamp, old/new status and suitable service classification, subject to confirming grain, access, ordering, coverage, status semantics and join cardinality. Potential downstream indicators are FIR incidence, repeat FIR, FIR status duration and status loops. **Status: enhancement candidate, not approved and not yet validated for episode-duration calculation.**

DataHub/data-owner review determines implementation scope, governance and timetable. Until a governed source is available, any interim query-based dashboard indicator must be marked with its evidence status, owner and retirement/migration condition. Do not use `vwsupport_enriched`.

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

**Discovery reported 13 October 2026; source mappings and Power BI parity not yet validated.** No Genie configuration or production deployment is represented as complete.

## Next action

Validate the Salesforce Permit application case vs BLA workflow using service configuration and Databricks schema metadata. Establish customer transaction versus internal processing case grain before changing measures or views.
