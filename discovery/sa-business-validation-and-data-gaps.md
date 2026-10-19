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

### Datamart application category findings (19 October 2026)

User-executed grouping of `datahub_datamart.customer_account_management.vwpermit` by `category`: `Permit` holds **120,033** records and distinct `application_id` values across **33 distinct `permit_type` identifiers**. Other recorded categories include Skip Bins (4,532), Mobile plant equipment (1,825), Exclusion area/drop zone (1,192), Location of works (1,080), Waste Bin (884), crane/equipment/construction categories and two JSON-like category values (8 and 5 records). These category labels are source data, not a verified taxonomy of permit eligibility.

| ID | Observation / issue | Impact / decision | Status |
| --- | --- | --- | --- |
| SA-APP-05 | `vwpermit.category='Permit'` identifies a large, distinct subset (120,033) but many other categories may still be permit-related operational services. | Use category exactly as stored for an initial Permit-only **reporting filter**, **retain all categories in the base application model**, and avoid representing the subset as all permit-related CoM activity. | Reporting scope choice; taxonomy gap logged |
| SA-APP-06 | `vwpermit.category` contains JSON-like configuration strings (two observed groups of 8 and 5 records), alongside readable labels. | Expose as recorded, exclude from presentation-friendly grouped labels until a governed category mapping is available; no ad hoc manual recoding. | Known category quality gap; deferred |

| SA-APP-07 | The remaining `vwpermit.category` values beyond `Permit` include Skip Bins, mobile plant, construction equipment/works, location categories and malformed configuration values. Their end-to-end treatment in CRM workflows, Datamart classifications, portal journeys and reporting has not been assessed. | **Future investigation:** Establish how each non-`Permit` category is processed and represented (BLA/Case/permit workflows, application versus issued permit, portal versus staff-assisted activity, service mapping), and whether it belongs in self-service experience reporting or should remain a separate segment. Avoid assuming the current `category='Permit'` filter captures all permit services. | Open: deferred investigation; does not block current dashboard development |

### Application-to-BLA join failure (19 October 2026)

**Observed:** User-executed query joining `datahub_datamart.customer_account_management.vwpermit p` to `datahub_refined_source.salesforce.vwbusinesslicenseapplication b` on `p.application_id = b.Id` returned **135,039 Datamart rows, zero matched BLA rows, and 135,039 unmatched**. Consequently `sa_application_monthly` (with `b.CreatedDate IS NOT NULL`) was empty. Do not treat the assumed keys as equivalent or deploy the previous joined SQL unchanged.

| ID | Observation / issue | Dashboard impact / next action | Status |
| --- | --- | --- | --- |
**Follow-up result (19 October 2026):** Grouping BLA rows by `ApplicationCaseId` and joining to `vwpermit.case_id` returned 135,039 Datamart application rows; 8,428 with no Case ID; 126,611 matched; **zero** non-null Case IDs unmatched; 37 Datamart application rows have a Case matching more than one BLA (maximum 3). The 37 figure is affected **application rows**, not necessarily 37 unique Case IDs. This confirms the Case relationship has broad coverage but is not one-to-one. Do not use an arbitrary `ROW_NUMBER` choice to assign a BLA.

| SA-REL-04 | `vwpermit.application_id` is a BLA business reference and does **not** match Salesforce BLA `Id`. A direct join to Salesforce BLA `Name` matches all 135,039 records without duplication, including 8,428 without Case IDs. | Replace the incorrect SQL joins in both application datasets; preserve separate Salesforce record ID and business reference. | **Resolved: join verified in current snapshot** |

### Missing Case ID distribution by category and status (19 October 2026)

User-run aggregate over `datahub_datamart.customer_account_management.vwpermit` where `case_id IS NULL`: **8,428** applications; **8,321 (98.7%)** are `category='Permit'` with `application_status='Draft'`. Other Permit statuses: Lapsed 42, Submitted 37, Withdrawn 24, Declined 1 (104 in total). Non-Permit: one Draft record with a JSON-like category, one withdrawn Crane <150 ton, and one lapsed Mobile plant equipment (3 in total). All returned category/status groups had count equal to distinct application count.

**Interpretation boundary:** Strong association with drafts suggests Case creation may happen later in the workflow, but that mechanism is unconfirmed; missing Case ID does not prove missing Salesforce BLA. Do not label drafts as abandoned without elapsed-time/activity evidence. Preserve these records in the application foundation; avoid inner joins to Case for application population counts.

| ID | Observation / issue | Dashboard impact / next action | Status |
| --- | --- | --- | --- |
| SA-REL-05 | 8,321 of 8,428 missing-Case applications are Permit/Draft. 107 are not Permit/Draft, including 37 Permit/Submitted. | Account for drafts separately from Case-linked operational processing. Record non-draft exceptions as a known completeness gap; revisit only when lifecycle or linking functionality requires it. | Observed pattern; workflow cause unconfirmed, deferred |

### Draft as the start of the portal application journey (19 October 2026)

**Business clarification:** Draft is the expected **first status** for portal applications. Draft records are valuable early-journey evidence and a potential customer-friction signal; they must not be dropped merely because no Salesforce Case ID exists. The user confirmed this workflow interpretation. The observed concentration (8,321 Permit/Draft applications without Case IDs) is consistent with it but does not alone establish the moment when Cases are created.

| ID | Observation / business interpretation | Dashboard design implication | Status |
| --- | --- | --- | --- |
| SA-FR-03 | Draft is the initial application status and potentially signals friction at the start of the digital self-service journey. | Retain draft records in `sa_application`, including those with no Case ID. Distinguish **current draft stock**, **drafts created by month**, **draft age**, and **draft-to-submitted progression** (latter requires status transitions or reliable submission event dates). Potential signals include prolonged draft state and repeat draft activity; do not interpret draft status alone as abandonment or customer difficulty. | Business interpretation confirmed; measurement definitions/evidence pending |

**Analytical safeguard:** Snapshot `application_status='Draft'` is current state, not historical count of applications that entered Draft during a month. Application creation dates can anchor creation cohorts if linked correctly; transition timing and abandonment require further event evidence. Log missing history rather than inventing lifecycle timestamps. This insight belongs to the **Understand & Prepare / Apply & Submit** stages and the **Process Interruptions** domain.

### Application reference format clarified (19 October 2026)

User-shared sample of 30 Case-linked Permit records shows `vwpermit.application_id` is formatted `BLA-0000000044` etc., whereas Salesforce BLA `Id` is a Salesforce ID formatted `0f0...`. `vwpermit.case_id` and `BLA.ApplicationCaseId` match in the shown sample. `SavedApplicationRefId` is null in the sample. **Next candidate:** inspect Salesforce BLA `Name` (or other application-number field) for the `BLA-...` value and test match/cardinality before implementing a join. Draft records with populated Case IDs occur in the sample, so Case absence is not equivalent to Draft status.

### BLA join resolved by application reference (19 October 2026)

User-run grouped join test on `vwpermit.application_id = vwbusinesslicenseapplication.Name` returned **135,039 Datamart applications, 135,039 matched, zero unmatched, zero ambiguous reference matches, max one BLA row per reference**, including **all 8,428** records where `vwpermit.case_id IS NULL`. This establishes a complete unique current-snapshot linkage suitable for the `sa_application` foundation; `BLA.Id` is a Salesforce record ID, not the Datamart's `BLA-...` application reference. Keep these identifiers separate and retain drafts irrespective of Case availability.

**Resolution of SA-REL-04:** Join-key gap resolved in current snapshot by `p.application_id = b.Name`. The missing Case relation remains a distinct, documented workflow/coverage observation (SA-REL-05), not missing BLA records. Previous `sa_application` / `sa_application_monthly` SQL that joined by `b.Id` must be replaced.

## Journey Outcomes: current application status inventory (19 October 2026)

**Evidence:** User-executed `COUNT(DISTINCT application_id)` by `vwpermit.category` and `application_status`. Initial reporting scope `category='Permit'` totals **120,033** applications: Issued **70,646**, Draft **30,845**, Lapsed **8,036**, Withdrawn **5,246**, Declined **2,217**, Pending Payment **1,456**, Submitted **517**, Renewed **452**, Further Information Requested **350**, In Progress **229**, Extended **39**. These are **current status counts**, not historical status-change event counts.

| ID | Observation / question | Dashboard treatment | Status |
| --- | --- | --- | --- |
| SA-OUT-01 | Current statuses mix application processing (Draft, Submitted, In Progress, FIR, Pending Payment), recorded decisions (Issued, Declined, Withdrawn) and possible permit lifecycle statuses (Lapsed, Renewed, Extended). Their workflow semantics differ by service. | Initially display exact Datamart `application_status`; optional **provisional** status-group presentation only with explicit label. Do not classify Lapsed/Renewed/Extended as successes/failures without future business interpretation. | Known semantic gap; deferred |
| SA-OUT-02 | `vwpermit.application_status` is a current-state snapshot, not an outcome timestamp. | Do not show quarterly issued/declined/withdrawn **events** or historical approval rates by application creation month. Future capability needs validated status history / outcome dates and cohort maturity. | Known time-series gap; future Datamart enhancement |
| SA-OUT-03 | Status `Further Information Requested` appears in the current BLA/permit population, while prior FIR incidence was calculated from Salesforce CaseHistory status transitions. | Treat current FIR status stock and ever-had-FIR Case incidence as different signals and denominators. Never sum or equate them. | Measurement distinction documented |

### Journey outcomes: positive/negative and fastest Issued paths (19 October 2026)

**Business design direction:** Separate **application decision outcomes** from **permit continuation outcomes**, and examine the fastest low-interruption journey to Issued. These are reporting lenses, not replacements for source `application_status`.

- Application decision: `Issued` as positive; `Declined` as adverse; `Withdrawn` as discontinued/unissued, **not necessarily failed**. Permit-category current snapshot: Issued **70,646**, Declined **2,217**, Withdrawn **5,246**. Do not treat draft/processing status as decided.
- Permit continuation: `Renewed` (**452**) and `Extended` (**39**) as favourable continuation outcomes; `Lapsed` (**8,036**) as a potentially unfavourable or natural lifecycle ending. These are **current state** counts, not evidence of renewal eligibility, renewal events or failed renewal; do not calculate a success rate from these counts without a defined eligible population.
- Fastest successful journey: investigate application CreatedDate → *actual issuance event date*, duration distribution (median, p25), direct path with **no recorded FIR/rework**, and service-level differentiation. **No validated issuance timestamp has yet been established** in this model. Do not substitute `period_start` (permit validity start) or `Application_status_timestamp__c` (semantic unverified) for issuance time. Use CaseHistory/BLA event evidence where validated, and keep immature cohorts separate. Fast ≠ frictionless by itself.

| ID | Question / gap | Measurement impact / proposed treatment | Status |
| --- | --- | --- | --- |
| SA-OUT-04 | Does Declined versus Withdrawn represent negative decision versus customer-initiated discontinuation? | Keep both separately visible; do not combine into a binary failure rate without explanatory labels. | Provisional grouping; business context deferred |
| SA-OUT-05 | When is Lapsed natural expiry versus an unsuccessful expected renewal? Which permits are renewal/extension eligible? | Keep Lapsed separate and compare with Renewed/Extended as **status composition**, not a continuation-success rate. | Known denominator/semantic gap; deferred |
| SA-OUT-06 | What authoritative event identifies time when an application first becomes Issued? Can repeat/FIR/rework transitions be linked to BLA deterministically? | Required for time-to-Issued and direct/low-friction issued-journey metrics; status snapshot alone insufficient. | Future event-source enhancement; targeted investigation when implementing measure |
| SA-OUT-07 | How should fastest paths be benchmarked across services and creation cohorts? | Use appropriate time-to-issued distribution, filter to comparable service cohorts, examine FIR/rework independently; avoid declaring one fastest record representative or equivalent to frictionless. | Design decision; dependent on SA-OUT-06 |

### Issued event-source discovery (19 October 2026)

A user-run query of `datahub_refined_source.salesforce.vwcasehistory` joined to `vwpermit.case_id`, filtered for status history values containing `issu` and `category='Permit'`, returned **no rows**. This shows the tested CaseHistory path cannot currently supply an Issued transition; it does **not** establish that BLA issuance events do not exist. Next contained investigation: discover a Salesforce BLA/application-history table in refined/source, then evaluate event semantics before computing Created-to-Issued durations. Do not substitute permit validity start or unverified current-status timestamp.

### Candidate BLA history sources discovered (19 October 2026)

User-run `system.information_schema.tables` search found `datahub_refined_source.salesforce.vwbusinesslicenseapplication_cdh_history`, `vwpermit_stage__c_cdh_history`, and `vwbusinesslicense_cdh_history`, alongside Accela permit history views. These names establish source availability only; whether they contain status-change events, CDC snapshots, or reliable Issued timestamps is unverified. **Next scoped step:** inspect the BLA history view's columns and row examples before designing Created-to-Issued measures. Other history views are fallback candidates; no need for broad exploration yet.

### BLA CDH history schema inspection (19 October 2026)

User-shared complete column inventory for `datahub_refined_source.salesforce.vwbusinesslicenseapplication_cdh_history`. Relevant fields present: `Id`, `Name`, `Status`, `CreatedDate`, `ApprovedDate`, `Submitted__c`, `WasReturned`, `Application_status_timestamp__c`, `_effective_from`, `_effective_to`. These support a **candidate** version-based application timeline, but existence alone does not prove each status transition's timestamp corresponds to `_effective_from` or that history is complete. Next scoped test: count history versions and distinct statuses per BLA, then examine statuses and observed Issued version boundaries without using permit validity start. Keep `ApprovedDate` distinct from an actual issuance event until confirmed.

### BLA history coverage profiling (19 October 2026)

User-run aggregate on `datahub_refined_source.salesforce.vwbusinesslicenseapplication_cdh_history`: **135,730 distinct BLA IDs**, **433,650 version rows**, **110,619** BLAs with >1 version, **44,062** with >1 distinct observed status, **85,281** with at least one `Issued` version, and **2,802** with both `Draft` and `Issued` observed. `_effective_from` ranges **2025-02-17 13:04:46 UTC** through **2026-10-19 01:27:18 UTC** in this extraction. Multiple version rows do not necessarily indicate a status change. A small Draft→Issued observed overlap does not mean applications never passed through Draft: history may be incomplete for earlier transitions.

| ID | Observation / question | Model consequence | Status |
| --- | --- | --- | --- |
| SA-OUT-08 | BLA history records multiple historical versions and Issued statuses, but the reliability of the first observed Issued `_effective_from` as the actual issuance event date has not been established. | Compare each application's first Issued version to the preceding version/status, and examine first-history-row Issued and status timestamp relationships; do not calculate certified time-to-issued until interpretation is sound. | Evidence discovered; event timestamp validation pending |
| SA-OUT-09 | BLA history begins February 2025 in observed data; 85,281 applications have Issued observed, but only 2,802 have both Draft and Issued observed. | Clearly label left-censored cohorts and avoid treating first observed status as start of lifecycle; distinguish observed transitions from applications already Issued when history begins. | Known history coverage gap |

### First observed BLA Issued transitions (19 October 2026)

User-run status-version query on Salesforce BLA CDH history returned **85,281** applications ever observed Issued; **53,816** already Issued in their first recorded history version; **31,465** with an observed change from a non-Issued prior status to Issued. These groups sum to the ever-observed-Issued population in this query. The 53,816 first-version Issued applications are **left-censored**, not evidence of an issuance event at that first history date. The 31,465 transitions are promising for time-to-issued; confirm effective-from timestamp semantics and sequence ties before certification. Do not conflate ever-Issued historical population with currently Issued Datamart snapshot.

### Provisional creation-to-Issued duration benchmark (19 October 2026)

User-run read-only BLA CDH analysis linked first observed transition from a non-Issued version to Issued via `Name = vwpermit.application_id`, scoped to `vwpermit.category='Permit'`. Results: **30,884** observed Permit journeys; **0 negative durations**; p25 **3.44 days**, median **8.61 days**, p75 **22.97 days**, measured `CreatedDate` to first observed Issued `_effective_from` (calendar days). This is a **provisional benchmark for observed transitions**, not all issued permits. Historical left censoring, status-version ordering ties and event-time semantics are not yet certified. Interpret the percentiles only for this selected cohort; do not call these frictionless journeys without separate FIR/rework evidence.

### FIR and time-to-Issued association (19 October 2026)

**Evidence:** User-run exploratory SQL linked 30,884 Permit applications with observed BLA transitions into Issued to Salesforce CaseHistory FIR entries, grouped by total observed FIR count per Case. No FIR observed: **26,274** journeys, p25 **3.08**, median **7.61**, p75 **22.09** calendar days. One FIR: **4,032**, p25 **6.58**, median **12.61**, p75 **24.64**. Repeat FIR: **578**, p25 **14.41**, median **22.53**, p75 **39.75**. Median differences versus no-FIR are +5.00 days (one) and +14.92 days (repeat). The comparison is observational and not causal. The tested SQL counts all observed FIR transitions by Case without checking whether they occurred **before** the issuance timestamp; CaseHistory coverage is also limited and service mix may confound results.

| ID | Observation / question | Dashboard impact / next action | Status |
| --- | --- | --- | --- |
| SA-FR-04 | Larger FIR counts are associated with longer creation-to-Issued journeys in the 30,884 observed Permit issuance cohort. | Strong candidate for Customer Friction → Journey Outcomes. Keep distributions and sample sizes, avoid causal claims, and compare within permit service/type. For a production metric, restrict FIR to on/before each first observed issuance event and report unknown CaseHistory coverage explicitly. | Exploratory finding; temporal scope and service mix not yet aligned |

### FIR before issuance, compared within permit type (19 October 2026)

User-run diagnostic limits CaseHistory FIR transitions to `CreatedDate >= BLA.CreatedDate` and `<= first observed BLA Issued _effective_from`, then reports p25/median/p75 elapsed days by `vwpermit.permit_type` and FIR group, with at least 20 journeys per displayed group. The output covers 33 displayed combinations and uses opaque Salesforce Product2 identifiers pending a readable service mapping.

**Comparable groups with at least 100 journeys in both No-FIR and One-FIR:**

| Permit type ID suffix | No FIR n / median days | One FIR n / median days | Difference days |
| --- | ---: | ---: | ---: |
| GVjm | 344 / 4.53 | 134 / 5.69 | +1.16 |
| GVjp | 152 / 31.69 | 124 / 65.58 | +33.89 |
| GVjq | 617 / 10.58 | 159 / 28.45 | +17.87 |
| GVjr | 3,670 / 7.40 | 1,180 / 12.41 | +5.01 |
| Gtb0 | 1,396 / 7.76 | 594 / 14.50 | +6.74 |
| 1gUx | 17,117 / 6.96 | 1,362 / 9.50 | +2.54 |

**Repeat FIR observed groups:** GVjr 301 / median 22.43 days; Gtb0 166 / 22.37 days; 1gUx 49 / 21.71 days. Within each of these three types, repeat-FIR median exceeds one-FIR median, which exceeds no-FIR median. This is not a causal estimate. A separate type (`004eiEh`) has 37 No-FIR and 228 One-FIR observations and should not be compared as if the sample sizes were equivalent.

| ID | Finding / gap | Dashboard implication | Status |
| --- | --- | --- | --- |
| SA-FR-05 | Within comparable permit types, recorded FIR before issuance is associated with longer median application-to-Issued time; repeated FIR also aligns with longer durations in 3 types with reported repeat groups. | Strong candidate for service-level benchmark visual. Retain sample sizes, quartiles, permit-type cohorts and timestamps; guard against CaseHistory coverage bias, complexity and cohort differences. Resolve readable `permit_type` service names before visual publication. | Exploratory evidence captured; mapping and production methodology open |

## Other confirmed modelling gaps to track

| ID | Existing evidence | Impact / question | Status |
| --- | --- | --- | --- |
| SA-APP-01 | `vwpermit.period_start` is permit validity start, **not** application submission date; Salesforce BLA has `CreatedDate`, `Submitted__c`, and `AppliedDate` but latter may be populated on drafts. | Validate submission events before modelling completed journeys or quarterly submissions. | Open |
| SA-APP-02 | Application/BLA and Salesforce Case grains differ; some cases and applications have non-1:1 relationships. | Define canonical BLA-Case relationships and appropriate population/denominators. | Open |
| SA-FR-01 | FIR 1+ and repeat-FIR Case-based incidence reproduced in SQL; 24,745 eligible, 6,286 FIR and 695 repeat at last snapshot. | Keep Case-created cohorts and event-timed measures distinct. Recent cohorts are immature. | Reproduced, methodological caveats |
| SA-FR-02 | 7,141 FIR entries, 6,938 completed episodes; 203 without exits (171 current FIR, 32 other current status). Median completed episode 5.86 calendar days. | Duration is provisional and includes customer + internal time; investigate mismatched exits and censoring for later gold status-history enhancement. | Provisional |
| SA-SUP-01 | Available reviewed Genesys conversation sources cannot be deterministically joined to Salesforce Cases. | Do not claim per-application cross-channel repeat contact from inferred identities. | Unresolved linkage |

**Hygiene:** Append new IDs when a genuine new issue emerges. Update existing rows when validation resolves them. Record date, evidence and explicit decision; avoid separate parallel issue inventories.
