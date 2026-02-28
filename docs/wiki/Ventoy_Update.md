# Ventoy Update Runbook

Use this guide **on `primary-host` itself** when you need to update the Ventoy bootloader
installed on the USB stick. This preserves the data partition (ISOs) while updating
Ventoy's boot files.

## Prerequisites

- Ventoy USB is attached (typically `/dev/sde` on `primary-host`).
- Network access to GitHub for downloads.
- Packages on the host: `curl`, `tar`.

## Procedure

1. Identify the Ventoy device. Confirm the label is `Ventoy` before proceeding:

   ```bash
   lsblk -f
   ```

2. Download and run the latest Ventoy update:

   ```bash
   cd /tmp
   url=$(curl -Ls -o /dev/null -w "%{url_effective}" https://github.com/ventoy/Ventoy/releases/latest)
   ver=${url##*/}
   ver_num=${ver#v}
   tarball="ventoy-${ver_num}-linux.tar.gz"
   curl -L -o "$tarball" "https://github.com/ventoy/Ventoy/releases/download/${ver}/${tarball}"
   rm -rf /tmp/ventoy-update
   mkdir /tmp/ventoy-update
   tar -xzf "$tarball" -C /tmp/ventoy-update
   /tmp/ventoy-update/ventoy-${ver_num}/Ventoy2Disk.sh -u /dev/sde
   ```

3. Set auto-boot to the current ISO and auto-accept the secondary menu (prevents sitting in the menu after a power cut):

   ```bash
   dmsetup remove ventoyfull 2>/dev/null || true
   dmsetup create ventoyfull --table "0 $(blockdev --getsz /dev/sde1) linear /dev/sde1 0"
   mkdir -p /mnt/ventoy
   mount.exfat-fuse /dev/mapper/ventoyfull /mnt/ventoy

   mkdir -p /mnt/ventoy/ventoy
   cat > /mnt/ventoy/ventoy/ventoy.json <<'EOF'
   {
     "control": [
       { "VTOY_MENU_TIMEOUT": "5" },
       { "VTOY_DEFAULT_IMAGE": "/yggdrasil-YYYYMMDDHHMM-amd64.hybrid.iso" },
       { "VTOY_SECONDARY_BOOT_MENU": "1" },
       { "VTOY_SECONDARY_TIMEOUT": "5" }
     ]
   }
   EOF

   sync
   umount /mnt/ventoy
   dmsetup remove ventoyfull
   ```

4. Keep the last known-good ISO on the USB as a fallback. Only replace the "current"
   build when updating the Ventoy data partition.

## Notes

- `-u` updates Ventoy in place and preserves the data partition.
- Always double-check `/dev/sde` before running `Ventoy2Disk.sh`.
- Update `VTOY_DEFAULT_IMAGE` whenever a new ISO is copied so auto-boot targets
  the latest build.
- The secondary boot menu is the screen that offers "Boot in normal mode" and
  "Boot in grub2 mode" after selecting an ISO.
