# Portal draft application analytics — opportunity

Status: Candidate enhancement; feasibility and metric definitions unverified.

## Confirmed operational context

The self-service portal is built on Salesforce and customers can save draft applications (confirmed by project stakeholder, 13 October 2026). Salesforce Business License Application has specialised Customer DRAFT BLA and Portal Customer Booking BLA page layouts; page layouts alone do not prove lifecycle states or data retention.

## Opportunity

Track the application journey before submission, while preserving the Power BI baseline transaction definition. Candidate metrics:

- Drafts created over time (distinguish first creation from subsequent saves)
- Open or inactive drafts by age cohort
- Draft-to-submission conversion by service and cohort
- Elapsed time from draft creation to submission

Do not describe unfinished drafts as abandonment or service failure without time thresholds, workflow knowledge and validation.

## Discovery questions

1. What field/object represents application lifecycle (draft, submitted, withdrawn, etc.)?
2. Are drafts retained in BusinessLicenseApplication or a separate object?
3. Can one customer create multiple drafts for one intended application? What is the stable identifier?
4. Which timestamps identify creation, first save, submission and change history?
5. Are drafts and transitions available in the Databricks schema? Is historical state retained, or only current status?
6. Does the workflow differ for Permit applications and BLA?
7. Are deleted/expired drafts observable? What are their retention rules?
8. Can these data be aggregated without revealing applicant PII?

## Reporting guardrails

Retain separate reporting concepts: application started, draft saved, application submitted, internal processing case, and licence/permit outcome. Do not include drafts in submitted transaction denominators. Validate completeness and dates before publishing trends or conversion rates.
