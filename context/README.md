# Context map

| File | Structure and contents |
| --- | --- |
| [scope.json](scope.json) | Unchanged scope object: `codebases[]` contains repository directory, revision, `in_scope_files[]` and pull request metadata. |
| [statements.json](statements.json) | Unchanged object with `version` and `files[]`; each file has a `path` and `statements[]` with `id`, `lines`, `quote` and per-dimension `dimensions` containing `level` and `fails`. Judges cite statement IDs. |
| [entries.json](entries.json) | Document array: `id`, workspace-relative `path`, `dimensions`, one-line `description`, `written_for_engagement`, `level`, `visibility` and `sources[]` with `path` and `location`. |
| [severity-set.json](severity-set.json) | Ordered array of the five complete severity definitions, preserving `identifier`, `display_name` and `body_markdown`. |

Entry dimensions are `tf`, `scope`, `known` and `severity`; an empty array identifies background. The statements file uses `true_finding` for `tf`. Entry levels describe the document’s attachment level (`platform`, `customer`, `repo`, `engagement`); statement levels describe the individual statement. An inventory label is not a verdict.

Queries run from the workspace root:

```sh
python - <<'PY'
import json

# Documents relevant to known status; change to tf, scope or severity.
entries = json.load(open("context/entries.json"))
for entry in entries:
    if "known" in entry["dimensions"]:
        print(entry["id"], entry["path"])

# Exact statement and source by ID.
statements = json.load(open("context/statements.json"))
for source in statements["files"]:
    for statement in source["statements"]:
        if statement["id"] == "S1":
            print(json.dumps({"path": source["path"], **statement}, indent=2))
PY
```

To select statements by dimension, replace the ID test with `if "scope" in statement["dimensions"]:` (or `true_finding` or `known`). Preserve each matching quotation and its level when reading the results.

[Source catalog](catalog.md) links the full documents, original PDFs and repository statement sources. [Source reading notes](source-reading-notes.md) explain identifier and link differences. [Judging guidelines](judging-guidelines.md), the [repository scope rule](scope-rules/uniswap__v4-periphery.md), [deduplication instructions](deduplication-guidelines.md) and the [finding report standard](finding-report-standard.md) contain the associated guidance.
