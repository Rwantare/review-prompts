# QEMU Review Prompts for AI-Assisted Code Review

AI-assisted code review, debugging, and maintainer prompts optimized for the QEMU codebase.

## Overview

QEMU is a large, high-performance systems emulator and virtualizer written in C (along with Python, Meson, QAPI, and Rust). While sharing many low-level concepts with the Linux kernel (concurrency, hardware emulation, device drivers, mailing list review workflow), QEMU has distinct architectural frameworks, coding standards, error handling paradigms, and security boundaries.

These prompts give AI coding assistants deep context on QEMU-specific subsystems, conventions, and invariants to perform high-accuracy patch reviews, regression analysis, crash debugging, and false positive elimination.

## Quick Start

### Installation

Install the QEMU skill and slash commands for your agent using the unified setup script from the root of this repository:

```bash
./setup.sh <agent> qemu
```

Where `<agent>` is one of: `claude`, `codex`, `gemini`, `opencode`, `goose`, `kiro-cli`.

This installs:
- **QEMU Skill** (`<agent-skills-dir>/qemu/SKILL.md`) - Automatically loads context when working in a QEMU source tree.
- **Slash Commands** (`<agent-commands-dir>/`):
  - `/qemu-review` - Deep-dive regression analysis of a patch or commit.
  - `/qemu-series` - Commit-by-commit review of a patch series or pull request.
  - `/qemu-debug` - Debug QEMU crashes, assertions, and guest hangs.
  - `/qemu-verify` - Verify potential findings against false positive patterns.

## Available Slash Commands

| Command | Purpose | Input | Output |
|---------|---------|-------|--------|
| `/qemu-review` | Single commit or diff regression analysis | Commit hash, diff, or HEAD | `review-inline.txt` |
| `/qemu-series` | Multi-commit series or pull request review | Git range (e.g., `origin/master..HEAD`) | Per-commit reviews |
| `/qemu-debug` | Crash, abort, or hang investigation | Stack trace, core dump, or error log | `debug-report.txt` |
| `/qemu-verify` | False-positive check on findings | Proposed issue and code snippets | Verified / Eliminated verdict |

## File Structure

```
qemu/
├── README.md                 # This file
├── technical-patterns.md     # Core QEMU invariants and patterns (always loaded)
├── review-core.md            # Main review protocol and checklist
├── debugging.md              # Crash, assertion, and deadlock debugging protocol
├── false-positive-guide.md   # Systematic checklist to eliminate false positives
├── inline-template.md        # Plain-text review report template for qemu-devel
├── coding-style.md           # QEMU coding style rules (indentation, braces, types)
├── fixes-tag.md              # Fixes:, Resolves:, and stable tag validation
├── pointer-guards.md         # Pointer guard and condition analysis
├── maintainer-workflow.md    # Subtree management, pull requests, testing, and CI
├── skills/
│   └── qemu.md               # Skill definition template
├── slash-commands/
│   ├── qemu-review.md        # /qemu-review command definition
│   ├── qemu-series.md        # /qemu-series command definition
│   ├── qemu-debug.md         # /qemu-debug command definition
│   └── qemu-verify.md        # /qemu-verify command definition
└── subsystems/
    ├── subsystem.md          # Subsystem index and trigger table
    ├── qom.md                # QEMU Object Model (lifecycle, realize, reset)
    ├── memory.md             # Memory API (MemoryRegion, AddressSpace, DMA, IOMMU)
    ├── block.md              # Block layer (AioContext, coroutines, drained sections)
    ├── migration.md          # Live migration & VMState compatibility
    ├── virtio.md             # Virtio, vring, and vhost-user
    ├── pci.md                # PCI/PCIe device models, BARs, and MSI-X
    ├── tcg-accel.md          # TCG, KVM, CPUState, and execution loops
    └── qapi.md               # QAPI schema, QMP commands, and visitor APIs
```

## Key Differences: QEMU vs. Linux Kernel

Reviewers and AI assistants familiar with Linux kernel conventions must keep these critical differences in mind:

| Dimension | Linux Kernel | QEMU |
|-----------|--------------|------|
| **Indentation** | 8-column tabs | **4 spaces**, strictly **no tabs** |
| **Braces `{}`** | Omitted for single-line statement blocks | **Mandatory** on all `if`/`else`/`while`/`for` blocks, even 1-liners |
| **Integer Types** | `u8`, `u16`, `u32`, `u64`, `s32` | Standard C99: `uint8_t`, `uint32_t`, `uint64_t`, `bool` |
| **Structures** | `struct list_head`, `struct inode` (snake_case) | `typedef struct MyDevice MyDevice;` (**CamelCase**) |
| **Memory Allocation** | `kmalloc`, `kfree` (NULL on OOM, must check) | `g_malloc`, `g_new0` (**aborts on OOM; checking for NULL is an error, unless size is 0**) |
| **Auto Cleanup** | `__free`, `guard(mutex)` | `g_autofree`, `g_autoptr()`, `QEMU_LOCK_GUARD()` |
| **Error Handling** | Negative errno (`-EINVAL`) or `ERR_PTR()` | `Error **errp`, `ERRP_GUARD()`, functions return `bool` or int |
| **Error Output** | `pr_err()`, `dev_err()`, `printk()` | `error_report()`, `warn_report()`, `qemu_log_mask(LOG_GUEST_ERROR, ...)` |
| **Guest Boundary** | Kernel trusts hardware; checks user syscalls | **Guest OS is untrusted**: device models must **never** assert or abort on guest actions |
| **Concurrency** | Spinlocks, mutexes, RCU, preemption | **Big QEMU Lock (BQL)**, IOThreads, **Coroutines** (`coroutine_fn`), Userspace RCU |
| **State Serialization** | Hibernation/kexec | **Live Migration (`VMStateDescription`)**: strict forward/backward stream compatibility |

## Subsystem Trigger Matrix

| Changed Path or Symbol | Subsystem Guide |
|------------------------|-----------------|
| `qom/`, `TypeInfo`, `OBJECT_DECLARE_*`, `dc->realize` | `subsystems/qom.md` |
| `memory.c`, `MemoryRegion`, `AddressSpace`, `dma_memory_*` | `subsystems/memory.md` |
| `block/`, `BlockDriverState`, `coroutine_fn`, `AioContext` | `subsystems/block.md` |
| `migration/`, `VMStateDescription`, `VMSTATE_*`, `hw_compat_*` | `subsystems/migration.md` |
| `hw/virtio/`, `VirtIODevice`, `VirtQueue`, `vhost` | `subsystems/virtio.md` |
| `hw/pci/`, `PCIDevice`, `pci_register_bar`, `msi_*` | `subsystems/pci.md` |
| `accel/tcg/`, `target/`, `accel/kvm/`, `CPUState` | `subsystems/tcg-accel.md` |
| `qapi/`, `qmp_`, `*.json`, `Visitor` | `subsystems/qapi.md` |

## Semcode Integration

These prompts work seamlessly with [semcode](https://github.com/facebookexperimental/semcode) to provide fast symbol lookup, callers/calls inspection, and callchain navigation across the QEMU tree.
