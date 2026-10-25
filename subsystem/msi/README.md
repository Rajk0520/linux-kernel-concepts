# MSI and PCI Interrupts for Linux

Learning notes on legacy PCI interrupts, MSI/MSI-X, and interrupt handling APIs in Linux.

> **Scope.** Legacy INTx, MSI, and MSI-X are distinct ways for a PCI function to notify the CPU that it needs service. The platform routes the signal/message to a Linux IRQ; the driver handles the device-specific cause.

## Contents

1. [Interrupts: INTx, MSI, and MSI-X](#1-interrupts-intx-msi-and-msi-x)
   1. [Legacy pin-based INTx](#11-legacy-pin-based-intx)
   2. [MSI](#12-msi)
   3. [MSI-X](#13-msi-x)
   4. [Why prefer MSI/MSI-X](#14-why-prefer-msi-msi-x)
2. [MSI/MSI-X in a Linux driver](#2-msi-msi-x-in-a-linux-driver)
3. [DMA visibility and interrupt readback](#3-dma-visibility-and-interrupt-readback)
4. [Interrupt glossary](#4-interrupt-glossary)
5. [References](#5-references)

## 1. Interrupts: INTx, MSI, and MSI-X

An interrupt is a notification that a device needs CPU attention. Interrupt delivery has several layers: device signaling, host bridge/interrupt-controller routing, Linux IRQ number/domain, and the registered driver handler.

### 1.1 Legacy pin-based INTx

Each PCI function supports at most one legacy interrupt pin. Its **Interrupt Pin** configuration-space field reports which pin it uses: INTA#, INTB#, INTC#, or INTD#. This pin identity is usually fixed by the function’s hardware design; neither firmware nor a normal driver selects it. The function asserts/deasserts its interrupt when its device-specific interrupt logic needs service.

The four labels are not four processor IRQs. On conventional PCI, the signals are routed through slots/bridges and platform wiring. Bridge **swizzling** rotates pin identity as it crosses a bridge, and motherboard/platform routing maps the result to an interrupt controller input. On PCIe, legacy INTx is transported as Assert_INTx/Deassert_INTx messages to the root complex, which maps them to system IRQs. ACPI `_PRT` can describe legacy PCI interrupt routing on x86.

Two functions that both report INTA# **may** end up on the same system/Linux IRQ, but INTA# alone does not decide the route. Device location, bridge swizzling, and platform routing matter. Different INTx pins may also converge on the same IRQ.

Legacy interrupts are level-triggered and can be shared. In Linux, multiple drivers can register handlers on the same IRQ using `IRQF_SHARED` and a unique `dev_id`. When that IRQ fires, the IRQ subsystem invokes the handlers associated with it. Each handler checks a device status/cause register: it services its own device and returns `IRQ_HANDLED`, or returns `IRQ_NONE` if the interrupt came from another sharer. The device must clear/deassert its interrupt cause. A driver that leaves its interrupt asserted can cause an interrupt storm (“screaming interrupt”).

### 1.2 MSI

**Message Signaled Interrupts (MSI)** replace the legacy pin assertion with a programmed memory write. Linux allocates one or more interrupt vectors and programs the message address/data into the endpoint’s MSI capability. When an event occurs, the device emits the MSI write; the root complex/interrupt controller interprets it as an interrupt.

MSI vectors are exclusive in the sense that the allocated vector is not shared by unrelated devices. A device may have multiple MSI vectors, but classic MSI allocation requires a power-of-two contiguous vector count. MSI is still a device notification, not the data transfer itself.

### 1.3 MSI-X

MSI-X is a more flexible message-signaled scheme. The device provides a table of independently configurable vectors, allowing more vectors and less constrained allocation than MSI (up to 2048 table entries in the standard). Each vector can be individually masked and assigned an address/data message. Drivers commonly associate vectors with queues, ports, or event classes.

### 1.4 Why prefer MSI/MSI-X

- No shared physical INTx pin/IRQ handler chain in the normal allocation model; the handler knows which vector fired.
- Avoids legacy pin routing and swizzling complications.
- MSI signaling is ordered with preceding DMA writes under PCI ordering rules, avoiding the specific INTx DMA visibility race described in the Linux MSI guide.
- Multiple vectors allow parallel or purpose-specific handling (for example, queue-per-vector).
- MSI-X supports many independently configurable vectors and per-vector masking.
- Avoids some level-triggered shared-interrupt storm failure modes.

MSI/MSI-X still require correct device acknowledgment/clearing and DMA ordering discipline; they do not eliminate every race or driver bug.

## 2. MSI/MSI-X in a Linux driver

The common modern API is `pci_alloc_irq_vectors()`. The driver should allow fallback to INTx when it supports it, unless MSI-only operation is required by the hardware/driver design.

```c
int nvec;

nvec = pci_alloc_irq_vectors(pdev, 1, max_vectors,
                             PCI_IRQ_MSI | PCI_IRQ_MSIX | PCI_IRQ_INTX);
if (nvec < 0)
        return nvec;

for (int i = 0; i < nvec; i++) {
        int irq = pci_irq_vector(pdev, i);

        ret = request_irq(irq, example_irq_handler, 0,
                          "example", per_vector_context[i]);
        if (ret)
                goto err_free_irqs;
}
```

On teardown, call `free_irq(pci_irq_vector(pdev, i), dev_id)` for each successfully registered vector, then `pci_free_irq_vectors(pdev)`. Error paths must free only handlers that were successfully registered. `pci_request_irq()` is also available as a helper on supported kernels.

High-level sequence:

1. Quiesce/mask device interrupt generation and clear stale causes.
2. Request vectors using `pci_alloc_irq_vectors()` with minimum/maximum counts and allowed types.
3. Obtain each Linux IRQ number with `pci_irq_vector()`.
4. Register handlers with `request_irq()` or `pci_request_irq()`; choose the handler strategy based on whether Linux granted MSI/MSI-X or INTx if the behavior differs.
5. Program device-specific interrupt routing/queue-to-vector registers.
6. Enable device interrupt generation.
7. During teardown, disable device interrupts, synchronize/free handlers, then free vectors.

For a simple single-vector driver, set min=max=1. For MSI-X queue scaling, choose a useful maximum and handle fewer allocated vectors gracefully. Never assume the requested number was granted: use the positive return value from `pci_alloc_irq_vectors()`.

## 3. DMA visibility and interrupt readback

A device may DMA data into host memory and then signal legacy INTx. With pin-based interrupts, the CPU can begin handling the interrupt before the device's earlier DMA writes are visible to the CPU. The Linux MSI guide describes reading a safe register from the interrupting device to ensure the earlier writes have arrived before the handler consumes the DMA data. This is a device MMIO register (typically in a BAR), not normally a PCI configuration-space register. The exact register and required ordering are device-specific.

MSI signaling is a memory write and PCI ordering rules ensure preceding DMA writes are visible before the MSI is delivered. This removes that particular pin-interrupt visibility race, but drivers must still follow the DMA API and device-specific ordering requirements.

## 4. Interrupt glossary

- **INTA#–INTD# are pin identities**, not four processor IRQ lines. Each function reports one fixed legacy pin in its configuration space; platform routing and bridge swizzling determine the resulting system IRQ.
- **Shared IRQs use a handler chain.** Linux invokes each handler registered for a shared IRQ; each driver checks its own device's interrupt status, typically in a BAR-mapped register, and returns `IRQ_HANDLED` or `IRQ_NONE`.
- **MSI/MSI-X use message writes**, not a shared legacy pin. Linux maps allocated vectors to IRQ numbers and dispatches the matching handler.

## 5. References

- Linux kernel, [How To Write Linux PCI Drivers](https://docs.kernel.org/PCI/pci.html)
- Linux kernel, [MSI Driver Guide HOWTO](https://docs.kernel.org/PCI/msi-howto.html)
- Linux kernel, [PCI Support Library](https://docs.kernel.org/driver-api/pci/pci.html)
- Linux kernel, [PCI host controller drivers](https://docs.kernel.org/PCI/controller/pci-controller-drivers.html)
- Linux kernel, [ACPI considerations for PCI host bridges](https://docs.kernel.org/PCI/acpi-info.html)
- Linux kernel, [Generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html)

