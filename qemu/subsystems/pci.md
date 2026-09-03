# QEMU PCI & PCIe Subsystem Guide

The PCI subsystem (`hw/pci/`, `include/hw/pci/`) models standard PCI and PCI Express endpoints, bridges, root complexes, and host controllers.

---

## 1. `PCIDevice` Lifecycle & Realization

Devices inherit from `TYPE_PCI_DEVICE`:

```c
static void my_pci_realize(PCIDevice *pci_dev, Error **errp)
{
    MyPCIState *s = MY_PCI_DEVICE(pci_dev);
    ERRP_GUARD();

    /* 1. Initialize configuration space */
    pci_config_set_vendor_id(pci_dev->config, PCI_VENDOR_ID_MY);
    pci_config_set_device_id(pci_dev->config, PCI_DEVICE_ID_MY);

    /* 2. Initialize MMIO BAR */
    memory_region_init_io(&s->mmio, OBJECT(s), &my_pci_ops, s,
                          "my-pci-mmio", 0x1000);
    pci_register_bar(pci_dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &s->mmio);

    /* 3. Initialize MSI / MSI-X */
    if (msi_init(pci_dev, 0x00, 1, true, false, errp)) {
        return; /* Error already set */
    }
}
```

---

## 2. Base Address Registers (BARs)

- **Registration**: `pci_register_bar(PCIDevice *pci_dev, int region_num, uint8_t type, MemoryRegion *memory)`
- **BAR Types**:
  - `PCI_BASE_ADDRESS_SPACE_MEMORY` (32-bit MMIO)
  - `PCI_BASE_ADDRESS_SPACE_MEMORY | PCI_BASE_ADDRESS_MEM_TYPE_64` (64-bit MMIO)
  - `PCI_BASE_ADDRESS_SPACE_IO` (Port I/O)
- **Power of Two Invariant**: The size of a BAR memory region **must be a power of two**. Registering a non-power-of-two BAR causes an assertion failure during machine startup.

---

## 3. Interrupts: INTx, MSI, and MSI-X

### 3.1 Interrupt Mechanisms

| Type | Initialization | Cleanup | Trigger |
|------|---------------|---------|---------|
| **Legacy INTx** | `pci_config_set_interrupt_pin()` | Automatic | `pci_set_irq(dev, level)` |
| **MSI** | `msi_init()` | `msi_uninit()` | `msi_notify(dev, vector)` |
| **MSI-X** | `msix_init()` / `msix_init_exclusive_bar()` | `msix_uninit()` / `msix_uninit_exclusive_bar()` | `msix_notify(dev, vector)` |

### 3.2 Cleanup Invariant

- If `msi_init()` or `msix_init()` succeeds during `realize`:
  - It **must** be paired with `msi_uninit(dev)` / `msix_uninit(dev, table_bar, pba_bar)` (or `msix_uninit_exclusive_bar(dev)`) in `unrealize`.
  - It **must** be undone if a subsequent step in `realize` fails before returning.
  - Failure to call `msix_uninit()` or `msix_uninit_exclusive_bar()` leaks MSI-X table memory regions.

---

## 4. PCI DMA Helpers

PCI devices should use the PCI DMA wrappers (`include/hw/pci/pci_device.h`):
```c
MemTxResult pci_dma_read(PCIDevice *dev, dma_addr_t addr, void *buf, dma_addr_t len);
MemTxResult pci_dma_write(PCIDevice *dev, dma_addr_t addr, const void *buf, dma_addr_t len);
```
- Both functions return `MemTxResult` (`MEMTX_OK` on success). Callers must verify the return value rather than assuming success.
- These route DMA through the PCI device's assigned `AddressSpace` (which handles guest IOMMUs / vIOMMUs such as Intel VT-d or ARM SMMU).
- Do NOT bypass `pci_get_address_space(pci_dev)` to write directly to system physical memory, or vIOMMU protection in the guest will be completely broken.
