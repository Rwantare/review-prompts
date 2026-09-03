# QEMU Debugging Protocol

## Overview

This protocol guides systematic debugging of QEMU crashes (SIGSEGV, SIGABRT), assertion failures, deadlocks, guest/host hangs, memory corruption, and migration aborts.

---

## Pre-Debug Setup

1. ALWAYS load `technical-patterns.md` first.
2. Identify the failing subsystem (QOM, Block, Memory, TCG, KVM, Virtio, Migration, PCI).
3. Load the corresponding subsystem guide from `subsystems/`.

---

## Component & Subsystem Identification

Map crash reports or logs to relevant QEMU components:

| Binary / Context / Log Marker | Subsystem | Documentation Guide |
|-------------------------------|-----------|---------------------|
| `OBJECT_CHECK`, `qdev_`, `device_realize`, `resettable_` | QOM / Devices | `subsystems/qom.md` |
| `address_space_`, `memory_region_`, `dma_memory_`, `flatview_` | Memory API & DMA | `subsystems/memory.md` |
| `bdrv_`, `qcow2_`, `coroutine_`, `aio_`, `AioContext` | Block & Coroutines | `subsystems/block.md` |
| `vmstate_`, `migrate_`, `qemu_loadvm_`, `post_load` | Live Migration | `subsystems/migration.md` |
| `virtio_`, `vring_`, `virtqueue_`, `vhost_` | Virtio & Vhost | `subsystems/virtio.md` |
| `pci_`, `msi_`, `msix_`, `pcie_` | PCI / PCIe | `subsystems/pci.md` |
| `tcg_`, `cpu_exec`, `helper_`, `tb_lookup`, `kvm_` | Accelerators & CPU | `subsystems/tcg-accel.md` |
| `qmp_`, `qobject_`, `visit_type_`, `qapi_` | QAPI / QMP | `subsystems/qapi.md` |

---

## Debug Tasks

### DEBUG.1: Crash & Failure Information Extraction

From the bug report or backtrace, extract:
- **Crash Signal**: SIGSEGV, SIGABRT, SIGBUS, SIGFPE, or hang/deadlock.
- **Faulting instruction / address**: e.g., `0x0000000000000000` (NULL deref) or unmapped host address.
- **Full Call Stack**: All frames across active threads (`thread apply all bt full`).
- **Assertion / Error message**: Look for `assertion "..." failed`, `error_abort` trace, or GLib message.
- **QEMU Log**: Inspect `-d guest_errors,unimp` output leading up to the failure.
- **Command line**: QEMU machine type, accelerator (`-accel kvm` vs `-accel tcg`), device options.

---

### DEBUG.2: Stack Trace Analysis

For each frame in the crashing thread's call stack:
1. Identify the source file and function name.
2. Check whether the function runs with BQL held or in an IOThread.
3. Check whether the frame is inside a coroutine context (`coroutine_fn`).
4. Identify local variables and pointers passed as arguments.

---

### DEBUG.3: Root Cause Classification

Match symptoms against QEMU-specific failure patterns:

#### A. SIGSEGV (Segmentation Fault)
1. **NULL Pointer Dereference**:
   - Device model accessing uninitialized pointer from `instance_init` or missing realize step.
   - `Error **errp` dereferenced without `ERRP_GUARD()` when caller passed `NULL`.
   - Memory region accessed without being mapped to an AddressSpace.
2. **Invalid QOM Object Cast**:
   - `OBJECT_CHECK(MyType, obj, TYPE_MY_TYPE)` called on an object of a different type. In debug builds this asserts; in non-debug builds it returns invalid pointers.
3. **Use-After-Free / Double-Free**:
   - Device unparented/unrealized while pending bottom half (BH) or timer executes.
   - Pointer freed by `g_autofree` and also explicitly freed with `g_free()`.
   - RCU reader accessing an element freed without waiting for the grace period.

#### B. SIGABRT (Abort)
1. **Guest-Triggered Assertion**:
   - An emulated device called `assert()` or `abort()` on an unexpected register write from the guest.
   - **Root Cause**: Missing validation of guest input. Must be converted to `LOG_GUEST_ERROR`.
2. **`error_abort` Triggered**:
   - A function passed `&error_abort` encountered a runtime error that was not a programming invariant.
3. **Recursive BQL Acquisition**:
   - Calling `bql_lock()` from a thread that already holds the BQL triggers an immediate assertion abort: `g_assert(!bql_locked())`.
4. **GLib Abort**:
   - `g_malloc` failed due to host OOM (check if guest requested an absurd allocation size).
   - `g_assert()` failure or corrupted GLib container (`GHashTable`).

#### C. Deadlocks & Hangs
1. **Lock Order Inversion & Deadlocks**:
   - Thread lock order inversion between BQL and subsystem-specific mutexes.
   - Worker thread or IOThread waiting synchronously on main loop while main loop waits for worker thread.
2. **Coroutine Hang**:
   - `qemu_coroutine_yield()` called, but no timer, callback, or event loop handler is scheduled to call `aio_co_wake()` to resume it.
   - Coroutine blocked on a `QemuMutex` instead of `CoMutex`, halting the whole thread.
3. **vCPU Lockup**:
   - vCPU thread stuck in infinite loop in TCG generated code or failing to handle an interrupt.

#### D. Live Migration Abort
1. **Stream Desynchronization**:
   - Sender and receiver disagree on the number or size of `VMStateField` entries.
   - Missing subsection or version bump when state fields were added.
2. **`post_load` Rejection**:
   - `post_load` returned `< 0` because loaded register values or buffer indices exceeded hardware bounds.

---

### DEBUG.4: Code Path Tracing & Preconditions

1. Trace backwards from the crash point to the entry point (MMIO read/write, QMP command, packet handler, or timer).
2. What guest inputs or parameters were supplied?
3. What state prerequisites were violated?
4. Could the bug be triggered by a reentrant access (e.g. device DMA writing to its own MMIO registers)?

---

### DEBUG.5: Reproduction Analysis

Determine conditions required to reproduce:
- **Deterministic vs. Timing-dependent**: Can it be triggered by a minimal QTest testcase?
- **Configuration-dependent**: Does it require specific machine types, CPU models, or backend configurations?
- **Guest-reproducible**: Can a malicious guest kernel or user program trigger it?

---

## Output Format

Generate `debug-report.txt` with the following structure:

```
=== QEMU Debug Analysis Report ===

Summary: <One-line description of the bug>
Severity: <CRITICAL | HIGH | MEDIUM | LOW>
Subsystem: <QOM / Memory / Block / Migration / Virtio / PCI / TCG / KVM>
Crash Type: <SIGSEGV | SIGABRT | BQL Deadlock | Coroutine Hang | Migration Abort>

Root Cause:
<Detailed technical explanation of why the failure occurred, citing specific code lines>

Triggering Conditions:
<Exact prerequisites, input values, or race conditions that cause execution to reach the bug>

Call Chain:
<Entry point> -> <Caller> -> <Failing Function:Line>

Impact:
<Host crash, Guest denial-of-service, Host memory disclosure, Arbitrary code execution, Migration failure>

Proposed Fix:
<Concrete code diff or patch recommendation showing how to fix the issue cleanly according to QEMU invariants>
```
