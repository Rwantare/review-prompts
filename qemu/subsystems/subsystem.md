# QEMU Subsystem Guide Index

Load subsystem guides based on the files, functions, and symbols modified by the patch. Each guide contains subsystem-specific invariants, API contracts, common bug patterns, and anti-patterns.

A single patch can touch multiple subsystems. When reviewing, load **every** matching subsystem guide.

---

## Subsystem Trigger Matrix

> **Path resolution**: All files in the "File" column are relative to **this file's directory**: `review-prompts/qemu/subsystems/`.

| Subsystem | Triggers (Paths, Symbols, APIs) | File |
|-----------|--------------------------------|------|
| **QOM & Devices** | `qom/`, `hw/core/`, `TypeInfo`, `OBJECT_DECLARE_*`, `OBJECT_CHECK`, `DeviceState`, `DeviceClass`, `realize`, `unrealize`, `instance_init`, `instance_finalize`, `Resettable`, `qdev_` | `qom.md` |
| **Memory & DMA** | `system/memory.c`, `include/system/memory.h`, `include/exec/memory.h`, `MemoryRegion`, `MemoryRegionOps`, `AddressSpace`, `FlatView`, `dma_memory_*`, `address_space_*`, `pci_dma_*`, `IOMMUMemoryRegion` | `memory.md` |
| **Block & Coroutines** | `block/`, `include/block/`, `BlockDriverState`, `BlockBackend`, `BdrvChild`, `coroutine_fn`, `qemu_coroutine_*`, `AioContext`, `aio_*`, `bdrv_drained_*`, `BlockJob` | `block.md` |
| **Live Migration** | `migration/`, `include/migration/`, `VMStateDescription`, `VMSTATE_*`, `post_load`, `pre_load`, `pre_save`, `hw_compat_*`, `savevm` | `migration.md` |
| **Virtio & Vhost** | `hw/virtio/`, `include/hw/virtio/`, `VirtIODevice`, `VirtQueue`, `vring`, `virtqueue_pop`, `virtqueue_push`, `vhost`, `vhost-user`, `vhost-net` | `virtio.md` |
| **PCI & PCIe** | `hw/pci/`, `include/hw/pci/`, `PCIDevice`, `PCIDeviceClass`, `pci_register_bar`, `pci_dma_*`, `msi_*`, `msix_*`, `pcie_*`, `PCIExpressHost` | `pci.md` |
| **TCG & Accelerators** | `accel/tcg/`, `target/`, `accel/kvm/`, `include/exec/cpu-common.h`, `include/system/cpus.h`, `CPUState`, `CPUArchState`, `TranslationBlock`, `gen_intermediate_code`, `tcg_gen_*`, `kvm_*` | `tcg-accel.md` |
| **QAPI & QMP** | `qapi/`, `qobject/`, `*.json`, `qmp_*`, `hmp_*`, `Visitor`, `visit_type_*`, `QObject`, `QDict`, `QList` | `qapi.md` |
