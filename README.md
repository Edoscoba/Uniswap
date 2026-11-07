# Uniswap v4 margin trading engagement

The [context map](context/README.md) explains the structured files and lookup commands. Read the [statements](context/statements.json) alongside the [document inventory](context/entries.json): statements record accepted behavior, scope and known issues for this engagement, with IDs, exact quotations, source lines and levels. Apply the [judging guidelines](context/judging-guidelines.md), including their order of authority in §2.

[uniswap__v4-periphery/](uniswap__v4-periphery/) audits pull request **#563, feat: add margin trading periphery**. The head is checked out. Run `git diff merge-base HEAD` inside that directory to see the change; read its [pull request scope rule](context/scope-rules/uniswap__v4-periphery.md).

| Revision | Commit |
| --- | --- |
| Audited head | `1c86348d3fa3f91660b2c75b325807cb12872caf` |
| Merge base | `c6e4754dab11bd38c6f08ac51c3c868569e93d64` |

The [scope manifest](context/scope.json) contains the 20 selected files and revision metadata. The developer-written pull request title and description in that manifest are background only and never bind a verdict.

| Evidence needed | Reference |
| --- | --- |
| Architecture, components and user flows | [Protocol document §§1–2](context/Uniswap_Context_File.md#1-protocol-overview); [integration guide](uniswap__v4-periphery/docs/margin-trading.md) |
| Chains, tokens and integrations | [Protocol document §3](context/Uniswap_Context_File.md#3-deployment-environment) |
| Roles, permissions and trust assumptions | [Protocol document §4](context/Uniswap_Context_File.md#4-roles-permissions--trust-model) |
| Prior reviews, disclosures, accepted behavior and fix-status notes | [Protocol document §§5–6](context/Uniswap_Context_File.md#5-previous-audits); [complete report catalog](context/catalog.md#reports-and-technical-papers); [source reading notes](context/source-reading-notes.md) |
| Build, tests and PoC setup | [Protocol document §7](context/Uniswap_Context_File.md#7-build-test--poc-setup); [Foundry configuration](uniswap__v4-periphery/foundry.toml) |
| Validity, scope and known-status standards | [Judging guidelines §§2–3](context/judging-guidelines.md#2-order-of-authority) |
| Complete severity definitions, including every Medium condition | [Severity set](context/severity-set.json); [severity reference](context/severities.md) |
| Comparing submitted findings | [Deduplication guidelines](context/deduplication-guidelines.md) |

The [finding report standard](context/finding-report-standard.md) provides guidance for writing findings. Judging and deduplication assess a finding’s content, not whether it follows that standard.
