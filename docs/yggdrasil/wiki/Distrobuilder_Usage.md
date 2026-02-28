# Using Distrobuilder for Custom LXC Images on Yggdrasil

This document describes how to use `distrobuilder` to create custom LXC container images for the Yggdrasil system. `distrobuilder` is a tool developed by the LXC/LXD team for building images of various Linux distributions. It serves a similar purpose to `imgadm` in SmartOS for managing custom images.

## Overview

`distrobuilder` uses YAML definition files to describe how an image should be built. This includes:
*   The base distribution and release.
*   Package repositories.
*   Packages to install or remove.
*   Customization scripts to run during the build process.
*   Image metadata (architecture, creation date, etc.).

The output is typically a tarball containing the root filesystem of the container, along with a metadata tarball.

## Installation of Distrobuilder

`distrobuilder` is a Go application. It needs to be installed on a system where you will build the images (this can be the Yggdrasil host itself, or another Debian/Linux system).

```bash
# Ensure Go is installed (e.g., version 1.18 or newer)
sudo apt install golang-go

# Build and install distrobuilder
# (Adjust version as needed, check for latest stable release)
go install github.com/lxc/distrobuilder/distrobuilder@latest
# This will typically install it to ~/go/bin/distrobuilder
# Ensure ~/go/bin is in your PATH
export PATH="${HOME}/go/bin:${PATH}"
```

## Creating a Distrobuilder YAML Definition

Let's create an example YAML file for a minimal Debian Unstable (Sid) image, which could serve as a base for Yggdrasil LXC containers.

File: `debian-sid-minimal.yaml`
```yaml
image:
  description: Minimal Debian Sid (Unstable)
  distribution: debian
  release: sid
  architecture: amd64
  expiry: "30d" # How long the image stays in local cache
  # serial: "YYYYMMDD_HHMM" # Optionally set a serial

source:
  downloader: debootstrap
  url: http://deb.debian.org/debian # Or a local mirror
  keys:
    - /usr/share/keyrings/debian-archive-keyring.gpg
  variant: minbase

packages:
  manager: apt
  update: true
  upgrade: true
  install_recommends: false
  repositories:
    - source: http://deb.debian.org/debian sid main contrib non-free non-free-firmware
      # You can add more repositories here, e.g., security or custom ones.
  # Packages to remove after bootstrap (if any)
  remove:
    # - package-to-remove
  # Packages to install
  install:
    - systemd-sysv
    - init # systemd-sysv should provide this
    - iproute2
    - openssh-server
    - vim-tiny
    - less
    - locales
    - sudo
    # Add any other essential packages for your base image

actions:
  # Runs after packages are installed, before any cleanup.
  # Useful for initial setup within the chroot.
  post-packages:
    - command: |
        set -eux;
        echo "en_US.UTF-8 UTF-8" > /etc/locale.gen;
        locale-gen en_US.UTF-8;
        update-locale LANG=en_US.UTF-8;
        echo "Etc/UTC" > /etc/timezone;
        ln -sf /usr/share/zoneinfo/Etc/UTC /etc/localtime;
        dpkg-reconfigure -f noninteractive tzdata;
        # Add a default user (optional, can be done in container creation)
        # useradd -m -s /bin/bash -G sudo ygg_user;
        # echo "ygg_user:your_password" | chpasswd; # Not recommended for base images
        # Setup SSH
        sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config;
        # Clean apt cache
        apt-get clean;
        rm -rf /var/lib/apt/lists/*;
        # Remove machine-id to ensure uniqueness on container start
        rm -f /etc/machine-id;
        touch /etc/machine-id; # systemd will populate it on first boot

targets:
  # Output LXC container image (rootfs tarball and metadata tarball)
  lxc:
    compression: gzip # or xz for smaller size but slower compression

# Mappings (e.g., for unprivileged containers, often handled by LXC itself)
# mappings:
#   profiles:
#     # Default profile
#     default:
#       uid:
#         - "0:1000000:1" # map host root to container root
#       gid:
#         - "0:1000000:1"
```

## Building the Image

Once the YAML file is ready:

```bash
# Create a directory for the build
mkdir -p ~/distrobuilder_cache
mkdir -p ~/distrobuilder_images

# Run distrobuilder
distrobuilder build-lxc debian-sid-minimal.yaml \
    --cache-dir ~/distrobuilder_cache \
    --output-dir ~/distrobuilder_images

# On successful build, you will find two files in ~/distrobuilder_images:
# - rootfs.tar.gz (or .tar.xz)
# - lxc.meta.tar.gz (or .tar.xz)
# They might be named based on serial or hash.
```

**Explanation of options:**
*   `build-lxc`: Command to build an LXC image.
*   `debian-sid-minimal.yaml`: Path to your YAML definition file.
*   `--cache-dir`: Directory to store downloaded packages and other build artifacts. Reusing this speeds up subsequent builds.
*   `--output-dir`: Directory where the final image tarballs will be placed.

## Using the Built Image with LXC

1.  **Transfer images:** Copy the `rootfs.tar.*` and `lxc.meta.tar.*` (if distrobuilder names them that way, or the single tarball if it packages them together) to your Yggdrasil host, for example, into a directory like `/zroot/lxc/images/`.
    If `distrobuilder` outputs separate `rootfs.tar.gz` and `meta.tar.gz`, you often combine them for LXC:
    ```bash
    # Assuming output files are rootfs.tar.gz and lxc.meta.tar.gz in current dir
    tar -czf my-debian-sid-minimal.tar.gz rootfs.tar.gz lxc.meta.tar.gz
    # Or simply provide the rootfs tarball to lxc-create and it might find the meta automatically if in the same dir or use -F for metadata
    ```
    However, `lxc-create -t local` usually expects a single rootfs tarball. The metadata tarball (`lxc.meta.tar.gz`) is typically used by LXD or when importing into `lxc image import`. For `lxc-create`, you might only need the `rootfs.tar.gz`.

2.  **Create an LXC container using the image:**

    If using a single tarball containing both (or just the rootfs):
    ```bash
    # Assuming your image is at /zroot/lxc/images/my-debian-sid-minimal.tar.gz
    # And your LXC container path is, e.g., /zroot/lxc/containers/
    sudo lxc-create -n my-new-container \
        -t local -- \
        -p /zroot/lxc/containers \
        -i /zroot/lxc/images/my-debian-sid-minimal.tar.gz
    ```

    If you are using ZFS backend for LXC directly with `lxc-create` (which is the Yggdrasil goal):
    ```bash
    # 1. Ensure the ZFS dataset for the container exists or will be created
    #    (e.g., zfs create zroot/lxc/containers/my-new-container)
    # 2. Create the container
    sudo lxc-create -n my-new-container \
        -t local \
        -B zfs --zfsroot=zroot/lxc/containers -- \
        -i /zroot/lxc/images/my-debian-sid-minimal.tar.gz # Path to rootfs tarball
        # The exact options for ZFS backend can vary; consult lxc-create --help
    ```
    Refer to [LXC Configuration](LXC_Configuration.md) for specifics on integrating with ZFS. `distrobuilder` itself just creates the rootfs tarball.

## Tips and Advanced Usage

*   **Source Dates:** For reproducible builds, you can use `SOURCE_DATE_EPOCH` environment variable.
*   **Custom Scripts:** Use the `actions` section for complex customizations. Scripts are run inside the chroot.
*   **Overlays:** You can overlay custom files into the image using the `files` section in the YAML.
*   **Multiple Architectures:** `distrobuilder` supports building for different architectures (e.g., `arm64`) if your build environment supports it (e.g., via qemu-user-static).
*   **Debugging:** If a build fails, `distrobuilder` usually leaves the chroot environment in the cache directory, allowing you to `chroot` into it and debug.

This provides a basic framework for using `distrobuilder` to create the custom LXC images Yggdrasil will utilize. You can create more specialized images by modifying the YAML definition (e.g., an image with specific GPU drivers pre-installed, or a web server image).
