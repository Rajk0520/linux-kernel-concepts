# PCI and PCIe for Linux

Learning notes on PCI/PCIe architecture, enumeration, configuration space, BAR resources, and MMIO in Linux.

> **Scope and terminology.** PCIe keeps much of the PCI software model—configuration space, BDF addresses, BARs, and bridges—but changes the electrical and link-level transport. Linux APIs hide many platform-specific details. These notes describe the common model; device and platform specifications remain authoritative.

## Contents

1. [PCI and PCIe](#1-pci-and-pcie)
2. [Core terminology and topology](#2-core-terminology-and-topology)
3. [PCI versus PCIe](#3-pci-versus-pcie)
4. [PCIe generations](#4-pcie-generations)
5. [Enumeration and configuration access](#5-enumeration-and-configuration-access)
6. [PCI configuration space](#6-pci-configuration-space)
7. [PCIe protocol layers](#7-pcie-protocol-layers)
8. [PCI address spaces and BARs](#8-pci-address-spaces-and-bars)
9. [BAR sizing and assignment](#9-bar-sizing-and-assignment)
10. [Linux BAR claim and mapping](#10-linux-bar-claim-and-mapping)
11. [What happens during an MMIO access](#11-what-happens-during-an-mmio-access)
12. [Posted writes and ordering](#12-posted-writes-and-ordering)
13. [Glossary and corrections](#13-glossary-and-corrections)
14. [References](#14-references)

## 1. PCI and PCIe

**PCI (Peripheral Component Interconnect)** is a bus standard that lets a processor system discover peripherals, configure them, assign address ranges, and exchange data. Conventional PCI is a shared parallel bus: devices share bus signals, and only one transaction can use a bus segment at a time.

**PCI Express (PCIe)** is the successor interconnect. It preserves the familiar software model—configuration space, bus/device/function addresses, BARs, and bridges—but sends packetized transactions over point-to-point serial links. PCIe is not simply a faster shared parallel PCI bus.

The PCI software model describes resources and transactions. The host bridge/root complex connects that model to the CPU, memory system, and platform interrupt infrastructure.

## 2. Core terminology and topology

- **Lane:** one transmit differential pair and one receive differential pair. PCIe is full duplex; data can travel in both directions at once.
- **Link:** one or more bonded lanes between two PCIe ports. Common widths are x1, x2, x4, x8, and x16. The link negotiates a supported width and speed during link training.
- **Root Complex (RC):** the PCIe hierarchy’s connection to the processor and system memory. It contains or connects to one or more root ports and host bridges.
- **Host bridge / root bus:** the Linux/platform representation of the bridge between the CPU-side address and transaction domains and a PCI bus hierarchy. A host bridge exposes configuration access and resource windows to the PCI core.
- **Endpoint:** a device/function at the edge of the hierarchy, such as a Wi-Fi adapter, NVMe controller, or GPU. It typically implements configuration space and one or more BARs.
- **Bridge:** connects PCI buses or PCIe links. A PCIe switch contains an upstream port and downstream ports and behaves as a set of bridges for routing and enumeration.
- **Bus:** a logical PCI segment identified by a bus number. PCIe links are point-to-point, but the PCI software hierarchy still uses bus numbers.
- **Function:** an independently configurable PCI function. One physical device can expose multiple functions; each has its own configuration space and BDF.
- **BDF:** Bus, Device, Function. Linux commonly prints `domain:bus:device.function`, for example `0000:03:00.0`. The domain (also called segment) distinguishes PCI hierarchies.

Example topology:

```text
CPU / system memory
        |
  Root Complex
    /       \
Root Port  Root Port
   |          |
 Switch    Endpoint
  /   \
EP     EP
```

The root complex itself is platform-discovered, often through ACPI on x86 or Device Tree on many embedded systems. Endpoints and downstream bridges are ordinarily discovered by PCI configuration-space scanning.

## 3. PCI versus PCIe

| Aspect | Conventional PCI | PCI Express |
|---|---|---|
| Electrical structure | Shared parallel bus | Point-to-point serial links |
| Transfer | Shared bus cycles | Packetized transactions (TLPs) |
| Concurrency | One active transaction per bus segment | Independent links can carry traffic concurrently; switches route packets |
| Configuration model | Bus/device/function and configuration cycles | Same BDF software model; configuration requests transported through PCIe or accessed using ECAM |
| Interrupts | INTx pins, conventionally active-low level signaling | MSI/MSI-X memory writes; legacy INTx represented by Assert_INTx/Deassert_INTx messages |
| Software compatibility | PCI model | Retains PCI config/BAR/resource model, with PCIe capabilities added |

PCIe is backward-compatible at the software resource-model level, not electrically compatible with a conventional PCI connector/device without a bridge.

## 4. PCIe generations

Each generation generally doubles the signaling rate per lane. The values below are per direction; PCIe is full duplex, so aggregate both-direction figures are twice these values. `GT/s` is transfers per second, not directly the same as payload `GB/s`; encoding and protocol overhead reduce usable bandwidth.

| Generation | Signaling rate | Encoding / notable change | Approx. x1 bandwidth per direction after line encoding |
|---|---:|---|---:|
| 1.x | 2.5 GT/s | 8b/10b | 250 MB/s |
| 2.x | 5.0 GT/s | 8b/10b | 500 MB/s |
| 3.x | 8.0 GT/s | 128b/130b; improved encoding efficiency | 985 MB/s |
| 4.0 | 16 GT/s | 128b/130b | 1.97 GB/s |
| 5.0 | 32 GT/s | 128b/130b | 3.94 GB/s |
| 6.0 | 64 GT/s | PAM4, FLIT mode, FEC and CRC | About 7.56 GB/s before higher-layer overhead |
| 7.0 | 128 GT/s | PAM4, FLIT mode, FEC; doubles Gen 6 rate | About 15.1 GB/s before higher-layer overhead |

For an x16 link, multiply the per-lane figure by 16. The actually negotiated speed/width is bounded by the capabilities of both ends and the link quality. A Gen 5 x16 card in a Gen 4 x8 slot, for example, operates at the common supported generation and width (at most Gen 4 x8).

Generational changes include both signaling rate and, in newer generations, changes needed to preserve signal integrity and error handling. Gen 6 introduced PAM4 and fixed-size FLIT transport with forward error correction; Gen 7 doubles its signaling rate while retaining this direction. A released specification does not mean all devices or platforms implement that generation.

## 5. Enumeration and configuration access

### 5.1 Enumeration is a scan; BDF is the address

The PCI core discovers devices by probing configuration space. It begins at a known root bus, checks device/function addresses for valid Vendor ID and Device ID values, identifies bridges, assigns or reads bus numbers, and scans buses behind those bridges recursively. It doesn’t blindly scan every possible BDF in a single depth-first pass: normally, functions 1–7 are checked when function 0 indicates a multifunction device, subject to platform and kernel quirks.

A BDF identifies a function, for example `0000:03:00.0`:

- `0000`: PCI domain/segment
- `03`: bus number
- `00`: device number (slot number in classic terminology)
- `.0`: function number

There can be up to 256 bus numbers, 32 device numbers per bus, and 8 functions per device number. A bridge’s **Primary**, **Secondary**, and **Subordinate Bus Number** fields describe the bus it sits on, the bus immediately below it, and the highest bus number reachable below it. The kernel uses these to scan and route transactions through the hierarchy.

### 5.2 Configuration transactions: Type 0 and Type 1

For conventional PCI configuration cycles, a **Type 0** cycle targets a device directly attached to the local bus. A **Type 1** cycle targets a downstream bus through a PCI-to-PCI bridge; bridges forward it toward the requested bus. In PCIe, configuration requests are packetized and routed by BDF through the hierarchy; the Type 0/Type 1 distinction remains relevant to config routing/compatibility but should not be confused with the Type 0 and Type 1 *configuration-header layouts* described later.

### 5.3 CF8/CFC: x86 Configuration Mechanism #1

On x86 systems supporting legacy configuration mechanism #1, software uses two I/O port operations:

1. Write a selector to `0xCF8`, encoding enable, bus, device, function, and register offset.
2. Read or write the selected data through `0xCFC` (with byte/word offsets for narrower accesses).

This is an **access mechanism**. The BDF still chooses the function. It is not a separate discovery algorithm.

### 5.4 ECAM: Enhanced Configuration Access Mechanism

ECAM maps PCIe configuration space into a memory-mapped aperture. In the conventional ECAM layout, each bus gets 1 MiB, each device 32 KiB, each function 4 KiB. The address is computed from the ECAM base and BDF plus the register offset:

```text
ECAM address = base
             + (bus - start_bus) * 1 MiB
             + device * 32 KiB
             + function * 4 KiB
             + register_offset
```

The host bridge converts accesses to that aperture into configuration transactions. ECAM is not ordinary system RAM and does not itself enumerate devices; it is one way to access their configuration registers. On ACPI systems, the MCFG table describes ECAM base addresses and bus ranges. Linux can use platform-specific configuration mechanisms too; PCIe does not imply every OS access must use ECAM.

## 6. PCI configuration space

Each PCI function has its own configuration space. The first 256 bytes are the conventional configuration space; PCIe defines extended configuration space up to 4 KiB per function. Configuration space contains identity, control/status, BAR, capability, and interrupt-related registers.

There are two common **header layouts**:

- **Type 0 header:** endpoint/normal device.
- **Type 1 header:** PCI-to-PCI bridge.

These header types are encoded in the Header Type register. They are distinct from the Type 0/Type 1 *configuration transactions* above.

### 6.1 Important Type 0 (endpoint) header fields

| Offset | Register(s) | Purpose |
|---:|---|---|
| `0x00` | Vendor ID, Device ID | Identifies vendor and device; invalid/absent functions commonly read as all ones |
| `0x04` | Command, Status | Enables I/O decode, memory decode, bus mastering, and reports status |
| `0x08` | Revision ID, Prog IF, Subclass, Class Code | Identifies device class and programming interface |
| `0x0C` | Cache Line Size, Latency Timer, Header Type, BIST | Header type tells software which layout follows |
| `0x10`–`0x24` | BAR0–BAR5 | Describe endpoint I/O or memory resources; 64-bit BAR consumes two adjacent slots |
| `0x2C` | Subsystem Vendor ID, Subsystem ID | Board/product identity beyond the chip identity |
| `0x30` | Expansion ROM BAR | Optional expansion ROM aperture |
| `0x34` | Capabilities pointer | Links standard capabilities such as MSI and PCIe |
| `0x3C` | Interrupt Line, Interrupt Pin | Legacy platform IRQ hint and fixed INTA#–INTD# pin identity |

### 6.2 Important Type 1 (bridge) header fields

The first 16 bytes have the same identity/command/status/class/header fields. The rest describes bridge bus numbers and forwarding windows.

| Offset | Register(s) | Purpose |
|---:|---|---|
| `0x18` | Primary, Secondary, Subordinate Bus Number; Secondary Latency Timer | Defines the buses on and behind this bridge |
| `0x1C` | I/O Base, I/O Limit; Secondary Status | I/O port window forwarded downstream |
| `0x20` | Memory Base, Memory Limit | Non-prefetchable memory window forwarded downstream |
| `0x24` | Prefetchable Memory Base, Limit | Prefetchable memory window forwarded downstream |
| `0x28`, `0x2C` | Upper 32 bits of prefetchable window | Extends the prefetchable window above 4 GiB when supported |
| `0x30` | I/O Base/Limit Upper 16 bits; optional ROM BAR | Extends I/O window; bridge expansion ROM |
| `0x34` | Capabilities pointer | Standard capability list |
| `0x3C` | Interrupt Line, Interrupt Pin; Bridge Control at `0x3E` | Interrupt routing fields and bridge controls |

The PCI core configures bridge windows so downstream BAR ranges can be forwarded upstream. A BAR assignment must fit through every bridge window on its route.

The PCI **Command** register is in configuration space at offset `0x04`. Linux `pci_set_master()` sets Bus Master Enable so the function may initiate transactions such as DMA; it does not configure DMA buffers or IOVAs. Memory Space Enable and I/O Space Enable control BAR decoding in their respective address spaces.

## 7. PCIe protocol layers

PCIe is commonly described in three layers. These are conceptual protocol layers; actual implementations can combine functions in hardware.

### 7.1 Transaction Layer

Creates and consumes **Transaction Layer Packets (TLPs)** for operations such as memory reads/writes, configuration reads/writes, completions, and messages. It handles transaction addressing, attributes, byte enables, ordering, and request/completion tracking. A memory read is a request that normally receives a Completion with Data; a memory write is generally posted and does not require a completion.

### 7.2 Data Link Layer

Provides reliable delivery across one link between adjacent ports. It adds link-level sequence information and CRC (LCRC), detects errors, and uses acknowledgements/replay mechanisms. Link-level flow control uses credits so a sender doesn’t overrun the receiver’s available buffering. DLLPs carry link-management information such as acknowledgements and flow-control updates.

### 7.3 Physical Layer

Performs link training and initialization, lane configuration, serialization/deserialization, encoding/modulation, electrical signaling, and recovery. It sends and receives symbols/bits over the lane wires. PCIe link training negotiates supported speed and width.

Each link hop terminates and regenerates link-layer handling. Switches route TLPs based on address or ID; they do not turn the fabric into one shared electrical bus.

## 8. PCI address spaces and BARs

PCI functions expose three conceptual spaces:

1. **Configuration space:** identity and control registers, accessed by BDF via configuration mechanisms.
2. **Memory space:** memory-mapped device resources, accessed by CPU loads/stores through MMIO mappings or by device bus-master transactions.
3. **I/O port space:** legacy port-mapped resources, accessed using I/O instructions/APIs where the architecture supports them.

BARs (Base Address Registers) advertise a function’s I/O or memory resources. A BAR describes the size/type and stores the assigned **PCI bus address**. A memory BAR generally represents registers or device memory; it is not backed by host DRAM just because its address is in the CPU’s physical address space.

### 8.1 BAR attributes

- **I/O BAR vs Memory BAR:** BAR bit 0 identifies I/O space (1) or memory space (0).
- **32-bit vs 64-bit memory BAR:** Memory BAR bits 2:1 encode the address type. A 64-bit BAR consumes two adjacent BAR registers: the first is the low 32 bits, the next is the high 32 bits. It can be assigned above 4 GiB if platform windows and addressability permit. I/O BARs are not 64-bit BARs.
- **Prefetchable:** A property of a memory BAR. Traditionally, it promises that reads have no side effects and that certain writes can be merged; it allows safe prefetch/forwarding optimizations in parts of the PCI fabric. It does **not** mean the CPU automatically caches the BAR as normal RAM. Non-prefetchable is appropriate for side-effecting registers such as FIFO/data-pop or clear-on-read status registers. A memory-like framebuffer may be prefetchable.
- **Size:** The BAR size is a power-of-two aperture encoded by the alignment mask returned during probing. The required base alignment is normally the resource size.

## 9. BAR sizing and assignment

BAR sizing is normally performed by firmware or the PCI core during enumeration, not by the endpoint driver. The endpoint hardware implements the BAR mask. Conceptually, software temporarily writes ones, reads the mask, derives the size, then restores the original value. Real implementations must account for decode enable state, 64-bit BAR pairs, I/O versus memory type bits, and preserve/restore the original registers safely.

### 9.1 32-bit memory BAR sizing (conceptual)

1. Save the original BAR value.
2. Ensure decoding is safely disabled as required by the enumeration procedure.
3. Write all ones to the BAR and read it back.
4. Mask off the BAR attribute bits (low 4 bits for a memory BAR). The remaining implemented address bits form the mask.
5. Invert the address mask and add one to derive the aperture size.
6. Restore the original BAR and continue resource assignment.

For an I/O BAR, the low two bits are attributes and the mask/alignment rules differ. Don’t apply the memory-BAR mask formula indiscriminately.

### 9.2 64-bit memory BAR sizing (conceptual)

The low BAR indicates that it is a 64-bit BAR and the next BAR slot holds the high 32 address bits. Software saves both registers, writes all ones to both halves, reads both masks, combines them into one 64-bit mask, masks the low attribute bits, computes the size, and restores both values. The next BAR index is consumed by this 64-bit resource and is not an independent BAR.

### 9.3 Assigning an address

Once size/type are known, the resource allocator finds an appropriately aligned free interval in the host bridge’s available PCI I/O or memory windows, respecting bridge windows and address constraints. It reserves the interval in Linux’s resource tree and programs the base address into the BAR (low/high halves for 64-bit BARs).

The available windows come from platform description/initialization: on Device Tree systems, host bridge `ranges`; on ACPI systems, host bridge `_CRS` and related firmware data. These describe address windows and translations, not necessarily each endpoint’s final BAR assignment. Firmware may already have assigned BARs; Linux can retain or reassign them.

Linux APIs involved in driver-level resource use include:

- `pci_assign_resource(pdev, bar)`: ask the PCI core to assign a resource if it is unassigned; drivers seldom need to do this because the core assigns resources during enumeration.
- `pci_enable_device(pdev)`: enable the device and its I/O/memory decoding as appropriate.
- `pci_request_region(pdev, bar, name)` / `pci_request_regions()`: claim the assigned range in the kernel resource tree.
- `pci_resource_start()`, `pci_resource_len()`, `pci_resource_flags()`: inspect the CPU-visible resource start, length, and flags.
- `pci_iomap()` / `pcim_iomap()` or `pci_ioremap_bar()`: map a memory BAR into kernel virtual address space and return an `__iomem` pointer.
- `pci_iounmap()` and `pci_release_region()`: undo mapping and claim for non-managed APIs.

## 10. Linux BAR claim and mapping

Example flow (simplified):

```c
ret = pci_enable_device(pdev);
if (ret)
        return ret;

ret = pci_request_region(pdev, 0, "example");
if (ret)
        goto err_disable;

len = pci_resource_len(pdev, 0);
regs = pci_iomap(pdev, 0, 0); /* map whole BAR0 */
if (!regs)
        goto err_release;

/* Use readl()/writel() or appropriate accessors on regs. */
```

Cleanup reverses these operations: `pci_iounmap(pdev, regs)`, `pci_release_region(pdev, 0)`, `pci_disable_device(pdev)`. Prefer managed helpers such as `pcim_*` where appropriate.

The assignment is a PCI bus/CPU physical resource address. The mapping creates a **kernel virtual address** through which the CPU accesses that device resource. It does not allocate ordinary RAM to hold the BAR contents. Linux’s `__iomem` annotation reminds driver code that this pointer is I/O memory and should be accessed with I/O accessors, not normal `memcpy()` or ordinary dereferences unless a specific API documents that use.

### Example: ath12k

In the ath12k PCI path, `ATH12K_PCI_BAR_NUM` is 0. Its claim routine calls `pci_assign_resource(pdev, 0)`, `pci_enable_device()`, `pci_request_region(pdev, 0, "ath12k_pci")`, reads `pci_resource_len(pdev, 0)` into `ab->mem_len`, and maps BAR0 with `pci_iomap(pdev, 0, 0)` into `ab->mem`. That pointer is used for device register access. Cleanup calls `pci_iounmap()`, `pci_release_region()`, and `pci_disable_device()`. The DMA-mask and `pci_set_master()` calls in that path serve separate DMA/bus-master setup purposes.

## 11. What happens during an MMIO access

Suppose a driver executes `readl(regs + offset)` where `regs` came from mapping a PCI memory BAR:

1. The driver passes a **kernel virtual address** to `readl()`.
2. CPU page tables translate the virtual address to a CPU physical address, with I/O memory attributes established by the mapping API.
3. The CPU/system interconnect (for example, an SoC fabric or a CPU-to-I/O interconnect such as Intel UPI on multi-socket systems) carries the transaction to the root complex/host bridge. The exact internal fabric is platform-specific.
4. The host bridge recognizes the address as an outbound PCIe memory access and creates a Memory Read Request TLP with the PCIe address and requested length.
5. PCIe ports/switches route it to the endpoint whose BAR owns that address.
6. The endpoint decodes the offset and returns a Completion with Data TLP.
7. The completion travels back to the root complex, which returns the data through the CPU/system interconnect; `readl()` gives the value to the driver.

For `writel(value, regs + offset)`, the CPU sends an outbound Memory Write TLP. PCIe memory writes are **posted**: the CPU may continue before the endpoint has acted on the write. The endpoint eventually decodes and applies it. There is generally no Completion TLP for an ordinary memory write.

The address seen in `regs` is a virtual mapping; the BAR contains a PCI bus address. Host bridges can translate between CPU physical and PCI bus address domains. They need not be numerically identical.

## 12. Posted writes and ordering

PCI I/O port writes historically provide stronger completion behavior: the write reaches the device before the CPU proceeds past the I/O operation. MMIO writes can be **posted** in CPU, host bridge, or fabric buffers, allowing the CPU to continue before the endpoint observes them.

If software must ensure a posted MMIO write has reached the device before continuing, a common technique is a safe read from the same device after the write. The read completion cannot return until earlier ordered writes to that device have passed through the relevant path. Use the device’s documented safe readback register and the Linux I/O accessors; barriers alone and `readl()` have distinct ordering roles, so follow the device/platform documentation.

Prefetchability and write posting are related to device I/O behavior but are separate properties: prefetchability describes whether certain speculative/combined operations are safe; posting describes when a write is considered complete to the CPU.

## 13. Glossary and corrections

- **BDF is an identity/address for configuration access**, not an alternative to ECAM. ECAM or CF8/CFC is the access mechanism; BDF selects the function.
- **A BAR assignment is an address range, not RAM allocation.** For MMIO, accesses route to the endpoint or its device memory.
- **The BAR address and driver pointer are different.** The BAR stores a PCI address; `pci_iomap()` returns a CPU kernel virtual `__iomem` mapping.
- **`pci_assign_resource()` does not map the BAR.** It asks PCI core to assign a range when needed. Claiming and mapping are separate operations.
- **Prefetchable does not mean automatically CPU-cached.** It marks access properties for the PCI resource/fabric; CPU mapping attributes are controlled separately.
- **PCIe’s link is point-to-point, but its software view remains a bus hierarchy.** Bridges and switches create the tree used by BDF enumeration and routing.
- **CPU internal fabric terminology is platform-specific.** The safe generic term is CPU/system interconnect; examples include SoC fabric and Intel UPI. The root complex/host bridge translates CPU-side transactions to PCIe TLPs.


## 14. References

- Linux kernel, [How To Write Linux PCI Drivers](https://docs.kernel.org/PCI/pci.html)
- Linux kernel, [PCI Support Library](https://docs.kernel.org/driver-api/pci/pci.html)
- Linux kernel, [PCI host controller drivers](https://docs.kernel.org/PCI/controller/pci-controller-drivers.html)
- Linux kernel, [ACPI considerations for PCI host bridges](https://docs.kernel.org/PCI/acpi-info.html)
- Linux kernel, [Device I/O](https://docs.kernel.org/driver-api/device-io.html)
- PCI-SIG, [PCIe 6.0 FAQ](https://pcisig.com/faq?field_category_value%5B%5D=pci_express_6.0&keys=PAM4)
- PCI-SIG, [PCIe 7.0 FAQ](https://pcisig.com/faq?field_category_value%5B%5D=pci_express_7.0)
- PCI-SIG, [PCIe 5.0 rates and link widths](https://pcisig.com/what-bit-rates-does-pcie-50-specification-support-and-how-does-it-compare-prior-pci-express-specifications)
- PCI-SIG, [Why prefetchable terminology is changing](https://pcisig.com/PCIExpress/ECN/Base/RemovingPrefetchableTerminology)
- Linux ath12k, [`drivers/net/wireless/ath/ath12k/pci.c`](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/ath/ath12k/pci.c)
