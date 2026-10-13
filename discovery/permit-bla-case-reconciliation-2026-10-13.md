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

## Follow-up: source BLA lifecycle timestamps and corrected date semantics (13 October 2026)

**Evidence level:** Genie Code reported read-only queries and successful execution, shared by the project lead. The exact final SQL, filters, timezone conversion, joins and underlying records have not been independently inspected here. Figures below are **reported findings, not certified metrics**.

- Source located: `datahub_refined_source.salesforce.vwbusinesslicenseapplication` (reported 200 columns); unlike `datahub_datamart.customer_account_management.vwpermit`, it reportedly exposes `CreatedDate`, `AppliedDate`, `LastModifiedDate`, `Application_status_timestamp__c`, `PaymentDate`, `ApprovedDate`, `Is_Draft_Application__c`, `Submitted__c`, and `Created_By_Employee__c`.
- **Prohibited substitution:** `vwpermit.period_start` and `period_end` describe permit validity, **never** application creation/submission. A prior query replacing three nonexistent `transaction_date` references with `period_start` gave an invalid transaction-time axis. Do not use that result.
- The three repaired references in the reported SQL all address **application creation activity**: SELECT date axis = BLA `CreatedDate`; WHERE non-null `period_start` condition removed because source `CreatedDate` was reported 100% populated in the examined population; GROUP BY = `CAST(bla.CreatedDate AS DATE)`. Verify timezone and source query before adopting day grouping.
- **Submission remains a candidate interpretation:** `Submitted__c = TRUE` together with `AppliedDate` may approximate formal submission; `AppliedDate` was reportedly prepopulated at creation even for Drafts. Do **not** use `AppliedDate` alone as a submission count, treat every matching CreatedDate/AppliedDate as proof of same-session submission, or assert the observed gaps establish all draft-to-submit histories.
- **Case fallback:** `vwcase.date_time_opened` matched source `CreatedDate` by calendar date for 99.9% of a reported linked primary-parking subset (56,483 of 57,335 BLAs linked; 852 unlinked). This is a **linked-Case-created proxy**, not a source BLA creation event and not a submission date. Do not generalise this result to all services or eras.
- Source timestamps: `Application_status_timestamp__c` reportedly covered 61.1% overall and does not establish complete status history. `ApprovedDate` reportedly had 0% coverage for parking; `PaymentDate` 63.0%. These are not universal journey anchors.
- Reported primary-parking, customer-created sample: 7,623 Draft/false Submitted records, 7,978 Issued/true Submitted and 58 Pending Payment/true Submitted. Of Issued, 87.9% had the same calendar creation/applied date and 12.1% differed (reported average 18 days, median 5 days in the latter group). Validate sample filters, event semantics and time units before any conversion/time-to-submit metric.
- A reported creation-date spike of 13,182 employee-created records on 29 March was interpreted as a migration batch. **Migration cause and year need confirmation** using metadata and operational evidence; do not label as organic customer portal starts or silently exclude them from all reporting.

### Provisional metric mapping

| Journey event / proposed metric | Candidate field(s) | Readiness / caveat |
| --- | --- | --- |
| BLA record created | Source BLA `CreatedDate` | Candidate for **application record creation**; portal/customer initiation requires channel/employee and migration exclusion validation |
| Current draft stock | Source `Is_Draft_Application__c`, `Submitted__c`, current status | Candidate snapshot only; not historical draft-created cohorts or abandonment |
| Submitted applications | `Submitted__c = TRUE` and `AppliedDate` | **Not yet certified** as submission date or event-history metric |
| Case creation | `vwcase.date_time_opened` | Case-level proxy, separately labelled; check linkage and population |
| Status progression / interruptions | Status history object or event log if available | **Unresolved**: current status timestamp is insufficient for full transitions |
| Permit validity | `vwpermit.period_start` / `period_end` | Validity metrics only; exclude from activity/submission dates |

### Follow-up validation

Before dashboard charts or Genie metrics: inspect the **actual corrected SQL**, join uniqueness, status-field definitions/automation, calendar timezone, and source-to-datamart reconciliation. Verify whether `AppliedDate` changes at submission with historical evidence (not current-row correlations). Confirm employee/migration treatment and product classification. Keep Genesys/Contact Centre systemic-friction investigation separate until reviewed. No governed source changes authorised.
