# Asus_TUF_z270_Mark_2__2xP40_1x1080Ti_nvidia


This document describes a workaround for running two NVIDIA Tesla P40 GPUs (24 GB each) together with one GTX 1080 Ti (11 GB) on an ASUS TUF Z270 Mark 2 motherboard.


Motherboard: Asus TUF z270 Mark 2




## The Problem

The system freezes during POST (boot initialization) because the motherboard cannot properly allocate the required MMIO address space for the two Tesla P40 cards.

And you are working with the Above_4G feature, because, it is common, that if
not enabled you see no Nvidia Cards (with hugh VRAM)

Each P40 requires a large BAR1 memory window (32GB), and the Z270 platform becomes overloaded when Above 4G Decoding is enabled.

The Resizable BAR Feature does not solve the issue on this platform.





## With one Nvidia P40 and 1080Ti

No problem, you configure in the BIOS Above_4G and all is up and running.
You see the Nvidia 1080Ti and the P40.






## The Working Solution for me

Key observations:

- Two Tesla P40 cards require very large BAR1 mappings.
- Standard Z270 "Above 4G Decoding" causes a POST freeze.
- Resizable BAR does not resolve the issue on this platform.
- BIOS upgrade, latest from ASUS, upload from changed bios, even removing bios
  lock not possible, only with flashrom, but I want no risk to turn my
  motherboard into a brick




Instead of letting the BIOS handle the large MMIO allocations:

* Disable **Above 4G Decoding** in BIOS.
* Let the system boot normally.
* Override the ACPI DSDT table.
* Create a 1-TB 64-bit MMIO window directly in the Linux kernel.
* Use GRUB parameters to force PCI resource reallocation.





---

# Step 1 - BIOS Configuration

Configure the BIOS as follows:

| Setting | Value | Purpose |
|----------|----------|----------|
| Above 4G Decoding | Disabled | Prevents POST freeze |
| Launch CSM | Disabled | Forces pure UEFI boot |
| Primary Display | PCIe or iGPU | Prevents probing of headless Tesla GPUs |
| PCIe Link Speed | Gen2 (all slots) | Improves PCIe stability under full slot population |

> Do not leave the Primary Display option set to **Auto**.

---


# Step 2 - Extract and Decompile the DSDT

Install required tools:

```bash
sudo apt install acpica-tools cpio
```

Create a work directory:

```bash
mkdir ~/dsdt_override
cd ~/dsdt_override
```

Copy the current DSDT from the running system:

```bash
sudo cp /sys/firmware/acpi/tables/DSDT ./dsdt.dat
```


Decompile the table:

```bash
iasl -d dsdt.dat
```

This generates:

```text
dsdt.dsl
```


---

# Step 3 - Modify the DSDT Source

## 3.1 Increase the Revision Number

Open:

```bash
nano dsdt.dsl
```

Find the `DefinitionBlock(...)` entry near the top of the file.

Change the revision number from:

```text
2
```

to:

```text
3
```

This helps the kernel prioritize the modified table over the original one.




---

## 3.2 Create a 1‑TB PCIe MMIO Window

Remark: change to 64Gbyte was not enough


Locate the `QWordMemory` resource inside:

```text
Device (PCI0)
```

Replace the original entry with:

```asl
QWordMemory (
    ResourceProducer,
    PosDecode,
    MinFixed,
    MaxFixed,
    NonCacheable,
    ReadWrite,
    0x0000000000000000,
    0x0000010000000000,
    0x000001FFFFFFFFFF,
    0x0000000000000000,
    0x0000010000000000,
    ,
    ,
    _Y1X,
    AddressRangeMemory,
    TypeStatic
)
```

Additionally rename:

```text
_Y1E
```

to:

```text
_Y1X
```

throughout the affected resource section.



---

# Step 4 - Build the Modified ACPI Table

Compile the updated DSDT:

```bash
iasl -sa dsdt.dsl
```

Compilation should finish with:

```text
0 Errors
```

before proceeding.

---




# Step 5 - Create the ACPI Override Initrd

Prepare the directory structure:

```bash
mkdir -p kernel/firmware/acpi
```

Copy the compiled AML:

```bash
cp dsdt.aml kernel/firmware/acpi/dsdt.aml
```

Build the override archive:

```bash
find kernel | cpio -H newc --create > acpi_override
```

Install it:

```bash
sudo cp acpi_override /boot/acpi_override
```






---

# Step 6 - Configure GRUB

Edit:

```bash
sudo nano /etc/default/grub
```

Configure:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash pci=realloc pci=nocrs"
```

Add:

```bash
GRUB_EARLY_INITRD_LINUX_CUSTOM="acpi_override"
```

Explanation:

| Parameter | Function |
|------------|-----------|
| pci=realloc | Reallocate PCI resources during boot |
| pci=nocrs | Ignore firmware PCI resource assignments |
| acpi_override | Load the custom DSDT before kernel initialization |



---

# Step 7 - Apply Changes

Update GRUB:

```bash
sudo update-grub
```

Reboot:

```bash
sudo reboot
```



---

# Technical Summary

The solution works by moving PCI resource allocation away from the BIOS and into Linux:

1. BIOS boots with reduced PCI resource expectations.
2. A custom ACPI DSDT table is loaded during early boot.
3. The DSDT exposes a 1‑TB MMIO aperture.
4. Linux reallocates PCI resources using:
   - `pci=realloc`
   - `pci=nocrs`
5. Enough MMIO space becomes available for:
   - 2 × Tesla P40
   - 1 × GTX 1080 Ti

This procedure is specifically documented for:

- ASUS TUF Z270 Mark 2
- NVIDIA Tesla P40 (24 GB)
- NVIDIA GTX 1080 Ti (11 GB)



---

# Disclaimer

This procedure modifies core ACPI firmware behavior through a DSDT override. A typo in the DSDT or an incorrect AML compilation may prevent booting until GRUB or the override is reverted. Always keep a fallback boot option available for recovery.





