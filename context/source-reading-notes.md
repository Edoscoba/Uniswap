# Reading the source collection

The [inventory](entries.json) cites the unchanged documents and their source locations. The [catalog](catalog.md) provides links to both readable text and original PDFs. Report updates and fix-review notes are part of each full report.

The [protocol document §6.5](Uniswap_Context_File.md#65-openzeppelin-finding-status-at-head) and the [OpenZeppelin margin-trading report](derived/uniswap__v4-periphery/audits/OpenZeppelin_audit_margin_trading.md) use different finding identifiers. For example, the absolute-fill assertion appears as M-03 in the protocol document and M-01 in the report; the cached data provider appears as M-01 and L-01 respectively. The protocol document §5 lists three Medium and fourteen Low findings, while the report’s Summary lists one Medium and sixteen Low findings. Finding titles and mechanisms locate the corresponding passages. Both sources and the [supplied statements](statements.json) retain their original text and identifiers.

The standalone Pashov report, its severity annex, the Simao report, and the internal validation record referenced in the [protocol document §5](Uniswap_Context_File.md#5-previous-audits) are not separate files in this collection. The protocol document contains their disclosed findings, leads, corrections and fix-status notes in §§5–6. Its references to those reports do not link to additional local documents.

Repository-scan copies under `context/` retain their original relative links. Those links are relative to the original repository location. The catalog’s “Repository source” links open that location, and its report table links readable conversions and PDFs directly. Supplied PDF conversions retain their original formatting and external links; page numbers in their text can differ from the PDF viewer’s page index.

PDF conversions with `## PDF page N` headings count from the first PDF page. Page images accompany technical papers, formal-verification reports and graphical finding summaries. Verification check, cross, writing-hand and stopwatch markers are also transcribed beside their adjacent text where the PDF stores them as images.
