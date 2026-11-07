**Scope of a pull request.** This repository is audited as a pull request. The scope is the diff from the merge base to the head, in the files listed for this repository in `context/scope.json`. The whole repository at the head is context: read any file you need.

A finding is in scope when the diff in a listed file causes it:

1. **The diff introduces the defect.** The defect is in code that the diff adds or changes in a listed file. A defect that the diff causes by removing code from a listed file, such as a deleted check, counts as introduced.
2. **The diff makes an older defect reachable or worse.** The defect is in code that the diff does not change. Before the change, it could not be triggered, or its impact was smaller. The finding shows the path through a change in a listed file.

A finding is out of scope when the same defect exists at the merge base with the same reachability and the same impact. This holds even when the defect is in a listed file. A finding is also out of scope when only changes in files that are not listed cause it.

A finding can point at any file in the repository. Its severity is the impact at the head.

To decide, ask two questions. Does this defect exist at the merge base with the same impact? Does a change in a listed file cause it? The finding is in scope only when the answers are no and yes.
