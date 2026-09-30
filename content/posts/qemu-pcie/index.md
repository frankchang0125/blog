---
date: "2023-07-07T14:08:12+08:00"
title: "QEMU PCI/PCIe Emulation (DesignWare PCIe Host Controller)"
author: "Frank Chang"
categories:
  - "Emulator"
  - "PCIe"
tags:
  - "QEMU"
  - "PCIe"
---

- SW 透過 host controller 設定 BAR 的位址，實際上在 QEMU 中就是新增/改變對應到該 BAR memory region 的 memory subregion base address
    - BAR 空間記錄的是 PCI/PCIe 自己的 address space，而不是 memory address space，兩者實際上是不相通的。但由於大部分系統都會將 PCI/PCIe address space 映射到相同位址的 memory address space，因此”看起來”就像是 CPU 直接對 memory address space 做存取，可其實兩個 address spaces 是需要透過 host controller 做轉換的
        - CPU 存取 PCI/PCIe device 的時候，實際硬體是必須透過 host controller 來將 MMIO transaction (e.g., AXI transaction) / address 轉換成 PCI/PCIe transaction (TLP 封包) / address 才能存取到該 PCI/PCIe device 的。但 QEMU 的模擬並不會真的透過 host controller，而是直接存取前述之 BAR memory subregion
        - 反之，當 PCI/PCIe master device 做 DMA 的時候，也是需要透過 host controller 才能將其所發出的 PCI/PCIe transaction / address (TLP 封包) 轉換成 MMIO transaction (e.g., AXI transaction) / address
- 在配置 BAR 位址後，CPU 就可以透過存取 BAR 空間 (通常會被映射到 PCI/PCIe device 的 controller registers) 來控制該 PCI/PCIe device
    - e.g., PCIe BAR0 所記錄之 base address = `0x40000000`, size = `0x1000000` (16 MB)，CPU 就可以透過存取 `0x40000000 ~ 0x40FFFFFF` 來控制該 PCI/PCIe device

---

- **PCI/PCIe Device:**
    - **TYPE_PCI_DEVICE** (`struct PCIDevice`, `hw/pci/pci.c`)
        - **TYPE_DEVICE** (`struct DeviceState`, `hw/core/qdev.c`)
        - There no such **TYPE_PCIE_DEVICE**. PCIe related structure is embedded in `struct PCIDevice`:

            ```c
            struct PCIDevice {
                ...

                /* PCI Express */
                PCIExpressDevice exp;

                ....
            };
            ```

- **PCI Bus:**
    - **TYPE_PCI_BUS** (`struct PCIBus`, `hw/pci/pci.c`)
        - **TYPE_BUS** (`struct BusState`, `hw/core/bus.c`)
- **PCIe Bus:**
    - **TYPE_PCIE_BUS** (`hw/pci/pci.c`)
        - **TYPE_PCI_BUS** (`struct PCIBus`, `hw/pci/pci.c`)
            - **TYPE_BUS** (`struct BusState`, `hw/core/bus.c`)
- **CXL Bus:**
    - **TYPE_CXL_BUS** (`hw/pci/pci.c`)
        - **TYPE_PCIE_BUS** (`hw/pci/pci.c`)
            - **TYPE_PCI_BUS** (`struct PCIBus`, `hw/pci/pci.c`)
                - **TYPE_BUS** (`struct BusState`, `hw/core/bus.c`)
- **PCI-PCI Bridge** - PCI-PCI bridge itself is a `PCIDevice`, but also includes a `PCIBus`:

    ```c
    struct PCIBridge {
        /*< private >*/
        PCIDevice parent_obj;
        /*< public >*/

        /* private member */
        PCIBus sec_bus;

        ...
    };
    ```

    - **TYPE_PCI_BRIDGE** (`struct PCIBridge`, `hw/pci/pci_bridge.c`)
        - **TYPE_PCI_DEVICE** (`struct PCIDevice`, `hw/pci/pci.c`)
            - **TYPE_DEVICE** (`struct DeviceState`, `hw/core/qdev.c`)
- **PCIe-PCI Bridge:**
    - **TYPE_PCIE_PCI_BRIDGE_DEV** (`struct PCIBridgeDev`, `hw/pci-bridge/pcie_pci_bridge.c`)
        - **TYPE_PCI_BRIDGE** (`struct PCIBridge`, `hw/pci/pci_bridge.c`)
            - **TYPE_PCI_DEVICE** (`struct PCIDevice`, `hw/pci/pci.c`)
                - **TYPE_DEVICE** (`struct DeviceState`, `hw/core/qdev.c`)
- **PCIe Root Port:**
    - **TYPE_PCIE_ROOT_PORT** (`hw/pci-bridge/pcie_root_port.c`)
        - **TYPE_PCIE_SLOT**  (`struct PCIESlot`, `hw/pci/pcie_port.c`)
            - **TYPE_PCIE_PORT** (`struct PCIEPort`, `hw/pci/pcie_port.c`)
                - **TYPE_PCI_BRIDGE** (`struct PCIBridge`, `hw/pci/pci_bridge.c`)
                    - **TYPE_PCI_DEVICE** (`struct PCIDevice`, `hw/pci/pci.c`)
                        - **TYPE_DEVICE** (`struct DeviceState`, `hw/core/qdev.c`)
- **PCI Host Bridge** - For CPU to access the PCI bus, converts the memory loads/stores to PCI loads/stores. It also includes a `PCIBus`, which is created with `pci_register_root_bus()`.

    ```c
    struct PCIHostState {
        SysBusDevice busdev;

        ...

        PCIBus *bus;

        ...
    };
    ```

    - **TYPE_PCI_HOST_BRIDGE** (`struct PCIHostState`, `hw/pci/pci_host.c`)
        - **TYPE_SYS_BUS_DEVICE** (`struct SysBusDevice`,`hw/core/sysbus.c`)
- **PCIe Host Bridge** - For CPU to access the PCIe bus, converts the memory loads/stores to PCI loads/stores.
    - **TYPE_PCIE_HOST_BRIDGE** (`struct PCIExpressHost`, `hw/pci/pcie_host.c`)
        - **TYPE_PCI_HOST_BRIDGE** (`struct PCIHostState`, `hw/pci/pci_host.c`)
            - **TYPE_SYS_BUS_DEVICE** (`struct SysBusDevice`,`hw/core/sysbus.c`)

---

- DesignWare PCIe host controller (`hw/pci-host/designware.c`):
    - **TYPE_DESIGNWARE_PCIE_HOST** (`struct DesignwarePCIEHost`) - Host-facing part
        - **TYPE_PCI_HOST_BRIDGE** (`struct PCIHostState`, `hw/pci/pci_host.c`)
    - **TYPE_DESIGNWARE_PCIE_ROOT** (`struct DesignwarePCIERoot`) - PCI-facing part
        - **TYPE_PCI_BRIDGE** (`struct PCIBridge`, `hw/pci/pci_bridge.c`)
- QEMU Generic PCIe host controller (`hw/pci-host/gpex.c`)
    - **TYPE_GPEX_HOST** (`struct GPEXHost`) - Host-facing part
        - **TYPE_PCIE_HOST_BRIDGE** (`struct PCIExpressHost`, `hw/pci/pcie_host.c`)
    - **TYPE_GPEX_ROOT_DEVICE** (`struct GPEXRootState`) - PCI-facing part
        - **TYPE_PCI_DEVICE** (`struct PCIDevice`, `hw/pci/pci.c`)
- Xilinx PCIe host controller (`hw/pci-host/xilinx-pcie.c`)
    - **TYPE_XILINX_PCIE_HOST** (`struct XilinxPCIEHost`) - Host-facing part
        - **TYPE_PCIE_HOST_BRIDGE** (`struct PCIExpressHost`, `hw/pci/pcie_host.c`)
    - **TYPE_XILINX_PCIE_ROOT** (`struct XilinxPCIERoot`) - PCI-facing part
        - **TYPE_PCI_BRIDGE** (`struct PCIBridge`, `hw/pci/pci_bridge.c`)

---

```c
// include/hw/pci-host/designware.h

struct DesignwarePCIEHost {
    PCIHostState parent_obj;

    DesignwarePCIERoot root;

    struct {
        AddressSpace address_space;
        MemoryRegion address_space_root;

        MemoryRegion memory;
        MemoryRegion io;

        qemu_irq     irqs[4];
    } pci;

    MemoryRegion mmio;
};
```

```c
// include/hw/pci-host/designware.h

struct DesignwarePCIERoot {
    PCIBridge parent_obj;

    uint32_t atu_viewport;

#define DESIGNWARE_PCIE_VIEWPORT_OUTBOUND    0
#define DESIGNWARE_PCIE_VIEWPORT_INBOUND     1
#define DESIGNWARE_PCIE_NUM_VIEWPORTS        4

    DesignwarePCIEViewport viewports[2][DESIGNWARE_PCIE_NUM_VIEWPORTS];
    DesignwarePCIEMSI msi;
};
```

```c
// hw/pci-host/designware.c

static void designware_pcie_host_realize(DeviceState *dev, Error **errp)
{
    PCIHostState *pci = PCI_HOST_BRIDGE(dev);
    DesignwarePCIEHost *s = DESIGNWARE_PCIE_HOST(dev);
    SysBusDevice *sbd = SYS_BUS_DEVICE(dev);
    size_t i;

    ...

    memory_region_init(&s->pci.io, OBJECT(s), "pcie-pio", 16);

    // Initialize PCIe root sub-container MemoryRegion.
    memory_region_init(&s->pci.memory, OBJECT(s),
                       "pcie-bus-memory",
                       UINT64_MAX);

    // Create and assign DesignwarePCIEHost => PCIHostState->PCIBus
    // PCIBus's parent (DeviceState) is DesignwarePCIEHost.
    // pci.memory is the MemoryRegion for PCIe address space.
    // pci.io is the I/O MemoryRegion for PCIe address space.
    pci->bus = pci_register_root_bus(dev, "pcie",
                                     designware_pcie_set_irq,
                                     pci_swizzle_map_irq_fn,
                                     s,
                                     &s->pci.memory,
                                     &s->pci.io,
                                     0, 4,
                                     TYPE_PCIE_BUS);

    ....

    // Initialize PCIe root container MemoryRegion.
    memory_region_init(&s->pci.address_space_root,
                       OBJECT(s),
                       "pcie-bus-address-space-root",
                       UINT64_MAX);
    // Add pcie.memory as the subregion of PCIe root container MemoryRegion.
    memory_region_add_subregion(&s->pci.address_space_root,
                                0x0, &s->pci.memory);
    // Initialize PCIe root address space with root container MemoryRegion.
    address_space_init(&s->pci.address_space,
                       &s->pci.address_space_root,
                       "pcie-bus-address-space");
    // Specify pci.address_space as the address space of
    pci_setup_iommu(pci->bus, designware_pcie_host_set_iommu, s);

    // Set DesignwarePCIERoot's parent_bus to:
    // DesignwarePCIEHost => PCIHostState->PCIBus.
    qdev_realize(DEVICE(&s->root), BUS(pci->bus), &error_fatal);
}
```

```c
// hw/pci-host/designware.c

static AddressSpace *designware_pcie_host_set_iommu(PCIBus *bus, void *opaque,
                                                    int devfn)
{
    DesignwarePCIEHost *s = DESIGNWARE_PCIE_HOST(opaque);

    return &s->pci.address_space;
}
```

```c
// hw/pci/pci.c

// DesignwarePCIERoot is a PCIDevice.
static void pci_qdev_realize(DeviceState *qdev, Error **errp)
{
    PCIDevice *pci_dev = (PCIDevice *)qdev;
    PCIDeviceClass *pc = PCI_DEVICE_GET_CLASS(pci_dev);
    ObjectClass *klass = OBJECT_CLASS(pc);
    Error *local_err = NULL;
    bool is_default_rom;
    uint16_t class_id;

    ...

    // Add DesignwarePCIERoot to DesignwarePCIEHost => PCIHostState->PCIBus's
    // device list.
    pci_dev = do_pci_register_device(pci_dev,
                                     object_get_typename(OBJECT(qdev)),
                                     pci_dev->devfn, errp);
    if (pci_dev == NULL)
        return;

    // This will call designware_pcie_root_realize().
    if (pc->realize) {
        pc->realize(pci_dev, &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            do_pci_unregister_device(pci_dev);
            return;
        }
    }

    ...
}
```

```c
// hw/pci/pci.c

/* -1 for devfn means auto assign */
static PCIDevice *do_pci_register_device(PCIDevice *pci_dev,
                                         const char *name, int devfn,
                                         Error **errp)
{
    PCIDeviceClass *pc = PCI_DEVICE_GET_CLASS(pci_dev);
    PCIConfigReadFunc *config_read = pc->config_read;
    PCIConfigWriteFunc *config_write = pc->config_write;
    Error *local_err = NULL;
    DeviceState *dev = DEVICE(pci_dev);
    PCIBus *bus = pci_get_bus(pci_dev);

    ...

    pci_dev->devfn = devfn;
    pci_dev->requester_id_cache = pci_req_id_cache_get(pci_dev);
    pstrcpy(pci_dev->name, sizeof(pci_dev->name), name);

    memory_region_init(&pci_dev->bus_master_container_region, OBJECT(pci_dev),
                       "bus master container", UINT64_MAX);
    address_space_init(&pci_dev->bus_master_as,
                       &pci_dev->bus_master_container_region, pci_dev->name);

    ...

    if (!config_read)
        config_read = pci_default_read_config;
    if (!config_write)
        config_write = pci_default_write_config;
    pci_dev->config_read = config_read;
    pci_dev->config_write = config_write;
    // Add this PCIDevice to its parent bus's (PCIBus) device list.
    bus->devices[devfn] = pci_dev;
    pci_dev->version_id = 2; /* Current pci device vmstate version */
    return pci_dev;
}
```

```c
// hw/pci-host/designware.c

static DesignwarePCIEHost *
designware_pcie_root_to_host(DesignwarePCIERoot *root)
{
    // DesignwarePCIEHost => PCIHostState->PCIBus
    BusState *bus = qdev_get_parent_bus(DEVICE(root));
    // PCIBus's parent (DeviceState) is DesignwarePCIEHost
    return DESIGNWARE_PCIE_HOST(bus->parent);
}
```

```c
// hw/pci-host/designware.c

static void designware_pcie_root_realize(PCIDevice *dev, Error **errp)
{
    DesignwarePCIERoot *root = DESIGNWARE_PCIE_ROOT(dev);
    DesignwarePCIEHost *host = designware_pcie_root_to_host(root);
    MemoryRegion *address_space = &host->pci.memory;
    PCIBridge *br = PCI_BRIDGE(dev);
    DesignwarePCIEViewport *viewport;
    /*
     * Dummy values used for initial configuration of MemoryRegions
     * that belong to a given viewport
     */
    const hwaddr dummy_offset = 0;
    const uint64_t dummy_size = 4;
    size_t i;

    br->bus_name  = "dw-pcie";

    ...

    pci_bridge_initfn(dev, TYPE_PCIE_BUS);

    ...
}
```

```c
// include/hw/pci/pci_bridge.h

struct PCIBridge {
    /*< private >*/
    PCIDevice parent_obj;
    /*< public >*/

    /* private member */
    PCIBus sec_bus;
    /*
     * Memory regions for the bridge's address spaces.  These regions are not
     * directly added to system_memory/system_io or its descendants.
     * Bridge's secondary bus points to these, so that devices
     * under the bridge see these regions as its address spaces.
     * The regions are as large as the entire address space -
     * they don't take into account any windows.
     */
    MemoryRegion address_space_mem;
    MemoryRegion address_space_io;

    PCIBridgeWindows *windows;

    pci_map_irq_fn map_irq;
    const char *bus_name;
};
```

```c
// hw/pci/pci_bridge.c

void pci_bridge_initfn(PCIDevice *dev, const char *typename)
{
    // dev is DesignwarePCIERoot

    // DesignwarePCIERoot's parent_bus is:
    // DesignwarePCIEHost => PCIHostState->PCIBus.
    PCIBus *parent = pci_get_bus(dev);
    PCIBridge *br = PCI_BRIDGE(dev);
    PCIBus *sec_bus = &br->sec_bus;

    ...

    /*
     * If we don't specify the name, the bus will be addressed as &lt;id&gt;.0, where
     * id is the device id.
     * Since PCI Bridge devices have a single bus each, we don't need the index:
     * let users address the bus using the device name.
     */
    if (!br->bus_name && dev->qdev.id && *dev->qdev.id) {
            br->bus_name = dev->qdev.id;
    }

    // Initialize sec_bus.
    qbus_init(sec_bus, sizeof(br->sec_bus), typename, DEVICE(dev),
              br->bus_name);
    // sec_bus's parent_dev (PCIDevice) is DesignwarePCIERoot.
    sec_bus->parent_dev = dev;
    sec_bus->map_irq = br->map_irq ? br->map_irq : pci_swizzle_map_irq_fn;
    sec_bus->address_space_mem = &br->address_space_mem;
    memory_region_init(&br->address_space_mem, OBJECT(br), "pci_bridge_pci", UINT64_MAX);
    sec_bus->address_space_io = &br->address_space_io;
    memory_region_init(&br->address_space_io, OBJECT(br), "pci_bridge_io",
                       4 * GiB);
    br->windows = pci_bridge_region_init(br);
    QLIST_INIT(&sec_bus->child);
    // Add sec_bus to DesignwarePCIEHost => PCIHostState->PCIBus's child list.
    QLIST_INSERT_HEAD(&parent->child, sec_bus, sibling);
}
```

```text
        +--------------------------+    +--------------------------------------------------------------------------------------------------------+
        |                          |    |                                                                                                        |
        |    DesignwarePCIEHost    | <--+                                                                                                        |
        |                          |        +----------------------+   +-------------------------------------------------------------------------|----+
        +--------------------------+        |                      |   |                                                                         |    |
        | PCIHostState parent_obj; | =====> |     PCIHostState     |   |                                                                         |    |
        +--------------------------+        |                      |   |                                                                         |    |
   +--> | DesignwarePCIERoot root; | ===+   +----------------------+   |                                                                         |    |
   |    +--------------------------+    |   | SysBusDevice busdev; |   |    +------------------------------+                                     |    |
   |                                    |   +----------------------+   |    |                              |                                     |    |
   |                                    |   | PCIBus *bus;         | ==|==> |            PCIBus            |                                     |    |
   |                                    |   | ("pcie")             |   |    |                              |        +----------------------+     |    |
   |                                    |   +----------------------+   |    +------------------------------+        |                      |     |    |
   |                                    |               ^              |    | BusState qbus;               | =====> |       BusState       |     |    |
   |                                    v               |              |    +------------------------------+        |                      |     |    |
   |                        +-----------------------+   +--------------+    | PCIDevice *parent_dev;       |        +----------------------+     |    |
   |                        |                       |                       +------------------------------+        | Object obj;          |     |    |
   |                        |   DesignwarePCIERoot  |                       | QLIST_HEAD(, PCIBus) child;  | <--+   +----------------------+     |    |
   |                        |                       |                       +------------------------------+    |   | DeviceState *parent; | ----+    |
   |                        +-----------------------+                       | QLIST_ENTRY(PCIBus) sibling; |    |   +----------------------+          |
   |                        | PCIBridge parent_obj; | =============+        +------------------------------+    |                                     |
   |                        +-----------------------+              |                                            |                                     |
   |                                                               |                                            |                                     |
   |                                                               v                                            |                                     |
   |                                                   +-----------------------+                                |                                     |
   |                                                   |                       |   +----------------------------+                                     |
   |                                                   |       PCIBridge       |   |                                                                  |
   |                                                   |                       |   |    +-------------------+                                         |
   |                                                   +-----------------------+   |    |                   |                                         |
   |           +------------------------------+        | PCIDevice parent_obj; | ==|==> |     PCIDevice     |                                         |
   |           |                              |        +-----------------------+   |    |                   |        +-----------------------+        |
   |           |            PCIBus            | <===== | PCIBus sec_bus;       |   |    +-------------------+        |                       |        |
   |           |                              |        | ("dw-pcie")           |   |    | DeviceState qdev; | =====> |      DeviceState      |        |
   |           +------------------------------+        +-----------------------+   |    +-------------------+        |                       |        |
   |           | BusState qbus;               |                    |               |                                 +-----------------------+        |
   |           +------------------------------+                Added to            |                                 | Object parent_obj;    |        |
   +---------- | PCIDevice *parent_dev;       |                    |               |                                 +-----------------------+        |
               +------------------------------+                    +---------------+                                 | BusState *parent_bus; | -------+
               | QLIST_HEAD(, PCIBus) child;  |                                                                      +-----------------------+
               +------------------------------+
               | QLIST_ENTRY(PCIBus) sibling; |
               +------------------------------+
```

---

## PCIe Host Read/Write

```c
// hw/pci-host/designware.c

static void designware_pcie_host_realize(DeviceState *dev, Error **errp)
{
    ...

    memory_region_init_io(&s->mmio,
                          OBJECT(s),
                          &designware_pci_mmio_ops,
                          s,
                          "pcie.reg", 4 * 1024);
    sysbus_init_mmio(sbd, &s->mmio);

    ...
}
```

```c
// hw/pci-host/designware.c

static const MemoryRegionOps designware_pci_mmio_ops = {
    .read       = designware_pcie_host_mmio_read,
    .write      = designware_pcie_host_mmio_write,
    .endianness = DEVICE_LITTLE_ENDIAN,

    ...
    },
};
```

```c
// hw/pci-host/designware.c

static uint64_t designware_pcie_host_mmio_read(void *opaque, hwaddr addr,
                                               unsigned int size)
{
    PCIHostState *pci = PCI_HOST_BRIDGE(opaque);
    // Write to DesignwarePCIEHost will issue the PCIe config read
    // request to DesignwarePCIERoot.
    // device => DesignwarePCIERoot.
    PCIDevice *device = pci_find_device(pci->bus, 0, 0);

    return pci_host_config_read_common(device,
                                       addr,
                                       pci_config_size(device),
                                       size);
}
```

```c
// hw/pci-host/designware.c

static void designware_pcie_host_mmio_write(void *opaque, hwaddr addr,
                                            uint64_t val, unsigned int size)
{
    PCIHostState *pci = PCI_HOST_BRIDGE(opaque);
    // Read from DesignwarePCIEHost will issue the PCIe config read
    // request to DesignwarePCIERoot.
    // device => DesignwarePCIERoot.
    PCIDevice *device = pci_find_device(pci->bus, 0, 0);

    return pci_host_config_write_common(device,
                                        addr,
                                        pci_config_size(device),
                                        val, size);
}
```

```c
// hw/pci-host/designware.c

static void designware_pcie_root_class_init(ObjectClass *klass, void *data)
{
    ...

    k->config_read = designware_pcie_root_config_read;
    k->config_write = designware_pcie_root_config_write;

    ...
}
```

```c
// hw/pci-host/designware.c

static uint32_t
designware_pcie_root_config_read(PCIDevice *d, uint32_t address, int len)
{
    DesignwarePCIERoot *root = DESIGNWARE_PCIE_ROOT(d);
    DesignwarePCIEViewport *viewport =
        designware_pcie_root_get_current_viewport(root);

    uint32_t val;

    switch (address) {
    case DESIGNWARE_PCIE_PORT_LINK_CONTROL:
        /*
         * Linux guest uses this register only to configure number of
         * PCIE lane (which in our case is irrelevant) and doesn't
         * really care about the value it reads from this register
         */
        val = 0xDEADBEEF;
        break;

    ...

    case DESIGNWARE_PCIE_ATU_CR1:
    case DESIGNWARE_PCIE_ATU_CR2:
        val = viewport->cr[(address - DESIGNWARE_PCIE_ATU_CR1) /
                           sizeof(uint32_t)];
        break;

    default:
        val = pci_default_read_config(d, address, len);
        break;
    }

    return val;
}
```

```c
// hw/pci-host/designware.c

static void designware_pcie_root_config_write(PCIDevice *d, uint32_t address,
                                              uint32_t val, int len)
{
    DesignwarePCIERoot *root = DESIGNWARE_PCIE_ROOT(d);
    DesignwarePCIEHost *host = designware_pcie_root_to_host(root);
    DesignwarePCIEViewport *viewport =
        designware_pcie_root_get_current_viewport(root);

    switch (address) {
    case DESIGNWARE_PCIE_PORT_LINK_CONTROL:
    case DESIGNWARE_PCIE_LINK_WIDTH_SPEED_CONTROL:
    case DESIGNWARE_PCIE_PHY_DEBUG_R1:
        /* No-op */
        break;

    ...

    case DESIGNWARE_PCIE_ATU_CR1:
        viewport->cr[0] = val;
        break;
    case DESIGNWARE_PCIE_ATU_CR2:
        viewport->cr[1] = val;
        designware_pcie_update_viewport(root, viewport);
        break;

    default:
        pci_bridge_write_config(d, address, val, len);
        break;
    }
}
```

---

## ViewPort MEM/CFG Read/Write

```c
// hw/pci-host/designware.c

static void designware_pcie_root_realize(PCIDevice *dev, Error **errp)
{
    DesignwarePCIERoot *root = DESIGNWARE_PCIE_ROOT(dev);
    DesignwarePCIEHost *host = designware_pcie_root_to_host(root);
    MemoryRegion *address_space = &host->pci.memory;
    PCIBridge *br = PCI_BRIDGE(dev);
    DesignwarePCIEViewport *viewport;

    ...

    for (i = 0; i < DESIGNWARE_PCIE_NUM_VIEWPORTS; i++) {
        MemoryRegion *source, *destination, *mem;
        const char *direction;
        char *name;

        viewport = &root->viewports[DESIGNWARE_PCIE_VIEWPORT_INBOUND][i];
        viewport->inbound = true;
        viewport->base    = 0x0000000000000000ULL;
        viewport->target  = 0x0000000000000000ULL;
        viewport->limit   = UINT32_MAX;
        viewport->cr[0]   = DESIGNWARE_PCIE_ATU_TYPE_MEM;

        source      = &host->pci.address_space_root;
        destination = get_system_memory();
        direction   = "Inbound";

        /*
         * Configure MemoryRegion implementing PCI -> CPU memory
         * access
         */
        // Assign ViewPort's MEM MemoryRegion as the alias of System memory
        // (destination).
        // The actual offset/size will be updated in
        // designware_pcie_update_viewport().
        mem  = &viewport->mem;
        name = designware_pcie_viewport_name(direction, i, "MEM");
        memory_region_init_alias(mem, OBJECT(root), name, destination,
                                 dummy_offset, dummy_size);
        // Add ViewPort's MEM MemoryRegion as the subregion of PCIe
        // Host Controller root memory (source).
        // => PCIe Host Controller root memory -> ViewPort MEM -> System memory.
        memory_region_add_subregion_overlap(source, dummy_offset, mem, -1);
        memory_region_set_enabled(mem, false);
        g_free(name);

        viewport = &root->viewports[DESIGNWARE_PCIE_VIEWPORT_OUTBOUND][i];
        viewport->root    = root;
        viewport->inbound = false;
        viewport->base    = 0x0000000000000000ULL;
        viewport->target  = 0x0000000000000000ULL;
        viewport->limit   = UINT32_MAX;
        viewport->cr[0]   = DESIGNWARE_PCIE_ATU_TYPE_MEM;

        destination = &host->pci.memory;
        direction   = "Outbound";
        source      = get_system_memory();

        /*
         * Configure MemoryRegion implementing CPU -> PCI memory
         * access
         */
        // Assign ViewPort's MEM MemoryRegion as the alias of PCIe Host
        // Controller root memory (destination).
        // The actual offset/size will be updated in
        // designware_pcie_update_viewport().
        mem  = &viewport->mem;
        name = designware_pcie_viewport_name(direction, i, "MEM");
        memory_region_init_alias(mem, OBJECT(root), name, destination,
                                 dummy_offset, dummy_size);
        // Add ViewPort's MEM MemoryRegion as the subregion of System memory
        // (source).
        // System memory -> ViewPort MEM -> PCIe Host Controller root memory.
        memory_region_add_subregion(source, dummy_offset, mem);
        memory_region_set_enabled(mem, false);
        g_free(name);

        /*
         * Configure MemoryRegion implementing access to configuration
         * space
         */
        mem  = &viewport->cfg;
        name = designware_pcie_viewport_name(direction, i, "CFG");
        memory_region_init_io(&viewport->cfg, OBJECT(root),
                              &designware_pci_host_conf_ops,
                              viewport, name, dummy_size);
        // Add ViewPort's CFG MemoryRegion as the subregion of System memory
        // (source).
        // System memory -> ViewPort CFG -> designware_pci_host_conf_ops().
        memory_region_add_subregion(source, dummy_offset, mem);
        memory_region_set_enabled(mem, false);
        g_free(name);
    }

    /*
     * If no inbound iATU windows are configured, HW defaults to
     * letting inbound TLPs to pass in. We emulate that by exlicitly
     * configuring first inbound window to cover all of target's
     * address space.
     *
     * NOTE: This will not work correctly for the case when first
     * configured inbound window is window 0
     */
    viewport = &root->viewports[DESIGNWARE_PCIE_VIEWPORT_INBOUND][0];
    viewport->cr[1] = DESIGNWARE_PCIE_ATU_ENABLE;
    designware_pcie_update_viewport(root, viewport);

    memory_region_init_io(&root->msi.iomem, OBJECT(root),
                          &designware_pci_host_msi_ops,
                          root, "pcie-msi", 0x4);
    /*
     * We initially place MSI interrupt I/O region a adress 0 and
     * disable it. It'll be later moved to correct offset and enabled
     * in designware_pcie_root_update_msi_mapping() as a part of
     * initialization done by guest OS
     */
    memory_region_add_subregion(address_space, dummy_offset, &root->msi.iomem);
    memory_region_set_enabled(&root->msi.iomem, false);
}
```

```c
// hw/pci-host/designware.c

static const MemoryRegionOps designware_pci_host_conf_ops = {
    .read = designware_pcie_root_data_read,
    .write = designware_pcie_root_data_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .valid = {
        .min_access_size = 1,
        .max_access_size = 4,
    },
};
```

```c
// hw/pci-host/designware.c

static uint64_t designware_pcie_root_data_read(void *opaque, hwaddr addr,
                                               unsigned len)
{
    return designware_pcie_root_data_access(opaque, addr, NULL, len);
}
```

```c
// hw/pci-host/designware.c

static void designware_pcie_root_data_write(void *opaque, hwaddr addr,
                                            uint64_t val, unsigned len)
{
    designware_pcie_root_data_access(opaque, addr, &val, len);
}
```

```c
// hw/pci-host/designware.c

static uint64_t designware_pcie_root_data_access(void *opaque, hwaddr addr,
                                                 uint64_t *val, unsigned len)
{
    DesignwarePCIEViewport *viewport = opaque;
    DesignwarePCIERoot *root = viewport->root;

        // Config the PCIe devices (BDF) managed by PCIe Host Controller.
    const uint8_t busnum = DESIGNWARE_PCIE_ATU_BUS(viewport->target);
    const uint8_t devfn  = DESIGNWARE_PCIE_ATU_DEVFN(viewport->target);
    PCIBus    *pcibus    = pci_get_bus(PCI_DEVICE(root));
    PCIDevice *pcidev    = pci_find_device(pcibus, busnum, devfn);

    if (pcidev) {
        addr &= pci_config_size(pcidev) - 1;

        if (val) {
            pci_host_config_write_common(pcidev, addr,
                                         pci_config_size(pcidev),
                                         *val, len);
        } else {
            return pci_host_config_read_common(pcidev, addr,
                                               pci_config_size(pcidev),
                                               len);
        }
    }

    return UINT64_MAX;
}
```

---

```c
    dmpcie@1200000000 {
        compatible = "snps,dw-pcie";
        interrupt-map-mask = <0x00 0x00 0x00 0x07>;
        interrupt-map = <0x00 0x00 0x00 0x01 0x0f 0x10 0x04 0x00 0x00 0x00 0x02 0x0f 0x11 0x04 0x00 0x00 0x00 0x03 0x0f 0x12 0x04 0x00 0x00 0x00 0x04 0x0f 0x13 0x04>;
        interrupt-names = "inta", "intb", "intc", "msi";
        #interrupt-cells = <0x01>;
        interrupts = <0x0f 0x04 0x10 0x04 0x11 0x04 0x12 0x04 0x13 0x04>;
        interrupt-parent = <0x0f>;
        dma-ranges = <0x2000000 0x00 0x80000000 0x00 0x80000000 0x00 0x80000000>;
        ranges = <0x81000000 0x00 0x60080000 0x00 0x60080000 0x00 0x10000
            0x82000000 0x00 0x60090000 0x00 0x60090000 0x00 0xff70000
            0x82000000 0x00 0x70000000 0x00 0x70000000 0x00 0x10000000
            0xc3000000 0x20 0x00 0x20 0x00 0x01 0x00>;
        bus-range = <0x00 0xff>;
        num-lanes = <0x04>;
        dma-coherent;
        device_type = "pci";
        #size-cells = <0x02>;
        #address-cells = <0x03>;
        reg-names = "config", "dbi", "mgmt";
        reg = <0x12 0x00 0x00 0x10000000
               0x11 0x00 0x01 0x00
               0x00 0x20004000 0x00 0x1000>;
        interrupt-controller {
            interrupt-controller;
            #address-cell = <0x00>;
        };
    };
```

- `regs`:
    - `config`: Read/write the PCIe devices' configuration space registers (e.g., Vendor ID, Device ID, BAR 0, BAR 1… etc) under DW PCIe host controller
        - base: `0x12_00000000`
        - size: `0x10000000`
            - `hw/pci-host/designware.c`:
                - `viewport->cfg`
                    - `designware_pci_host_conf_ops`:
                        - .read = `designware_pcie_root_data_read()`
                        - .write = `designware_pcie_root_data_write()`
                            - `designware_pcie_root_data_access()`, get PCIe device's BDF from `viewport->target`
                                - `pci_host_config_write_common()`
                                - `pci_host_config_read_common()`
    - `dbi`: Data Bus Interface, read/write DW PCIe host controller itself's configuration space registers (e.g., Vendor ID, Device ID, BAR 0, BAR 1… etc)
        - base: `0x11_00000000`
        - size: `0x1_00000000`
            - `hw/pci-host/designware.c`:
                - `designware_pci_mmio_ops`:
                    - .read = `designware_pcie_host_mmio_read()`, Device/Function ID are hard-coded to `0`/`0` (i.e. DWPCIe host controller itself)
                        - `pci_host_config_read_common()`
                    - .write: `designware_pcie_host_mmio_write()`
                        - `pci_host_config_write_common()`
                        - `designware_pcie_root_class_init()`
                            - k->config_read = `designware_pcie_root_config_read()`
                            - k->config_write = `designware_pcie_root_config_write()`
    - `mgmt`: PHY management
        - base: `0x20004000`
        - size: `0x1000`
- `ranges`:
    - Outbound viewports (CPU address space → PCIe address space):
        - e.g., Configure BAR to store the base address (in PCI address space) of PCIe device's MMIO registers…etc. CPU can then access PCIe device's MMIO register by the CPU address. The value written to BAR is allocated by the Linux from the given memory regions specified in `ranges`.

        ```c
        destination = &host->pci.memory;
        direction   = "Outbound";
        source      = get_system_memory();
        ```

- `dma-ranges`:
    - Inbound viewports (PCIe address space → CPU address space):
        - e.g., PCIe device DMA.

        ```c
        source      = &host->pci.address_space_root;
        destination = get_system_memory();
        direction   = "Inbound";
        ```


---

- DW PCIe host controller registers root PCI bus's MemoryRegion:
    - `host->pci.address_space` (AddressSpace)
        - Container: `host->pci.address_space_root` (MemoryRegion; size: `UINT64_MAX`)
            - Aliases: `root->viewports[inbound][i]->mem` (MemoryRegion; base and size are run-time configurable)
            - Container: `host->pci.memory` (MemoryRegion; base: `0x0`, size: `UINT64_MAX`)
                - Aliases: `root->viewports[outbound][i]->mem` (MemoryRegion; base and size are run-time configurable)
                - `root->msi.iomem` (MemoryRegion; base is run-time configurable, size: `0x4`)

    ```c
    // hw/pci-host/designware.c

    static void designware_pcie_host_realize(DeviceState *dev, Error **errp)
    {
        ...

        pci->bus = pci_register_root_bus(dev, "pcie",
                                         designware_pcie_set_irq,
                                         pci_swizzle_map_irq_fn,
                                         s,
                                         &s->pci.memory,
                                         &s->pci.io,
                                         0, 4,
                                         TYPE_PCIE_BUS);

        memory_region_init(&s->pci.address_space_root,
                           OBJECT(s),
                           "pcie-bus-address-space-root",
                           UINT64_MAX);
        memory_region_add_subregion(&s->pci.address_space_root,
                                    0x0, &s->pci.memory);
        address_space_init(&s->pci.address_space,
                            &s->pci.address_space_root,
                           "pcie-bus-address-space");
        pci_setup_iommu(pci->bus, designware_pcie_host_set_iommu, s);

        qdev_realize(DEVICE(&s->root), BUS(pci->bus), &error_fatal);
    }
    ```

    ```c
    // hw/pci/pci.c

    PCIBus *pci_register_root_bus(DeviceState *parent, const char *name,
                                  pci_set_irq_fn set_irq, pci_map_irq_fn map_irq,
                                  void *irq_opaque,
                                  MemoryRegion *address_space_mem,
                                  MemoryRegion *address_space_io,
                                  uint8_t devfn_min, int nirq,
                                  const char *typename)
    {
        PCIBus *bus;

        bus = pci_root_bus_new(parent, name, address_space_mem,
                               address_space_io, devfn_min, typename);
        pci_bus_irqs(bus, set_irq, map_irq, irq_opaque, nirq);
        return bus;
    }
    ```

    ```c
    // hw/pci/pci.c

    PCIBus *pci_root_bus_new(DeviceState *parent, const char *name,
                             MemoryRegion *address_space_mem,
                             MemoryRegion *address_space_io,
                             uint8_t devfn_min, const char *typename)
    {
        PCIBus *bus;

        bus = PCI_BUS(qbus_new(typename, parent, name));
        pci_root_bus_internal_init(bus, parent, address_space_mem,
                                   address_space_io, devfn_min);
        return bus;
    }
    ```

    ```c
    // hw/pci/pci.c

    static void pci_root_bus_internal_init(PCIBus *bus, DeviceState *parent,
                                           MemoryRegion *address_space_mem,
                                           MemoryRegion *address_space_io,
                                           uint8_t devfn_min)
    {
        assert(PCI_FUNC(devfn_min) == 0);
        bus->devfn_min = devfn_min;
        bus->slot_reserved_mask = 0x0;
        bus->address_space_mem = address_space_mem;
        bus->address_space_io = address_space_io;
        bus->flags |= PCI_BUS_IS_ROOT;

        /* host bridge */
        QLIST_INIT(&bus->child);

        pci_host_bus_register(parent);
    }
    ```

- PCIe device's `bus_master_as`:
    - PCI device `bus_master_as`, e.g., DW PCIe host controller + e1000e:
        - `pci-dev->bus_master_as` (AddressSpace, e1000e):
            - Container: `pci_dev->bus_master_container_region` (MemoryRegion, e1000e)
                - Alias: `pci_dev->bus_master_enable_region` (MemoryRegion, e1000e), alias of `host->pci.address_space`'s (AddressSpace, DW PCIe) root MemoryRegion ⇒ Container: `host->pci.address_space_root` (MemoryRegion, DW PCIe, size: `UINT64_MAX`); base: `0x0`, size: `host->pci.address_space_root`'s size ⇒ `UINT64_MAX`.
                    - Aliases: `root->viewports[inbound][i]->mem` (MemoryRegion; base and size are run-time configurable)
                    - Container: `host->pci.memory` (MemoryRegion; base: `0x0`, size: `UINT64_MAX`)
                        - Aliases: `root->viewports[outbound][i]->mem` (MemoryRegion; base and size are run-time configurable)
                        - `root->msi.iomem` (MemoryRegion; base is run-time configurable, size: `0x4`)

    ```c
    // hw/pci/pci.c

    /* -1 for devfn means auto assign */
    static PCIDevice *do_pci_register_device(PCIDevice *pci_dev,
                                             const char *name, int devfn,
                                             Error **errp)
    {
        PCIDeviceClass *pc = PCI_DEVICE_GET_CLASS(pci_dev);
        PCIConfigReadFunc *config_read = pc->config_read;
        PCIConfigWriteFunc *config_write = pc->config_write;
        Error *local_err = NULL;
        DeviceState *dev = DEVICE(pci_dev);
        PCIBus *bus = pci_get_bus(pci_dev);

        ...

        memory_region_init(&pci_dev->bus_master_container_region, OBJECT(pci_dev),
                           "bus master container", UINT64_MAX);
        address_space_init(&pci_dev->bus_master_as,
                           &pci_dev->bus_master_container_region, pci_dev->name);

        if (phase_check(MACHINE_INIT_PHASE_READY)) {
            pci_init_bus_master(pci_dev);
        }
        ...

        return pci_dev;
    }
    ```

    ```c
    // hw/pci/pci.c

    static void pci_bus_realize(BusState *qbus, Error **errp)
    {
        PCIBus *bus = PCI_BUS(qbus);

        bus->machine_done.notify = pcibus_machine_done;
        qemu_add_machine_init_done_notifier(&bus->machine_done);

        vmstate_register(NULL, VMSTATE_INSTANCE_ID_ANY, &vmstate_pcibus, bus);
    }
    ```

    ```c
    // hw/pci/pci.c

    static void pcibus_machine_done(Notifier *notifier, void *data)
    {
        PCIBus *bus = container_of(notifier, PCIBus, machine_done);
        int i;

        for (i = 0; i < ARRAY_SIZE(bus->devices); ++i) {
            if (bus->devices[i]) {
                pci_init_bus_master(bus->devices[i]);
                pci_init_mem_attrs(bus->devices[i]);
            }
        }
    }
    ```

    ```c
    // hw/pci/pci.c

    static void pci_init_bus_master(PCIDevice *pci_dev)
    {
        AddressSpace *dma_as = pci_device_iommu_address_space(pci_dev);

        memory_region_init_alias(&pci_dev->bus_master_enable_region,
                                 OBJECT(pci_dev), "bus master",
                                 dma_as->root, 0, memory_region_size(dma_as->root));
        memory_region_set_enabled(&pci_dev->bus_master_enable_region, false);
        memory_region_add_subregion(&pci_dev->bus_master_container_region, 0,
                                    &pci_dev->bus_master_enable_region);
    }
    ```

    ```c
    // hw/pci/pci.c

    AddressSpace *pci_device_iommu_address_space(PCIDevice *dev)
    {
        PCIBus *bus = pci_get_bus(dev);
        PCIBus *iommu_bus = bus;
        uint8_t devfn = dev->devfn;

        while (iommu_bus && !iommu_bus->iommu_fn && iommu_bus->parent_dev) {
            PCIBus *parent_bus = pci_get_bus(iommu_bus->parent_dev);

            ...

            if (!pci_bus_is_express(iommu_bus)) {
                PCIDevice *parent = iommu_bus->parent_dev;

                if (pci_is_express(parent) &&
                    pcie_cap_get_type(parent) == PCI_EXP_TYPE_PCI_BRIDGE) {
                    devfn = PCI_DEVFN(0, 0);
                    bus = iommu_bus;
                } else {
                    devfn = parent->devfn;
                    bus = parent_bus;
                }
            }

            iommu_bus = parent_bus;
        }
        if (!pci_bus_bypass_iommu(bus) && iommu_bus && iommu_bus->iommu_fn) {
            return iommu_bus->iommu_fn(bus, iommu_bus->iommu_opaque, devfn);
        }
        return &address_space_memory;
    }
    ```

    ```c
    // hw/pci-host/designware.c

    static void designware_pcie_host_realize(DeviceState *dev, Error **errp)
    {
        ...

        memory_region_init(&s->pci.address_space_root,
                           OBJECT(s),
                           "pcie-bus-address-space-root",
                           UINT64_MAX);
        memory_region_add_subregion(&s->pci.address_space_root,
                                    0x0, &s->pci.memory);
        address_space_init(&s->pci.address_space,
                           &s->pci.address_space_root,
                           "pcie-bus-address-space");
        pci_setup_iommu(pci->bus, designware_pcie_host_set_iommu, s);

        qdev_realize(DEVICE(&s->root), BUS(pci->bus), &error_fatal);
    }
    ```

    ```c
    // hw/pci/pci.c

    void pci_setup_iommu(PCIBus *bus, PCIIOMMUFunc fn, void *opaque)
    {
        bus->iommu_fn = fn;
        bus->iommu_opaque = opaque;
    }
    ```

    ```c
    // hw/pci-host/designware.c

    static AddressSpace *designware_pcie_host_set_iommu(PCIBus *bus, void *opaque,
                                                        int devfn)
    {
        DesignwarePCIEHost *s = DESIGNWARE_PCIE_HOST(opaque);

        return &s->pci.address_space;
    }
    ```

- PCI device's DMA:

    ```c
    // hw/pci/pci.h

    /**
     * pci_dma_read: Read from an address space from PCI device.
     *
     * Return a MemTxResult indicating whether the operation succeeded
     * or failed (eg unassigned memory, device rejected the transaction,
     * IOMMU fault).  Called within RCU critical section.
     *
     * @dev: #PCIDevice doing the memory access
     * @addr: address within the #PCIDevice address space
     * @buf: buffer with the data transferred
     * @len: length of the data transferred
     */
    static inline MemTxResult pci_dma_read(PCIDevice *dev, dma_addr_t addr,
                                           void *buf, dma_addr_t len)
    {
        return pci_dma_rw(dev, addr, buf, len,
                          DMA_DIRECTION_TO_DEVICE, MEMTXATTRS_UNSPECIFIED);
    }
    ```

    ```c
    // hw/pci/pci.h

    /**
     * pci_dma_rw: Read from or write to an address space from PCI device.
     *
     * Return a MemTxResult indicating whether the operation succeeded
     * or failed (eg unassigned memory, device rejected the transaction,
     * IOMMU fault).
     *
     * @dev: #PCIDevice doing the memory access
     * @addr: address within the #PCIDevice address space
     * @buf: buffer with the data transferred
     * @len: the number of bytes to read or write
     * @dir: indicates the transfer direction
     */
    static inline MemTxResult pci_dma_rw(PCIDevice *dev, dma_addr_t addr,
                                         void *buf, dma_addr_t len,
                                         DMADirection dir, MemTxAttrs attrs)
    {
        return dma_memory_rw(pci_get_address_space(dev), addr, buf, len,
                             dir, attrs);
    }
    ```

    ```c
    // hw/pci/pci.h

    /* DMA access functions */
    static inline AddressSpace *pci_get_address_space(PCIDevice *dev)
    {
        return &dev->bus_master_as;
    }
    ```
