---
name: qemu-debug
description: Debug QEMU crashes, assertions, deadlocks, and guest hangs
---

Using the prompt {{REVIEW_DIR}}/debugging.md, systematically investigate a crash,
assertion failure, abort, deadlock, or guest hang in QEMU.

Load {{REVIEW_DIR}}/technical-patterns.md first, then follow debugging.md.

Expected input:
- Crash backtrace (gdb backtrace, coredumpctl info)
- QEMU log output (`-d guest_errors,unimp`, `-D /tmp/qemu.log`)
- Error message (e.g. `error_abort` or assertion message)
- QEMU command line and reproduction steps

Investigation steps:
1. Extract faulting address, instruction, and call frame chain.
2. Identify failing subsystem (QOM, Block, Memory, TCG, KVM, Virtio, Migration).
3. Check for typical QEMU bug classes:
   - Untrusted guest triggering host assert/abort
   - Missing or recursive BQL lock / lock order inversion
   - Coroutine re-entrancy or calling `coroutine_fn` outside coroutine context
   - Double-free or use-after-free in `g_autofree` / `g_autoptr` / QOM object lifecycle
   - Migration stream deserialization mismatch or missing `post_load` check
4. Formulate root cause and concrete reproducer or fix.
5. Generate `debug-report.txt`.
