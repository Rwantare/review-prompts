---
name: qemu-series
description: Review an entire QEMU patch series or pull request commit-by-commit
---

Using the prompt {{REVIEW_DIR}}/review-core.md, review a series of commits or pull
request range (e.g. `origin/master..HEAD` or a git range provided in the prompt).

Load {{REVIEW_DIR}}/technical-patterns.md first.

Workflow:
1. Print a numbered list of all commits in the range: `# <hash> <subject>`.
2. Inspect the series architecture: check for bisectability (does each commit build and maintain invariants?).
3. For each commit in sequence:
   - Perform the regression analysis using review-core.md.
   - Load relevant subsystem guides for changed code.
   - If a bug in an earlier commit is fixed in a later commit in the same series, note it and confirm whether the earlier commit breaks bisection.
4. Verify overall series tags: `Signed-off-by:`, `Reviewed-by:`, `Fixes:`, `Resolves:`.
5. Aggregate all verified regressions and issues into `review-inline.txt`.
