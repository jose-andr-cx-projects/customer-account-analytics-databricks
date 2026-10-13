# Databricks schema discovery — 13 October 2026

**Status:** Reported by Genie Code; not independently validated against source SQL, Power BI, or Salesforce configuration.

**Source:** Genie Code read-only discovery of `datahub_datamart.customer_account_management`, supplied in conversation 13 October 2026.

## Reported inventory

All eight objects were reported as masking views over corresponding objects in `datahub_datamart_eng.customer_account_management`. There are no base tables in the scoped schema.

| View | Reported rows | Reported grain | Initial decision |
| --- | ---: | --- | --- |
| `vwaccount` | 772,081 | account_id | Candidate for adoption and account mix |
| `vwcase` | 2,156,813 | case_id | Candidate case fact; validate workflow classification |
| `vwmanagedcase_casedetails` | 2,156,813 | case_number | Defer; meaning of snapshot/activity unresolved |
| `vwmanagedcase_contact` | 2,156,813 | case_id | Defer; PII and no confirmed unique metric |
| `vwpermit` | 134,365 | application_id | Candidate, not yet validated as Power BI transaction denominator |
| `vwservice_enablement` | 33 | service_key | Candidate reference mapping |
| `vwsupport` | 145,763 | case_id | Candidate ASK support source; confirm eligibility rules |
| `vwsupport_enriched` | 312 | source_id | Defer; inadequate demonstrated coverage |

## Reported concerns and verification requirements

1. **Transaction base uncertain:** `vwcase` has reportedly ~126K APPLY cases while `vwpermit` has 134,365 applications. Do not equate them or infer a one-to-one mapping. Determine distinct application/case lifecycle, Permit vs BLA configuration, account links and the exact Power BI denominator.
2. **Account attribution incomplete:** ~721K cases and ~56K support cases reportedly have null `account_id`. Confirm the denominator, the population/date scope, and any authorised alternative linkage before reporting account-level support.
3. **Masked-view and transformation boundary:** Schema views reportedly delegate to `datahub_datamart_eng`. Confirm underlying definitions and permissions before diagnosing performance or recommending source changes; no writes authorised.
4. **Unreliable categories:** `vwpermit.permit_type` reportedly contains identifier-like values and `category` includes malformed-looking JSON. Verify datatype, mapping authority and actual frequency of defects.
5. **Sparse enrichment:** 312 rows in `vwsupport_enriched`; no evidence of sufficient coverage for historical themed-support metrics.
6. **Potential duplicate/multi-stage cases:** Identical counts for the three case-oriented views do **not** establish 1:1 uniqueness or safe joins. Test distinct keys, nulls and join cardinality.
7. **Date semantics — explicit exclusion:** `vwpermit.period_start` is the **permit validity start**, not application submission, transaction, creation or first-draft date. The Genie Code substitution of nonexistent `transaction_date` with `period_start` may run successfully but is **semantically invalid for transaction reporting**; reject any resulting transaction trend or volume until rebuilt with a verified event date. Future validity starts (including 2027) may be legitimate and must not be automatically filtered out. Identify an authoritative submission event/timestamp from Salesforce application status and Applied Date metadata, verify its presence and meaning in curated Databricks sources, and reconcile it against the Power BI baseline before defining transaction metrics.
8. **Account onboarding reconciliation:** Report says 39,994 portal accounts versus 25,856 NEW + 14,169 EXISTING = 40,025; populations or filters are inconsistent (difference 31). Reconcile before using the split.
9. **Support source equivalence unproven:** `vwsupport` is reported as ASK subset of `vwcase`, but portal eligibility and service-name logic may differ. Compare matched case IDs and effective-date rules.
10. **Record type != workflow:** Do not assume every APPLY case is one portal transaction or use BLA suffix heuristics as canonical workflow classification. Validate against Salesforce service configuration.
11. **Permits category/source reliability:** Permit vs BLA mapping must be validated using service definitions, record types, cases/applications, process states and source-system ownership.

## Phase 1 decision

Proceed with **Power BI-to-Databricks metric reconciliation**, not dashboard build or Genie Agent configuration yet. Proposed minimum candidate views are `vwaccount`, `vwcase`, `vwsupport`, `vwservice_enablement`, and `vwpermit`. Candidate does not imply approved.

## Next contained validation task

Build a **read-only Permit vs BLA workflow and transaction-grain reconciliation**:

- Identify Salesforce service-configuration evidence/authoritative mappings for Permit and BLA.
- Inspect available columns and definitions in `vwcase`, `vwpermit`, and `vwservice_enablement` (and upstream metadata if authorised).
- Count distinct case IDs and application IDs by workflow classification, record type, period and service, reporting nulls, unclassified and ambiguous matches.
- Check whether case-to-permit relationships are 1:1, 1:many, many:1 or unmatched. Do not expose PII.
- Compare with current Power BI APPLY definition and document variances; if Power BI logic not supplied, label comparison pending.
- Produce recommendations only, without modifying objects.

**Guardrails:** read-only; bounded queries; respect masking and permissions; never export personal/customer-level data into GitHub; distinguish observation from assumption.
