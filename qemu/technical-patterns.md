# QEMU Technical Deep-Dive Patterns

## Core Instructions

- **Trace full execution flow**: Gather additional context from call chains and callers to confirm full understanding of device models, bus hierarchies, and execution context.
- **Never make assumptions**: Do not assume that error checks, `assert()`, comments, or return types behave as expected without inspecting the actual code path.
- **Concrete proof required**: Never report a regression or bug unless you can prove that the failing path is physically reachable and triggers incorrect behavior in practice.
- **Universal quantifiers**: When a commit or comment claims "all devices", "never called without BQL", "strictly under coroutine", verify by searching the codebase for all invocations rather than sampling only the modified files.
- **Check implementation over comments**: QEMU comments and documentation can be outdated. Always verify against the actual C implementation.
- **Code changed from review feedback**: Scrutinize reviewer-suggested changes with the same rigor as new code. Suggestions often swap helper functions while inadvertently altering locking, error propagation, or ownership semantics.

---

## 1. QEMU Object Model (QOM) Invariants

QOM is QEMU's object-oriented framework for devices and buses. Violations of QOM lifecycle rules lead to resource leaks, crashes during introspection (`-device help`), or failed migrations.

### 1.1 `instance_init` vs. `realize` Separation

- **`instance_init` (`TypeInfo.instance_init`)**:
  - Runs whenever an object is instantiated, including during machine introspection, QOM querying (`qom-list-types`), and help output (`-device <name>,help`).
  - **Permitted**: Initializing simple scalar fields, setting default property values, initializing child QOM sub-objects (`object_initialize_child`), initializing memory regions (`memory_region_init_io` without mapping them to a bus).
  - **FORBIDDEN**:
    - Allocating host OS resources (file descriptors, sockets, threads).
    - Accessing host devices, network, or filesystems.
    - Talking to other devices or buses (the bus connection does not exist yet).
    - Mapping memory regions to system address spaces.
    - Registering reset handlers (`qemu_register_reset`).
    - Exiting with `exit()` or setting `error_fatal` / `error_abort` due to user parameters.
  - **Rule**: If an allocation in `instance_init` cannot be cleaned up by `instance_finalize` without the device ever having been realized, it is a bug.

- **`realize` (`DeviceClass.realize`)**:
  - Runs when the device is attached to the machine and configured.
  - Validates user-supplied properties.
  - Connects IRQs (`sysbus_init_irq`, `qdev_init_gpio_out`), maps memory regions.
  - Allocates backend buffers, opens connections.
  - **Error Unwinding**: If `realize` fails at any point:
    - It must set `errp` (e.g. `error_setg(errp, ...)`).
    - Note on return type: `DeviceClass.realize` returns `void` (legacy pre-QOM qdev `init` returned `int`). Failure is signaled exclusively by setting `*errp` before returning.
    - **CRITICAL**: It must undo and free ALL resources allocated during that `realize` invocation before returning! QEMU will not call `unrealize` if `realize` failed.

- **`unrealize` (`DeviceClass.unrealize`)**:
  - Symmetric inverse of `realize`.
  - Must free or close everything allocated/opened in `realize`.
  - Must disconnect IRQs and unmap memory regions.

- **`instance_finalize` (`TypeInfo.instance_finalize`)**:
  - Symmetric inverse of `instance_init`.
  - Cleans up child objects or memory allocated in `instance_init`.

### 1.2 Device Reset Lifecycle (`ResettableClass`)

- Modern devices implement the `ResettableClass` interface, which divides reset into three phases:
  1. `enter`: Hold the device in reset, disable outputs, reset state registers to power-on values. No side-effects on other devices.
  2. `hold`: Devices are in reset; perform complex state transitions if needed.
  3. `exit`: Release from reset; propagate new state, trigger IRQs if active.
- **Legacy reset** (`device_class_set_legacy_reset`): Deprecated in modern QEMU (`docs/devel/reset.rst`, `include/hw/core/qdev.h`). Only sets device registers back to defaults. New device models must implement `ResettableClass`.
- **Rule**: Reset callbacks must NEVER reallocate buffers, reopen host connections, or reinitialize QOM properties. They must only reset hardware register state to power-on defaults.

---

## 2. Error Handling & `Error **errp` Rules

QEMU's standard error mechanism uses GLib-like `Error` pointers defined in `include/qapi/error.h`. Misuse of `errp` causes crashes, lost error messages, or silent failures.

### 2.1 The `ERRP_GUARD()` Macro

- **Why**: `errp` passed to a function can be `NULL` (caller ignores error), `&error_fatal` (aborts on error), or `&error_abort`. Dereferencing `*errp` when `errp == NULL` causes an instant SIGSEGV! Furthermore, `error_prepend()` and `error_append_hint()` cannot operate on `&error_fatal` without a valid intermediate error object.
- **Mandatory Rule**: If a function inspects `*errp` (e.g., `if (*errp)`), or calls `error_prepend()` / `error_append_hint()` on `errp`, it **MUST** include `ERRP_GUARD();` at the very beginning of the function:
  ```c
  bool do_something(DeviceState *dev, Error **errp)
  {
      ERRP_GUARD();

      if (!step_one(dev, errp)) {
          return false;
      }
      if (!step_two(dev, errp)) {
          error_prepend(errp, "failed step two: ");
          return false;
      }
      return true;
  }
  ```
- `ERRP_GUARD()` guarantees `errp` is non-NULL and points to a valid local pointer if the caller passed `NULL` or `&error_fatal`.
- **Calling Multiple Fallible Functions**:
  - Always check each function's return value and return immediately on failure.
  - If a function returns `bool` and simply passes `errp` to other functions returning early on failure (`if (!foo(errp)) return false;`), `ERRP_GUARD()` is **not** strictly required unless `*errp` is dereferenced or modified.
  - Never call a second function passing `errp` if the first call failed and set `*errp`; doing so triggers an immediate assertion failure (`assert(*errp == NULL)` in `error_setv`).
- **Anti-Pattern**: Do not use local `Error *local_err = NULL;` and `error_propagate(errp, local_err);` in new code. `ERRP_GUARD()` is the modern, preferred idiom.

### 2.2 Function Return Values for Error Propagation

- Functions taking `Error **errp` should return:
  - `bool`: `true` on success, `false` on failure.
  - Or a pointer: valid pointer on success, `NULL` on failure.
  - Or an integer: `>= 0` on success, `< 0` on failure.
- **Caller Rule**: Callers must check the function's return value (`if (!foo(..., errp))`), NOT `if (*errp)` or `if (err != NULL)`.
- Never call `error_setg()` if `*errp` is already set; doing so triggers an assertion failure.

### 2.3 `error_abort` and `error_fatal` Restraint

- Passing `&error_abort` or `&error_fatal` causes QEMU to immediately terminate.
- **CRITICAL**: Never pass `&error_abort` or `&error_fatal` on code paths that can be triggered by:
  - Untrusted guest execution (MMIO read/write, DMA, guest hypercalls).
  - Dynamic user commands via QMP/HMP.
  - Network packet reception.
- They are only permissible during machine initialization / startup before the guest runs, or for internal invariants that represent non-recoverable coding bugs.

### 2.4 User Error Output

- Do NOT use `printf()` or `fprintf(stderr, ...)` for error messages.
- Use:
  - `error_report()`: Report an error to the human monitor / stderr.
  - `warn_report()`: Report a warning.
  - `info_report()`: Informational status.
  - `qemu_log_mask(LOG_GUEST_ERROR, ...)`: Invalid guest operations.
  - `qemu_log_mask(LOG_UNIMP, ...)`: Unimplemented guest hardware features.

---

## 3. Memory Allocation & GLib Conventions

QEMU relies extensively on GLib (`glib.h`) for memory management, containers, and utilities.

### 3.1 Allocations that Abort on OOM

- `g_malloc()`, `g_malloc0()`, `g_new()`, `g_new0()`, `g_strdup()`:
  - **Never return NULL**. If host memory is exhausted, GLib immediately aborts the process.
  - **False Positive Alert**: Never flag code for "missing NULL check after g_malloc/g_new". These checks are dead code.
- **Fallible Allocations**:
  - When allocating memory where the size is influenced or controlled by the guest (e.g. packet buffer, firmware image size, guest command payload), use `g_try_malloc()`, `g_try_new()`, or `g_try_malloc0()`.
  - Code using `g_try_*` **MUST** explicitly check for `NULL` and handle the failure gracefully (e.g. returning `-ENOMEM` or `LOG_GUEST_ERROR`).

### 3.2 Automatic Cleanup Attributes

QEMU uses GLib's automatic cleanup macros:
- `g_autofree`: Free with `g_free()` when the variable goes out of scope.
  ```c
  g_autofree char *path = NULL;
  path = g_strdup_printf("%s/%s", dir, file);
  ```
- `g_autoptr(Type)`: Calls the cleanup function registered via `G_DEFINE_AUTOPTR_CLEANUP_FUNC()`.
  ```c
  g_autoptr(GString) buf = g_string_new(NULL);
  g_autoptr(QObject) obj = NULL;
  ```
- **Invariants for Auto-Cleanup**:
  1. **Always initialize**: Variables with `g_autofree` or `g_autoptr` must be initialized (usually to `NULL`) at declaration. Uninitialized pointers will cause `g_free()` on a garbage pointer when returning early.
  2. **Ownership transfer**: To return an allocated pointer or transfer ownership to a long-lived struct, use `g_steal_pointer(&ptr)`.
  3. **No double-free**: Never call `g_free(ptr)` explicitly on a `g_autofree` pointer without setting `ptr = NULL;`.
  4. **Do not mix allocators**: Never use `free()` on memory allocated with `g_malloc`, and never use `g_free()` on memory allocated with libc `malloc()`.

---

## 4. Hypervisor Security Boundary & Guest Isolation

The guest operating system is considered **untrusted and potentially hostile**. Any flaw that allows a guest to crash QEMU, leak host memory, or execute code on the host is a severe security vulnerability (CVE).

### 4.1 Guest Input Validation

- **No Assertions on Guest Data**:
  - An emulated device model must NEVER call `assert()`, `abort()`, or `g_assert()` based on values supplied by the guest via MMIO, PIO, DMA, configuration registers, or virtqueues.
  - If a guest writes an invalid register value or out-of-range index:
    - Log it with `qemu_log_mask(LOG_GUEST_ERROR, "...")`.
    - Ignore the write or clamp the value to valid hardware limits.
    - Set device error status bits if defined by hardware specifications.
- **Array and Buffer Indexing**:
  - Device models with register arrays (e.g. `s->regs[offset >> 2]`) must strictly validate `offset < sizeof(s->regs)` before access.
  - Multi-byte accesses: Verify `offset + size <= total_size`.
- **Loop Counters and Lengths**:
  - If a guest supplies a packet length or descriptor count, validate it against maximum buffer capacity before looping or copying.

### 4.2 DMA Safety

- Guest physical addresses (GPA) must be accessed using QEMU's DMA/AddressSpace APIs:
  - `dma_memory_read(&s->dma_as, addr, buf, len, MEMTXATTRS_UNSPECIFIED)`
  - `dma_memory_write(&s->dma_as, addr, buf, len, MEMTXATTRS_UNSPECIFIED)`
  - `pci_dma_read(pci_dev, addr, buf, len)`
- **Check Return Values**: DMA operations return `MemTxResult`. A failed DMA read/write must not leave the device in an undefined state or assume the buffer was populated.
- **DMA Reentrancy**:
  - Beware of DMA operations triggering MMIO access back into the same device model (DMA reentrancy). QEMU provides built-in DMA reentrancy protection via `dev->mem_reentrancy_guard` in `DeviceState` (enabled by default for device memory regions). Devices intentionally designed to perform reentrant IO into their own regions must explicitly set `mr->disable_reentrancy_guard = true`. Always ensure device state is consistent before initiating DMA.

### 4.3 Host Memory Disclosure

- When returning data to the guest via MMIO reads or DMA writes:
  - Every byte of the response buffer must be explicitly written or zeroed.
  - Uninitialized padding bytes in C structs written to the guest leak host stack or heap contents.
  - Use `memset(buf, 0, sizeof(*buf))` before filling fields.

---

## 5. Concurrency, Locking, and Event Loops

QEMU combines multi-threading, an event-driven main loop, coroutines, and Read-Copy-Update (RCU).

### 5.1 Big QEMU Lock (BQL)

- The BQL protects the global VM state, device emulation state, and memory region updates.
- **API**: `bql_lock()`, `bql_unlock()`, `BQL_LOCK_GUARD()` (formerly `qemu_mutex_lock_iothread()`).
- **Contexts that Hold BQL**:
  - Main loop dispatchers, timer callbacks, bottom halves (BH) by default.
  - vCPU threads executing TCG emulation.
  - MMIO read/write callbacks for devices attached to the system bus.
- **Contexts Running WITHOUT BQL**:
  - IOThreads handling block or network I/O.
  - vCPU threads executing inside KVM guest code.
  - Background worker threads (e.g. migration threads, RCU thread).
- **Invariants**:
  - Code running in an IOThread or worker thread must NEVER access BQL-protected devices or memory regions without taking the BQL or using thread-safe primitives.
  - Taking the BQL from a thread that already holds it triggers an immediate assertion failure (`g_assert(!bql_locked())`) causing `SIGABRT`. Note that `BQL_LOCK_GUARD()` safely handles conditional acquisition if already held.

### 5.2 Coroutines (`coroutine_fn` and `no_coroutine_fn`)

Coroutines provide cooperative multitasking for asynchronous I/O (especially in the block layer).

- **Marking**: Any function that calls `qemu_coroutine_yield()` or any other `coroutine_fn` must be marked with `coroutine_fn`.
- **Restrictions**:
  - `coroutine_fn` functions can ONLY be invoked from a coroutine context. Calling them from regular thread context will crash or malfunction.
  - Non-coroutine code wishing to run coroutine functions must spawn a coroutine (`qemu_coroutine_create()` + `qemu_coroutine_enter()`), run a nested event loop via `BDRV_POLL_WHILE()` / `AIO_WAIT_WHILE()`, or use generated coroutine wrappers (`co_wrapper`).
  - **`no_coroutine_fn`**: Functions that block synchronously or acquire non-coroutine locks (such as `bdrv_graph_wrlock()`) must be annotated with `no_coroutine_fn` (`include/qemu/osdep.h`). Calling a `no_coroutine_fn` from inside a coroutine risks blocking the underlying OS thread.
- **Locking in Coroutines**:
  - **NEVER** hold a `QemuMutex` across a `qemu_coroutine_yield()`. If the coroutine yields, the thread halts with the mutex held, deadlocking other coroutines or threads.
  - Inside coroutines, use `CoMutex` (`qemu_co_mutex_lock()`, `qemu_co_mutex_unlock()`) and `CoQueue`. `CoMutex` yields execution to other coroutines when contended.
  - **RAII / Lock Guards**: `QemuLockable` (`include/qemu/lockable.h`) supports `CoMutex`. You can use `QEMU_LOCK_GUARD(&co_mutex)` and `WITH_QEMU_LOCK_GUARD(&co_mutex)` safely in coroutine code.

### 5.3 Bottom Halves (BH)

- Bottom halves defer work to the event loop (`qemu_bh_schedule()`, `aio_bh_schedule_oneshot()`).
- Invariants:
  - If an object can be deleted while a BH is pending, the BH must be deleted or canceled in the destructor (`qemu_bh_delete()`).
  - BH handlers must handle cases where the device state has changed or been reset since the BH was scheduled.

### 5.4 Userspace RCU

QEMU includes a userspace RCU implementation (`include/qemu/rcu.h`) used for flat views, memory mappings, and fast lookups.
- **Readers**: Wrap accesses in `rcu_read_lock()` / `rcu_read_unlock()`.
- **Updates**:
  - Write new state with `qatomic_rcu_set()`.
  - Readers access with `qatomic_rcu_read()`.
  - Free old memory with `call_rcu()` or `g_free_rcu(obj, field)`. Note: `g_free_rcu` requires an embedded `struct rcu_head field;` at offset 0 of the allocated struct.
- **Mandatory Order**: An element must be **unlinked/removed** from the data structure FIRST, before calling `call_rcu()` or `g_free_rcu()`. Freeing before or during unlinking creates use-after-free for concurrent readers.

### 5.5 Atomics and Barriers

- Use `qatomic_*` macros (`qatomic_read`, `qatomic_set`, `qatomic_cmpxchg`, `qatomic_inc`).
- Memory barriers: `smp_mb()`, `smp_rmb()`, `smp_wmb()`, `smp_mb_acquire()`, `smp_mb_release()`.
- Do not mix plain variable access with atomic updates when concurrent threads are reading.

---

## 6. Live Migration & VMState Invariants

QEMU supports live migration of running virtual machines between different QEMU processes and versions. Stateful device models must serialize their state using `VMStateDescription`.

### 6.1 Stream Compatibility Rules

- **Never reorder or delete fields**: The migration stream is a packed binary sequence. Changing the order, size, or type of existing fields in a `VMStateDescription` breaks migration from older QEMU versions.
- **Adding New Fields**:
  - Unconditionally adding a new `VMSTATE_*` entry breaks backwards and forwards migration with existing QEMU releases.
  - **Preferred Solution (Subsections)**: Place new fields in a `VMStateDescription` subsection with a `.needed` predicate function:
    ```c
    static bool my_feature_needed(void *opaque)
    {
        MyDeviceState *s = opaque;
        return s->has_new_feature_enabled;
    }

    static const VMStateDescription vmstate_my_device_feature = {
        .name = "my-device/feature",
        .version_id = 1,
        .minimum_version_id = 1,
        .needed = my_feature_needed,
        .fields = (const VMStateField[]) {
            VMSTATE_UINT32(new_register, MyDeviceState),
            VMSTATE_END_OF_LIST()
        }
    };
    ```
    If the feature is inactive (default), the subsection is omitted from the stream, preserving compatibility.
  - **Machine Type Compatibility**: If a new feature or register must be enabled by default, tie it to a property in `hw_compat_*` arrays (in `hw/core/machine.c`) so older machine types disable the feature.

### 6.2 Validation on Load (`post_load` / `post_load_errp`)

- The migration stream comes from the network and can be corrupted or maliciously modified.
- Every stateful device that has array indices, buffer lengths, or hardware states loaded from the migration stream **MUST** validate them in a `post_load` or `post_load_errp` callback:
  ```c
  /* Modern errp variant (preferred): */
  static bool my_device_post_load_errp(void *opaque, int version_id, Error **errp)
  {
      MyDeviceState *s = opaque;
      if (s->ring_head >= MY_RING_SIZE || s->buffer_len > MAX_BUFFER_LEN) {
          error_setg(errp, "invalid ring indices: head=%u len=%u",
                     s->ring_head, s->buffer_len);
          return false; /* abort migration safely */
      }
      return true;
  }

  /* Legacy variant returning int: */
  static int my_device_post_load(void *opaque, int version_id)
  {
      MyDeviceState *s = opaque;
      if (s->ring_head >= MY_RING_SIZE || s->buffer_len > MAX_BUFFER_LEN) {
          return -EINVAL; /* abort migration safely */
      }
      return 0;
  }
  ```
- Modern code should prefer `post_load_errp`, `pre_load_errp`, and `pre_save_errp` (`docs/devel/migration/main.rst`) returning `bool` with an explicit `Error **errp`.
- Failure to validate in `post_load` / `post_load_errp` turns any migration stream corruption into a potential host arbitrary code execution vulnerability (always classify as **CRITICAL**).

---

## 7. QEMU Coding Style Requirements

QEMU follows `docs/devel/style.rst` (formerly `CODING_STYLE.rst`). Key mechanical invariants to verify:

1. **Indentation**: Exactly **4 spaces**. Tabs are forbidden anywhere in C source code (enforced by `scripts/checkpatch.pl`).
2. **Braces `{}`**: Braces are **mandatory** on all `if`, `else`, `while`, `for`, and `do` statements, even when the body contains only a single line.
3. **Types**: Use standard C99 types: `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`, `int32_t`, `int64_t`, `bool`, `size_t`. Avoid kernel-style `u32`, `s32`.
4. **Naming**:
   - Types and structs: `CamelCase` (e.g. `MyDeviceState`, `VirtQueue`).
   - Functions and variables: `snake_case` (e.g. `my_device_realize`).
   - Macros and constants: `UPPER_CASE` (e.g. `TYPE_MY_DEVICE`).
5. **Line Length**: Lines should not exceed 80 characters.
6. **Comments**: Use C-style block comments:
   ```c
   /*
    * This is a standard
    * QEMU block comment.
    */
   ```
