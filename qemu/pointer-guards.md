# QEMU Pointer Guard & Condition Analysis

Defensive programming should be avoided when implicit or explicit preconditions already guarantee safety. Recommending defensive checks that can never trigger clutters the codebase and signals a lack of understanding of QEMU's control flow.

This prompt guides systematic analysis of guard conditions protecting pointers and state variables in QEMU to prevent false positive reviews.

---

## 1. Common QEMU Guard Idioms

### 1.1 The `ERRP_GUARD()` Macro
- **Idiom**:
  ```c
  bool foo(..., Error **errp)
  {
      ERRP_GUARD();
      ...
      if (*errp) { ... }
  }
  ```
- **Guard Mechanism**: If `errp` is `NULL` or `&error_fatal`, `ERRP_GUARD()` redirects `errp` to point to a valid local pointer. Therefore, `*errp` is guaranteed safe to dereference throughout the function.
- **Rule**: If `ERRP_GUARD()` is present, never report that dereferencing `*errp` could crash.

### 1.2 Device Realization Guard (`qdev_is_realized`)
- **Idiom**:
  ```c
  if (!qdev_is_realized(DEVICE(s))) {
      return;
  }
  ```
- **Guard Mechanism**: In many device models, certain callbacks (such as timer events or incoming backend data) can only occur after the device has completed `realize`.
- **Rule**: If an initialization step or buffer allocation happens in `realize`, checking `qdev_is_realized(DEVICE(s))` guards against accessing an uninitialized pointer. (Avoid direct member access `DEVICE(s)->realized` as `DeviceState` internals are encapsulated).

### 1.3 GLib Allocation Invariant
- **Idiom**:
  ```c
  s->buf = g_malloc0(s->buf_size);
  s->buf[0] = 0x42;
  ```
- **Guard Mechanism**: For any non-zero size, `g_malloc0()` guarantees that if control reaches the next line, `s->buf` is non-NULL (GLib aborts on OOM).
- **Rule**: Never flag missing NULL checks after `g_malloc()`, `g_new()`, or `g_strdup()`.
- **Exception (Zero-Size Allocations)**: In GLib, `g_malloc(0)` and `g_malloc0(0)` explicitly return `NULL`. If `s->buf_size` can evaluate to 0 (e.g. parsed from untrusted guest descriptors or config headers), `s->buf` will be `NULL`, and an immediate dereference will crash with SIGSEGV. When reviewing code with variable allocation sizes, verify that `size > 0` or that zero-length requests are rejected before allocation.

### 1.4 Queue and List Iterators
- **Idiom**:
  - `QTAILQ_FOREACH(item, &head, next)`
  - `QLIST_FOREACH(item, &head, node)`
- **Guard Mechanism**: The iterator loop condition checks `item != NULL` before each iteration.
- **Rule**: Inside the loop body, `item` is guaranteed non-NULL unless modified explicitly.

### 1.5 Caller Preconditions
- Device models, memory dispatchers, and accelerators have strict entry contracts:
  - An MMIO read/write callback receives `opaque` as the pointer registered during `memory_region_init_io()`. The memory core guarantees this pointer is non-NULL.
  - vCPU run loop helpers receive `CPUState *cpu`, guaranteed valid by the accelerator.

---

## 2. Step-by-Step Guard Tracing Workflow

When evaluating whether a pointer dereference could be unsafe:

### Step 1: Trace from Dereference to Origin
1. Identify the line where the dereference happens (e.g. `val = ptr->field;`).
2. Trace backwards to where `ptr` was assigned or passed as a parameter.
3. List all conditional branches between assignment and dereference:
   - Early returns: `if (!ptr) { return; }`
   - Assertions: `assert(ptr != NULL);`
   - Ternary operators or loop filters.

### Step 2: Trace Caller Invariants
If no local check exists:
1. Examine all direct callers of the function.
2. Does the caller already perform a check or allocate the object?
3. Is the function static and only invoked from safe call sites?
4. If ALL callers guarantee the pointer is non-NULL, the absence of a local check is NOT a bug.

### Step 3: Prove Reachability Before Reporting
- Can you construct a concrete execution path where `ptr` is actually NULL and reaches the dereference?
- If YES: Include the full call chain and conditions in `review-inline.txt`.
- If NO: Suppress the finding as a false positive.
