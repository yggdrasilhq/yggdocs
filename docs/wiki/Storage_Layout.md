# Storage Layout

This page captures the storage layout on `primary-host.example` so future updates can rehydrate the same dataset structure.

## ZFS pools
- `zroot` is the primary pool for the live host, LXC rootfs datasets, and shared data.
- `zbackup` is a secondary pool used for backup datasets.

## LXC rootfs datasets
- LXC rootfs datasets live under `zroot/lxc/<container>` and mount at `/zroot/lxc/<container>/rootfs`.
- The host LXC path is pinned to `/zroot/lxc` (`/etc/lxc/lxc.conf`).
- For container inventory, see [LXC fleet recipes](./lxc/containers.md).

## Shared data datasets
- Shared data for containers lives under `zroot/data`.
- Common subtrees used by containers include:
  - `zroot/data/lxcfs` (shared bind mounts like `books`, `calibre`, `peertube`)
  - `zroot/data/smbfs` (Samba shares)
  - `zroot/data/archive` (archive datasets)
  - `zroot/data/immich*` (Immich photo libraries)

## Inventory command
Run this on `primary-host` to see the full dataset inventory:

```bash
zfs list -r -o name,mountpoint,used,avail,refer zroot/lxc zroot/data
```
