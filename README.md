{Report / App / Dashboard Name}
Release Notes
Audience: Business users & stakeholders   Maintained by: Data & AI Team   Last updated: {dd/mm/yyyy}
Worked example. The dashboard name, dates, version content and requirement references below are illustrative — replace them with your real release detail. The structure is the bit to keep: it is the level of detail requested for business-facing notes.
About these release notes
These notes summarise what has changed in each release of the dashboard, why the change was made, and what (if anything) users need to do. They are issued to the business so stakeholders can understand how each version meets new or updated requirements. The most recent release appears first.
How to read each entry
•	New features & enhancements — capabilities added in this version, each tied to the requirement it addresses.
•	Changes to existing functionality — anything that behaves differently from the previous version.
•	Bug fixes — defects corrected in this version.
•	Data & model changes — changes to sources, refresh schedules or the underlying model that may affect figures.
•	Known issues / limitations — anything not yet resolved, with a workaround where available.
•	Action required — steps users should take after this release.
Requirement references in square brackets (e.g. [REQ-118]) link each change back to the requirement or change request it satisfies, so the business can see how the version meets the new requirements.
Version history at a glance
This summary gives a quick, structured view for management information and audit; the detailed notes follow below.
Version	UAT Date	Released	Status	Headline Change
2.1	25/12/2025	01/01/2026	Live	New home page + migration to certified semantic model
2.0	01/03/2025	15/03/2025	Superseded	Re-designed data model; scheme-level drill-through added
1.0	20/06/2024	01/07/2024	Superseded	Initial release of the dashboard
 January 2026
Version 2.1  |  Deployed to UAT: 25/12/2025  |  Released to Production: 01/01/2026  |  Status: Live
Release summary
This release refreshes the dashboard landing experience and moves reporting onto the new certified semantic model. The redesigned home page gives users a single overview of headline metrics, while the data source migration aligns the dashboard with the approved data governance standard and improves refresh reliability. There is no change to the figures users are familiar with, other than the data quality improvements noted below.
New features & enhancements
•	Redesigned home page — the landing page now presents a single, role-aware overview with the most-used KPIs, a clearer navigation panel and a “last refreshed” timestamp. This reduces the number of clicks needed to reach the most common views.  [REQ-118: single entry point]
•	In-page guided tour — first-time users see optional tooltips explaining each section, lowering the support burden during roll-out.  [REQ-119: self-service onboarding]
Changes to existing functionality
•	Migrated data source — the dashboard now reads from the certified semantic model rather than the legacy SQL view. Slicers, totals and drill-throughs behave as before; the change is transparent to users but standardises definitions across reports.  [REQ-122: certified single source of truth]
•	Refresh schedule — the scheduled refresh has moved from 06:00 to 05:30 so data is ready before the start of business.  [REQ-124]
Bug fixes
•	Month-to-date totals — corrected an issue where MTD values double-counted on the first working day of the month when a scheme filter was applied.
•	Export to Excel — fixed column headers being truncated when exporting the transactions table.
Data & model changes
•	New source: certified semantic model (replaces legacy SQL view).
•	Definition alignment: “Settled value” now uses the governed measure; historical figures are unchanged but are now consistent with other certified reports.
Known issues / limitations
•	On very first load after the migration, pinned dashboard tiles may show a cached value for up to one refresh cycle. Workaround: refresh the page once.
Action required
•	Re-pin any personal bookmarks created before this release, as the page layout has changed.
•	No action required for scheduled subscriptions — these continue to work automatically.
March 2025
Version 2.0  |  Deployed to UAT: 01/03/2025  |  Released to Production: 15/03/2025  |  Status: Superseded
Release summary
A major release that re-designed the underlying data model to support faster filtering and new scheme-level analysis. This version introduced drill-through from summary to transaction detail, addressing the business need to investigate variances without leaving the dashboard.
New features & enhancements
•	Scheme-level drill-through — users can right-click any summary figure to view the underlying transactions for that scheme and period.  [REQ-101: variance investigation]
•	Date-range comparison — added a period-over-period view so users can compare the current period against the prior period or the same period last year.  [REQ-104]
Changes to existing functionality
•	Re-designed the data model — moved to a star schema with conformed dimensions, improving filter responsiveness and enabling consistent cross-report definitions.  [REQ-100]
•	Renamed measures — for clarity, “Volume” is now “Transaction Count” and “Value” is now “Settled Value”. Old bookmarks continued to work.
Bug fixes
•	Resolved slow load times on the summary page caused by redundant relationships in the previous model.
•	Fixed incorrect totals when more than one region was selected at once.
Data & model changes
•	Model: single flat table replaced with a star schema (fact + conformed dimensions).
•	New dimension: Scheme dimension added to support drill-through and filtering.
Known issues / limitations
•	Period-over-period comparison did not yet support custom (non-calendar) periods. Addressed in a later release.
Action required
•	Users relying on the old measure names in personal reports should update references to “Transaction Count” and “Settled Value”.
July 2024
Version 1.0  |  Deployed to UAT: 20/06/2024  |  Released to Production: 01/07/2024  |  Status: Superseded
Release summary
Initial release of the dashboard, providing the business with a single view of operational performance to replace the previous manual spreadsheet pack.
New features & enhancements
•	Operational overview — headline KPIs for volume and value, with monthly trend and region breakdown.  [REQ-001: replace manual MI pack]
•	Self-service filtering — users can filter by date, region and product without requesting a new report.  [REQ-002]
Data & model changes
•	Source: initial load from the operational data warehouse, refreshed daily.
Known issues / limitations
•	Scheme-level detail was not available in this version (delivered in v2.0).
Action required
•	None — first release.
 Template for future releases
Copy the block below for each new version and complete the bracketed fields. Keep newest at the top. Delete any sections that do not apply to a given release (but keep at least Release summary and Action required, even if Action required is “None”).
{Month Year}
Version {x.x}  |  Deployed to UAT: {dd/mm/yyyy}  |  Released to Production: {dd/mm/yyyy}  |  Status: {Live / Superseded / In UAT}
Release summary
{Two to four sentences in plain English: what this release does and the business need it addresses. Avoid jargon — this is the part most stakeholders will read.}
New features & enhancements
•	{Feature name} — {what it does and the benefit to the user}.  [REQ-xxx]
Changes to existing functionality
•	{What changed} — {how it now behaves differently and why}.  [REQ-xxx]
Bug fixes
•	{Defect corrected, described from the user's point of view}.
Data & model changes
•	{Source / measure / model}: {what changed and any impact on figures}.
Known issues / limitations
•	{Anything outstanding, with a workaround if one exists}.
Action required
•	{What users need to do, or “None”}.
Tip: every change in “New features” and “Changes to existing functionality” should carry a requirement reference so the business can trace how the version meets the new requirements — this is the main thing the notes need to demonstrate.
