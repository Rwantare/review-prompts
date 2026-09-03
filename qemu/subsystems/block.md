# QEMU Block Layer & Coroutines Subsystem Guide

The QEMU block layer (`block/`, `include/block/`) provides virtual disk emulation, image format parsing (qcow2, raw, vmdk), protocol handling (file, nbd, iscsi), and asynchronous I/O scheduling.

---

## 1. Coroutine Invariants (`coroutine_fn`)

Coroutines provide cooperative multitasking for non-blocking I/O.

### 1.1 The `coroutine_fn` Annotation

- Any function that yields (`qemu_coroutine_yield()`) or calls another function marked `coroutine_fn` **must** itself be marked `coroutine_fn`.
- **Calling Rule**: A `coroutine_fn` can **ONLY** be called from within a running coroutine. Calling a `coroutine_fn` from normal thread context causes crashes or undefined behavior.
- **Dispatch from Non-Coroutine Context**: To invoke coroutine code from synchronous context:
  ```c
  Coroutine *co = qemu_coroutine_create(my_coroutine_func, opaque);
  qemu_coroutine_enter(co);
  ```

### 1.2 Locking Inside Coroutines

- **FORBIDDEN**: Never hold a `QemuMutex` across a `qemu_coroutine_yield()`.
  - If Coroutine A yields with `QemuMutex` held, the thread moves on to run other tasks. If another coroutine or thread attempts to acquire that `QemuMutex`, it deadlocks.
- **MANDATORY**: Use `CoMutex` (`qemu_co_mutex_lock()`, `qemu_co_mutex_unlock()`) inside coroutines. When a `CoMutex` is contended, the calling coroutine yields cleanly without blocking the OS thread.
- **RAII / Lock Guards**: `QemuLockable` (`include/qemu/lockable.h`) polymorphically supports `CoMutex`. You can safely use `QEMU_LOCK_GUARD(&s->co_mutex)` and `WITH_QEMU_LOCK_GUARD(&s->co_mutex)` inside coroutines.

### 1.3 The `no_coroutine_fn` Annotation

- Functions that block synchronously or must never run within a coroutine context (such as `bdrv_graph_wrlock()`, `bdrv_unref()`, or blocking I/O) should be annotated with `no_coroutine_fn` (`include/qemu/osdep.h`).
- Calling a `no_coroutine_fn` from inside a coroutine risks blocking the underlying OS thread or corrupting coroutine state.

---

## 2. Block Graph Locking & Threading Model

QEMU annotates block layer functions to enforce thread safety:

| Annotation | Meaning | Permitted Thread Context |
|------------|---------|--------------------------|
| `GLOBAL_STATE_CODE()` | Modifies block graph, adds/removes drives | Main loop thread only (under BQL) |
| `IO_CODE()` | Fast-path read/write I/O operations | Any thread (Main loop or IOThread) |

### 2.1 Block Graph R/W Lock
- When modifying the block graph (attaching child nodes, inserting filters, taking snapshots):
  - Must run in the main loop thread with `GLOBAL_STATE_CODE()`.
  - Must acquire the graph write lock:
    ```c
    bdrv_graph_wrlock();
    /* modify bdrv children */
    bdrv_graph_wrunlock();
    ```
- **Readers of the block graph**:
  - In coroutine context: acquire `bdrv_graph_co_rdlock()` / `bdrv_graph_co_rdunlock()` (or use `GRAPH_RDLOCK_GUARD()`).
  - In main-loop non-coroutine context: acquire `bdrv_graph_rdlock_main_loop()` / `bdrv_graph_rdunlock_main_loop()`.

---

## 3. Drained Sections (`bdrv_drained_begin` / `bdrv_drained_end`)

- **What It Does**: Temporarily stops all in-flight I/O requests for a `BlockDriverState` and its children, waiting for outstanding requests to complete.
- **When Required**:
  - Before modifying the block graph (inserting or deleting filter nodes).
  - Before changing a node's `AioContext`.
  - Before closing or detaching a block backend.
- **Pattern**:
  ```c
  bdrv_drained_begin(bs);
  /* perform graph reconfiguration or detach */
  bdrv_drained_end(bs);
  ```
- **Bug Pattern**: Modifying a block node without a drained section leads to use-after-free when in-flight coroutines complete and access freed BDS structures.

---

## 4. Reentrancy Across Yield Points

In cooperative coroutines, any call to `qemu_coroutine_yield()` or a helper that yields yields control to other events:
- **State Invalidation**: The device or block node state may change while the coroutine is yielded (e.g. the disk may be detached, closed, or resized).
- **Rule**: Re-verify pointer validity, buffers, and offsets after returning from a yield point. Do NOT assume cached local variables reflect the global state across a yield.

---

## 5. Common Block Bug Checklist

- [ ] Is a `coroutine_fn` function called from a non-coroutine site?
- [ ] Is a `no_coroutine_fn` function called from within a coroutine?
- [ ] Is `QemuMutex` used instead of `CoMutex` inside a coroutine?
- [ ] Does a block graph modification occur without holding `bdrv_graph_wrlock()`?
- [ ] Is a block device detached or reconfigured without `bdrv_drained_begin()`?
- [ ] Does an I/O error path forget to decrease in-flight counters or wake waiting coroutines?
