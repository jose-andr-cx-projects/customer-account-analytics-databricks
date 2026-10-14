# FIR by service — provisional Genie Code findings

**Status:** Reported read-only Genie Code results, supplied 14 October 2026. Not independently rerun; do not treat as approved KPIs or causal diagnosis.

## Cohort and headline

Reported customer-created, submitted Apply/Web population: **24,700 Cases** associated with **24,670 distinct BLAs**. **6,286 Cases** had at least one observed Further Information Requested (FIR) status transition (25.45%); **695 Cases** had at least two (2.81%). Reported top five services contribute **5,316 of 6,286 FIR Cases (84.6%)**. Count unique Cases with FIR rather than number of FIR events.

| Service | Cases | FIR 1+ Cases | FIR rate | FIR 2+ Cases | Repeat FIR rate |
| --- | ---: | ---: | ---: | ---: | ---: |
| Parking Permit | 9,673 | 2,116 | 21.9% | 85 | 0.9% |
| Space Occupancy Permit | 5,484 | 1,748 | 31.9% | 335 | 6.1% |
| Consent For Works Permit | 2,129 | 925 | 43.5% | 191 | 9.0% |
| Filming Permit | 426 | 319 | 74.9% | 8 | 1.9% |
| Out Of Hours On Private Land | 601 | 208 | 34.6% | 19 | 3.2% |

Reported service classification: Case `Service_Name__c` populated for all 24,700 eligible Cases, with 28 unique services; `Service_Type__c` reportedly NULL throughout this cohort. Check source service taxonomy authority before treating names as approved groupings.

## Excluded population reconciliation

From 49,223 reported Apply/Web non-Draft Cases: 24,700 customer-created and submitted; 22,626 employee-created and submitted; remainder **1,897**, reportedly comprised of 1,330 customer-created not submitted, 546 employee-created not submitted, and 21 with no BLA link. These are diagnostic categories, not evidence that all excluded records are non-portal or irrelevant to customer experience.

## Interpretation and constraints

- High FIR incidence **does not prove** unclear guidance or a failed portal. Some services may routinely request information as part of lawful assessment; validate workflow expectations.
- Service breakdown is a **secondary diagnostic filter**, not the dashboard's primary information architecture. Use five journey stages: Access & Identity; Understand & Prepare; Apply & Submit; Track & Respond; Outcome & Manage.
- Source event history reportedly starts **2025-04-10**; assess population-specific completeness, recent ingestion lag, follow-up maturity and whether the 2025 residential parking migration changes the denominator.
- The reported top-five total and individual figures reconcile arithmetically. Case-to-BLA cardinality is not precisely one-to-one (24,700 Cases vs 24,670 BLAs); future metric specification must state whether it is Case- or application-level.
- A service with high volume is not necessarily the one with greatest remediable effort. Measure repeats, wait time, reasons for requests, and actual communication quality before making recommendations.
- `vwpermit.period_start` is a **permit validity start**, not a submission date.

## Suggested bounded CX investigation

Prioritise **Space Occupancy Permit** and **Consent For Works Permit** as candidates for a joint Portal Analytics × CX Communication Governance assessment: they contribute 526 of the reported 695 repeat-FIR Cases and allow testing of the existing prototype `request-additional-information` communication standard against authorised real operational messages.

The communication prototype was validated on synthetic Salesforce templates, not production communication content. Do not infer cause or approve message changes without real template review, business-owner validation, and before/after evidence. Keep the governance capability in its own repository and keep organisational records in their authorised systems.

## Next decision

Discovery is sufficient to **begin dashboard design**, while treating FIR rates as provisional and gating publication on SQL reproduction, service/workflow validation and follow-up-window checks. Do not open another broad data-discovery workstream.
