# Xen Hypervisor on Yggdrasil
Last updated: 2024-04-01 (with additions from general Xen setup guide)

This document outlines the setup, configuration, usage, and troubleshooting of the Xen Hypervisor within the Yggdrasil project. Xen is included as an optional virtualization layer, primarily for running full virtual machines when LXC containers are not suitable.

## Inclusion in Build

Xen support can be compiled into the Yggdrasil ISO by using the `--with-xen` flag when running `mkconfig.sh`:

```bash
./mkconfig.sh --with-xen
```

This will ensure:
*   The Xen hypervisor packages are installed (e.g., `xen-hypervisor-amd64`, `xen-tools`, `xen-utils`).
*   GRUB is configured to offer a Xen boot option.
*   Necessary kernel modules for Xen are included.

## System Requirements

*   **Hardware Virtualization:** Ensure your CPU (EPYC Zen 2) has virtualization extensions enabled in the BIOS/UEFI (AMD-V).
*   **RAM:** Sufficient RAM for the host (dom0) and guest domains (domUs). Given 512GB, this should not be an issue.
*   **Disk Space:** Adequate space on `zroot` or other storage for virtual machine disk images.

## Critical Issues, Solutions, and Setup Notes

This section combines general setup information with common "gotchas" and solutions encountered during Yggdrasil's Xen configuration.

### 1. Bootloader Configuration (GRUB2 Recommended)

The most critical issue was getting Xen to boot properly. Syslinux/isolinux can be problematic. **GRUB2 is the recommended bootloader** as it handles Xen booting natively and works in both BIOS and UEFI modes.

**Problematic syslinux configuration (DO NOT USE):**
```bash
# DO NOT USE - Problematic syslinux configuration
LABEL xen
    KERNEL mboot.c32
    APPEND /boot/xen-4.17-amd64.gz --- vmlinuz console=tty0 --- initrd.img
```
**Issues with syslinux:** `mboot.c32` module missing, incorrect path references, memory allocation issues.

**Working GRUB2 Configuration Example (from live-build context):**
This should be adapted based on your `mkconfig.sh` and `live-build` setup.
```bash
menuentry "Live system with Xen" {
    # Adjust path to xen.gz as necessary
    multiboot2 /live/xen.gz dom0_mem=max:8192M # Example dom0 memory
    # KERNEL_LIVE and APPEND_LIVE are placeholders for live-build variables
    module2 @KERNEL_LIVE@ @APPEND_LIVE@
    module2 @INITRD_LIVE@
}
```
Ensure the paths to `xen.gz`, your kernel (`vmlinuz`), and `initrd.img` are correct for the live environment.

### 2. Kernel Location in Live Systems

**Problem:** Xen hypervisor (`xen.gz`) or Dom0 kernel/initrd not found during boot because live-build places these files in specific locations within the live filesystem (often `/live/` or `/casper/` for Ubuntu-based systems, check Debian live-build specifics).

**Investigation:**
- Standard systems: kernels typically in `/boot`.
- Live systems: often in `/live` or a similar directory within the image.
- The Xen hypervisor itself might need to be copied to this live directory.

**Solution (Example Binary Hook for live-build):**
This hook copies the Xen hypervisor to the `live/` directory of the ISO.
```bash
# config/hooks/normal/9030-copy-xen-to-live.hook.binary
#!/bin/bash
# Find the Xen hypervisor in the chroot (adjust find path if needed)
source_file=$(find ../chroot/boot/ -name 'xen*.gz' -type f -print -quit)
if [ -n "$source_file" ]; then
  cp -va "$source_file" live/xen.gz
else
  echo "Xen hypervisor not found in ../chroot/boot/" >&2
  exit 1
fi
```

### 3. Dom0 Memory Allocation

**Problems Encountered:**
- Dom0 taking too much or too little memory.
- VMs failing to start due to insufficient memory for Dom0 or DomUs.
- System instability.

**Solution:** Configure `dom0_mem` parameter in GRUB configuration for Xen.
```bash
# Conservative static setting
# dom0_mem=2048M

# Dynamic setting (recommended, allows Dom0 to use up to 8GB, but can shrink)
dom0_mem=max:8192M

# For systems with lots of RAM, providing a minimum and maximum
# dom0_mem="4096M,max:8192M"
```
Choose a value appropriate for your total system RAM and expected workload.

### 4. Package Dependencies

Ensure all required packages for Xen functionality are included in the Yggdrasil build when `--with-xen` is selected:
*   `xen-hypervisor-amd64` (or specific version)
*   `xen-tools`
*   `xen-utils-common`
*   `xen-utils-<version>`
*   `qemu-system-xen` (for HVM guests)
*   `qemu-utils`
*   `ovmf` (for UEFI guests)
*   `seabios` (for BIOS guests, usually a QEMU dependency)
*   `libvirt-daemon-system` and `libvirt-clients` (if using libvirt to manage Xen)

### 5. Network Bridge Configuration

**Problem:** Default network bridge for Xen (e.g., `xenbr0`) might conflict with other bridges (like `lxcbr0` for LXC) or require specific setup.

**Solution:** Configure a dedicated bridge for Xen VMs. This often involves creating a bridge manually and enslaving a physical interface to it.
Example for `/etc/network/interfaces.d/xenbr0` (adjust for your network):
```
# This is a basic example. Your actual config may differ based on Yggdrasil's network setup.
# Ensure this doesn't conflict with how your primary network interface is managed.
# auto eth0
# iface eth0 inet manual

# auto xenbr0
# iface xenbr0 inet static
#     bridge_ports eth0  # Or the relevant physical interface
#     address 10.20.0.10 # Static IP for the bridge/dom0 on this bridge
#     netmask 255.255.255.0
#     gateway 10.20.0.1
#     dns-nameservers 8.8.8.8 8.8.4.4
#     bridge_stp off
#     bridge_waitport 0
#     bridge_fd 0
```
Alternatively, if Dom0 networking is handled differently (e.g., NetworkManager, systemd-networkd), configure the bridge there. VMs will then connect their VIFs to this bridge.

### 6. UEFI Boot Issues for Xen and Guests

**Problems:**
- UEFI Secure Boot conflicts with Xen or unsigned kernels/modules.
- Missing OVMF firmware for UEFI guests.
- GRUB UEFI modules for Xen multiboot missing or misconfigured.

**Solutions:**
- **Secure Boot:** For `live-build`, you might use `--uefi-secure-boot disable` or ensure all components (GRUB, Xen, kernel) are properly signed if Secure Boot is required.
- **OVMF:** Ensure `ovmf` package is installed for UEFI HVM guests.
- **GRUB UEFI Modules:** `live-build` and standard Debian GRUB packages usually handle this. Ensure GRUB is installed correctly for UEFI boot. Required modules like `efi_gop`, `multiboot2`, `xen_boot` should be available.

### 7. Post-Installation Setup & Verification

1.  **Boot into Xen:** When booting Yggdrasil from USB, select the GRUB menu entry that includes "Xen hypervisor".
2.  **Verify Xen Installation and Dom0:**
    ```bash
    # Check Xen is running and basic info
    xl info

    # List domains (should show Domain-0)
    xl list

    # Check available memory for DomUs
    xl info | grep free_memory

    # Check for errors in kernel log
    dmesg | grep -i xen
    ```

3.  **Network Configuration (Dom0):**
    *   Ensure Dom0 has network connectivity.
    *   Verify the Xen bridge (e.g., `xenbr0` or `br0`) is up and configured.
    *   Review Debian's documentation on Xen networking and `/etc/xen/xend-config.sxp` (if using `xend`, though `xl` toolstack is modern) or `/etc/xen/xl.conf` for network script settings.

## Creating and Managing VMs (DomUs)

Xen virtual machines (DomUs) are typically defined by configuration files (e.g., `my_vm.cfg`).

### Example VM Configuration File (`my_vm.cfg`)

```
# Kernel and Ramdisk (if using PV guest with host kernel/initrd)
# kernel = "/boot/vmlinuz-guest"
# ramdisk = "/boot/initrd.img-guest"

# Bootloader (for PV guests booting from their own disk image, or for HVM/PVH)
# For PV guest with its own kernel:
# bootloader = "pygrub"
# For HVM guest (recommended for broader OS compatibility):
builder = "hvm"
# device_model_override = "/usr/lib/xen/bin/qemu-system-i386" # or x86_64, path may vary

# VM Name
name = "my-debian-vm"

# Memory
memory = 4096 # MB

# VCPUs
vcpus = 2
# Optionally pin vCPUs for performance:
# cpuid = ...
# cpus = "0,1" # Host CPUs for this VM

# Disk(s)
# Example: ZFS zvol for disk
disk = [ 'phy:/dev/zvol/zroot/xen-vms/my-debian-vm-disk,xvda,w',  # Main disk
         'file:/path/to/iso/debian-installer.iso,hdc:cdrom,r' ] # Installer ISO (temporary)
# After installation, remove the ISO line or comment it out.

# Network Interface(s)
vif = [ 'bridge=br0' ] # Adjust to your bridge name (e.g., xenbr0)

# Boot options
on_poweroff = "destroy"
on_reboot   = "restart"
on_crash    = "restart"

# Graphics (for HVM guests, if VNC access is needed)
# vnc = 1
# vnclisten = "0.0.0.0" # Listen on all interfaces
# vncdisplay = 0 # VNC display number, e.g. :0 for port 5900
# stdvga = 1 # Enable standard VGA device

# Other options
# For PV guest console:
# extra = "console=hvc0"
# For HVM guest serial console (if guest OS is configured for ttyS0):
# serial = "pty"
```

**Notes on Disk Images:**
*   It's highly recommended to use ZFS zvols for VM disk images on Yggdrasil:
    ```bash
    zfs create -V 20G zroot/xen-vms/my-debian-vm-disk
    ```
*   For installation, add the OS installation ISO as a cdrom device. Remove it after installation.

### Managing VMs with `xl`

*   **Create (and start) a VM:**
    ```bash
    xl create /path/to/my_vm.cfg
    ```
*   **List running VMs:**
    ```bash
    xl list
    ```
*   **Connect to a VM's console:**
    *   PV guest (if `extra = "console=hvc0"`): `xl console my-debian-vm`
    *   HVM guest (if `serial = "pty"` and guest OS uses serial console): `xl console my-debian-vm`
    *   (Ctrl + ] to detach from console)
    *   For VNC, use a VNC client to connect to Dom0's IP on port 5900 + vncdisplay.
*   **Shutdown a VM gracefully:**
    ```bash
    xl shutdown my-debian-vm
    ```
*   **Force power-off a VM:**
    ```bash
    xl destroy my-debian-vm
    ```
*   **Save VM state to file:**
    ```bash
    xl save my-debian-vm /path/to/savefile
    ```
*   **Restore VM state from file:**
    ```bash
    xl restore /path/to/savefile
    ```

## GPU Passthrough with Xen

GPU passthrough to Xen DomUs allows a VM to have dedicated access to a GPU. This is complex and typically involves:
1.  Ensuring IOMMU (AMD-Vi for EPYC) is enabled in BIOS/UEFI and active in the Dom0 kernel (`amd_iommu=on` kernel parameter).
2.  Binding the GPU and its associated PCI devices (e.g., audio controller) to the `pciback` or `vfio-pci` driver in Dom0. This prevents Dom0 from using the GPU.
3.  Assigning the PCI device(s) to the DomU via its configuration file (e.g., `pci = ['0000:03:00.0', '0000:03:00.1']`).
4.  Installing appropriate GPU drivers within the guest VM.

This will require specific kernel command-line parameters for Dom0 and careful PCI device identification. Refer to `GPU_Passthrough.md` for more general details which can be adapted for Xen.

## Storage for Xen VMs

*   **ZFS zvols:** Recommended. Create a parent dataset like `zroot/xen-vms` and then zvols for each VM's disks.
    ```bash
    zfs create zroot/xen-vms
    zfs create -V <size_in_G>G zroot/xen-vms/<vm-name>-disk0
    ```
    Then use `phy:/dev/zvol/zroot/xen-vms/<vm-name>-disk0` in the VM config.
*   **File-backed images:** Can be stored on any filesystem (e.g., `/zroot/xen-vm-images/vm.img`). Generally slower than zvols. Create with `dd` or `qemu-img create`.

## Common Issues and Fixes (Troubleshooting)

1.  **"Dom0 kernel not found" or "Xen hypervisor not found":**
    *   Check kernel/hypervisor paths in `grub.cfg` are correct for the live environment.
    *   Verify any binary hooks (like `9030-copy-xen-to-live.hook.binary`) correctly copied `xen.gz` and other necessary files.
2.  **"Not enough memory" for Dom0 or DomU:**
    *   Adjust `dom0_mem` parameter for Dom0.
    *   Ensure sufficient free memory before starting a DomU (`xl info | grep free_memory`).
    *   Check for memory fragmentation.
3.  **"Device not found" for passthrough devices:**
    *   Ensure correct PCI IDs are used in `pci` list in DomU config.
    *   Verify the device is bound to `pciback` or `vfio-pci` in Dom0 (`lspci -k`).
    *   Load necessary kernel modules in Dom0 (though `pciback`/`vfio-pci` should handle this if configured right).
4.  **Network issues for DomUs:**
    *   Verify bridge configuration in Dom0 (`brctl show`).
    *   Check `vif` line in DomU config, ensure bridge name is correct.
    *   Ensure network scripts for Xen are correctly configured (e.g., `/etc/xen/scripts/network-bridge`).
    *   Check firewall rules on Dom0.

## Performance Optimization

Recommended Xen parameters and considerations for optimal performance (add to GRUB Xen command line or DomU config as appropriate):
*   `dom0_mem=max:8192M` (adjust based on total RAM and needs)
*   `dom0_vcpus_pin` (pin Dom0 VCPUs to physical CPUs)
*   `dom0_max_vcpus=4` (limit Dom0 VCPUs, adjust based on core count)
*   `smt=true` (if your CPU has SMT/Hyper-Threading and it benefits your workload)
*   Consider CPU pinning for DomU VCPUs in their config files.
*   Use PVHVM or PV drivers in HVM guests where possible for better I/O performance.

## Further Considerations

*   **Toolstack:** Yggdrasil uses the `xl` toolstack, which is the modern standard. Older `xm` (based on `xend` daemon) is deprecated.
*   **PV vs HVM vs PVH Guests:**
    *   **PV (Paravirtualized):** Guests are aware they are virtualized. Requires a Xen-aware kernel in the guest. Generally good I/O performance. Linux guests are well-suited.
    *   **HVM (Hardware Virtualized Machine):** Guests are not aware they are virtualized and run unmodified OSes (e.g., Windows). Requires CPU virtualization extensions.
    *   **PVH:** A hybrid mode using PV drivers for I/O and boot within an HVM container. Often provides a good balance of performance and compatibility. Increasingly the preferred mode for modern Linux guests.
*   **Security:**
    *   Follow Xen security best practices (see Xen Project documentation).
    *   Enable XSM (Xen Security Modules) if needed for advanced policy enforcement.
    *   Carefully manage device passthrough.
    *   Implement network isolation between VMs and Dom0 as required.
*   **Backup:**
    *   For VMs on ZFS zvols, use ZFS snapshots for consistent backups.
    *   `xl save` and `xl restore` can be used for live state backup/restore but are not ideal for routine filesystem backups.
    *   Consider guest-level backup agents or filesystem-level backups via Dom0 if ZFS zvols are not used.
*   **Live Migration:** Xen supports live migration if shared storage is used for VM disks (e.g., iSCSI, NFS, or ZFS over network). This is an advanced topic.

This document serves as a combined reference. More specific configurations and troubleshooting steps will be added as the Yggdrasil project evolves and Xen usage is further tested.
```
