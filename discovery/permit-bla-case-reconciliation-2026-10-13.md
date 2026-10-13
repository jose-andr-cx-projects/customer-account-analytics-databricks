# Permit applications versus Salesforce Cases — reconciliation, 13 October 2026

**Status:** Provisional / read-only Genie Code findings. No Databricks source SQL, Salesforce runtime records, or Power BI parity independently validated. Do not deploy classifications as governed metrics.

## Main conclusion

Salesforce Object Manager independently confirms a standard `BusinessLicenseApplication` object and a Case lookup named `Business_Permit_Application__c`, which `BLA_Id__c` references through `Business_Permit_Application__r.Id`. The BLA page layout exposes Application Status, Application Case, Applied Date, Approved Date, Cases, Licence Activities, payments and reservations. Databricks Genie Code reports `vwpermit` at application grain and `vwcase` at Case grain, with linked and unlinked records.

**Distinguish:** application record, draft state, submission event, related Case, post-submission operational activities and licence outcome. The previous APPLY Case count remains the Power BI reconciliation baseline; it is not automatically the definitive measure of submitted applications.

## Genie Code reported aggregates

| Finding | Count |
| --- | ---: |
| `vwpermit` applications / distinct application_id | 134,365 |
| `vwpermit` applications with NULL case_id | 8,403 |
| NULL-case applications currently Draft | 8,296 |
| Distinct case_id values referenced by vwpermit | 125,943 |
| Linked application rows matched in vwcase | 125,962 (as reported; verify denominator) |
| Apply cases in vwcase | 126,106 |
| Apply cases absent from vwpermit | 197 |
| Cases linked to multiple BLA applications | 18 |
| Issued applications | 82,975 |

Counts are copied from the read-only Genie result for validation planning, not certified figures.

## Issues to resolve before using results

1. **Identifier mapping:** `application_id` reportedly looks like `BLA-...`, whereas the Salesforce `BusinessLicenseApplication` record also has a native Salesforce Id. Confirm which field is represented and whether `vwpermit.case_id` maps to the BLA Application Case or to any related Case.
2. **Join arithmetic:** 134,365 total minus 8,403 null-case rows = 125,962 nonnull application rows. 125,943 distinct cases implies 19 additional application rows, consistent with 17 doubles and one triple (17 + 2 = 19), but Genie also claims *18* multi-application Cases; verify exact grouped distribution. Distinguish application rows, distinct case IDs and matched joined rows.
3. **Submission state:** BLA `Application Status` and `Applied Date` are visible in Salesforce layout. They, not simply nonnull `case_id`, are candidate submission evidence. Confirm available field API names, values, underlying persistence and date semantics.
4. **Drafts:** 8,296 null-case Draft rows are not proven abandoned. Other statuses and draft records with linked Cases need investigation. Do not call unsubmitted or inactive automatically abandoned.
5. **Product lookup:** `01t` indicates Salesforce Product2 Id; not a Salesforce RecordType Id. Resolve Product2/service mapping with source metadata; do not assume single mapping from historic Case text.
6. **Channel:** Web and Face-to-Face are recorded channels, not guaranteed end-to-end customer submission paths. Confirm operational process and whether assisted transactions count in dashboard.
7. **Workflow complexity:** Salesforce service catalogue includes configured Permit Application / Referral / Inspection Workorder variants. Their actual Cases/activities may be absent from curated Databricks views; cannot conclude no internal workflow exists.
8. **Notifications:** `record_type='Apply'` may include Notification of Completion cases. Confirm inclusion or exclusion with Power BI and service owners, rather than blanket name matching.
9. **Historical analytics:** Current application status alone cannot reconstruct first draft date, submission date or transitions. Confirm BLA history, CreatedDate, Applied Date and extract/retention coverage.
10. **Access:** `datahub_datamart_eng` read reportedly blocked by USE CATALOG permission. Do not bypass or claim complete upstream lineage.
11. **Model integrity:** Distinguish which counts are case-level and application-level; verify all channel/service group totals sum to their appropriate populations. Audit source query and filter conditions before reuse.

## Agreed scope decision

`vwsupport_enriched` has been decommissioned from this project's analytical scope, not physically dropped.

## Next contained task

Using Salesforce setup, inspect **Business License Application → Application Status** source/values and the **Application Case** relationship. Confirm relevant identifiers, states, Applied Date and workflow transitions, then rerun one narrowly scoped Databricks reconciliation. No view/dashboard/agent writes until metric definitions pass parity testing.
