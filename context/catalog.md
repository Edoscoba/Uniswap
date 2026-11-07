# Source catalog

[Document inventory](entries.json) records every supplied document, each PDF with its readable text, and repository documents cited by the [statements](statements.json). Dimensions identify relevant evidence; they are not verdicts.

For known-status assessment, consult [judging guidelines §3.3](judging-guidelines.md#33-known), the statements, and the full report including its per-finding updates. Report severity labels and audit scopes belong to those reports; the engagement uses its [selected severity set](severity-set.json) and [scope rule](scope-rules/uniswap__v4-periphery.md).

## Engagement documents

| Document | Contents |
| --- | --- |
| [Protocol document](Uniswap_Context_File.md) | Architecture and user flows (§§1–2), deployment environment (§3), trust and permissions (§4), prior reviews (§5), disclosures and current fix-status table (§6), build and test setup (§7). |
| [Statements](statements.json) | Exact quotations, IDs, source lines and levels. |
| [Scope manifest](scope.json) | Repository, revision, selected files and developer-written pull request background. |
| [Judging guidelines](judging-guidelines.md) | Authority, validity, scope, known status, severity rules, duplication and mechanics. |
| [Severity reference](severities.md) | Supplied definitions, abbreviations and weights. |
| [Finding report standard](finding-report-standard.md) | Guidance for writing finding fields. |
| [Source reading notes](source-reading-notes.md) | Identifier differences, referenced report availability and source-link navigation. |

## Reports and technical papers

Rows group byte-identical PDFs. Every occurrence remains separately inventoried by the listed entry IDs. Each text link includes the complete document; originals remain at their repository locations.

| Document | Readable text | Original | Inventory locations |
| --- | --- | --- | --- |
| ABDK: v4 Periphery and Universal Router (draft, September 2024) | [Text](<derived/uniswap__v4-periphery/audits/DRAFT_ABDK_audit_periphery_universal_router.md>) | [PDF](<../uniswap__v4-periphery/audits/DRAFT_ABDK_audit_periphery_universal_router.pdf>) | pdf-01, pdf-22, pdf-31 |
| Spearbit: v4 Periphery (draft, September 2024) | [Text](<derived/uniswap__v4-periphery/audits/DRAFT_Spearbit_audit_periphery.md>) | [PDF](<../uniswap__v4-periphery/audits/DRAFT_Spearbit_audit_periphery.pdf>) | pdf-02, pdf-32 |
| OpenZeppelin: Uniswap Margin Trading (August 2026) | [Text](<derived/uniswap__v4-periphery/audits/OpenZeppelin_audit_margin_trading.md>) | [PDF](<../uniswap__v4-periphery/audits/OpenZeppelin_audit_margin_trading.pdf>) | pdf-03 |
| OpenZeppelin: v4 Periphery and Universal Router (September 2024) | [Text](<derived/uniswap__v4-periphery/audits/OpenZeppelin_audit_periphery_universal_router.md>) | [PDF](<../uniswap__v4-periphery/audits/OpenZeppelin_audit_periphery_universal_router.pdf>) | pdf-04, pdf-24, pdf-33 |
| Cantina: Permissioned Pools (August 2026) | [Text](<derived/uniswap__v4-periphery/audits/permissionedPools/Cantina_report_permissioned_pools.md>) | [PDF](<../uniswap__v4-periphery/audits/permissionedPools/Cantina_report_permissioned_pools.pdf>) | pdf-05 |
| OpenZeppelin: v4 Periphery PR #585 Review (August 2026) | [Text](<derived/uniswap__v4-periphery/audits/permissionedPools/OpenZeppelin_permissioned_pools_fix_review_report.md>) | [PDF](<../uniswap__v4-periphery/audits/permissionedPools/OpenZeppelin_permissioned_pools_fix_review_report.pdf>) | pdf-06 |
| OpenZeppelin: Permissioned Pools Force-Exit Fix Review (September 2026) | [Text](<derived/uniswap__v4-periphery/audits/permissionedPools/OpenZeppelin_permissioned_pools_force_exit_fix_review_report.md>) | [PDF](<../uniswap__v4-periphery/audits/permissionedPools/OpenZeppelin_permissioned_pools_force_exit_fix_review_report.pdf>) | pdf-07 |
| OpenZeppelin: Permissioned Pools (May 2026) | [Text](<derived/uniswap__v4-periphery/audits/permissionedPools/OpenZeppelin_permissioned_pools_report.md>) | [PDF](<../uniswap__v4-periphery/audits/permissionedPools/OpenZeppelin_permissioned_pools_report.pdf>) | pdf-08 |
| OpenZeppelin: Morpho Labs Blue (October 2023) | [Text](<derived/uniswap__v4-periphery/lib/morpho-blue/audits/2023-10-13-morpho-blue-and-speed-jump-irm-open-zeppelin.md>) | [PDF](<../uniswap__v4-periphery/lib/morpho-blue/audits/2023-10-13-morpho-blue-and-speed-jump-irm-open-zeppelin.pdf>) | pdf-09 |
| Cantina: Morpho Blue (draft, November 2023) | [Text](<derived/uniswap__v4-periphery/lib/morpho-blue/audits/2023-11-13-cantina-managed-review-draft.md>) | [PDF](<../uniswap__v4-periphery/lib/morpho-blue/audits/2023-11-13-cantina-managed-review-draft.pdf>) | pdf-10 |
| Morpho Blue Whitepaper (October 2023) | [Text](<derived/uniswap__v4-periphery/lib/morpho-blue/morpho-blue-whitepaper.md>) | [PDF](<../uniswap__v4-periphery/lib/morpho-blue/morpho-blue-whitepaper.pdf>) | pdf-11 |
| ABDK: Permit2 (November 2022) | [Text](<derived/uniswap__v4-periphery/lib/permit2/audits/ABDKAudit.md>) | [PDF](<../uniswap__v4-periphery/lib/permit2/audits/ABDKAudit.pdf>) | pdf-12, pdf-26, pdf-34 |
| ChainSecurity: Permit2 (November 2022) | [Text](<derived/uniswap__v4-periphery/lib/permit2/audits/Chainsecurity Audit.md>) | [PDF](<../uniswap__v4-periphery/lib/permit2/audits/Chainsecurity Audit.pdf>) | pdf-13, pdf-27, pdf-35 |
| Level K: OpenZeppelin v2.0 (October 2018) | [Text](<derived/uniswap__v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2018-10.md>) | [PDF](<../uniswap__v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2018-10.pdf>) | pdf-14, pdf-28, pdf-36, pdf-46, pdf-65 |
| Fixed Point Solutions: Solmate Library (December 2021) | [Text](<derived/uniswap__v4-periphery/lib/permit2/lib/solmate/audits/v6-Fixed-Point-Solutions.md>) | [PDF](<../uniswap__v4-periphery/lib/permit2/lib/solmate/audits/v6-Fixed-Point-Solutions.pdf>) | pdf-15, pdf-29, pdf-30, pdf-37, pdf-54, pdf-73 |
| Ackee: Solady Tokens and Utils (May 2023) | [Text](<derived/uniswap__v4-periphery/lib/solady/audits/ackee-blockchain-solady-report.md>) | [PDF](<../uniswap__v4-periphery/lib/solady/audits/ackee-blockchain-solady-report.pdf>) | pdf-16 |
| Cantina: Solady (September 2023) | [Text](<derived/uniswap__v4-periphery/lib/solady/audits/cantina-solady-report.md>) | [PDF](<../uniswap__v4-periphery/lib/solady/audits/cantina-solady-report.pdf>) | pdf-17 |
| Spearbit: Coinbase Solady (January 2025) | [Text](<derived/uniswap__v4-periphery/lib/solady/audits/cantina-spearbit-coinbase-solady-report.md>) | [PDF](<../uniswap__v4-periphery/lib/solady/audits/cantina-spearbit-coinbase-solady-report.pdf>) | pdf-18 |
| Shung: Solady ERC721 Security Review | [Text](<derived/uniswap__v4-periphery/lib/solady/audits/shung-solady-erc721-audit.md>) | [PDF](<../uniswap__v4-periphery/lib/solady/audits/shung-solady-erc721-audit.pdf>) | pdf-19 |
| Xu Winnie: Solady cbrt and cbrtWad Proof (July 2024) | [Text](<derived/uniswap__v4-periphery/lib/solady/audits/xuwinnie-solady-cbrt-proof.md>) | [PDF](<../uniswap__v4-periphery/lib/solady/audits/xuwinnie-solady-cbrt-proof.pdf>) | pdf-20 |
| ABDK: Universal Router v2.0 | [Text](<derived/uniswap__v4-periphery/lib/universal-router/audit/ABDK_Uniswap_UniversalRouter_v_2_0.md>) | [PDF](<../uniswap__v4-periphery/lib/universal-router/audit/ABDK_Uniswap_UniversalRouter_v_2_0.pdf>) | pdf-21 |
| Spearbit: v4 Universal Router (draft, September 2024) | [Text](<derived/uniswap__v4-periphery/lib/universal-router/audit/DRAFT_Spearbit_audit_UR.md>) | [PDF](<../uniswap__v4-periphery/lib/universal-router/audit/DRAFT_Spearbit_audit_UR.pdf>) | pdf-23 |
| OpenZeppelin: Universal Router 2.1 and v4 Periphery (November 2025) | [Text](<derived/uniswap__v4-periphery/lib/universal-router/audit/Universal Router 2.1 And V4 Periphery Audit.md>) | [PDF](<../uniswap__v4-periphery/lib/universal-router/audit/Universal Router 2.1 And V4 Periphery Audit.pdf>) | pdf-25 |
| Known Effects of Hook Permissions | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/security/Known_Effects_of_Hook_Permissions.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/security/Known_Effects_of_Hook_Permissions.pdf>) | pdf-38, pdf-57 |
| ABDK: v4 Core (draft, September 2024) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/security/audits/DRAFT_ABDK_audit_core.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/security/audits/DRAFT_ABDK_audit_core.pdf>) | pdf-39, pdf-58 |
| Certora: v4 Core Security Assessment and Formal Verification (draft, July 2024) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/security/audits/DRAFT_Certora_audit_core.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/security/audits/DRAFT_Certora_audit_core.pdf>) | pdf-40, pdf-59 |
| Spearbit: v4 Core (draft, September 2024) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/security/audits/DRAFT_Spearbit_audit_core.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/security/audits/DRAFT_Spearbit_audit_core.pdf>) | pdf-41, pdf-60 |
| OpenZeppelin: v4 Core (July 2024) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/security/audits/OpenZeppelin_audit_core.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/security/audits/OpenZeppelin_audit_core.pdf>) | pdf-42, pdf-61 |
| Trail of Bits: v4 Core Security Assessment | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/security/audits/TrailOfBits_audit_core.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/security/audits/TrailOfBits_audit_core.pdf>) | pdf-43, pdf-62 |
| Uniswap v4 Core Whitepaper (June 2023) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/whitepaper/whitepaper-v4.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/whitepaper/whitepaper-v4.pdf>) | pdf-44, pdf-63 |
| Uniswap v4 Core Whitepaper, Chinese translation (June 2023) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/docs/whitepaper/whitepaper-zh.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/docs/whitepaper/whitepaper-zh.pdf>) | pdf-45, pdf-64 |
| OpenZeppelin: Checkpoints Library (November 2022) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2022-10-Checkpoints.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2022-10-Checkpoints.pdf>) | pdf-47, pdf-66 |
| OpenZeppelin: ERC4626 Tokenized Vault (November 2022) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2022-10-ERC4626.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2022-10-ERC4626.pdf>) | pdf-48, pdf-67 |
| OpenZeppelin: Contracts 4.9 Release (May 2023) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2023-05-v4.9.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2023-05-v4.9.pdf>) | pdf-49, pdf-68 |
| OpenZeppelin: Contracts 5.0 Release (October 2023) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2023-10-v5.0.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2023-10-v5.0.pdf>) | pdf-50, pdf-69 |
| Certora: OpenZeppelin Governance Formal Verification (October–November 2021) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/certora/reports/2021-10.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/certora/reports/2021-10.pdf>) | pdf-51, pdf-70 |
| Certora: OpenZeppelin Formal Verification (March–April 2022) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/certora/reports/2022-03.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/certora/reports/2022-03.pdf>) | pdf-52, pdf-71 |
| Certora: OpenZeppelin Formal Verification (May–June 2022) | [Text](<derived/uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/certora/reports/2022-05.md>) | [PDF](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/certora/reports/2022-05.pdf>) | pdf-53, pdf-72 |
| ABDK: Uniswap v3 Core | [Text](<derived/uniswap__v4-periphery/lib/v3-core/audits/abdk/audit.md>) | [PDF](<../uniswap__v4-periphery/lib/v3-core/audits/abdk/audit.pdf>) | pdf-55 |
| Trail of Bits: Uniswap v3 Core (March 2021) | [Text](<derived/uniswap__v4-periphery/lib/v3-core/audits/tob/audit.md>) | [PDF](<../uniswap__v4-periphery/lib/v3-core/audits/tob/audit.pdf>) | pdf-56 |

## Repository-scan documents

| Document | Repository source | Inventory |
| --- | --- | --- |
| [New Alchemy OpenZeppelin audit (March 2017), repository-scan copy.](<uniswap-v4-periphery__lib-permit2-lib-openzeppelin-contracts-audit-2017-03.md>) | [lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md](<../uniswap__v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md>) | attachment-02 |
| [Universal Router audit-directory guide and report list.](<uniswap-v4-periphery__lib-universal-router-audit-CLAUDE.md>) | [lib/universal-router/audit/CLAUDE.md](<../uniswap__v4-periphery/lib/universal-router/audit/CLAUDE.md>) | attachment-03 |
| [New Alchemy OpenZeppelin audit (March 2017), repository-scan copy.](<uniswap-v4-periphery__lib-universal-router-lib-permit2-lib-openzeppelin-contracts-audit-2017-03.md>) | [lib/universal-router/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md](<../uniswap__v4-periphery/lib/universal-router/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md>) | attachment-04 |
| [New Alchemy OpenZeppelin audit (March 2017), repository-scan copy.](<uniswap-v4-periphery__lib-universal-router-lib-v4-periphery-lib-permit2-lib-openzeppelin-contracts-audit-2017-03.md>) | [lib/universal-router/lib/v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md>) | attachment-05 |
| [New Alchemy OpenZeppelin audit (March 2017), repository-scan copy.](<uniswap-v4-periphery__lib-universal-router-lib-v4-periphery-lib-v4-core-lib-openzeppelin-contracts-audits-2017-03.md>) | [lib/universal-router/lib/v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md>) | attachment-06 |
| [OpenZeppelin audit and formal-verification report index.](<uniswap-v4-periphery__lib-universal-router-lib-v4-periphery-lib-v4-core-lib-openzeppelin-contracts-audits-README.md>) | [lib/universal-router/lib/v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/README.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/README.md>) | attachment-07 |
| [Trail of Bits v3 verification-property documentation and tool instructions.](<uniswap-v4-periphery__lib-v3-core-audits-tob-README.md>) | [lib/v3-core/audits/tob/README.md](<../uniswap__v4-periphery/lib/v3-core/audits/tob/README.md>) | attachment-08 |
| [New Alchemy OpenZeppelin audit (March 2017), repository-scan copy.](<uniswap-v4-periphery__lib-v4-core-lib-openzeppelin-contracts-audits-2017-03.md>) | [lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md>) | attachment-09 |
| [OpenZeppelin audit and formal-verification report index.](<uniswap-v4-periphery__lib-v4-core-lib-openzeppelin-contracts-audits-README.md>) | [lib/v4-core/lib/openzeppelin-contracts/audits/README.md](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/README.md>) | attachment-10 |

## Repository statement sources

Each source below is referenced in the supplied statements. Repository files remain at their original paths.

| Source | Statement IDs |
| --- | --- |
| [src/AaveLendingAdapter.sol](<../uniswap__v4-periphery/src/AaveLendingAdapter.sol>) | S108, S109, S110 |
| [src/AaveV4LendingAdapter.sol](<../uniswap__v4-periphery/src/AaveV4LendingAdapter.sol>) | S111 |
| [src/CompoundV3LendingAdapter.sol](<../uniswap__v4-periphery/src/CompoundV3LendingAdapter.sol>) | S112 |
| [src/MarginAccount.sol](<../uniswap__v4-periphery/src/MarginAccount.sol>) | S113, S114 |
| [src/MarginAccountFactory.sol](<../uniswap__v4-periphery/src/MarginAccountFactory.sol>) | S115, S116 |
| [src/MarginRouter.sol](<../uniswap__v4-periphery/src/MarginRouter.sol>) | S117, S118, S119, S120, S121, S122, S123, S124, S125, S126, S127, S128, S129, S130 |
| [src/MorphoLendingAdapter.sol](<../uniswap__v4-periphery/src/MorphoLendingAdapter.sol>) | S131 |
| [src/interfaces/IAmountResolver.sol](<../uniswap__v4-periphery/src/interfaces/IAmountResolver.sol>) | S132 |
| [src/interfaces/ILendingAdapter.sol](<../uniswap__v4-periphery/src/interfaces/ILendingAdapter.sol>) | S133, S134, S135 |
| [src/interfaces/IMarginAccount.sol](<../uniswap__v4-periphery/src/interfaces/IMarginAccount.sol>) | S136, S137, S138, S139 |
| [src/interfaces/IMarginRouter.sol](<../uniswap__v4-periphery/src/interfaces/IMarginRouter.sol>) | S140, S141, S142, S143, S144, S145, S146, S147, S148 |
| [src/libraries/MarginActions.sol](<../uniswap__v4-periphery/src/libraries/MarginActions.sol>) | S149, S150, S151 |
| [src/libraries/MarginCalldataDecoder.sol](<../uniswap__v4-periphery/src/libraries/MarginCalldataDecoder.sol>) | S152 |
| [src/libraries/TransientPlanState.sol](<../uniswap__v4-periphery/src/libraries/TransientPlanState.sol>) | S153 |
| [src/types/PositionData.sol](<../uniswap__v4-periphery/src/types/PositionData.sol>) | S154 |
| [CONTRIBUTING.md](<../uniswap__v4-periphery/CONTRIBUTING.md>) | S155 |
| [docs/margin-trading.md](<../uniswap__v4-periphery/docs/margin-trading.md>) | S156, S157, S158, S159, S160, S161, S162, S163, S164, S165, S166, S167, S168, S169, S170, S171, S172, S173, S174, S175, S176, S177, S178, S179, S180, S181, S182, S183, S184, S185 |
| [lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md](<../uniswap__v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md>) | S186, S187, S188, S189, S190, S191, S192, S193, S194, S195, S196, S197, S198, S199, S200, S201, S202, S203, S204, S205, S206 |
| [lib/universal-router/.claude/prompts/claude-pr-review.md](<../uniswap__v4-periphery/lib/universal-router/.claude/prompts/claude-pr-review.md>) | S207, S208, S209, S210, S211, S212, S213, S214, S215, S216, S217 |
| [lib/universal-router/contracts/deploy/CLAUDE.md](<../uniswap__v4-periphery/lib/universal-router/contracts/deploy/CLAUDE.md>) | S218 |
| [lib/universal-router/deploy-addresses/CLAUDE.md](<../uniswap__v4-periphery/lib/universal-router/deploy-addresses/CLAUDE.md>) | S219 |
| [lib/universal-router/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md](<../uniswap__v4-periphery/lib/universal-router/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md>) | S220, S221, S222, S223, S224, S225, S226, S227, S228, S229, S230, S231, S232, S233, S234, S235, S236, S237, S238, S239 |
| [lib/universal-router/lib/v3-periphery/bug-bounty.md](<../uniswap__v4-periphery/lib/universal-router/lib/v3-periphery/bug-bounty.md>) | S240, S241, S242, S243, S244, S245 |
| [lib/universal-router/lib/v4-periphery/CONTRIBUTING.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/CONTRIBUTING.md>) | S246, S247, S248 |
| [lib/universal-router/lib/v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/lib/permit2/lib/openzeppelin-contracts/audit/2017-03.md>) | S249, S250, S251, S252, S253, S254, S255, S256, S257, S258, S259, S260, S261, S262, S263, S264, S265, S266, S267, S268, S269 |
| [lib/universal-router/lib/v4-periphery/lib/v4-core/CONTRIBUTING.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/lib/v4-core/CONTRIBUTING.md>) | S270, S271, S272 |
| [lib/universal-router/lib/v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md](<../uniswap__v4-periphery/lib/universal-router/lib/v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md>) | S273, S274, S275, S276, S277, S278, S279, S280, S281, S282, S283, S284, S285, S286, S287, S288, S289, S290, S291 |
| [lib/v3-core/bug-bounty.md](<../uniswap__v4-periphery/lib/v3-core/bug-bounty.md>) | S292, S293, S294, S295, S296, S297 |
| [lib/v4-core/CONTRIBUTING.md](<../uniswap__v4-periphery/lib/v4-core/CONTRIBUTING.md>) | S298, S299 |
| [lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md](<../uniswap__v4-periphery/lib/v4-core/lib/openzeppelin-contracts/audits/2017-03.md>) | S300, S301, S302, S303, S304, S305, S306, S307, S308, S309, S310, S311, S312, S313, S314, S315, S316, S317, S318, S319 |
