# Deduplication guidelines

The selected [Sherlock judging guidelines §4](judging-guidelines.md#4-duplication-guidelines) govern duplication. Engagement context can explain their application and cannot override them. The instructions below reference that section; use the full source for details.

For each comparison, identify the **target issue** and the **potential duplicate**. The potential duplicate must meet all three requirements:

1. Identify the root cause.
2. Identify at least a Low impact.
3. Identify a valid attack path or vulnerability path.

Only when the potential duplicate meets all three requirements is it grouped with the target issue. All duplicates receive the highest severity identified among them. When the report fails any requirement, do not group it with that target; assess whether it duplicates another issue separately, and assess its True Finding, scope, known and severity verdicts under their respective rules. Source: [§4, requirements and consequences](judging-guidelines.md#4-duplication-guidelines). The complete [severity set](severity-set.json) supplies the severity definitions.

Issues appearing in multiple places, including different components, may share the same logic mistake or conceptual mistake. If their underlying code implementations, impact or fixes differ, they may be treated separately. Where an issue has many duplicates, group reports that identify the core issue and show a valid impact. Source: [§4, “Root cause groupings” and “Identifies the core issue”](judging-guidelines.md#4-duplication-guidelines).

Use the definitions of root cause, attack path and vulnerability path in the [glossary (§6)](judging-guidelines.md#6-glossary). The [finding report standard](finding-report-standard.md) guides writing; assess the finding’s content when applying the three requirements.

The [protocol document §§5–6](Uniswap_Context_File.md#5-previous-audits) and [report catalog](catalog.md#reports-and-technical-papers) provide disclosure context. Their report cross-references do not replace the submitted-report comparison required by §4. Known status has its own [guideline §3.3](judging-guidelines.md#33-known) and [statements](statements.json).
