# Service Account dashboard visual standard

Status: Adopted design direction, 19 October 2026. Reference: Executive Overview concept approved in conversation. Apply within actual Databricks AI/BI dashboard capabilities; do not promise exact rendering parity with concept illustrations.

## Principles
- Executive clarity: concise, plain-English labels and focused visual purpose.
- Four ordered dashboard domains: 01 Account Management & Digital Access, 02 Customer Activity, 03 Customer Friction, 04 Journey Outcomes & Efficiency.
- Neutral page surface, restrained colour, generous consistent spacing, aligned chart sizes, no gratuitous decoration.
- KPI cards: short label, prominent numerical value, short context (period/population/caveat). No decorative badges.
- Consistent title hierarchy: dashboard title > numbered section > chart title > axis/tooltip/caption.
- Line charts for monthly time trends; bars for categorical and FIR group comparisons; meaningful ordering; start applicable count/rate bars at zero.
- Colour semantics: neutral/default for regular trends; green for favourable outcome; red for adverse decision; amber for friction/attention. Colour must not be sole carrier of meaning and should meet WCAG 2.2 AA contrast when possible.
- Keep rates properly defined: ratios from summed numerators and denominators, not mean of rates; end-of-quarter population for stock; event counts summed; percentiles computed from application-level journey records, not mean of monthly medians.
- Monthly is the model's primary aggregate grain; fiscal quarters/years are reporting views. Expose date anchor for each chart (account created, portal-enabled, application created, Case created, or observed Issued date).
- Status and draft current-state metrics are not historical outcomes. Issued duration from BLA CDH _effective_from is provisional. FIR comparisons observational, not causal. Keep denominator, coverage, cohort maturity visible in tooltips/notes.
- Avoid global service filters incorrectly applying to CRM account metrics when relationship is not established; prefer domain-appropriate filters.
- Responsive design in Databricks constraints: consistent widths, charts readable without scrolling horizontally, avoid overloaded legends and excessive precision.

## Repeatable chart specification
For each visual record: business question; source dataset; chart type; x-axis dimension / time anchor; y-axis and aggregation; grouping; period treatment; source filter; colour meaning; title; description/tooltip; caveat; accepted screenshot / QA status.

## Initial chart specification
- Domain: 01 Account Management & Digital Access
- Title: New CRM accounts by month
- Dataset: sa_account_monthly
- Chart: line
- X: month_start (continuous/chronologically sorted)
- Y: SUM(new_crm_accounts)
- Group: none initially
- Time anchor: CRM Account created_date
- Main caption: New Salesforce CRM Account records created each month; not unique people, nor total active account stock.
- Layout: half-width of two-chart section, matching portal enablement trend.
- Next companion chart: Portal enablement / adoption over time; percent from aligned numerator/denominator, not averaged monthly percentages.

## Implementation limits
Design concept is a visual reference; Databricks supported controls decide actual appearance. Keep SQL datasets independent of presentation where practical. Reuse chart settings manually or via available dashboard duplication tooling rather than assuming a custom theme import exists.
