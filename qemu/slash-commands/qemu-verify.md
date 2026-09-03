---
name: qemu-verify
description: Verify potential QEMU findings against false positive patterns
---

Using the prompt {{REVIEW_DIR}}/false-positive-guide.md, verify that a reported
issue or regression is genuine and not a false positive.

Load {{REVIEW_DIR}}/technical-patterns.md first.

For each reported issue:
1. Trace the exact concrete code path that triggers the problem.
2. Verify that the condition is physically reachable in real execution.
3. Check QEMU-specific false positive rules:
   - Does it flag `g_malloc`/`g_new` without NULL check? (False positive: GLib aborts on OOM, unless size can be 0).
   - Does it assume BQL is absent when caller holds it? (False positive: MMIO dispatch holds BQL).
   - Does it flag `*errp` dereference when `ERRP_GUARD()` is present? (False positive: guarded).
   - Is it demanding defensive checks for internal state when input is already validated?
   - Is it complaining about mandatory QEMU coding style (e.g. braces on single-line ifs)?
4. Provide concrete code proof (call chain, struct values, trigger condition).

Output:
- `VERIFIED ISSUE`: Real bug with concrete proof and execution path.
- `ELIMINATED`: False positive with exact rationale.
