---
name: qemu
description: Load anytime the working directory is a QEMU tree. QEMU-specific knowledge, subsystem details, code review, debugging protocols, and maintainer workflows. Read this anytime you're in the QEMU tree.
invocation_policy: automatic
---

## ALWAYS READ
1. Load `{{QEMU_REVIEW_PROMPTS_DIR}}/technical-patterns.md`

These files are MANDATORY. This skill exists as a framework for loading additional QEMU prompts and subsystem knowledge.

## Configuration

The review prompts directory is configured during installation:
- **QEMU_REVIEW_PROMPTS_DIR**: {{QEMU_REVIEW_PROMPTS_DIR}}

This variable is set by the installation script when the skill is installed.

## Capabilities

### Patch Review
When asked to review a QEMU patch, commit, series of commits, or pull request:
1. Load `{{QEMU_REVIEW_PROMPTS_DIR}}/review-core.md`
2. Follow the complete review protocol defined there
3. Load subsystem-specific files as directed by `{{QEMU_REVIEW_PROMPTS_DIR}}/subsystems/subsystem.md`
4. If reviewing commit tags or backports, consult `{{QEMU_REVIEW_PROMPTS_DIR}}/fixes-tag.md`

### Debugging
When asked to debug a QEMU crash, assertion, guest hang, abort, or deadlock:
1. Load `{{QEMU_REVIEW_PROMPTS_DIR}}/debugging.md`
2. Follow the complete debugging protocol defined there
3. Use stack trace and crash log information as entry points into code analysis

### False Positive Verification
When validating potential bugs or regressions identified during review:
1. Load `{{QEMU_REVIEW_PROMPTS_DIR}}/false-positive-guide.md`
2. Follow the elimination checklist before reporting any issue

### Maintainer Workflows & Pull Requests
When preparing patch queues, reviewing git trees, or preparing pull requests:
1. Load `{{QEMU_REVIEW_PROMPTS_DIR}}/maintainer-workflow.md`
2. Follow the commit message guidelines, Signed-off-by requirements, and PULL request checklist

### Pointer & Condition Guards
When analyzing pointer dereferences, NULL check redundancy, or branch conditions:
1. Load `{{QEMU_REVIEW_PROMPTS_DIR}}/pointer-guards.md`
2. Follow the pointer guard evaluation rules

### Subsystem Context
When working on QEMU code in specific subsystems, load the appropriate context files from `{{QEMU_REVIEW_PROMPTS_DIR}}/`:

1. Always read `technical-patterns.md` before loading subsystem-specific files.
2. Check `{{QEMU_REVIEW_PROMPTS_DIR}}/subsystems/subsystem.md` and load matching subsystem guides:

| Subsystem | Triggers | Guide File |
|-----------|----------|------------|
| QOM & Device Lifecycle | `qom/`, `TypeInfo`, `OBJECT_DECLARE_*`, `dc->realize`, `Resettable` | `subsystems/qom.md` |
| Memory & DMA | `memory.c`, `MemoryRegion`, `AddressSpace`, `dma_memory_*`, `IOMMU` | `subsystems/memory.md` |
| Block Layer | `block/`, `BlockDriverState`, `coroutine_fn`, `AioContext`, `drained` | `subsystems/block.md` |
| Migration & VMState | `migration/`, `VMStateDescription`, `VMSTATE_*`, `hw_compat_*` | `subsystems/migration.md` |
| Virtio & Vhost | `hw/virtio/`, `VirtIODevice`, `VirtQueue`, `vring`, `vhost` | `subsystems/virtio.md` |
| PCI & PCIe | `hw/pci/`, `PCIDevice`, `pci_register_bar`, `msi_*`, `pcie_*` | `subsystems/pci.md` |
| TCG & Accelerators | `accel/tcg/`, `target/`, `accel/kvm/`, `CPUState`, `TranslationBlock` | `subsystems/tcg-accel.md` |
| QAPI & QMP | `qapi/`, `qmp_`, `*.json`, `Visitor`, `QObject` | `subsystems/qapi.md` |

## Semcode Integration

When available, use semcode MCP tools for efficient code navigation across the QEMU codebase:
- `find_function` / `find_type`: Get definitions for functions, structs, and typedefs
- `find_callchain`: Trace call relationships up and down
- `find_callers` / `find_calls`: Explore caller/callee trees
- `grep_functions`: Search function bodies with regular expressions
- `diff_functions`: Identify changed functions in patches

## Output Formats

- Patch reviews produce `review-inline.txt` formatted for `qemu-devel@nongnu.org` (plain text, 78 column wrap, concrete snippets)
- Debug sessions produce `debug-report.txt` with root cause hypotheses, reproduction notes, and recommended fixes

## Key QEMU Invariants Quick Reference

1. **Coding Style**: 4 spaces indentation (NO tabs), braces `{}` on all control blocks, C99 types (`uint32_t`, `bool`).
2. **Memory Allocation**: `g_malloc` / `g_new0` aborts on OOM — never check return for NULL (note: `g_malloc(0)` returns NULL). Use `g_try_malloc` only for guest-controlled sizes.
3. **QOM Lifecycle**: `instance_init` must never allocate host resources or register external handlers; that belongs in `realize`. Clean up on `realize` failure.
4. **Error Handling**: Functions taking `Error **errp` return `bool` (or int/pointer); callers check return value, not `*errp`. Use `ERRP_GUARD()` when dereferencing `*errp` or calling `error_prepend`/`error_append_hint`.
5. **Guest Security**: Guest OS is untrusted. Never `assert()`, `abort()`, or crash on guest actions. Use `qemu_log_mask(LOG_GUEST_ERROR, ...)`.
6. **Concurrency**: BQL covers MMIO dispatch and timers. Coroutines (`coroutine_fn`) must yield or use `CoMutex`, never block with `QemuMutex`.
7. **Migration**: Never alter existing `VMStateDescription` fields. Use subsections with `.needed` callbacks for new fields. Validate loaded state in `post_load`.
