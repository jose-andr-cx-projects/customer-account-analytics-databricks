# IFD parking infringements — Databricks evidence and linkage backlog

**Logged:** 2026-10-31  
**Status:** Discovery evidence; financial join blocked; DataHub enhancement requested (not submitted to DataHub)  
**Project:** Intelligent Front Door (parking infringements)  
**Storage note:** This GitHub repository is a staging/reference copy. City of Melbourne's approved Bitbucket/Confluence/Jira records remain authoritative after review and publishing.

## Customer question
Of parking tickets issued, how many are paid, enter infringement review, enter driver nomination, or enter more than one pathway? Payment is a financial outcome; review and nomination are customer actions, so they are not mutually exclusive.

## Validated observations supplied from Databricks
- **PEMS issuance**: four views `datahub_refined_source.pems.vwmccp`, `vwmccb`, `vwmccda`, `vwmccl`. 958,251 distinct ticket numbers across full available history (2022-03-28 to 2026-07-28), almost no cross-view overlap. Duplicated rows require deduplication by ticket number. Status is unsuitable for lifecycle/payment classification: often null, codes undocumented, and some columns appear shifted. Quarterly refresh is a *hypothesis*, not confirmed.
- **Fixed issued cohort**: 310,302 distinct tickets issued 2025-04-10 through 2026-07-28.
- **Salesforce nominations**: `datahub_refined.customer.vwcase`, `Service_Name = 'Parking Infringement Nomination'`; do **not** filter `Case_Type='Nomination'` (infringement cases have `Case_Type='Customer Case'`). Across all available history, 129,879 nomination cases, 59,454 (45.8%) with infringement number, 55,204 distinct nominated infringement numbers, 39,159 matching PEMS, 16,045 unmatched. Related Resolved/Expert Enquiry/Transferred services excluded from this core definition.
- **90-day nomination proxy**: of the 310,302 issued tickets, 13,955 have a linked Salesforce nomination opened within 90 days: **4.50% observed**. This is not the true nomination rate: missing case infringement numbers and non-Salesforce nomination pathways mean undercoverage. July 1–28 2026: 18,757 issued, 819 nominated (4.37%). The 90-day window has matured by 2026-10-31. Monthly rates previously observed mostly 4.3–4.9%. A reported 'typical' 47-day delay needs median/mean definition confirmed.
- **Pathway legacy infringement CRM**: `datahub_refined_source.pathway.vwlifotin` contains 7.8M offence rows with `TFKLIFINFR` internal parent ID, `AMOUNT`, `BALANCE`, `PAYBYDATE`, late fees and `TFKLIFISTS` status key. `vwlifists` provides status descriptions. Zero balance is **not** proof of payment. `vwlifish1` and `vwlifstrt` share `TFKLIFINFR`, but do not expose external ticket number. Missing parent `LIFINFR`/equivalent header in available Pathway schema. `vwcrcallc.REFERENCE` is masked 'REDACTED'; do not request unmasking as first choice. PEMS `vwsession_transaction` is parking session payments, not infringement payments.
- **Current website**: pay, review and driver nomination are distinct entry paths. Review and nomination webforms ask whether the vehicle/fine is registered in the user's name; review eligibility guidance directs non-owner drivers to nomination/authorisation. Relationship-led routing is a design hypothesis supported by these observed rules, not a measured causal effect.

## DataHub / Databricks analytics connection backlog — DRAFT request
**Title:** Expose governed Pathway infringement-header mapping for PEMS ↔ Pathway ↔ Salesforce analysis  
**Priority:** Targeted dependency; does not block current IFD proxy analysis.  
**Ask:** Publish a minimally scoped authorised view of Pathway's infringement parent/header (e.g., source `LIFINFR` or equivalent) exposing (1) `TFKLIFINFR` stable internal infringement ID, (2) external infringement notice/ticket number, (3) issuance date and any necessary source-system identifiers, and (4) appropriate lifecycle fields and data definitions. Confirm parent/child grain, history, refresh cadence, masking and access rules. Do not request names, addresses or other personal data.
**Join validation acceptance criteria:**
1. Match external notice number to normalised PEMS `TicketNumber`, and independently to Salesforce `Infringement_Number`; report match rates by issuance month and source.
2. Check uniqueness and one-to-many relationships, duplicate/reissued notices, missing IDs, date compatibility and record completeness.
3. Link `TFKLIFINFR` to `vwlifotin` and `vwlifists`; verify statuses, voids, waivers, adjustments, instalments, receipts, refunds and write-offs before labelling 'paid'.
4. Validate Pathway refresh and PEMS apparent July 2026 cutoff with data owners.
5. Build a governed ticket-grain reporting view and reconcile cohort totals against the established 310,302 baseline.
**Fallback:** Keep Pathway financial analysis separate from PEMS/Salesforce until mapping exists; do not unmask `vwcrcallc.REFERENCE` without a separately justified and approved access request.
**Owner / ticket / due date:** Not assigned. **Submission:** Not yet submitted.

## Interim proxy analysis — next
1. Preserve fixed PEMS issued cohort (2025-04-10 to 2026-07-28).
2. Use Salesforce `Service_Name` to isolate infringement review and nomination; count distinct linked ticket numbers with valid case opened date, compare 90-day rates.
3. Produce **review only / nomination only / both / neither observed** at ticket grain; 'neither' is not 'paid'.
4. Audit missing infringement numbers, unmatched IDs, date ordering, review subtypes and potential multiple cases per ticket. Report all source coverage caveats.
5. Separately profile Pathway statuses and balances without claiming a PEMS cohort payment rate.

## Decision
Proceed with the best available **PEMS + Salesforce** proxy now; hold ticket-level paid-rate conclusions until the Pathway parent mapping is available and validated. No production Databricks sources modified.

## Follow-up: 90-day review vs nomination proxy (2026-10-31)

Databricks operator reran analysis using actual Salesforce review naming: `Service_Name LIKE 'Infringement Review - Parking%'`. The earlier exact value `Parking Infringement Review` is **not** a valid current Service_Name filter. Nomination filter remains `Service_Name = 'Parking Infringement Nomination'`.

For the same 310,302 PEMS tickets issued 2025-04-10 through 2026-07-28, a single run returned:

| Exclusive observed group | Distinct tickets | % issued |
| --- | ---: | ---: |
| Review only | 19,317 | 6.23% |
| Nomination only | 13,422 | 4.33% |
| Both | 499 | 0.16% |
| Neither observed | 277,064 | 89.29% |
| **Total** | **310,302** | **100% (rounding)** |

Derived: 19,816 tickets with review (6.39%); 13,921 with nomination (4.49%); 33,238 with either (10.71%); 499 / 13,921 = 3.58% of nominated tickets also reviewed. 'Neither observed' is **not** 'paid'.

**Volatility warning:** `vwcase` is being updated intraday, including historical infringement identifiers: 59,454 cases with infringement number earlier vs 59,564 later for cases opened before 2026-10-28. Nominated ticket counts varied between 13,955, 14,043 and 13,921 across runs. Do not combine denominators and numerators from different runs or present these figures as frozen. Save a consistent as-of snapshot or materialise the case cohort in an authorised workspace for reproducibility; capture query version, extraction timestamp, timezone and source freshness. This is a data-consistency need, not permission to change production sources.

**Next analytical action:** Inspect review service classifications and event sequence for the 499 overlapping tickets; measure completeness of `Infringement_Number` for review and nomination cases by month; repeat with a reproducible single-run snapshot. Continue to treat payment as unmeasured pending Pathway parent mapping.
