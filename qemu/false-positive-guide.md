# QEMU False Positive Elimination Guide

## Core Principle

**Never report an issue unless you can prove with concrete code paths and QEMU-specific rules that it is physically reachable, violates project invariants, and causes incorrect behavior.**

Low-signal, speculative, or incorrect review comments frustrate developers and maintainers. Use this guide to systematically filter out false positives before creating `review-inline.txt`.

---

## 1. Top QEMU False Positive Traps

### Trap 1: Missing NULL Check on `g_malloc` / `g_new`
- **Incorrect Finding**: "The return value of `g_malloc()`, `g_new()`, or `g_strdup()` is not checked for NULL."
- **Why It Is False**: GLib memory allocation functions (`g_malloc`, `g_malloc0`, `g_new`, `g_new0`, `g_strdup`, `g_strdup_printf`) **NEVER return NULL**. If memory allocation fails, GLib immediately prints an error and terminates the process via `abort()`. Adding a NULL check is dead code.
- **Rule**: Only flag missing NULL checks if the allocation uses `g_try_malloc()`, `g_try_new()`, or `g_try_malloc0()`.

### Trap 2: Believing BQL is Not Held in MMIO / Timer Callbacks
- **Incorrect Finding**: "The function accesses shared device state without taking `bql_lock()`."
- **Why It Is False**: The Big QEMU Lock (BQL) is automatically acquired by QEMU's main event loop before invoking:
  - Device MMIO read and write callbacks (`MemoryRegionOps.read` / `write`).
  - Timer expiration callbacks (`QEMUTimer`).
  - Standard bottom halves (`QEMUBH`).
  - CPU execution loops in TCG mode.
- **Rule**: Device MMIO callbacks universally execute with the BQL held by the main loop. Unless code specifically runs in an IOThread or background worker thread, BQL is already held. Taking BQL when already held triggers an immediate assertion failure (`g_assert(!bql_locked())`).

### Trap 3: Flagging `*errp` Dereference When `ERRP_GUARD()` Is Present
- **Incorrect Finding**: "Dereferencing `*errp` is unsafe because caller might pass `NULL`."
- **Why It Is False**: If the function contains `ERRP_GUARD();` at its entry, the macro automatically substitutes a dummy pointer if the caller passed `NULL` or `&error_fatal`. In the presence of `ERRP_GUARD()`, inspecting `*errp` (e.g. `if (*errp)`) is completely safe.
- **Rule**: Check the top of the function for `ERRP_GUARD()`. If present, `*errp` dereference is safe.

### Trap 4: Demanding `err != NULL` Check When Return Value Is Checked
- **Incorrect Finding**: "Caller passes `errp` but does not check whether `err != NULL` after the call."
- **Why It Is False**: QEMU error conventions mandate that functions taking `Error **errp` must return `bool` (`true` for success, `false` for error) or an integer/pointer with an error value. Callers are explicitly instructed to check the function's return value:
  ```c
  /* CORRECT QEMU IDIOM: */
  if (!foo(dev, errp)) {
      return false;
  }
  ```
- **Rule**: If the caller checks the boolean return value, do NOT flag it for not checking `*errp`.

### Trap 5: Confusing Linux Kernel Style with QEMU Style
- **False Inventions**:
  - ❌ "Single-line `if` block should omit braces `{}`." -> **FALSE**: In QEMU, braces `{}` are **MANDATORY** for all control blocks.
  - ❌ "Indentation should use 8-space tabs." -> **FALSE**: In QEMU, indentation is **4 spaces**, strictly **no tabs**.
  - ❌ "Types should use `u32` / `u64`." -> **FALSE**: QEMU uses standard C99 `uint32_t`, `uint64_t`, `bool`.
  - ❌ "Must check `g_free(p)` with `if (p != NULL)`." -> **FALSE**: `g_free(NULL)` is an explicit no-op.

### Trap 6: Defensive Programming on Trusted Internal Invariants
- **Incorrect Finding**: "Function `foo_internal()` should validate that its parameter `bar` is not NULL."
- **Why It Is False**: Internal static helper functions within a file whose callers are all local and pass non-NULL pointers do NOT need defensive NULL checks.
- **Rule**: Only demand input validation if the value originates from untrusted sources (guest OS, network, QMP user input) or if the API contract explicitly allows NULL.

### Trap 7: Assertions in Test and QTest Code
- **Incorrect Finding**: "`assert()` or `g_assert_cmpint()` could terminate QEMU."
- **Why It Is False**: Files under `tests/qtest/` or `tests/unit/` are test cases. Their entire purpose is to assert on test failures.
- **Rule**: Do not flag assertions in `tests/` unless they break test framework rules.

---

## 2. Systematic Elimination Checklist

Before including any finding in the final review, answer these five questions:

1. [ ] **Is the execution path physically reachable?**
   - Can you trace a concrete call chain from an entry point to the affected code?
   - Are there preceding checks (e.g. `if (!dev->realized)` or feature flags) that prevent reaching this state?

2. [ ] **Is the input untrusted or trusted?**
   - If untrusted (guest MMIO, guest DMA, QMP command): Is bounds checking or error handling truly missing?
   - If trusted (internal QEMU static helper): Is defensive programming being inappropriately demanded?

3. [ ] **Have you inspected the actual implementation of called functions?**
   - Did you read the real function body rather than guessing from its name or header comments?
   - Does a helper function already perform the check or acquire the lock?

4. [ ] **Does the finding respect QEMU conventions?**
   - Does it account for `ERRP_GUARD()`, `g_malloc` semantics, and BQL dispatch rules?
   - If analyzing pointer safety or proposed NULL checks, has it been checked against `pointer-guards.md`?

5. [ ] **Is this production code or test code?**
   - Is it in `hw/`, `target/`, `block/`, or in `tests/`?

---

## 3. Verdict Decision Table

| Analysis Result | Action |
|-----------------|--------|
| Proven concrete bug with reachable path | **Include in `review-inline.txt`** |
| Missing proof or unverifiable assumption | **Eliminate — do not report** |
| Defensive programming request on internal code | **Eliminate — do not report** |
| Suggesting kernel style (omitting braces, adding tabs) | **Eliminate — do not report** |
| Claiming `g_malloc` missing NULL check | **Eliminate — do not report** |
| MMIO handler lacking BQL lock | **Eliminate — do not report** |
