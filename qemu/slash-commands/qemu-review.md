---
name: qemu-review
description: Deep-dive regression analysis of a QEMU commit or patch
---

Using the prompt {{REVIEW_DIR}}/review-core.md, run a deep dive regression
analysis of the specified commit, diff, or range.

If no commit is specified, analyze the top commit (HEAD).

Load {{REVIEW_DIR}}/technical-patterns.md first, then follow the complete
review protocol in review-core.md.

For the commit being analyzed:
1. Understand the commit's intent and subsystem scope.
2. Identify all changed files and functions.
3. Load matching subsystem files from {{REVIEW_DIR}}/subsystems/subsystem.md.
4. Verify memory safety (GLib, g_autofree), QOM lifecycle, BQL/locking, and error handling.
5. Check hypervisor security: ensure guest inputs cannot trigger crashes, asserts, or out-of-bounds access.
6. Verify migration compatibility (VMState) if device state changed.
7. Run false positive checks before reporting.
8. Create review-inline.txt if issues or regressions are found.
