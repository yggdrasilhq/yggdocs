# ZFS Setup and Best Practices on Yggdrasil

This document details the ZFS (Zettabyte File System) configuration, management, and best practices within the Yggdrasil project. ZFS is a cornerstone of Yggdrasil, providing robust storage for the host OS (via live boot persistence if configured, though primarily for data pools) and for LXC container filesystems.

## Overview

Yggdrasil aims to leverage ZFS for:

*   **Data Integrity:** Checksums, self-healing, and copy-on-write.
*   **Flexibility:** Snapshots, clones, send/receive for backups and migration.
*   **Performance:** ARC (Adaptive Replacement Cache), L2ARC, SLOG.
*   **Container Storage:** Dedicated datasets for LXC containers.
*   **Data Pools:** Management of the primary data pool `zroot`.

## `zroot` Pool: The Primary Data Store

The main ZFS pool in Yggdrasil will be named `zroot`.

### Initial `zroot` Setup (Single Drive)

Initially, to facilitate the transition from the existing SmartOS `zones` pool, `zroot` will be created on a single new 12TB HDD.

**Steps:**

1.  **Identify the new HDD:** After booting Yggdrasil, use `lsblk`, `fdisk -l`, or `dmesg` to identify the device name for the new 12TB drive (e.g., `/dev/sdX`).
2.  **Partitioning (Optional but Recommended):**
    *   It's good practice to use GPT partitions. You can create a single large partition spanning most of the disk, leaving some space at the end if desired (though ZFS can use whole disks).
    *   Use `gdisk /dev/sdX`. Create a partition of type `BF00` (Solaris /usr & Apple ZFS) or `BF01` (Solaris Reserved 1) if you want a specific ZFS type, or a standard Linux filesystem type if you prefer tools to recognize it easily before ZFS claims it.
    *   Example using `sgdisk` for a whole-disk partition for ZFS (replace `sdX`):
        ```bash
        sgdisk --zap-all /dev/sdX
        sgdisk --new=1:0:0 --typecode=1:BF00 --change-name=1:"zfs-zroot-1" /dev/sdX
        partprobe /dev/sdX
        ```
    *   Use `/dev/sdX1` (or the partition ID like `/dev/disk/by-partlabel/zfs-zroot-1`) for pool creation. Using IDs (`/dev/disk/by-id/...` or partlabels) is generally more robust than `/dev/sdX` names.

3.  **Creating the `zroot` Pool:**
    ```bash
    # Using a partition label (recommended)
    zpool create -f \
        -o ashift=12 \             # Assuming 4K sector drive (common for large HDDs)
        -o autotrim=on \           # If using SSDs in future, good to have
        -O compression=lz4 \       # Good default compression
        -O acltype=posixacl \
        -O xattr=sa \
        -O dnodesize=auto \
        -O normalization=formD \   # Good for cross-platform compatibility
        -O relatime=on \
        -m none \                  # Mountpoint managed by datasets
        zroot /dev/disk/by-partlabel/zfs-zroot-1 # Or /dev/sdX1
    ```
    *   `-f`: Force. Use if the disk/partition contains existing data/signatures.
    *   `ashift=12`: Crucial for 4K native drives. Check your drive's sector size. `ashift=13` for 8K.
    *   `-m none`: Prevents `zroot` itself from being mounted at `/zroot`. Mountpoints will be set on child datasets.

4.  **Basic Dataset Structure for `zroot`:**
    ```bash
    # Parent dataset for LXC containers
    zfs create -o mountpoint=/var/lib/lxc zroot/lxc # Or symlink /var/lib/lxc to /zroot/lxc
    # If /var/lib/lxc is used by lxc tools directly, it might be better to:
    # zfs create -o mountpoint=none zroot/lxc # if lxc tools/templates manage sub-datasets
    # zfs create -o mountpoint=/zroot/lxc zroot/lxc # if you want it at /zroot/lxc

    # Parent dataset for general data, to be migrated from 'zones'
    zfs create -o mountpoint=/zroot/data zroot/data

    # Potentially for KVM/Xen images if used
    zfs create -o mountpoint=/zroot/vms zroot/vms
    zfs create -o recordsize=128k zroot/vms # Larger recordsize for VM images can be beneficial

    # Other common datasets
    zfs create -o mountpoint=/opt zroot/opt
    zfs create -o mountpoint=/srv zroot/srv
    # etc.
    ```

### Transitioning to RAID-Z1

Once data is migrated from the `zones` pool (which consists of 2x 12TB HDDs in a mirror) to the single-drive `zroot`, the plan is to incorporate the two older drives into `zroot` to form a 3x 12TB RAID-Z1 configuration.

**IMPORTANT:** Converting a single-disk pool to RAID-Z1 by adding disks is **NOT** a direct `zpool add` operation. You must *replace* the single disk with a RAID-Z1 vdev, or more practically, create a new pool and transfer data.

**Scenario A: Recreate `zroot` as RAID-Z1 (Requires data backup/restore or temporary storage)**

1.  **Ensure all data from the old `zones` pool AND the temporary single-disk `zroot` is backed up elsewhere.** This is the safest.
2.  Destroy the old `zones` pool to free up its two 12TB HDDs.
3.  Destroy the temporary single-disk `zroot` pool to free up the third 12TB HDD.
4.  Identify all three 12TB HDDs (e.g., `/dev/sdX1`, `/dev/sdY1`, `/dev/sdZ1` using partition labels or by-id paths).
5.  Create the new RAID-Z1 `zroot` pool:
    ```bash
    zpool create -f \
        -o ashift=12 \
        -O compression=lz4 \
        # ... other options as before ...
        -m none \
        zroot raidz1 /dev/disk/by-partlabel/zfs-zroot-1 /dev/disk/by-partlabel/zfs-zones-1 /dev/disk/by-partlabel/zfs-zones-2
        # (Adjust disk names/paths as identified)
    ```
6.  Recreate the desired dataset structure (e.g., `zroot/lxc`, `zroot/data`).
7.  Restore data from backup to the new `zroot` pool.

**Scenario B: Gradual transition (more complex, involves `zfs send/recv`)**

This is only feasible if you have enough temporary free space to shuffle data. If the data on the single-disk `zroot` is significant, this becomes hard without a fourth large drive. A full backup and recreate (Scenario A) is often simpler and safer for this kind of vdev topology change.

If you *must* attempt an in-place style migration using the existing disks:

1.  **Data migration:** Ensure all critical data from `zones` is on the single-disk `zroot`.
2.  **Free up one `zones` disk:** If `zones` is a mirror, you can `zpool detach zones <disk1>` to free one disk. *The `zones` pool will become degraded.*
3.  **Attach to `zroot` as mirror:** Add this freed disk to `zroot` as a mirror: `zpool attach zroot <current_zroot_disk> <freed_zones_disk1>`. Wait for resilvering. `zroot` is now a 2-disk mirror.
4.  **Free up second `zones` disk:** Detach the remaining disk from `zones`: `zpool detach zones <disk2>`. Destroy the (now empty or single-disk) `zones` pool: `zpool destroy zones`.
5.  **The Challenge:** You now have a 2-disk mirror `zroot` and one free disk. You cannot directly convert this 2-disk mirror into a 3-disk RAID-Z1. ZFS vdevs cannot be reshaped in this way.
    *   You would need to create a *new* temporary pool on the third disk, `zfs send/recv` all data from the 2-disk mirror `zroot` to this new temporary pool, destroy the 2-disk mirror `zroot`, and then re-create `zroot` as a 3-disk RAID-Z1 using all three disks, and finally `zfs send/recv` the data back. This is very convoluted and risky.

**Recommendation:** For changing `zroot` from a single disk to a 3-disk RAID-Z1, **Scenario A (backup, destroy, recreate, restore)** is strongly recommended for simplicity and safety.

## ZFS for LXC Containers

*   **Parent Dataset:** `zroot/lxc` (or a similar structure like `zroot/containers`).
*   **Individual Container Datasets:** Each LXC container should have its own child dataset, e.g., `zroot/lxc/<container_name>`.
    *   This allows for independent snapshots, clones, and `zfs send/recv` operations for each container.
    *   Example: `zfs create zroot/lxc/mywebserver`
*   **LXC Configuration:** LXC needs to be configured to use these ZFS datasets for its root filesystems.
    *   This might involve setting `lxc.rootfs.path = zfs:zroot/lxc/<container_name>` in the container's config, or using LXC templates that are ZFS-aware.
    *   The actual mount point inside the container would be `/`, but its backing store is the ZFS dataset.
*   **Snapshots:** Regularly snapshot container datasets before updates or major changes.
    `zfs snapshot zroot/lxc/mywebserver@before-upgrade`
*   **Clones:** Quickly provision new containers from a base snapshot/template.
    `zfs clone zroot/lxc/base-debian@latest zroot/lxc/new-debian-container`

## General ZFS Best Practices on Yggdrasil

*   **Monitoring:**
    *   Regularly check pool health: `zpool status -v zroot`
    *   Monitor disk errors (`smartctl -a /dev/sdX`).
    *   Set up ZFS event daemon (`zed`) for notifications on pool events/errors.
*   **Scrubbing:** Schedule regular ZFS scrubs to detect silent data corruption:
    `zpool scrub zroot` (Typically once a week or once a month for large pools).
*   **`ashift`:** Ensure `ashift=12` (for 4K sector drives) or `ashift=13` (for 8K) is set at pool creation. This cannot be changed later.
*   **Record Size:**
    *   Default is 128K. Good for general file storage.
    *   For databases or VM images, consider larger record sizes if benchmarks show improvement (e.g., 1M). Test this for your specific workloads.
    *   For datasets with many small files, a smaller recordsize might be considered, but often `compression=lz4` handles this well enough.
*   **Compression:** `lz4` is generally recommended as it provides a good balance of compression ratio and low CPU overhead.
    `zfs set compression=lz4 zroot` (can be set on parent and inherited).
*   **ARC Tuning (Advanced):**
    *   Debian ZFS usually has reasonable defaults.
    *   Given 512GB RAM, ZFS will use a significant portion for ARC. Monitor with `arc_summary` (from `zfsutils-linux` or `zfs-zed` package).
    *   You can limit ARC size via `/etc/modprobe.d/zfs.conf` if needed (e.g., `options zfs zfs_arc_max=<bytes>`), but usually letting ZFS manage it is fine with ample RAM.
*   **Snapshots:** Use snapshots liberally for rollback points and backups. Manage snapshot retention (e.g., using tools like `znapzend` or custom scripts).
*   **Backups:** ZFS snapshots are local. Use `zfs send/recv` to send snapshots to another pool, another server, or an external drive for actual backups.
*   **TRIM/Discard:** If using SSDs in `zroot` (even as cache or SLOG), ensure `autotrim=on` is set on the pool (or run `zpool trim` periodically). For HDDs, this is not applicable.

## `zones` Pool Migration

The existing SmartOS `zones` pool (2x 12TB mirror) needs its data migrated to `zroot/data`.

1.  **Mount `zones` pool (if possible on Yggdrasil):**
    *   Yggdrasil's ZFS version should be compatible with the SmartOS ZFS pool version.
    *   `zpool import -f zones` (You might need to specify `-R /mnt/zones_import` if there are mountpoint conflicts).
2.  **Data Transfer:** Use `rsync` or `zfs send/recv` (if creating an intermediate dataset on `zroot` that matches the structure of a dataset on `zones`).
    *   **`rsync` (simpler for differing dataset structures):**
        ```bash
        rsync -avP /mnt/zones_import/path/to/data/ /zroot/data/target_path/
        ```
    *   **`zfs send/recv` (more efficient if dataset structure allows):**
        This requires taking a snapshot on the `zones` dataset and receiving it into a `zroot` dataset.
        ```bash
        # On SmartOS (or if zones is imported read-only on Yggdrasil temporarily)
        # zfs snapshot zones/dataset@migration
        # zfs send zones/dataset@migration | ssh yggdrasil_host zfs recv zroot/data/dataset_migrated
        # OR, if zones is imported on Yggdrasil:
        # zfs send zones/dataset@migration | zfs recv zroot/data/dataset_migrated
        ```
3.  **Verification:** Thoroughly verify all data has been copied correctly.
4.  **Decommission `zones`:** Once data is safely on `zroot` and verified, the `zones` pool can be exported and its disks repurposed.
    `zpool export zones`

This ZFS setup provides a robust foundation for Yggdrasil. Further tuning and specific dataset properties can be adjusted based on observed workloads and specific application needs.