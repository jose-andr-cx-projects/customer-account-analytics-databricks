# Contact Centre sources and systemic friction — discovery review, 13 October 2026

**Status:** Provisional Genie Code schema discovery provided by project lead; source SQL, population coverage, joins and field value distributions not independently verified. Read-only investigation. Not approved metric specifications.

## Decision frame

The Portal Self-Service Experience initiative seeks to understand **where self-service works, where customers encounter friction and what CX improvements are needed**. Organise analysis by journey: **Access & Identity; Understand & Prepare; Apply & Submit; Track & Respond; Outcome & Manage**. Use **Clarity Gap; Access Barrier; Technical Failure; Process Complexity; Visibility Gap; Recovery Failure** as candidate interpretive categories. Product and service are secondary filters, not main dashboard sections.

Separate **directly measured customer effort**, **possible signals of friction** and **attributed causes**. A contact, transfer, unresolved Case or long-lived draft is not by itself evidence that the portal failed.

## Reported sources

- Genesys: `datahub_refined.customer.vwinteraction` (conversation plus participant grain reported; validate uniqueness), `vwtranscriptsummary` (conversation), `vwtranscript` (phrases). Conversation identifier and operational measures including duration, transfers, queue and wrap-up fields; transcripts may be protected.
- Salesforce: `datahub_refined.customer.vwcase` (Case grain claimed; inspect uniqueness), `datahub_refined_source.salesforce.vwcasehistory` (field-history event), `vwemailmessage` (message linked to Case), `vwcasecomment` (schema pending).
- Customer intelligence: `datahub_datamart.customer_intelligence.vwsurvey_feedback`, `vwtarget`; enterprise performance `vwfact_customer_service_performance` for monthly branch ageing summaries.
- Configuration: `datahub_refined.customer.vwservice_group` and `datahub_refined_source.salesforce.vwproduct2`.
- `datahub_datamart.customer_account_management.vwsupport_enriched` was rediscovered (reported 312 rows) but **remains excluded from this project's analytical scope**. Do not recommend scaling it or using it for a metric without a separate explicit scope decision.

Genie described 16 accessible objects but the named inventory and its access claims should be checked against catalog/query permissions. Catalog labels, grain and row coverage are provisional.

## Core structural limitation

No deterministic `conversationId`–Salesforce `Case_Id` linkage was identified in the **examined accessible views**. This is not evidence no upstream integration or authorised link exists. Phone-number / ANI plus timestamp proximity is an uncertain identity- and privacy-sensitive inference, **not a production join** or approved customer-level linkage.

Thus cross-channel "portal application led to call" or end-to-end recovery cannot be calculated defensibly now. Genesys conversation metrics and Salesforce Case metrics must retain their distinct denominators and populations.

## Measurement assessment

| Candidate signal | Source / grain | Proposed time anchor | Initial readiness | Interpretation safeguard |
| --- | --- | --- | --- | --- |
| Case resolution time | refined Salesforce `vwcase` / Case | `Date_Time_Opened` and closed timestamp | Testable, not yet validated | Case processing duration ≠ customer self-service delay |
| Case ageing and SLA | Case-level metrics or branch monthly snapshots | Case opened, or snapshot `period_date` | Testable with field/SLA value checks | Different entitlements and snapshot bases cannot be mixed |
| Transfers per conversation | Genesys `vwinteraction` / conversation | `conversationStart` | Testable after participant-row dedupe | Transfer count = handling effort, not proven CX process complexity |
| Abandons per inbound interaction | Genesys / conversation | `conversationStart` | Testable after short-abandon rules | Contact-centre queue abandon ≠ portal application abandonment |
| Salesforce status churn | `vwcasehistory` / history event per Case | History `CreatedDate` | Requires `Field` and transition profiling | Status changes ≠ unnecessary rework without expected workflow |
| Same-account repeat Cases | `vwcase` / account and Case | Case opened | Proxy only; reported ~33% missing account IDs | More than one Case is not automatically a repeat contact about same issue |
| Same-ANI repeated conversations | Genesys / conversation | `conversationStart` | Proxy; sensitive linkage review | Shared phone numbers; voice-only; no reliable Salesforce account mapping |
| Survey-reported contacts/ease | Case survey fields and survey feedback | Survey date | Requires coverage, definitions and response-rate QA | Selection/response bias; survey estimate ≠ observed contacts |
| Current open drafts / age | Source Salesforce BLA / application | `CreatedDate`, `LastModifiedDate` | Candidate snapshot | Long-lived draft ≠ abandonment, technical error or recovery failure |
| Submission and progression delay | Source Salesforce BLA / application | `AppliedDate` plus `Submitted__c` | Provisional from lifecycle investigation | Verify event semantics/history; `period_start` is validity start, never submission |
| Portal error / login failure rate | Portal or identity event stream | Event timestamp | Not measurable in discovered objects | Genesys `nError` is not a portal technical error |
| Cross-channel escalation | No confirmed Case–conversation link | Unresolved | Blocked | Do not infer from near timestamps/ANI |

For each future metric define **numerator, denominator, source grain, time anchor, eligible population, join key, missingness, exclusions and validation outcome** before charting. Never count Genesys participant rows as unique conversations without an explicit aggregation test.

## Friction classification: evidence gap rather than confirmed cause

- **Clarity Gap:** ASK reason text, survey topics or approved transcript analysis could support classification; not verified yet.
- **Access Barrier:** relevant service/account access cases are possible proxies, not authentication failure event counts.
- **Technical Failure:** no validated portal-side error events found; cannot infer from Genesys interaction `nError` or old drafts.
- **Process Complexity:** transfers, status churn and time taken expose effort; causation requires reason/expected-workflow review.
- **Visibility Gap:** repeat status enquiries, communication history could contribute if reasons for contact can be validated.
- **Recovery Failure:** reopen patterns and inability to complete after error need case/status/history evidence; drafts alone do not establish it.

## Next contained validation

Before defining the first chart, profile **Salesforce Case history** in read-only mode: count distinct Cases and history entries where `Field` represents status; list observed `OldValue` → `NewValue` transitions and missing timestamps at aggregate level; assess whether reopened/returned states are distinguishable. Avoid personal data. This is a candidate only; prioritisation can change when the full Genie evidence is reviewed.

## Boundaries

Original records remain in authorised Salesforce/Genesys/Databricks systems. No source, view or dashboard writes implied. GA4 remains on-demand diagnostic context. This GitHub file stores metadata, caveats and safe interpretation only.
