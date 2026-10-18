# Service Account analytics: business validation and data-gap log

**Purpose:** Keep a concise, living record of observations, business-rule questions, source gaps, decisions and their impact on the Service Account digital self-service portal and wider CRM Account Management dashboard. This complements, but does not replace, the data owner's future Customer Account Management Datamart enhancement request.

**Working rules:** Every entry states source/evidence, measurement impact, question, owner to consult, and status. Do not turn an unvalidated interpretation into a KPI definition. Reuse existing Datamart logic when validated; supplement with refined/source views during development (Option A). No production object changes authorised.

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
| SA-AM-02 | 32 Organisation accounts are ASSISTED and dated for enablement, but `customer_portal = false`; one Non-Statutory Person account is `customer_portal = true` with NEVER-ENABLED and no date. | Is this intentional eligibility/portal behaviour, legacy migration, an ETL-rule difference or data-quality issue? What is the authoritative portal-enabled definition? | Keep flag, onboarding and date as separate fields, surface reconciliation QA, do not silently auto-correct. | Open: business + data |
| SA-AM-03 | Datamart `new_or_existing` populated for ASSISTED rows; original Power BI classified NEW/EXISTING from account creation versus first portal enablement date. | Does the Datamart classification follow the same rule, including same-day cases and missing dates? Is it authoritative? | Run cross-tab comparing stored classification to date-based classification (SQL previously supplied), then inspect exceptions. | Open: SQL test |
| SA-AM-04 | Earlier Power BI portal KPI counted distinct nonblank `User_Id` for non-staff person accounts with portal enablement date. Datamart discovery is at `account_id` grain. | Is one CRM Account equivalent to one eligible portal User? How are multi-user accounts, staff, organisation and shared accounts represented? | Verify source user/identity IDs, uniqueness and account-to-user cardinality before adopting account counts as portal-user counts. | Open: data + business |
| SA-AM-05 | Current `vwaccount` snapshot covers account creation dates back to 2017 and current portal status; no historical snapshots proven. | For quarterly reporting, is the requested KPI quarter-end **account stock**, accounts **created during quarter**, portal accounts **enabled during quarter**, or all three? Can historical active/enabled populations be reconstructed? | Establish independent date anchors and status history before quarter-end adoption rates or forecasts. | Open: business + data |
| SA-AM-06 | Datamart is the existing gold reporting layer, target Option A. Detailed event history has been investigated in Salesforce source views. | Which new event facts / joins should DataHub add versus which existing gold definitions already meet need? | Continue dashboard read-only SQL build; request targeted Datamart enhancements after validation. | Decided approach, technical scope open |
| SA-AM-07 | CRM Account may not equal a unique natural person or organisation; duplicate/merge and relationships were not validated. | How should customers with multiple accounts, mergers or duplicates be represented in CRM-wide and portal adoption figures? | Report account-record counts distinctly from unique customer counts; validate relationship/source definitions before customer-level measures. | Open: business |

### Immediate next action

1. Execute the `new_or_existing` versus date-rule cross-tab; capture only aggregate outputs.
2. Confirm the intended CRM account inclusion rules and portal-enabled numerator/denominator with the CRM and Service Account owners.
3. Inspect refined/source account/identity mappings before defining `sa_crm_account` and `sa_portal_identity` as certified datasets.

## Other confirmed modelling gaps to track

| ID | Existing evidence | Impact / question | Status |
| --- | --- | --- | --- |
| SA-APP-01 | `vwpermit.period_start` is permit validity start, **not** application submission date; Salesforce BLA has `CreatedDate`, `Submitted__c`, and `AppliedDate` but latter may be populated on drafts. | Validate submission events before modelling completed journeys or quarterly submissions. | Open |
| SA-APP-02 | Application/BLA and Salesforce Case grains differ; some cases and applications have non-1:1 relationships. | Define canonical BLA-Case relationships and appropriate population/denominators. | Open |
| SA-FR-01 | FIR 1+ and repeat-FIR Case-based incidence reproduced in SQL; 24,745 eligible, 6,286 FIR and 695 repeat at last snapshot. | Keep Case-created cohorts and event-timed measures distinct. Recent cohorts are immature. | Reproduced, methodological caveats |
| SA-FR-02 | 7,141 FIR entries, 6,938 completed episodes; 203 without exits (171 current FIR, 32 other current status). Median completed episode 5.86 calendar days. | Duration is provisional and includes customer + internal time; investigate mismatched exits and censoring for later gold status-history enhancement. | Provisional |
| SA-SUP-01 | Available reviewed Genesys conversation sources cannot be deterministically joined to Salesforce Cases. | Do not claim per-application cross-channel repeat contact from inferred identities. | Unresolved linkage |

**Hygiene:** Append new IDs when a genuine new issue emerges. Update existing rows when validation resolves them. Record date, evidence and explicit decision; avoid separate parallel issue inventories.
