# linux-cachyos-kvm-stealth

Patched CachyOS kernel and QEMU for KVM GPU passthrough stealth.
Reduces VM detection from 12/92 to 7/92 on pafish/vmmdetect.

**Author:** [MarekSurek](https://github.com/MarekSurek)  
**License:** GPL-2.0  
**Base:** CachyOS Linux 6.19.7-1

---

## What this does

- Hides KVM signature (`KVMKVMKVM` → `AuthenticAMD`)
- Zeros KVM CPUID features leaf
- Patches ACPI OEM strings (`BOCHS`/`QEMU` → `ALASKA`/`A M I`)
- Patches SMBIOS strings
- AMD SVM TSC ratio tweak to trigger RDTSC intercept
- RDTSC interception handler to mask VM exit timing

## Hardware tested

- CPU: AMD Ryzen 5 3600
- Motherboard: ASUS TUF GAMING X570-PLUS
- Host GPU: Radeon RX 550
- Guest GPU: Radeon RX 6900 XT (passthrough)
- Host OS: CachyOS (Arch Linux)

## Results

| Tool | Before | After |
|------|--------|-------|
| pafish | 12/92 | 7/92 |
| vmmdetect | 10/92 | 7/92 |
| DFT Pro | ❌ | ✅ |
| Fortnite EAC | ❌ | ❌ (work in progress) |

## Patches

### kvm-stealth-kernel.patch
Applied to CachyOS kernel 6.19.7-1 source.

Changes:
- `arch/x86/include/uapi/asm/kvm_para.h` — KVM_SIGNATURE → AuthenticAMD
- `arch/x86/kvm/cpuid.c` — zero KVM_CPUID_FEATURES and KVM_CPUID_SIGNATURE
- `arch/x86/kvm/svm/svm.c` — RDTSC intercept handler + INTERCEPT_RDTSC
- `arch/x86/kvm/svm/svm.h` — TSC ratio tweak
- `arch/x86/include/asm/svm.h` — SVM_TSC_RATIO_DEFAULT tweak
- `arch/x86/kvm/x86.c` — TSC noise in kvm_read_l1_tsc

### kvm-stealth-qemu.patch
Applied to QEMU 10.2.0 source.

Changes:
- `hw/acpi/aml-build.c` — ACPI OEM ID: INTEL → ALASKA, PTL → A M I
- `include/hw/acpi/aml-build.h` — ACPI_BUILD_APPNAME6
- `hw/smbios/smbios.c` — DDR4, DIMM form factor, correct memory speed
- `include/hw/pci/pci.h` — PCI vendor ID tweak
- `hw/smbios/smbios.c` — processor socket AM4

## How to build

### Kernel

```bash
# Download CachyOS kernel source
wget https://github.com/CachyOS/linux/releases/download/cachyos-6.19.7-1/cachyos-6.19.7-1.tar.gz
tar xf cachyos-6.19.7-1.tar.gz
cd cachyos-6.19.7-1

# Apply patch
git apply ../kvm-stealth-kernel.patch

# Copy config from running kernel
zcat /proc/config.gz > .config
make olddefconfig

# Build
make -j$(nproc)

# Install
sudo cp arch/x86/boot/bzImage /boot/vmlinuz-linux-cachyos-patched
sudo cp System.map /boot/System.map-linux-cachyos-patched
sudo mkinitcpio -k 6.19.7-1-cachyos -g /boot/initramfs-linux-cachyos-patched.img
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### QEMU

```bash
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
git checkout v10.2.0
git submodule update --init --recursive

# Apply patch
git apply ../kvm-stealth-qemu.patch

# Build
./configure --prefix=/opt/qemu-patched --target-list=x86_64-softmmu --enable-kvm --disable-werror
make -j$(nproc)
sudo make install
```

## Updating to newer CachyOS kernel

```bash
# Download new version
wget https://github.com/CachyOS/linux/releases/download/cachyos-X.XX.X-1/cachyos-X.XX.X-1.tar.gz
tar xf cachyos-X.XX.X-1.tar.gz
cd cachyos-X.XX.X-1

# Try to apply patch
git apply ../kvm-stealth-kernel.patch

# If conflicts, fix manually then rebuild
```

## Related

- [pve-emu-realpc](https://github.com/AICodo/pve-emu-realpc) — Proxmox patched QEMU and OVMF
- [vendor-reset](https://github.com/gnif/vendor-reset) — AMD GPU reset fix
- CachyOS — https://cachyos.org

## Disclaimer

This project is provided for educational purposes and legitimate use cases such as 
phone unlocking and diagnostics tools (DFT Pro and similar) that incorrectly block 
virtual machines.

The patches are provided "as is", without warranty of any kind. The author takes no 
responsibility for any damage to your system, data loss, kernel panics, or any other 
issues that may arise from using these patches. Use at your own risk.

Modifying kernel code may cause system instability. Always keep a backup of your 
working kernel before applying any patches.

This project does not encourage or support cheating in online games or any other 
illegal activities.
