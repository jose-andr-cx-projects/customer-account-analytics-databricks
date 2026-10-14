# FIR by service — SQL methodology and reproducibility

**Status:** Query text supplied by project lead 14 October 2026. **Not independently executed by this reviewer.** Databricks query-editor reference: https://adb-118033547174904.4.azuredatabricks.net/editor/queries/1305078146177422?o=118033547174904 . Confirm that the editor's saved text/version matches the SQL below and retrieve a successful execution timestamp/result for audit.

## Business question

Among customer-created, submitted portal-related applications with eligible observed Salesforce Case history, how many Cases entered Further Information Requested (FIR) status, how many did so more than once, and which services contribute the most volume?

## Source and grain

- `datahub_refined_source.salesforce.vwcase` — Salesforce Cases; `sc.Id` Case key, `Service_Name__c` grouping, `CreatedDate` cohort anchor, `Business_Permit_Application__c` BLA lookup, `Origin` and current `Status`.
- `datahub_refined_source.salesforce.vwrecordtype` — Case record-type lookup; `rt.Name = 'Apply'`.
- `datahub_refined_source.salesforce.vwbusinesslicenseapplication` — BLA; `bla.Id` application key, `Created_By_Employee__c` and `Submitted__c` as current flags.
- `datahub_refined_source.salesforce.vwcasehistory` — Case history events, filtered to `Field = 'Status'` and `NewValue = 'Further Information Requested'`.
- `vwpermit.period_start` is permit validity start, **not** a transaction or submission timestamp. It is intentionally not used.

Cohort: Case Origin = Web, record type = Apply, Case current Status != Draft, Case CreatedDate >= 2025-04-10, BLA employee-created flag false, BLA submitted flag true, and a linked BLA. This cohort is **Case-created-date anchored**, not BLA submission-date anchored. Current flags may differ from flags at the time of cohort creation.

## Metric formulas

- `fir_rate_pct`: distinct eligible Cases with at least one qualifying FIR status transition / distinct eligible Cases × 100.
- `repeat_fir_pct`: distinct eligible Cases with at least two qualifying FIR status transitions / distinct eligible Cases × 100.
- `fir_rate_bla_pct`: distinct linked BLAs with ≥1 FIR-bearing Case / distinct linked BLAs × 100.
- `fir_volume_rank`: rank by distinct FIR-bearing Cases, not count of messages or status events.
- `oct26_partial`: count of cohort rows created 2026-10-01 or later; diagnostic flag, **not exclusion**.

## Saved SQL text provided by project lead

```sql
/*  Portal Self-Service: FIR by Service
    Validated cohort: customer-created, submitted, Apply/Web, non-Draft
    CaseHistory observation: 2025-04-10 onward
    Oct 2026 partial — flag incomplete follow-up  */

WITH cohort AS (
  SELECT
    sc.Id            AS case_id,
    bla.Id           AS bla_id,
    sc.Service_Name__c AS service_name,
    sc.CreatedDate   AS case_created
  FROM datahub_refined_source.salesforce.vwcase sc
  JOIN datahub_refined_source.salesforce.vwrecordtype rt
       ON sc.RecordTypeId = rt.Id AND rt.Name = 'Apply'
  JOIN datahub_refined_source.salesforce.vwbusinesslicenseapplication bla
       ON sc.Business_Permit_Application__c = bla.Id
  WHERE sc.Origin   = 'Web'
    AND sc.Status   != 'Draft'
    AND sc.CreatedDate >= '2025-04-10'
    AND bla.Created_By_Employee__c = false
    AND bla.Submitted__c           = true
),

fir_once AS (
  SELECT CaseId, COUNT(*) AS fir_transitions
  FROM datahub_refined_source.salesforce.vwcasehistory
  WHERE Field    = 'Status'
    AND NewValue = 'Further Information Requested'
  GROUP BY CaseId
),

by_service AS (
  SELECT
    c.service_name,
    /* --- Case-level --- */
    COUNT(DISTINCT c.case_id)        AS cases,
    COUNT(DISTINCT c.bla_id)         AS applications,
    COUNT(DISTINCT CASE WHEN f.CaseId IS NOT NULL THEN c.case_id END) AS fir_1plus_cases,
    COUNT(DISTINCT CASE WHEN f.fir_transitions >= 2 THEN c.case_id END) AS fir_2plus_cases,
    /* --- Application-level --- */
    COUNT(DISTINCT CASE WHEN f.CaseId IS NOT NULL THEN c.bla_id END)  AS fir_1plus_blas,
    /* --- Incomplete-period flag (Oct 2026) --- */
    SUM(CASE WHEN c.case_created >= '2026-10-01' THEN 1 ELSE 0 END)  AS oct26_partial
  FROM cohort c
  LEFT JOIN fir_once f ON c.case_id = f.CaseId
  GROUP BY c.service_name
)

SELECT
  service_name,
  cases,
  applications,
  fir_1plus_cases,
  ROUND(100.0 * fir_1plus_cases / cases, 2)  AS fir_rate_pct,
  fir_2plus_cases,
  ROUND(100.0 * fir_2plus_cases / cases, 2)  AS repeat_fir_pct,
  fir_1plus_blas,
  ROUND(100.0 * fir_1plus_blas / applications, 2) AS fir_rate_bla_pct,
  oct26_partial,
  RANK() OVER (ORDER BY fir_1plus_cases DESC)     AS fir_volume_rank
FROM by_service
ORDER BY fir_1plus_cases DESC;
```

## Reported reconciliation targets (not rerun here)

| Metric | Genie-reported value |
| --- | ---: |
| Eligible Case total | 24,700 |
| Distinct eligible BLAs | 24,670 |
| Eligible Cases with FIR ≥1 | 6,286 |
| Eligible Cases with FIR ≥2 | 695 |
| Case-level FIR rate | 25.45% |
| Case-level repeat-FIR rate | 2.81% |
| Top five FIR Cases | 5,316 (84.6% of FIR Cases) |

Service-level table and caveats: [FIR service distribution](fir-service-distribution-2026-10-14.md).

## Reproducibility and decision caveats

1. **No event-history date predicate:** `fir_once` counts all available FIR transitions by CaseId, regardless of history CreatedDate; the cohort Cases are created after the observed start date, but verify timestamp boundaries, ingestion freshness and lack of pre-creation anomalies. Case creation date is not a submission event.
2. **Right-censoring:** recently created submitted Cases may not yet have had an opportunity to receive an FIR. `oct26_partial` is only a flag. A fixed follow-up window or mature cohort is needed for robust comparisons.
3. **Current-state selection:** `Status != 'Draft'` can exclude submitted BLA records if Case status has a different business meaning, while `Submitted__c` might reflect current rather than historical submission. Verify with workflow owner.
4. **Scope:** `Origin = 'Web'` and `Created_By_Employee__c = false` are operational cohort filters, not independent proof of customer portal session origin. Confirm supported service categories.
5. **Joined grain:** Source views must be checked for duplicate primary keys, uniqueness in record type, and Case-to-BLA multiplicity. Per-service distinct BLA totals may double-count BLAs if a BLA appears under multiple services.
6. **Communication vs workflow:** an FIR status transition is not proof an email was sent or received; repeat transitions are not necessarily repeated customer messages or clarity defects. Review actual communication events and workflow expectations before causal claims.
7. **Execution lineage:** direct editor URL supplied, but run ID, warehouse, last execution time, SQL revision and result export/checksum are not in this repository. Preserve those within authorised Databricks/organisational systems, not raw customer records here.
8. **Cohort comparability:** confirm whether calendar date, timezone, exclusions and recent refresh align across all reported tables before presenting the rates as production governed metrics.

## Leadership-safe wording

> A read-only Databricks analysis of a defined Salesforce Case/BLA cohort reported that 6,286 of 24,700 eligible customer-created, submitted Apply/Web Cases (25.45%) had at least one recorded Further Information Requested status transition. This is an observed processing signal—not proof that portal guidance caused the request. The metric remains subject to independent SQL-result reproduction, event-maturity checks and workflow interpretation.
