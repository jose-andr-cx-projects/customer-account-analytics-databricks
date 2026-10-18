# Service Account analytics: business validation and data-gap log

**Purpose:** Keep a concise, living record of observations, business-rule questions, source gaps, decisions and their impact on the Service Account digital self-service portal and wider CRM Account Management dashboard. This complements, but does not replace, the data owner's future Customer Account Management Datamart enhancement request.

**Operating rule (19 October 2026):** The existing Power BI dashboard is operational, and its current Datamart-derived account calculations are an **accepted working baseline** for the new Service Account dashboard. Do **not** systematically revalidate or rederive these calculations in DAX or refined/source data. When a discrepancy, limitation or unresolved business interpretation emerges during new dashboard work, **record it explicitly here** with its evidence, affected measure, likely dashboard impact, and whether it blocks delivery. Continue with existing Datamart definitions unless the gap directly prevents a requested capability or requires a decision. Logging a gap does not automatically create a validation exercise or block development. Targeted assessment belongs in the future DataHub enhancement backlog, with an owner/decision only when necessary. Supplement existing Datamart views with read-only refined/source data when new analytical capabilities genuinely require it (Option A). No production changes authorised.

## Account management and portal enablement (19 October 2026)

**Source:** User-executed aggregate query against `datahub_datamart.customer_account_management.vwaccount`, grouped by `account_record_type`, `onboarding_type`, `new_or_existing` and `customer_portal`. These are snapshot values and have not yet been reconciled to the source view or historical logic.

**Arithmetic from returned groups:**
- CRM Account rows: **773,908** across five displayed account types: Non-Statutory Person **511,599**; Statutory Person **104,333**; Share **98,054**; Organisation **59,641**; generic Person **281**. The prior chat reply said 773,907 and Non-Statutory 511,598; those were arithmetic errors, not source discrepancies.
- `customer_portal = true`: **40,164** account rows, including **one** Non-Statutory Person row with `onboarding_type = NEVER-ENABLED` and no enablement date.
- Rows with `first_account_portal_on_date` present: **40,195**, comprising **40,163** true/ASSISTED person-account rows plus **32** Organisation rows with `customer_portal = false` and `onboarding_type = ASSISTED`. Thus enablement-date rows minus portal-flag rows = **31**, explained by 32 dated-but-false Organisation rows and 1 true-but-undated Non-Statutory Person row.
- All displayed groups had `account_rows = distinct_accounts`, but overall account-key uniqueness still requires a global distinct count.

| ID | Observation / issue | Business validation question | Dashboard impact / proposed next test | Status |
| --- | --- | --- | --- | --- |
| SA-AM-01 | Five CRM account types, including Organisation and Share Account, in the full snapshot. | Which types count in the **full CRM Account Management** population, and how should they be named/grouped? | Retain all source types initially; report type composition, avoid treating all Accounts as unique people. Validate owner definitions. | Open: business |
| SA-AM-02 | **Known gap in Organisation account coverage:** 32 Organisation accounts are classified ASSISTED (29 EXISTING, 3 NEW), have a portal enablement date, but `customer_portal = false`. Separately, one Non-Statutory Person account has `customer_portal = true`, NEVER-ENABLED and no enablement date. | Organisation portal enablement is inconsistently represented across the available fields. The cause and expected treatment are **not established**, and no further validation is requested at this stage. | Explicitly flag the Organisation account gap in dashboard model caveats and eventual Datamart enhancement scope; retain the fields as recorded and do not silently classify the 32 Organisation rows as portal-enabled. | **Documented gap; deferred** |
| SA-AM-03 | Datamart `new_or_existing` matches the Power BI date-based classification across **40,195 ASSISTED** accounts: **25,962 NEW→NEW** and **14,233 EXISTING→EXISTING**, with no mismatches or missing-date group returned. | Is the classification business-approved for all ASSISTED account types, including 32 Organisation rows with portal flag false? | **Reuse `new_or_existing`** in the `sa_` model for ASSISTED records without recalculating; retain separate enablement eligibility (SA-AM-02). | Calculation validated; business scope open |
| SA-AM-04 | Earlier Power BI portal KPI counted distinct nonblank `User_Id` for non-staff person accounts with portal enablement date. Datamart discovery is at `account_id` grain. | Is one CRM Account equivalent to one eligible portal User? How are multi-user accounts, staff, organisation and shared accounts represented? | Verify source user/identity IDs, uniqueness and account-to-user cardinality before adopting account counts as portal-user counts. | Open: data + business |
| SA-AM-05 | Current `vwaccount` snapshot covers account creation dates back to 2017 and current portal status; no historical snapshots proven. | For quarterly reporting, is the requested KPI quarter-end **account stock**, accounts **created during quarter**, portal accounts **enabled during quarter**, or all three? Can historical active/enabled populations be reconstructed? | Establish independent date anchors and status history before quarter-end adoption rates or forecasts. | Open: business + data |
| SA-AM-06 | Datamart is the existing gold reporting layer, target Option A. Detailed event history has been investigated in Salesforce source views. | Which new event facts / joins should DataHub add versus which existing gold definitions already meet need? | Continue dashboard read-only SQL build; request targeted Datamart enhancements after validation. | Decided approach, technical scope open |
| SA-AM-07 | CRM Account may not equal a unique natural person or organisation; duplicate/merge and relationships were not validated. | How should customers with multiple accounts, mergers or duplicates be represented in CRM-wide and portal adoption figures? | Report account-record counts distinctly from unique customer counts; validate relationship/source definitions before customer-level measures. | Open: business |

### Working next action

- Proceed with `sa_crm_account` and portal reporting using the existing operational Datamart calculations as a baseline; the NEW/EXISTING cross-tab already agreed for 40,195 ASSISTED accounts (25,962 NEW; 14,233 EXISTING).
- Retain the Organisation account inconsistency (SA-AM-02) as a documented, deferred gap; do not launch further validation merely because it exists.
- Add new gaps to this log as they are found. Investigate only when necessary for dashboard functionality, interpretation or a deliberate DataHub enhancement decision.

## Account-to-Case-to-Application schema discovery (19 October 2026)

**Evidence:** User-run `system.information_schema.columns` query on `datahub_datamart.customer_account_management.vwaccount`, `vwcase`, `vwpermit`, and `datahub_refined_source.salesforce.vwbusinesslicenseapplication`. This establishes field availability **only**, not referential integrity, match coverage, cardinality or business meaning.

- Datamart account → case candidate: `vwaccount.account_id = vwcase.account_id`.
- Datamart case → permit candidate: `vwcase.case_id = vwpermit.case_id`; `vwpermit.application_id` provides a separate application key.
- Salesforce BLA candidates: `AccountId`, `ApplicationCaseId`; `CoM_Account__c`, `Share_Account__c`, `Body_Corporate_Account__c` also exist and may carry different account roles.
- BLA `Submitted__c` and `Application_status_timestamp__c` exist, but field existence does not establish historical submission event semantics.
- Contact names, email and phone fields appear in the source; they are **not necessary** for the initial relationship map and should not be included in shared analytics datasets.

| ID | Observation / issue | Business / technical implication | Dashboard treatment | Status |
| --- | --- | --- | --- | --- |
| SA-REL-01 | Datamart `vwaccount.account_id`, `vwcase.account_id` and `vwpermit.case_id` support a possible Account → Case → Permit linkage. | Key availability alone does not guarantee a Case corresponds to a unique BLA or that every Case has an Account. | Use explicit keys and maintain Case/Application grains; do not create implicit one-to-one counts. | Candidate relationship; implement as needed |
| SA-REL-02 | Salesforce BLA exposes `AccountId` and `ApplicationCaseId` as well as `CoM_Account__c`, `Share_Account__c`, `Body_Corporate_Account__c`. | Account references may represent distinct business roles; choosing the wrong field could misattribute portal activity to customer segments. | Preserve role-specific fields; do not coalesce IDs into one assumed customer key. | Known semantic gap; deferred unless join needed |
| SA-REL-03 | Case-account association completeness remains a limitation noted in earlier discovery; account IDs can be null on Cases. | Joining Case activity to Accounts may omit activity and bias account-segment comparisons. | Make unmatched/unattributable Cases explicit when account-linked reporting is introduced. | Known coverage gap; deferred |

## BLA application classification discovery (19 October 2026)

**Evidence:** User-executed group by `ApplicationType`, `ApplicationCategory` on `datahub_refined_source.salesforce.vwbusinesslicenseapplication` (135,039 rows): New 94,838; Extend 33,085; Amend 7,114; null 2. `ApplicationCategory` is null throughout returned groups. These are record counts and current status flag distributions, not submission event counts.

| ID | Observation / issue | Impact / decision | Status |
| --- | --- | --- | --- |
| SA-APP-03 | Salesforce BLA `ApplicationType` is an action classification (New / Extend / Amend), not a service/permit taxonomy; `ApplicationCategory` has no populated values in the aggregate. | **Do not filter to `ApplicationType = 'Permit'`** or exclude amendments/extensions. Next inspect existing Datamart `vwpermit.category` and `permit_type` and service mapping before proposing any permit-only scope. | Classification gap recorded; contained source inspection |
| SA-APP-04 | BLA source has 135,039 rows in this run; prior `vwpermit` discovery saw 134,365 rows at another time. | Different snapshot dates and potentially mapping/filter differences; **do not treat the count difference as established missing records**. Log for later reconciliation if it affects dashboard scope. | Observation, deferred |

## Other confirmed modelling gaps to track

| ID | Existing evidence | Impact / question | Status |
| --- | --- | --- | --- |
| SA-APP-01 | `vwpermit.period_start` is permit validity start, **not** application submission date; Salesforce BLA has `CreatedDate`, `Submitted__c`, and `AppliedDate` but latter may be populated on drafts. | Validate submission events before modelling completed journeys or quarterly submissions. | Open |
| SA-APP-02 | Application/BLA and Salesforce Case grains differ; some cases and applications have non-1:1 relationships. | Define canonical BLA-Case relationships and appropriate population/denominators. | Open |
| SA-FR-01 | FIR 1+ and repeat-FIR Case-based incidence reproduced in SQL; 24,745 eligible, 6,286 FIR and 695 repeat at last snapshot. | Keep Case-created cohorts and event-timed measures distinct. Recent cohorts are immature. | Reproduced, methodological caveats |
| SA-FR-02 | 7,141 FIR entries, 6,938 completed episodes; 203 without exits (171 current FIR, 32 other current status). Median completed episode 5.86 calendar days. | Duration is provisional and includes customer + internal time; investigate mismatched exits and censoring for later gold status-history enhancement. | Provisional |
| SA-SUP-01 | Available reviewed Genesys conversation sources cannot be deterministically joined to Salesforce Cases. | Do not claim per-application cross-channel repeat contact from inferred identities. | Unresolved linkage |

**Hygiene:** Append new IDs when a genuine new issue emerges. Update existing rows when validation resolves them. Record date, evidence and explicit decision; avoid separate parallel issue inventories.
