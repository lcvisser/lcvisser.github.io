---
layout: post
title: "Getting the device tree for a QEMU ARM system"
tags: QEMU ARM DeviceTree
categories: QEMU ARM
---

[QEMU](https://www.qemu.org) is a machine emulator. It is capable of emulating [a lot of
targets](https://qemu-project.gitlab.io/qemu/system/targets.html), among them a variety of ARM-based boards. When
developing for Linux on ARM boards, you need a [Device Tree](https://www.devicetree.org) to tell the operating system
where all the hardware is. But also when you're doing bare-metal programming, you need to know where the hardware is.
However, there's no such documentation for the boards that QEMU can emulate.

There are two options: go through the [source code](https://github.com/qemu) of the corresponding board and find out
the memory addresses, or have QEMU output the device tree. Let's consider the [generic "virt"
board](https://qemu-project.gitlab.io/qemu/system/arm/virt.html).

## Examining the source code

The memory mappings are defined in [`hw/arm/virt.c`](https://github.com/qemu/qemu/blob/master/hw/arm/virt.c). The
relevant section is:

```
static const MemMapEntry base_memmap[] = {
    /* Space up to 0x8000000 is reserved for a boot ROM */
    [VIRT_FLASH] =              {          0, 0x08000000 },
    [VIRT_CPUPERIPHS] =         { 0x08000000, 0x00020000 },
    /* GIC distributor and CPU interfaces sit inside the CPU peripheral space */
    [VIRT_GIC_DIST] =           { 0x08000000, 0x00010000 },
    [VIRT_GIC_CPU] =            { 0x08010000, 0x00010000 },
    [VIRT_GIC_V2M] =            { 0x08020000, 0x00001000 },
    [VIRT_GIC_HYP] =            { 0x08030000, 0x00010000 },
    [VIRT_GIC_VCPU] =           { 0x08040000, 0x00010000 },
    /* The space in between here is reserved for GICv3 CPU/vCPU/HYP */
    [VIRT_GIC_ITS] =            { 0x08080000, 0x00020000 },
    /* This redistributor space allows up to 2*64kB*123 CPUs */
    [VIRT_GIC_REDIST] =         { 0x080A0000, 0x00F60000 },
    [VIRT_UART] =               { 0x09000000, 0x00001000 },
    [VIRT_RTC] =                { 0x09010000, 0x00001000 },
    [VIRT_FW_CFG] =             { 0x09020000, 0x00000018 },
    [VIRT_GPIO] =               { 0x09030000, 0x00001000 },
    [VIRT_SECURE_UART] =        { 0x09040000, 0x00001000 },
    [VIRT_SMMU] =               { 0x09050000, 0x00020000 },
    [VIRT_PCDIMM_ACPI] =        { 0x09070000, MEMORY_HOTPLUG_IO_LEN },
    [VIRT_ACPI_GED] =           { 0x09080000, ACPI_GED_EVT_SEL_LEN },
    [VIRT_NVDIMM_ACPI] =        { 0x09090000, NVDIMM_ACPI_IO_LEN},
    [VIRT_PVTIME] =             { 0x090a0000, 0x00010000 },
    [VIRT_SECURE_GPIO] =        { 0x090b0000, 0x00001000 },
    [VIRT_MMIO] =               { 0x0a000000, 0x00000200 },
    /* ...repeating for a total of NUM_VIRTIO_TRANSPORTS, each of that size */
    [VIRT_PLATFORM_BUS] =       { 0x0c000000, 0x02000000 },
    [VIRT_SECURE_MEM] =         { 0x0e000000, 0x01000000 },
    [VIRT_PCIE_MMIO] =          { 0x10000000, 0x2eff0000 },
    [VIRT_PCIE_PIO] =           { 0x3eff0000, 0x00010000 },
    [VIRT_PCIE_ECAM] =          { 0x3f000000, 0x01000000 },
    /* Actual RAM size depends on initial RAM and device memory settings */
    [VIRT_MEM] =                { GiB, LEGACY_RAMLIMIT_BYTES },
};
```

Here we find for example that there's a UART at `0x9000000`, probably the PL011 mentioned in the board description.
Probably most things can be found here, at least the ones you might be interested in.

## Device tree output from QEMU

When I said that QEMU can output the device tree, I lied: it can output the device tree blob (DTB), i.e. the compiled
version:

```
$ qemu-system-arm -M virt,dumpdtb=virt.dtb -nographic
```

This will bring up the "virt" machine, dump the DTB and then exit.

We need a device tree compiler to get a human-readable device tree source (DTS) file. You can get the compiler
[here](https://git.kernel.org/pub/scm/utils/dtc/dtc.git/), but chances are that your favorite package manager has a
package for it (e.g., I can just do `port install dtc` on MacOS). Then, to get the DTS, run:

```
$ dtc -I dtb -O dts virt.dtb >> virt.dts
```

In the resulting DTS file, we find our UART:

```
...
	pl011@9000000 {
		clock-names = "uartclk\0apb_pclk";
		clocks = <0x8000 0x8000>;
		interrupts = <0x00 0x01 0x04>;
		reg = <0x00 0x9000000 0x00 0x1000>;
		compatible = "arm,pl011\0arm,primecell";
	};
...
```
