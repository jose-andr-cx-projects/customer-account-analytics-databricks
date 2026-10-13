# Customer Account Analytics — Databricks

## Purpose

Build a trustworthy, reusable analytics capability for City of Melbourne portal accounts and customer account management, comprising:

1. A Databricks AI/BI dashboard reproducing the validated, Databricks-origin portion of the existing Service Accounts Power BI dashboard.
2. A Genie Agent allowing authorised users to ask natural-language questions about portal accounts and account management using the same validated business definitions.

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
- Eligible portal support per 100 transactions, only if source and denominator rules reconcile
- Broader account management exploration once the first dashboard is validated

Do not presume all areas are available from the Databricks schema or in scope for the first release. Exclude GA4 and Genesys-origin measures from the initial replication unless separately approved.

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

**Initiated — discovery pending.** No Databricks schema audit, Power BI parity validation, Genie configuration or production deployment is represented as complete.

## Next action

Recover the latest Power BI model or template and run a read-only schema inventory of `datahub_datamart.customer_account_management` in Databricks Genie Code. Then prepare a scoped source-to-metric mapping before making changes.
