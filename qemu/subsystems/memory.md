# QEMU Memory API & DMA Subsystem Guide

The Memory API (`include/system/memory.h`, formerly `include/exec/memory.h`) models guest physical memory, memory-mapped I/O (MMIO), port I/O (PIO), and DMA address translation.

---

## 1. MemoryRegion Types & Initializations

| Region Type | Constructor | Usage |
|-------------|-------------|-------|
| **MMIO / IO** | `memory_region_init_io()` | Hardware register read/write callbacks |
| **RAM** | `memory_region_init_ram()` | Guest physical RAM |
| **ROM** | `memory_region_init_rom()` | Read-only firmware / BIOS |
| **Container** | `memory_region_init()` | Parent container holding multiple subregions |
| **Alias** | `memory_region_init_alias()` | Window or slice into another memory region |
| **IOMMU** | `memory_region_init_iommu()` | DMA remapping / translation unit |

---

## 2. `MemoryRegionOps` Invariants

Every MMIO region specifies dispatch operations via `MemoryRegionOps`:

```c
static const MemoryRegionOps my_device_ops = {
    .read = my_device_read,
    .write = my_device_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .valid = {
        .min_access_size = 1,
        .max_access_size = 4,
        .unaligned = false,
    },
    .impl = {
        .min_access_size = 4,
        .max_access_size = 4,
    },
};
```

### 2.1 Read / Write Callbacks

- Signature:
  ```c
  uint64_t (*read)(void *opaque, hwaddr addr, unsigned size);
  void (*write)(void *opaque, hwaddr addr, uint64_t val, unsigned size);
  ```
- **Invariants**:
  1. `addr` is an offset relative to the base of this memory region, NOT the system address space.
  2. `size` is in bytes (1, 2, 4, or 8).
  3. **Array Bounds**: Never use `addr` as an array index directly:
     ```c
     /* BUG: addr=0x100 indexes far beyond 4-element array */
     return s->regs[addr];

     /* CORRECT: check bounds (including access size) and scale by register width */
     if (addr + size > sizeof(s->regs)) {
         qemu_log_mask(LOG_GUEST_ERROR,
                       "%s: out of bounds 0x%" HWADDR_PRIx " (size %u)\n",
                       __func__, addr, size);
         return 0;
     }
     return s->regs[addr >> 2];
     ```
  4. **Unimplemented Registers**: Log with `qemu_log_mask(LOG_UNIMP, ...)`. Do not assert or crash.

---

## 3. DMA Operations & AddressSpace

Guest physical memory access for device DMA must use `AddressSpace` or `dma_memory_*` APIs:

```c
MemTxResult dma_memory_read(AddressSpace *as, dma_addr_t addr,
                            void *buf, dma_addr_t len,
                            MemTxAttrs attrs);

MemTxResult dma_memory_write(AddressSpace *as, dma_addr_t addr,
                             const void *buf, dma_addr_t len,
                             MemTxAttrs attrs);
```

### 3.1 DMA Invariants & Safety Rules

1. **Verify Return Value**:
   - `dma_memory_read` and `dma_memory_write` return `MemTxResult`.
   - If `res != MEMTX_OK`, the transfer failed (e.g. invalid GPA or unmapped IOMMU entry). The device must record a DMA error and must NOT assume `buf` contains valid data.
2. **Buffer Length Validation**:
   - Validate guest-provided DMA lengths against maximum buffer capacities before reading or writing.
   - Prevent integer overflow when computing `addr + len`.
3. **Prevent Host Data Leaks**:
   - When copying host memory to guest via DMA or MMIO read, ensure any padding bytes in structs are cleared with `memset()`.

---

## 4. DMA Reentrancy Attacks

- **The Problem**: A malicious guest can program a device's DMA engine to write into the device's own MMIO registers. When the device executes DMA, the write triggers the device's own MMIO write callback synchronously while internal locks or state machines are active.
- **Protection**:
  - Built-in reentrancy guard: QEMU's memory dispatch tracks reentrancy per-device via `dev->mem_reentrancy_guard` in `DeviceState` and automatically rejects or warns on reentrant calls into device MMIO. Devices intentionally designed to perform reentrant IO must explicitly opt out via `mr->disable_reentrancy_guard = true`.
  - Complete internal state updates BEFORE initiating DMA operations.

---

## 5. Teardown & Destruction

- When unmapping a region in `unrealize`:
  ```c
  memory_region_del_subregion(parent_mr, &s->mmio);
  ```
- Failure to delete subregions before freeing the device leaves dangling pointers in the global `FlatView`, resulting in use-after-free on the next memory transaction.
