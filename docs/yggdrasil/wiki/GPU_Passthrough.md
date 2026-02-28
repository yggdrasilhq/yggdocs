# GPU Passthrough on Yggdrasil

This document provides guidance on configuring GPU passthrough to LXC containers (and potentially Xen/KVM VMs) on the Yggdrasil system. Yggdrasil is designed to support both an Intel Arc DG02 and an NVIDIA RTX 2060.

## Overview

GPU passthrough allows a container or virtual machine to have dedicated access to a physical GPU, offering near-native performance for graphics-intensive applications, GPGPU compute, and video encoding/decoding.

## Prerequisites (Host Setup)

1.  **Hardware Virtualization Support:**
    *   Ensure AMD-V (for EPYC Zen 2 CPU) is enabled in the server's BIOS/UEFI.
    *   Ensure IOMMU (Input/Output Memory Management Unit) is enabled in BIOS/UEFI. This is often called "AMD IOMMU", "SVM Mode", or similar.

2.  **Kernel Configuration:**
    *   The Yggdrasil kernel must be compiled with IOMMU support (`CONFIG_AMD_IOMMU=y`, `CONFIG_IOMMU_API=y`, `CONFIG_VFIO_IOMMU_TYPE1=y`, `CONFIG_VFIO=y`, `CONFIG_VFIO_PCI=y`). This should be standard in modern Debian kernels.
    *   Kernel command line parameters: Add `amd_iommu=on iommu=pt` (or `iommu=on` depending on kernel version and desired strictness) to the GRUB configuration (`/etc/default/grub`, then `update-grub`).

3.  **Identify GPU PCI Addresses:**
    Use `lspci -nnk` to find the PCI addresses and device IDs for both the Intel and NVIDIA GPUs. Note all functions associated with each GPU (e.g., VGA controller, Audio device, USB controller for VirtualLink on NVIDIA).

    Example output snippet:
    ```
    03:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106 [GeForce RTX 2060] [10de:1f08] (rev a1)
            Subsystem: ASUSTeK Computer Inc. TU106 [GeForce RTX 2060] [1043:869f]
            Kernel driver in use: vfio-pci
            Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
    03:00.1 Audio device [0403]: NVIDIA Corporation TU106 High Definition Audio Controller [10de:10f9] (rev a1)
            Kernel driver in use: vfio-pci
            Kernel modules: snd_hda_intel
    ...
    0a:00.0 VGA compatible controller [0300]: Intel Corporation DG2 [Arc A380] [8086:56a5] (rev 05)
            Subsystem: Intel Corporation Device [8086:3000]
            Kernel driver in use: vfio-pci
            Kernel modules: i915
    0a:00.1 Audio device [0403]: Intel Corporation Device [8086:56c8] (rev 05)
            Kernel driver in use: vfio-pci
            Kernel modules: snd_hda_intel
    ```
    You'll need all related PCI addresses for each GPU you intend to pass through.

4.  **Bind GPUs to `vfio-pci` Driver:**
    To make GPUs available for passthrough, they must be unbound from their host drivers (e.g., `nvidia`, `i915`, `nouveau`) and bound to `vfio-pci` early in the boot process.
    *   Create/edit `/etc/modprobe.d/vfio.conf` (or a similar file):
        ```
        options vfio-pci ids=10de:1f08,10de:10f9,8086:56a5,8086:56c8 # Add all relevant device IDs
        softdep nouveau pre: vfio-pci
        softdep nvidia pre: vfio-pci
        softdep nvidia_drm pre: vfio-pci
        softdep i915 pre: vfio-pci
        softdep snd_hda_intel pre: vfio-pci # If GPU audio devices also need vfio-pci
        ```
        *Note: Forcing `snd_hda_intel` to `vfio-pci` for all GPU audio might have unintended consequences if other non-passthrough GPUs use it. A more targeted approach using PCI bus IDs might be needed via early boot scripts if this becomes an issue.*

    *   Update initramfs: `update-initramfs -u -k all`
    *   Reboot. After reboot, verify with `lspci -nnk` that the GPUs intended for passthrough are using `vfio-pci`.

5.  **Host Drivers:**
    *   **Intel:** Even if passing through the Intel GPU, the host might still need basic Intel drivers (`i915` for console/early boot if it's the primary, or for another Intel GPU not being passed through). Firmware (`linux-firmware` package) is essential.
    *   **NVIDIA:** If the NVIDIA card is solely for passthrough, the host ideally shouldn't load proprietary NVIDIA drivers for it.

## GPU Passthrough to LXC Containers

LXC passthrough works by making the GPU's device nodes (`/dev/...`) accessible to the container and allowing the container's cgroup to access them.

### 1. Container Configuration (`config` file)

For each container needing GPU access, add the following to its configuration file (`/var/lib/lxc/<container-name>/config` or `${LXC_PATH}/<container-name>/config`):

```
# Allow access to the GPU device nodes (adjust based on your GPU)
# For NVIDIA:
lxc.cgroup2.devices.allow = c 195:* rwm  # /dev/nvidia* (character devices, major 195)
lxc.cgroup2.devices.allow = c 236:* rwm  # /dev/nvidia-uvm, /dev/nvidia-modeset, /dev/nvidiactl (check major numbers on your system)
# For Intel:
lxc.cgroup2.devices.allow = c 226:* rwm  # /dev/dri/* (DRM devices, major 226)

# Mount the device nodes into the container
# For NVIDIA:
lxc.mount.entry = /dev/nvidia0 dev/nvidia0 none bind,optional,create=file 0 0
lxc.mount.entry = /dev/nvidiactl dev/nvidiactl none bind,optional,create=file 0 0
lxc.mount.entry = /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file 0 0
lxc.mount.entry = /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file 0 0 # If present
lxc.mount.entry = /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file 0 0 # If present
# For Intel (dri/card0, dri/renderD128):
lxc.mount.entry = /dev/dri dev/dri none bind,optional,create=dir 0 0
# If you want to be more specific instead of mounting the whole /dev/dri:
# lxc.mount.entry = /dev/dri/card0 dev/dri/card0 none bind,optional,create=file 0 0
# lxc.mount.entry = /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0 0
```
*   **Major/Minor Numbers:** Verify the major numbers (e.g., 195, 226) for your system using `ls -l /dev/nvidia*` and `ls -l /dev/dri/*`.
*   **`optional`:** If the device doesn't exist on the host, the container will still start.
*   **`create=file` / `create=dir`:** Ensures the mount points are created if they don't exist.

### 2. Drivers Inside the Container

*   **Intel:**
    *   Install the Intel userspace drivers and libraries (e.g., `intel-media-va-driver-non-free`, `libmfx1`, `libigdgmm12`, `level-zero`, `intel-opencl-icd`, relevant parts of oneAPI).
    *   Ensure the container's kernel version (if it has its own, unlikely for LXC, which shares host kernel) or host kernel has the necessary features.
    *   Verify with `vainfo` and `clinfo` (or Intel-specific tools) inside the container.

*   **NVIDIA:**
    *   Install the NVIDIA proprietary drivers *inside the container*. The driver version should ideally match the host's kernel compatibility, but since the kernel driver isn't `nvidia` on the host (it's `vfio-pci`), this is less of a strict requirement than with direct host usage. However, it's good practice to use a recent driver.
    *   **NVENC Patch:** This is where the patch to `libnvidia-encode.so` to unlock concurrent NVENC sessions would be applied *within the container's NVIDIA driver installation*.
        *   Refer to resources like Keylase/nvidia-patch for the patch scripts.
        *   You'll need to build the patched library or apply the patch to the library provided by the NVIDIA driver installer.
    *   Install CUDA toolkit, cuDNN, etc., as needed within the container.
    *   Verify with `nvidia-smi` and specific application tests inside the container.

### 3. Permissions and AppArmor/Seccomp

*   You might need to adjust AppArmor profiles or Seccomp filters if they block access to the GPU devices or related system calls.
*   Running LXC containers as unprivileged adds another layer of complexity for device access. It might require careful UID/GID mapping for the device nodes.

## GPU Passthrough to Xen/KVM VMs

Passthrough to full VMs is more robust as it uses IOMMU directly.

### 1. Xen Configuration (`.cfg` file)

For a Xen DomU, you'd add the PCI devices to its configuration file:
```
# Example for NVIDIA GPU (replace with your actual PCI IDs from lspci)
pci = [ '03:00.0', '03:00.1' ] # VGA controller and Audio device
# Example for Intel GPU
# pci = [ '0a:00.0', '0a:00.1' ]
```
The `pciback` (or `vfio-pci` if Xen is configured to use it for passthrough) on Dom0 will manage these devices.

### 2. KVM/QEMU Configuration (Command line or libvirt XML)

Using `libvirt`:
```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x03' slot='0x00' function='0x0'/> <!-- NVIDIA VGA -->
  </source>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x03' slot='0x00' function='0x1'/> <!-- NVIDIA Audio -->
  </source>
</hostdev>
<!-- Add similar entries for the Intel GPU -->
```

### 3. Drivers Inside the VM

Install the respective Intel or NVIDIA drivers inside the guest OS of the VM, just as you would on a physical machine. The NVENC patch for NVIDIA would be applied inside the VM.

## Specific Considerations for Yggdrasil

*   **Intel Arc DG02:**
    *   Requires a recent kernel (e.g., Linux 5.19+ for good support, 6.2+ for Resizable BAR and better SR-IOV/GVT-g if those are ever explored). Debian Unstable should provide this.
    *   Ensure `linux-firmware` is up-to-date and includes GuC/HuC firmware for DG2.
    *   Host driver `i915` needs to be new enough.

*   **NVIDIA RTX 2060:**
    *   Standard NVIDIA proprietary drivers.
    *   NVENC patch is a userspace modification within the container/VM.

## Troubleshooting

*   **Check IOMMU groups:** `find /sys/kernel/iommu_groups/ -type l`. Ensure all functions of a GPU are in the same IOMMU group and that the group is "viable" (doesn't contain essential host devices like the PCIe root bridge unless you know what you're doing). ACS override patches might be needed for problematic IOMMU grouping, but this is a last resort and has security implications.
*   **`dmesg`:** Check for `vfio-pci`, IOMMU, and GPU driver errors on the host and in the container/VM.
*   **Permissions:** Double-check device node permissions inside the container.
*   **Driver versions:** Mismatches can cause issues.

This document provides a starting point. GPU passthrough can be complex and may require significant trial and error to get working perfectly for specific hardware and software combinations.