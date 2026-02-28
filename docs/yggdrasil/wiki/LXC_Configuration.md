# LXC Configuration and Management on Yggdrasil

This document details the setup, configuration, and management of LXC (Linux Containers) on the Yggdrasil system. LXC is the primary containerization technology for Yggdrasil, intended to be a modern replacement for SmartOS zones.

## Overview

Yggdrasil leverages LXC for its lightweight, OS-level virtualization. Key features of LXC on Yggdrasil include:

*   Integration with ZFS for container filesystems.
*   Support for both privileged and unprivileged containers.
*   Network configuration for containers (e.g., dedicated bridge, veth pairs).
*   GPU passthrough capabilities.

## Installation and Basic Setup

LXC is included by default in the Yggdrasil build.

### Verifying LXC Installation

After booting Yggdrasil, you can verify the LXC installation:

```bash
lxc-checkconfig
lxc-ls -f
```

`lxc-checkconfig` will show which kernel features required by LXC are enabled. `lxc-ls -f` will list existing containers (initially none).

### Default Configuration

*   Default LXC path: `/zroot/lxc` (configured via `/etc/lxc/lxc.conf` in the live image).
*   Default networking: macvlan on `eno1` (written into `/etc/lxc/default.conf` during build). The host uses a `mac0` macvlan interface for host-to-guest access.

## ZFS Integration

One of the core features of Yggdrasil is storing LXC container root filesystems on dedicated ZFS datasets. This provides benefits like:

*   Snapshots and clones for individual containers.
*   Data integrity.
*   Efficient storage utilization.

### Recommended ZFS Layout for LXC

It's recommended to create a parent ZFS dataset for all LXC containers, for example:

```bash
zfs create zroot/lxc
```

Each container's rootfs will then reside in a child dataset, e.g., `zroot/lxc/mycontainer`.

### Creating a Container with ZFS Backend

When creating a container, you can specify ZFS as the backing store. The `lxc-create` command might have options for this, or it might involve pre-creating the ZFS dataset and then pointing LXC to use it.

**Example (conceptual):**

```bash
# 1. Create the ZFS dataset for the container
zfs create zroot/lxc/my-web-server

# 2. Create the LXC container using this path as its rootfs
# The exact command might vary based on templates and LXC version.
# It might involve setting 'lxc.rootfs.path = zfs:zroot/lxc/my-web-server'
# or using a template that handles ZFS dataset creation.
lxc-create -t debian -n my-web-server -- -p /zroot/lxc/my-web-server --backend zfs
# OR, if using lxc.conf directly:
# lxc.rootfs.path = /zroot/lxc/my-web-server/rootfs
```
(The exact mechanism for integrating `lxc-create` with ZFS dataset creation/management needs to be detailed based on `mkconfig.sh` implementation or LXC best practices on Debian).

Alternatively, if LXC's ZFS backend support handles dataset creation:
```bash
lxc-create -t debian -n my-web-server -B zfs --zfsroot=zroot/lxc
```

## Container Creation

Containers can be created using `lxc-create` with various templates. `distrobuilder` will be the primary tool for creating custom base images/templates for Yggdrasil.

### Using `distrobuilder` Images

Refer to [Distrobuilder_Usage.md](Distrobuilder_Usage.md) for creating custom images. Once an image (as a tarball) is available:

```bash
lxc-create -t local -n <container-name> -- -i /path/to/distrobuilder-image.tar.gz
```

### Standard Templates

LXC also provides standard distribution templates (e.g., `debian`, `ubuntu`):

```bash
lxc-create -t debian -n mydebiancontainer
```

## Container Configuration (`config` file)

Each container has a configuration file located at `/var/lib/lxc/<container-name>/config` (or `${LXC_PATH}/<container-name>/config`).

Key configuration items:

*   **Networking:**
    ```
    lxc.net.0.type = veth
    lxc.net.0.link = lxcbr0 # or your custom bridge
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:xx:yy:zz
    # For static IP:
    # lxc.net.0.ipv4.address = 10.0.3.10/24
    # lxc.net.0.ipv4.gateway = 10.0.3.1
    ```
*   **Root Filesystem:**
    ```
    lxc.rootfs.path = /zroot/lxc/<container-name>/rootfs # Or zfs:zroot/lxc/<container-name>
    lxc.rootfs.mount = /zroot/lxc/<container-name>/rootfs # if path is not a mountpoint itself
    ```
*   **Mounts:**
    ```
    # Bind mount a directory from host to container
    lxc.mount.entry = /path/on/host mnt/host_share none bind 0 0
    # Mount a ZFS dataset directly
    lxc.mount.entry = zroot/data/container_data_dataset media/data none bind,rw,create=dir 0 0
    ```
*   **AppArmor/Seccomp Profiles:**
    ```
    lxc.apparmor.profile = generated  # or a custom profile
    lxc.seccomp.profile = /usr/share/lxc/config/common.seccomp
    ```
*   **GPU Passthrough:** See [GPU_Passthrough.md](GPU_Passthrough.md). This will involve `lxc.cgroup.devices.allow` and `lxc.mount.entry` for device nodes.

## Managing Containers

Standard `lxc-*` commands:

*   `lxc-start -n <container-name> -d` (start in background)
*   `lxc-stop -n <container-name>`
*   `lxc-attach -n <container-name>` (get a shell in the container)
*   `lxc-info -n <container-name>`
*   `lxc-ls -f` (list containers with details)
*   `lxc-freeze -n <container-name>`
*   `lxc-unfreeze -n <container-name>`
*   `lxc-destroy -n <container-name>` (Warning: this deletes the container and its filesystem if not carefully managed with ZFS snapshots separately).

### Autostart ordering on primary-host

`ygg-lxc-autostart.service` starts all containers with `lxc.start.auto = 1`. Use `lxc.start.order` and `lxc.start.delay` in each container config for ordering. On `primary-host`, `ygg-infisical-ensure.service` runs after autostart to ensure Infisical is up before dependent stacks (ownCloud) are refreshed. See the [LXC fleet inventory](./lxc/containers.md) for current ordering.

## Unprivileged Containers

For enhanced security, running containers as unprivileged users is recommended. This requires:

1.  User namespace support in the kernel (enabled in Yggdrasil).
2.  SubUID and SubGID ranges defined for the user running the container (e.g., `root` or a dedicated user).
    *   Edit `/etc/subuid` and `/etc/subgid`.
    *   Example for root running unprivileged containers:
        ```
        root:100000:65536
        root:100000:65536
        ```
3.  Container configuration adjustments (LXC often handles this automatically if `lxc.idmap` is not set, or you can set it explicitly).

Creating an unprivileged container:

```bash
lxc-create -t debian -n my-unprivileged-container
# (Additional configuration for rootfs path, networking, etc., as needed)
```
Starting an unprivileged container as root:
```bash
lxc-start -n my-unprivileged-container -d
```
The container processes will run as mapped UIDs/GIDs on the host, not as root.

## Networking Details

*   **Default Bridge (`lxcbr0`):** Often set up by `lxc-net` service or similar scripts. Provides NATed access for containers.
*   **Custom Bridge:** You might create your own bridge (e.g., `br0` linked to a physical interface) for direct network access for containers.
    *   Containers' `veth` pairs would then connect to this custom bridge.
*   **Static IPs vs DHCP:** Configure per container or manage via a DHCP server on the LXC bridge.
*   **No Network:** `lxc.net.0.type = empty`

### Host access to macvlan containers

When LXC guests are attached directly to the physical interface via macvlan (as configured in the build), the host itself cannot talk to those guests because Linux prohibits a parent interface from communicating with its macvlan children.  

To restore host connectivity, Yggdrasil now configures a dedicated host macvlan interface (`mac0`) on `eno1`:

```bash
ip link add mac0 link eno1 type macvlan mode bridge
ip addr add 10.10.0.250/24 dev mac0
ip link set mac0 up
ip route replace 10.10.0.0/24 dev mac0
```

The build drops `/usr/local/sbin/ygg-setup-mac0` and enables the `ygg-macvlan-mac0.service` systemd unit so this configuration persists across reboots. Adjust `MACVLAN_CIDR`/`MACVLAN_ROUTE` inside that helper script if your addressing scheme changes.

## GPU Passthrough for LXC

See [GPU_Passthrough.md](GPU_Passthrough.md) for detailed instructions. This typically involves:
*   Ensuring host drivers are loaded.
*   Allowing the container cgroup access to the GPU device nodes (`/dev/dri/*`, `/dev/nvidia*`).
*   Mounting the device nodes into the container.
*   Installing userspace drivers *inside* the container.

## Backup and Restore

*   **ZFS Snapshots:** The primary method for backing up LXC containers on Yggdrasil.
    ```bash
    zfs snapshot zroot/lxc/mycontainer@backup-timestamp
    zfs send zroot/lxc/mycontainer@backup-timestamp | zfs recv backup-pool/lxc-backups/mycontainer
    ```
*   **LXC Snapshots (if not using ZFS backend directly):** `lxc-snapshot -n <container-name>` (These are LVM or overlayfs snapshots, less ideal than ZFS).

## Security Considerations

*   Run containers as unprivileged where possible.
*   Use AppArmor and Seccomp profiles.
*   Keep container images updated.
*   Limit container capabilities.
*   Regularly review LXC and kernel security advisories.

## Troubleshooting

*   `lxc-checkconfig`: Identify missing kernel features.
*   Container logs: `/var/log/lxc/<container-name>.log` or `${LXC_PATH}/<container-name>/lxc.log`.
*   `dmesg` on the host for hardware or kernel-related issues.
*   `lxc-attach` to debug issues inside the container.

This document will be expanded with more specific examples and configurations as Yggdrasil is developed and tested.
