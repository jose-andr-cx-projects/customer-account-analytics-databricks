# Customer Account Analytics — Databricks

## Purpose

Establish how customers experience City of Melbourne portal self-service — from account access through application start, progress, submission, outcomes and ongoing service management — using reliable, reusable analytics. This comprises:

1. A Databricks AI/BI dashboard initially reproducing the validated, Databricks-origin portion of the existing Service Accounts Power BI dashboard, then extending to measurable stages of the portal self-service journey.
2. A Genie Agent allowing authorised users to explore portal self-service and account management using the same validated business definitions, while distinguishing observations from unverified causes.

This is a **fresh implementation**. The previous Genie Agent is historical evidence, not a configuration baseline.

## Data scope

Initial catalog and schema: `datahub_datamart.customer_account_management`.

Inspect existing tables and views before deciding which to reuse or improve. Source-system records and governed data remain in Databricks and their authorised systems of record.

## Initial delivery sequence

1. **Baseline:** inventory Power BI measures, filters, DAX, Power Query lineage and relationships; identify Databricks-origin measures only.
2. **Audit:** inspect schema objects for purpose, grain, duplicates, joins, coverage, freshness, reliability and performance.
3. **Refine:** repair only confirmed problems in the appropriate table, view or metric layer, after reviewing change impact and permissions.
4. **Replicate:** create Databricks dashboard and reconcile measures against the Power BI reference for the same periods and populations.
5. **Query:** configure a Genie Agent against validated data/metrics, with instructions and tested example questions.
6. **Validate:** run numerical parity, question-answer, permission, caveat and regression checks before use.

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
- Genie reproduces validated figures for an agreed set of questions and explains limitations.
- Data/table changes are reviewed for downstream impact before deployment.

## Current status

**Discovery reported 13 October 2026; source mappings and Power BI parity not yet validated.** No Genie configuration or production deployment is represented as complete.

## Next action

Validate the Salesforce Permit application case vs BLA workflow using service configuration and Databricks schema metadata. Establish customer transaction versus internal processing case grain before changing measures or views.
