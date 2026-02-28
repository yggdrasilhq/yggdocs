# Debian Live Build: Custom Kernel Compilation Crash Course

This document provides a detailed guide and a sample chroot hook script for compiling a custom Linux kernel, including ZFS and optional NVIDIA modules, during a Debian Live Build process. This is essential for creating a Yggdrasil live ISO with tailored kernel features.

## Overview

The core of this process is a `live-build` chroot hook. This hook script (`9000-custom-kernel.hook.chroot`) executes within the chroot environment during the `lb build` process. It performs the following key steps:

1.  **Environment Setup:** Determines host kernel version, sets up build directories, and defines build dependencies.
2.  **Dependency Installation:** Installs necessary packages for kernel compilation (e.g., `build-essential`, `libelf-dev`).
3.  **Source Code Acquisition:** Downloads the specified Linux kernel source tarball and extracts it.
4.  **External Module Integration:**
    *   Clones OpenZFS source code.
    *   Optionally clones NVIDIA open-gpu-kernel-modules if `WITH_NVIDIA` is true.
5.  **Kernel Configuration:**
    *   Starts with a default x86_64 configuration (`x86_64_defconfig`).
    *   Uses `scripts/config` to programmatically enable/disable specific kernel features tailored for Yggdrasil's target hardware (EPYC Zen 2) and use cases (virtualization, specific networking).
6.  **Kernel Compilation:** Compiles the kernel and creates Debian packages (`.deb`).
7.  **Kernel Package Installation:** Installs the newly built custom kernel image package.
8.  **External Module Compilation & Installation:**
    *   Builds and installs ZFS modules against the new custom kernel.
    *   If `WITH_NVIDIA` is true, builds and installs NVIDIA kernel modules.
9.  **System Optimization & Cleanup:** Removes build artifacts, documentation, build dependencies, and any stock kernel to reduce image size.
10. **Initramfs Generation:** Updates the initramfs for the newly installed custom kernel.

## The Chroot Hook Script

The following script is designed to be placed in `config/hooks/normal/9000-custom-kernel.hook.chroot` within your `live-build` configuration directory.

```bash
#!/bin/bash
#
# live-build chroot hook: 9000-custom-kernel.hook.chroot
# Purpose: Compile and install a custom Linux kernel with ZFS and optional NVIDIA modules.
#
# Script should be executable: chmod +x config/hooks/normal/9000-custom-kernel.hook.chroot

# Exit on any error
set -e
# Echo commands (optional, for debugging)
# set -x

# --- Configuration Variables ---
# HOST_KERNEL_VERSION: Determined dynamically. Can be overridden if a specific version is needed.
# KERNEL_VERSION_BASE: Extracted from uname -r. E.g., "6.1.0"
KERNEL_VERSION_BASE=$(uname -r | cut -d'-' -f1)
# Example: If you want to build a specific version, uncomment and set:
# KERNEL_VERSION_BASE="6.5.3" # Ensure this version is available on kernel.org

BUILDROOT="/tmp/kernel-build"
# BUILD_DEPS: Moved to a variable for clarity and reusability in purge.
# Added dkms, kmod, cpio as they are often useful for kernel/module management.
# pkg-config is often a dependency for ZFS or other modules.
# libsystemd-dev might be needed for ZFS with systemd integration.
BUILD_DEPS="build-essential flex bison libelf-dev libssl-dev bc rsync dkms kmod cpio pkg-config libsystemd-dev"
# Fetch number of available processor cores
CORES=$(nproc)
# Ensure at least 1 core is reported, use 2 if more are available for faster builds
# but be mindful of resource constraints in chroot environments.
COMPILE_CORES=${CORES:-1} # Default to 1 if nproc fails
if [ "$COMPILE_CORES" -gt 2 ]; then
    COMPILE_CORES=2 # Cap at 2 for this script example to avoid over-stressing typical build environments
fi

# WITH_NVIDIA: This variable should be passed into the chroot environment or set via lb config.
# For testing, you can uncomment and set it here:
# WITH_NVIDIA="true"

echo "=== Yggdrasil Custom Kernel Build Script ==="
echo "Host Kernel Version Detected (for base): $(uname -r)"
if [[ -z "$KERNEL_VERSION_BASE" ]]; then
    echo "Error: Could not determine kernel version base (e.g., 6.1.0) from uname -r."
    exit 1
fi
echo "Target Kernel Version to Build: ${KERNEL_VERSION_BASE}"
echo "Using ${COMPILE_CORES} cores for compilation."
echo "NVIDIA Modules: ${WITH_NVIDIA:-false}" # Default to false if not set

# === 1. Build Environment Orchestration ===
echo "Step 1: Setting up build environment..."
mkdir -p "${BUILDROOT}"
cd "${BUILDROOT}" || exit 1 # Critical to be in BUILDROOT

# === 2. Dependency Resolution ===
echo "Step 2: Installing build dependencies..."
apt-get update
# Using --no-install-recommends can save space but might miss some indirect dependencies.
# For kernel builds, it's usually safer to install recommends unless space is extremely critical.
apt-get install -y ${BUILD_DEPS}

# === 3. Kernel Source Integration ===
echo "Step 3: Acquiring kernel source for version ${KERNEL_VERSION_BASE}..."
# Constructing the major version (e.g., v6.x from 6.1.0)
KERNEL_MAJOR_VERSION=$(echo "${KERNEL_VERSION_BASE}" | cut -d'.' -f1)
KERNEL_SOURCE_URL="https://cdn.kernel.org/pub/linux/kernel/v${KERNEL_MAJOR_VERSION}.x/linux-${KERNEL_VERSION_BASE}.tar.xz"
echo "Downloading from: ${KERNEL_SOURCE_URL}"

wget -q "${KERNEL_SOURCE_URL}" -O "linux-${KERNEL_VERSION_BASE}.tar.xz"
tar -xf "linux-${KERNEL_VERSION_BASE}.tar.xz"
cd "linux-${KERNEL_VERSION_BASE}" || exit 1

# === 4. External Module Framework Integration ===
# ZFS Integration (OpenZFS)
ZFS_VERSION_TAG="zfs-2.2.2" # Specify a known good tag or branch for stability
echo "Step 4a: Integrating OpenZFS (${ZFS_VERSION_TAG})..."
# Clone outside kernel source tree, then copy/symlink as needed by ZFS build system
git clone --depth=1 --branch "${ZFS_VERSION_TAG}" https://github.com/openzfs/zfs.git "${BUILDROOT}/zfs-src"

# NVIDIA Open Kernel Module Integration
NVIDIA_MODULES_PATH="${BUILDROOT}/nvidia-kernel-src"
if [[ "${WITH_NVIDIA}" == "true" ]]; then
    echo "Step 4b: Integrating NVIDIA Open Kernel Modules..."
    # Consider specifying a release tag for NVIDIA modules if available/stable
    git clone --depth=1 https://github.com/NVIDIA/open-gpu-kernel-modules.git "${NVIDIA_MODULES_PATH}"
fi

# === 5. Kernel Configuration Matrix ===
echo "Step 5: Configuring kernel parameters..."
# Start with a distribution default or a generic defconfig
# make defconfig # or x86_64_defconfig as you had
# It's often better to copy the running host's config if it's similar enough and then modify
# cp /boot/config-$(uname -r) .config # If building for the same arch
# Or use the defconfig for the target architecture
make x86_64_defconfig

echo "Applying Yggdrasil specific kernel configurations..."
# Using scripts/config for targeted changes
# Ensure paths and CONFIG_ options are correct. Misspelled options are ignored.
# Grouping related options can improve readability.

# CPU Specific (AMD EPYC)
scripts/config --enable CONFIG_AMD_EPYC --enable CONFIG_NUMA --enable CONFIG_AMD_NUMA
# IOMMU: AMD IOMMU is generally needed for server/virtualization. Intel IOMMU can be disabled if no Intel passthrough.
scripts/config --enable CONFIG_AMD_IOMMU --disable CONFIG_INTEL_IOMMU_DEFAULT_ON --disable CONFIG_INTEL_IOMMU

# Power Management (for servers, schedutil is good)
scripts/config --enable CONFIG_CPU_FREQ_GOV_SCHEDUTIL --enable CONFIG_CPU_IDLE --enable CONFIG_ACPI_CPPC_LIB

# Security Enhancements
scripts/config --enable CONFIG_SECURITY_LANDLOCK --enable CONFIG_STACKPROTECTOR_STRONG --enable CONFIG_PAGE_TABLE_ISOLATION
# Consider CONFIG_KFENCE for debugging memory issues (slight performance overhead)
# scripts/config --enable CONFIG_KFENCE

# Feature Reduction (be careful not to remove essential features for a server/live OS)
# Disabling sound and basic USB HID might be too aggressive if emergency console access via USB keyboard is ever needed,
# or if any basic audio notifications are desired. Consider carefully.
scripts/config --disable CONFIG_SOUND --disable CONFIG_USB_KEYBOARD --disable CONFIG_USB_MOUSE --disable CONFIG_WIRELESS

# Networking (Broadcom focus, disable Intel/Realtek server NICs)
# This is very specific. Ensure it matches your hardware. Disabling all Intel/Realtek vendor support is broad.
scripts/config --enable CONFIG_NET_VENDOR_BROADCOM --enable CONFIG_TIGON3 --enable CONFIG_BNXT
# For Broadcom NICs like NetXtreme II, you might need CONFIG_CNIC, CONFIG_BNX2, CONFIG_BNX2X too.
# scripts/config --module CONFIG_BNX2 --module CONFIG_BNX2X
# Disabling all CONFIG_NET_VENDOR_INTEL removes support for e.g. e1000e, igb, ixgbe which are common.
scripts/config --disable CONFIG_NET_VENDOR_INTEL --disable CONFIG_NET_VENDOR_REALTEK

# Virtualization Support (Essential for Yggdrasil)
scripts/config --enable CONFIG_KVM --enable CONFIG_KVM_AMD --enable CONFIG_VFIO --enable CONFIG_VFIO_IOMMU_TYPE1 --enable CONFIG_VFIO_PCI
# For Xen host support (Dom0)
scripts/config --enable CONFIG_XEN_DOM0 --enable CONFIG_XEN_PVHVM --enable CONFIG_XEN_PV --enable CONFIG_XEN_EFI --enable CONFIG_XEN_FRONTEND # and others...

# Filesystems (ZFS will be built as module, ensure others are present if needed for boot/live system)
scripts/config --enable CONFIG_ISO9660_FS --enable CONFIG_SQUASHFS --enable CONFIG_EXT4_FS

# Ensure .config is saved
yes '' | make oldconfig # Process any new symbols with default values

# === 6. Kernel Build Process ===
echo "Step 6: Initiating kernel package compilation (bindeb-pkg) using ${COMPILE_CORES} cores..."
# Using bindeb-pkg is convenient. It creates .deb packages.
# The HOST_KERNEL_VERSION variable in your script was from uname -r, which includes extraversion (e.g., -amd64).
# For bindeb-pkg, the package version will be based on the kernel's Makefile version + localversion.
# You might want to set a LOCALVERSION for your custom kernel:
# echo "-yggdrasil" > .localversion-set # or append to .config with CONFIG_LOCALVERSION="-yggdrasil"
make LOCALVERSION="-yggdrasil" -j"${COMPILE_CORES}" bindeb-pkg

# === 7. Kernel Package Integration ===
echo "Step 7: Installing custom kernel Debian packages..."
# The packages will be in BUILDROOT (one level up from kernel source dir)
cd "${BUILDROOT}" || exit 1
# Install image, then headers (if needed for module builds later, though ZFS/NVIDIA build against source)
# Ensure the version matches what was built, including your LOCALVERSION if set.
KERNEL_PKG_VERSION="${KERNEL_VERSION_BASE}-yggdrasil" # Adjust if LOCALVERSION is different
dpkg -i "linux-image-${KERNEL_PKG_VERSION}"*.deb
# dpkg -i linux-headers-${KERNEL_PKG_VERSION}*.deb # Only if needed by DKMS later, but we build modules manually

# === 8. External Module Compilation Framework ===
echo "Step 8a: Building ZFS subsystem for kernel ${KERNEL_PKG_VERSION}..."
cd "${BUILDROOT}/zfs-src" || exit 1
# ZFS needs access to the KERNEL SOURCE, not just /lib/modules/.../build symlink if it points to a different source.
# The symlink /lib/modules/${KERNEL_PKG_VERSION}/build should point to ${BUILDROOT}/linux-${KERNEL_VERSION_BASE}
# Ensure this link is correct after kernel package install or explicitly provide path.
# Correct path to kernel source:
KERNEL_SOURCE_DIR="${BUILDROOT}/linux-${KERNEL_VERSION_BASE}"
# Make sure KERNEL_SOURCE_DIR is populated from previous steps.
./autogen.sh
./configure --with-linux="${KERNEL_SOURCE_DIR}" --with-linux-obj="${KERNEL_SOURCE_DIR}" # ZFS needs both for out-of-tree
make -j"${COMPILE_CORES}"
make install
# No need for depmod here, will be done after NVIDIA too.

# NVIDIA Module Compilation (if enabled)
if [[ "${WITH_NVIDIA}" == "true" ]]; then
    echo "Step 8b: Building NVIDIA kernel modules for kernel ${KERNEL_PKG_VERSION}..."
    cd "${NVIDIA_MODULES_PATH}" || exit 1
    # Build with consistent toolchain and specify kernel source directory
    # IGNORE_CC_MISMATCH=1 can be risky. Ensure your chroot gcc matches kernel's expected gcc if possible.
    make IGNORE_CC_MISMATCH=1 MODULE_SIGN_KEY= \
        NV_KERNEL_MODULE_NAME=nvidia \
        NV_KERNEL_OUTPUT_DIR="${NVIDIA_MODULES_PATH}/kernel-out" \
        NV_VERBOSE=1 \
        SYSSRC="${KERNEL_SOURCE_DIR}" \
        SYSOUT="${KERNEL_SOURCE_DIR}" \
        CONFTEST_PREOPTS="" CONFTEST_POSTOPTS="" \
        modules -j"${COMPILE_CORES}"

    echo "Installing NVIDIA kernel modules..."
    make modules_install \
        NV_KERNEL_MODULE_NAME=nvidia \
        NV_KERNEL_OUTPUT_DIR="${NVIDIA_MODULES_PATH}/kernel-out" \
        SYSSRC="${KERNEL_SOURCE_DIR}" \
        INSTALL_MOD_DIR=. # Install relative to /lib/modules/${KERNEL_PKG_VERSION}/kernel/
fi

# Update module dependencies for all new modules
echo "Updating module dependencies (depmod)..."
depmod -a "${KERNEL_PKG_VERSION}"

# === 9. System Optimization & Cleanup ===
echo "Step 9: Optimizing system footprint..."

# Remove build artifacts (kernel source, zfs source, nvidia source)
# This is done by removing BUILDROOT at the end.

# Remove unnecessary documentation and man pages from the chroot
rm -rf /usr/share/doc/* /usr/share/info/* /usr/share/man/*
# Some docs might be useful (e.g., ZFS, NVIDIA docs), be selective if needed.
# But for a minimal live ISO, this is often fine.

# Clean apt cache
apt-get clean
rm -rf /var/cache/apt/archives/*.deb # Remove downloaded .deb files
rm -rf /var/lib/apt/lists/* # Remove package lists

# Remove installed build dependencies
echo "Removing build dependencies..."
apt-get remove --purge -y ${BUILD_DEPS}
apt-get autoremove --purge -y

# Remove any stock kernel that might have been pulled in or was there before
# Be careful not to remove the custom kernel just installed!
# The glob linux-image-* is dangerous. Better to list specific stock kernel packages if known.
# For now, assume this means "any other kernel than the one we built".
# A safer way is to find kernel packages NOT matching KERNEL_PKG_VERSION.
# This part needs to be very careful.
echo "Attempting to remove stock/other kernels (use with caution)..."
# find /boot -name 'vmlinuz-*' ! -name "vmlinuz-${KERNEL_PKG_VERSION}" -print -delete
# find /lib/modules/* ! -name "${KERNEL_PKG_VERSION}" -print -delete
# apt-get remove --purge -y $(dpkg -l 'linux-image-*' | grep '^ii' | awk '{print $2}' | grep -v "${KERNEL_PKG_VERSION}" || true)

# === 10. Final Cleanup ===
echo "Step 10: Final cleanup of build directory..."
cd / # Change out of BUILDROOT before removing it
rm -rf "${BUILDROOT}"

# === 11. Initramfs Generation ===
echo "Step 11: Generating initramfs for custom kernel ${KERNEL_PKG_VERSION}..."
# update-initramfs -c -k all # '-k all' might try to build for removed kernels.
# Be specific:
update-initramfs -c -k "${KERNEL_PKG_VERSION}"
# Ensure ZFS and NVIDIA modules are included in initramfs if needed for early boot (ZFS for root, NVIDIA for early KMS).
# This typically requires /etc/initramfs-tools/modules to list them, or hooks.
# For ZFS on root, ensure /etc/default/zfs is configured and initramfs hooks are present.

echo "Build process completed successfully for kernel ${KERNEL_PKG_VERSION}."
exit 0
```

## Explanation of the Script Sections

### Initial Setup (`tee ... EOF`, `chmod`)
The script starts by using `tee` to write its content to `config/hooks/normal/9000-custom-kernel.hook.chroot`. The `chmod +x` (or `+777` as in your script, though `+x` is sufficient) makes it executable. `live-build` will pick up executable scripts from this directory.

*   **Suggestion:** Use `chmod +x` instead of `chmod +777`. The latter is excessively permissive.

### Configuration Variables
*   `KERNEL_VERSION_BASE`: Dynamically determines the kernel version (e.g., "6.1.0") from the build host's `uname -r`.
    *   **Suggestion:** Allow overriding this for reproducible builds of a *specific* kernel version, not just what the build host happens to run.
*   `BUILDROOT`: A temporary directory for all build operations.
*   `BUILD_DEPS`: A list of packages required for compilation.
    *   **Suggestion:** Add `dkms`, `kmod`, `cpio`, `pkg-config`, `libsystemd-dev` as these are often useful or required for kernel/module builds, especially ZFS.
*   `CORES` / `COMPILE_CORES`: Number of processor cores to use for compilation.
    *   **Suggestion:** Cap `COMPILE_CORES` (e.g., to 2 or 4) for stability in potentially resource-constrained chroot environments, even if `nproc` reports more. Your script already does this (`CORES=2 make`).
*   `WITH_NVIDIA`: An environment variable presumably passed to `live-build` or set in its config to control NVIDIA module inclusion.
    *   **Suggestion:** Add a default value display if the variable is not set (e.g., `echo "NVIDIA Modules: ${WITH_NVIDIA:-false}"`).

### Step 1: Build Environment Orchestration
Standard directory creation and navigation.

### Step 2: Dependency Resolution
Updates package lists and installs `BUILD_DEPS`.
*   **Suggestion:** Consider using `apt-get install -y --no-install-recommends ${BUILD_DEPS}` if image size is extremely critical, but be aware it might miss some indirect dependencies. For kernel building, usually safer to keep recommends.

### Step 3: Kernel Source Integration
Downloads and extracts the kernel source.
*   **Suggestion:** The URL construction `v6.x` is hardcoded. It's better to derive the major version (e.g., `v${KERNEL_MAJOR_VERSION}.x`) from `KERNEL_VERSION_BASE`.
*   Use `wget -q` for quieter downloads.

### Step 4: External Module Framework Integration
*   **ZFS:** Clones OpenZFS.
    *   **Suggestion:** Clone a specific, known-good ZFS version tag (e.g., `zfs-2.2.2`) for stability rather than the head of the default branch. Clone it *outside* the kernel source tree first (e.g., to `${BUILDROOT}/zfs-src`).
*   **NVIDIA:** Clones NVIDIA open kernel modules.
    *   **Suggestion:** Also clone this *outside* the kernel source tree (e.g., to `${BUILDROOT}/nvidia-kernel-src`).

### Step 5: Kernel Configuration Matrix
Uses `make x86_64_defconfig` and then `scripts/config` to customize.
*   **Critique:** `x86_64_defconfig` is very minimal. For a server/desktop live OS, you might want to start from the current host's config (`cp /boot/config-$(uname -r) .config`) if it's known to be good and relevant, or a distribution's generic config, and then apply specific changes.
*   **AMD_EPYC & NUMA:** Good for your target.
*   **IOMMU:** Enabling AMD IOMMU is correct. Disabling Intel IOMMU might be fine if you're certain no Intel passthrough is needed. `CONFIG_INTEL_IOMMU_DEFAULT_ON` is usually disabled by default if `CONFIG_INTEL_IOMMU` is `n`.
*   **Power Management:** `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` is a good modern default.
*   **Security:** Good selections. Consider `CONFIG_KFENCE` for debugging memory issues (has a slight performance overhead).
*   **Feature Reduction:**
    *   Disabling `CONFIG_SOUND` is common for servers.
    *   Disabling `CONFIG_USB_HID` (which implies `CONFIG_USB_KEYBOARD`, `CONFIG_USB_MOUSE`) is risky. If your live system ever needs console access via a USB keyboard (e.g., for emergency recovery, initramfs shell), this will prevent it. I strongly recommend keeping basic USB keyboard support.
    *   Disabling `CONFIG_WIRELESS` is fine for a server.
*   **Networking:**
    *   Enabling specific Broadcom drivers is good if that's your primary NIC.
    *   Disabling `CONFIG_NET_VENDOR_INTEL` and `CONFIG_NET_VENDOR_REALTEK` is very aggressive. This removes support for extremely common server NICs (e.g., Intel e1000e, igb, ixgbe) and Realtek NICs. Only do this if you are *absolutely certain* you will never encounter these NICs or need them as a fallback.
*   **Missing Essentials for Live OS / Virtualization Host:**
    *   **Virtualization:** `CONFIG_KVM`, `CONFIG_KVM_AMD`, `CONFIG_VFIO`, `CONFIG_VFIO_IOMMU_TYPE1`, `CONFIG_VFIO_PCI` are essential.
    *   **Xen Dom0 Support:** If Xen is an option, numerous `CONFIG_XEN_*` options are needed (e.g., `CONFIG_XEN_DOM0`, `CONFIG_XEN_PVHVM`, `CONFIG_XEN_EFI`).
    *   **Filesystems:** `CONFIG_ISO9660_FS` (for the live ISO itself), `CONFIG_SQUASHFS` (for the compressed live filesystem), `CONFIG_EXT4_FS` (common).
*   **Finalizing .config:** After `scripts/config`, run `yes '' | make oldconfig` to accept defaults for any new symbols introduced by your changes.

### Step 6: Kernel Build Process Orchestration
Uses `make -j${CORES} bindeb-pkg`. (Your script hardcodes `CORES=2` here).
*   **Suggestion:** Set `LOCALVERSION="-yggdrasil"` (e.g., `echo "-yggdrasil" > .localversion-set` or `scripts/config --set-str CONFIG_LOCALVERSION "-yggdrasil"`) to easily identify your custom kernel packages. This affects the package version.
*   The `HOST_KERNEL_VERSION` in your script comes from `uname -r` (e.g., `6.1.0-13-amd64`). `bindeb-pkg` will use the kernel Makefile version (e.g., `6.1.0`) plus your `CONFIG_LOCALVERSION`.

### Step 7: Package Integration
Installs the built `linux-image` deb.
*   **Suggestion:** The package name will include your `LOCALVERSION`. Ensure the `dpkg -i` command matches this. `linux-image-${HOST_KERNEL_VERSION}*.deb` might not match if `HOST_KERNEL_VERSION` had extras like `-amd64` and your built kernel is just `KERNEL_VERSION_BASE` + `LOCALVERSION`.

### Step 8: Module Compilation Framework
*   **ZFS:**
    *   It configures ZFS with `--with-linux=/lib/modules/$(uname -r)/build`. This symlink should point to your *newly built and installed* kernel's source tree. After `dpkg -i linux-image-...`, this link *should* be correct if the kernel package sets it up.
    *   **Better:** Explicitly provide `--with-linux=${BUILDROOT}/linux-${KERNEL_VERSION_BASE}` and `--with-linux-obj=${BUILDROOT}/linux-${KERNEL_VERSION_BASE}` to the ZFS configure script.
*   **NVIDIA:**
    *   `IGNORE_CC_MISMATCH=1`: Use with caution. It's better if the chroot's compiler matches what the kernel was built with.
    *   **Suggestion:** Explicitly pass `SYSSRC="${BUILDROOT}/linux-${KERNEL_VERSION_BASE}"` and `SYSOUT="${BUILDROOT}/linux-${KERNEL_VERSION_BASE}"` (or wherever the configured kernel source is) to the NVIDIA make command.
    *   `make modules_install` for NVIDIA: Ensure `INSTALL_MOD_DIR` is set appropriately if needed, or that it defaults to installing into the correct `/lib/modules/KERNEL_VERSION/` path.
*   **`depmod -a`:** Good. Should run after all modules are installed. Specify the kernel version: `depmod -a "${KERNEL_PKG_VERSION}"`.

### Step 9: System Optimization
Removes docs, man pages, apt cache, build deps, and stock kernel.
*   **Stock Kernel Removal:** `apt-get remove --purge -y linux-image-*` is **extremely dangerous** as it can remove your just-installed custom kernel if the glob matches!
    *   **CRITICAL SUGGESTION:** You MUST make this selective. For example, find installed `linux-image` packages that do *not* match your `KERNEL_PKG_VERSION` and remove those specifically.
    *   Example (needs refinement and testing): `apt-get remove --purge -y $(dpkg -l 'linux-image-*' | grep '^ii' | awk '{print $2}' | grep -v "${KERNEL_PKG_VERSION}" || true)`

### Step 10: Final Cleanup
Removes `BUILDROOT`. Good.

### Step 11: Initramfs Generation
`update-initramfs -c -k all`.
*   **Suggestion:** Be specific: `update-initramfs -c -k "${KERNEL_PKG_VERSION}"`. Using `-k all` might try to build initramfs for kernels you just removed or for which headers are gone.
*   **ZFS & NVIDIA in Initramfs:** If ZFS is used for the root filesystem, or if NVIDIA KMS needs to start very early, these modules might need to be included in the initramfs. This typically requires entries in `/etc/initramfs-tools/modules` and relevant hooks in `/etc/initramfs-tools/hooks/`. The ZFS utilities package usually installs hooks for ZFS.
