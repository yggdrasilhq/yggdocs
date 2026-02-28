# Ventoy ISO Copy Runbook

Use this guide **on `primary-host` itself** whenever you need to push a freshly built Yggdrasil ISO from the LXC build zone (`/zroot/lxc/dev`) onto the Ventoy USB stick that `primary-host` booted from. Do not run this from a different host; all device paths and mounts below assume you are logged into `primary-host`.

## Prerequisites

These packages must already be installed in the environment to avoid extra apt work mid-operation (the base image now includes them):

- `exfat-fuse`, `exfatprogs`
- `dmsetup`, `lvm2`
- `parted`
- `lsof`, `psmisc`
- `util-linux`

Also confirm you have the ISO under `/zroot/lxc/dev/rootfs/root/git/yggdrasil/` (e.g., `yggdrasil-YYYYMMDDHHMM-amd64.hybrid.iso`).

## Procedure

1. Identify the Ventoy data partition (usually `/dev/sde1` when booted from Ventoy). Verify with `lsblk` if unsure. The partition is mounted read-only by the live system, so we map it manually.
2. Reset/create the writable mapper and mount the exFAT filesystem:

   ```bash
   dmsetup remove ventoyfull 2>/dev/null || true
   dmsetup create ventoyfull --table "0 $(blockdev --getsz /dev/sde1) linear /dev/sde1 0"
   mkdir -p /mnt/ventoy
   mount.exfat-fuse /dev/mapper/ventoyfull /mnt/ventoy
   ```

3. Copy the ISO(s) from the build tree to Ventoy. Keep the last known-good ISO as a fallback and only replace the current build:

   ```bash
   # Keep the last known-good ISO (rename it if needed), prune everything else.
   BACKUP_ISO="yggdrasil-LAST-KNOWN-GOOD.hybrid.iso"
   for iso in /mnt/ventoy/yggdrasil-*.hybrid.iso; do
     case "$(basename "$iso")" in
       "$BACKUP_ISO") ;;
       *) rm -f "$iso" ;;
     esac
   done
   cp -av /zroot/lxc/dev/rootfs/root/git/yggdrasil/yggdrasil-*.hybrid.iso /mnt/ventoy/
   sync
   ```

4. Cleanly detach and remove the mapper:

   ```bash
   umount /mnt/ventoy || true
   dmsetup remove ventoyfull
   ```

5. (Optional) Remount and list contents to confirm:

   ```bash
   mount.exfat-fuse /dev/mapper/ventoyfull /mnt/ventoy
   ls -lh /mnt/ventoy
   umount /mnt/ventoy
   dmsetup remove ventoyfull
   ```

## Notes

- Always run `sync` before unmounting to ensure the copy is flushed to the USB stick.
- If `mount.exfat-fuse` complains about the device being busy, remove the mapper, wait a few seconds, and recreate it.
- Do **not** reuse the existing Ventoy mapper exposed by the live system—it’s read-only. The temporary `ventoyfull` mapper created above is writable and scoped to the copy session.
