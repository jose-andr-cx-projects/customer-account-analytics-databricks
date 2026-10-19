# Service Account dashboard — dataset registry

Status: Working inventory captured 19 October 2026. Not a certification of SQL implementations.

## Source of truth

Databricks AI/BI dashboard SQL is authoritative for actual dataset definitions, fields and execution. This repository records implementation intent, confirmed observations, decisions and caveats. Follow [project README](../README.md), [dashboard visual standard](dashboard-visual-standard.md), and [business validation and data-gap log](../discovery/sa-business-validation-and-data-gaps.md). Do not create replacement measures solely because an existing dataset has not yet been inventoried here.

## Observed dashboard inventory

All ten dataset names below were visible in the Databricks **Data Model > Datasets** panel in a user-provided screenshot on 19 October 2026. Grain and metric definitions are recorded only when supported by separate evidence.

| Dataset | Known use / intended domain | Evidence and remaining detail |
| --- | --- | --- |
| `sa_process_interruptions` | Process interruption / FIR reporting | Named in project README; current SQL, fields and relationship to prior `cx_process_interruptions` require runtime inspection |
| `sa_crm_account` | CRM account population | Logical grain one CRM Account ID described in README; verify current SQL implementation |
| `sa_account_monthly` | CRM account creation, portal enablement and their cumulative populations | SQL supplied in conversation; one row per month × `account_record_type` in its final grouped output; fields recorded below |
| `sa_application` | Application lifecycle and outcomes | Logical grain one BusinessLicenseApplication described in README; verify current SQL implementation |
| `sa_application_monthly` | Monthly application activity | Referenced in project discovery log; current SQL and fields not yet captured |
| `sa_draft_friction_monthly` | Draft-related customer friction | Name observed; business definition, source and fields not yet captured |
| `sa_friction_monthly` | Monthly friction trends | Name observed; source, grain, fields and measures not yet captured |
| `sa_outcomes_monthly` | Monthly application outcomes | Name observed; source, grain, fields and measures not yet captured |
| `sa_service` | Service classification/filtering | Logical service-key component described in README; verify current SQL implementation |
| `sa_journey_efficiency` | Journey duration / efficiency | Name observed; source, grain, fields and measures not yet captured |

## Confirmed account-monthly outputs

The user supplied the `sa_account_monthly` SQL during dashboard construction. It derives event months from `datahub_datamart.customer_account_management.vwaccount`:

- Dimensions: `month_start`, `shifted_quarter_start`, `financial_year_end`, `financial_quarter`, `financial_period`, `account_record_type`.
- Monthly flows: `new_crm_accounts`, `new_portal_enabled`.
- Cumulative counts: `cumulative_crm_accounts`, `cumulative_portal_enabled` (windowed within account type).
- Precalculated ratio: `digital_adoption_pct` = 100 × cumulative portal-enabled / cumulative CRM accounts, by month and account type, with zero-denominator protection.

**Interpretation and aggregation safeguards:** The dataset reflects account records, not distinct people. The numerator uses `customer_portal = true` and a non-null first enablement date, and the denominator uses accounts with a creation date. Historical enablement rates are provisional because the flag is current-state based; existing organisation-account inconsistencies are documented in the gap log. Do **not** sum cumulative populations across months or average account-type percentages for an overall KPI. Aggregate compatible numerator and denominator populations for a single reporting month before deriving the whole-population share; this is visualization-grain handling, not permission to introduce redundant measures.

## Implemented visuals observed during current session

| Visual | Dataset and field | State |
| --- | --- | --- |
| New CRM accounts by month | `sa_account_monthly`, `SUM(new_crm_accounts)` by `month_start` | Built; half-width smooth line with hidden markers |
| New portal-enabled accounts by month | `sa_account_monthly`, `SUM(new_portal_enabled)` by `month_start` | Built; half-width smooth line with hidden markers |
| Portal-enabled share of CRM accounts | Candidate `sa_account_monthly` ratio; whole-population KPI | Planned; not claimed complete; must use one reporting month with correct cross-account-type aggregation |

The two trend visuals are configured to ignore one another's cross-filtering. The approved theme is at [service-account-dashboard-theme.json](service-account-dashboard-theme.json). A lightened line observed during editing was attributed to Databricks selection highlighting rather than a change to the theme.

## Remaining lightweight inventory work

When each existing dataset is next used, capture its SQL-derived **source, grain, date anchor, key measure fields, filters and caveats** here and map it to a dashboard visual. No wholesale revalidation and no new SQL by default. Check the existing dataset first; only propose additional logic for a demonstrated gap.
