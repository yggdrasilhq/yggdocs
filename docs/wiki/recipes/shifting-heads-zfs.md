# Shifting Heads with ZFS (client-a/client-b <-> build-host LXC replicas)

This guide documents a host-replication workflow where each laptop/workstation ("head") can be pushed to, and pulled from, a canonical replica on `build-host`.

Goal:
- Keep host state reproducible and movable.
- Let any physical machine "become" another host identity by receiving that host's datasets.
- Support remote GUI work by connecting to the matching LXC replica on `build-host`.

## Model

- Source hosts: `client-a`, `client-b`, `client-c`, etc.
- Replica host: `root@build-host.example`.
- On `build-host`, each host has its own dataset tree under a shared parent:
  - `zfs/lxc/client-a/...`
  - `zfs/lxc/client-b/...`
- Replicas are updated with `zfs send | ssh ... zfs recv`.
- Working rule: whichever side you worked on last is the side that must push next.

## Design Rules

1. Use recursive snapshots with a consistent prefix, for example:
   - `@shift-YYYYMMDD-HHMMSS`
2. Do not receive directly into currently-mounted root datasets on a running machine.
3. Keep a "latest known good" snapshot before each receive.
4. Use one dataset namespace per host identity (`client-a` is always `.../client-a`).
5. Prefer incremental replication (`-I` or `-i`) after initial full seed.

## Example Dataset Mapping

Adjust these to your actual pools/datasets:

- `client-a` local root tree: `rpool/ROOT/debian`, `rpool/home/pi`, etc.
- `client-a` replica root on `build-host`: `zfs/lxc/client-a/ROOT/debian`, `zfs/lxc/client-a/home/pi`, etc.
- `client-b` replica root on `build-host`: `zfs/lxc/client-b/...`

If your root layout differs (for example only `rpool/ROOT`), keep the same pattern: one source tree -> one per-host replica tree.

## First Full Push (host -> build-host)

Run on source host (example: `client-a`):

```bash
set -euo pipefail

SRC="rpool/ROOT/debian"
DST="zfs/lxc/client-a/ROOT/debian"
STAMP="$(date +%Y%m%d-%H%M%S)"
SNAP="shift-${STAMP}"

sudo zfs snapshot -r "${SRC}@${SNAP}"
sudo zfs send -Rw "${SRC}@${SNAP}" \
  | ssh root@build-host.example "zfs recv -uF ${DST}"
```

Notes:
- `-R` sends descendants/properties.
- `-w` sends raw encrypted data if the dataset is encrypted.
- `-u` keeps received filesystems unmounted initially.
- `-F` rolls back destination to accept stream; use carefully.

## Periodic Incremental Push (host -> build-host)

Run on source host:

```bash
set -euo pipefail

SRC="rpool/ROOT/debian"
DST="zfs/lxc/client-a/ROOT/debian"
PREV="$(sudo zfs list -H -t snapshot -o name -s creation | grep "^${SRC}@shift-" | tail -n1 | cut -d@ -f2)"
STAMP="$(date +%Y%m%d-%H%M%S)"
NEXT="shift-${STAMP}"

sudo zfs snapshot -r "${SRC}@${NEXT}"
sudo zfs send -Rw -I "@${PREV}" "${SRC}@${NEXT}" \
  | ssh root@build-host.example "zfs recv -uF ${DST}"
```

If no previous snapshot exists, do a full push first.

## Pull Changes Back (build-host -> host)

Use this after you worked in the `client-a` LXC replica and want the physical `client-a` to catch up.

On `build-host`:

```bash
set -euo pipefail

SRC="zfs/lxc/client-a/ROOT/debian"
STAMP="$(date +%Y%m%d-%H%M%S)"
SNAP="shift-${STAMP}"

zfs snapshot -r "${SRC}@${SNAP}"
zfs send -Rw "${SRC}@${SNAP}" \
  | ssh client-a "sudo zfs recv -uF rpool/ROOT/debian"
```

For incremental pulls, use `-I` from prior shared snapshot name the same way as push.

## New Laptop Bootstrap ("become client-a")

Target outcome: first boot into live environment, receive datasets; second boot is effectively `client-a`.

1. Boot Debian Sid live media with ZFS tools available.
2. Wipe/install target pool (example: `rpool`) and create minimal bootable ZFS layout.
3. Receive `client-a` dataset tree from `build-host`.
4. Ensure bootloader + initramfs references the received root dataset.
5. Reboot into received root.

Example receive (run from live environment):

```bash
set -euo pipefail

ssh root@build-host.example "zfs snapshot -r zfs/lxc/client-a/ROOT/debian@shift-bootstrap"
ssh root@build-host.example "zfs send -Rw zfs/lxc/client-a/ROOT/debian@shift-bootstrap" \
  | zfs recv -uF rpool/ROOT/debian
```

Then set boot dataset if needed:

```bash
zpool set bootfs=rpool/ROOT/debian rpool
```

### Real Workflow Reference: `client-a` -> live `client-c` (`192.168.0.240`)

This is the concrete sequence used for first-boot head cloning to a blank internal SSD while booted from live media.

1. Prepare/verify target disk layout on the live host (`client-c`):

```bash
sudo sgdisk --zap-all /dev/nvme0n1
sudo sgdisk -n1:1M:+512M -t1:EF00 -c1:EFI /dev/nvme0n1
sudo sgdisk -n2:0:-16G -t2:BF01 -c2:ZFS /dev/nvme0n1
sudo sgdisk -n5:0:0 -t5:8200 -c5:swap /dev/nvme0n1
sudo mkfs.vfat -F32 /dev/nvme0n1p1
sudo mkswap /dev/nvme0n1p5
```

2. Create encrypted pool on `client-c` (zstd, passphrase key file during install):

```bash
printf '%s\n' "${POOL_PASSPHRASE:?set POOL_PASSPHRASE}" >/tmp/zroot.key
chmod 600 /tmp/zroot.key
sudo zpool create -f -o ashift=12 \
  -O atime=off -O xattr=sa -O acltype=posixacl -O dnodesize=auto \
  -O normalization=formD -O relatime=on -O compression=zstd \
  -O encryption=on -O keyformat=passphrase -O keylocation=file:///tmp/zroot.key \
  -O mountpoint=none zroot /dev/nvme0n1p2
sudo zfs create -o mountpoint=none zroot/ROOT
```

3. On `client-a`, create a replication snapshot:

```bash
sudo zfs snapshot -r zroot@headclone-$(date +%Y%m%d-%H%M%S)
```

4. Launch resilient replication from `client-a` as a transient unit:

```bash
sudo systemd-run --unit=clone-client-a-client-c \
  --description='Clone client-a zfs datasets to client-c' \
  --property=User=pi --property=Group=pi \
  --setenv=HOME=/home/pi --setenv=USER=pi --setenv=LOGNAME=pi \
  /tmp/clone_client_a_to_client_c_resume.sh
```

5. Monitor:

```bash
sudo systemctl status clone-client-a-client-c.service
sudo journalctl -u clone-client-a-client-c.service -f
ssh pi@192.168.0.240 'sudo zpool list -H -o alloc,free zroot'
```

6. Script responsibilities after data copy:
- set mountpoints/canmount and `bootfs`
- import pool under `/mnt`, mount root datasets
- sync `/boot/efi/EFI` from `client-a`
- write `/etc/zfs/zroot.key`
- set hostname to `client-c.gour.top`
- disable `ygg-kmonad-Y540.service`
- purge NVIDIA packages
- remove apt preference files containing `trixie`
- install latest sid kernel/meta (`linux-image-amd64`, `linux-headers-amd64`)
- apply hardware-specific firmware policy
- remove unneeded firmware/microcode families for the target hardware profile
- update swap UUID in `/etc/fstab`
- export pool cleanly

7. Re-run behavior after interruption:
- receive uses `zfs recv -s` and resumes from `receive_resume_token` automatically
- completed datasets are skipped by snapshot presence check
- safe to rerun the same script until `MIGRATION COMPLETE`

## Identity Switch (current client-a machine becomes client-b)

General sequence:

1. Push current `client-a` to `build-host` `zfs/lxc/client-a/...` (preserve latest state).
2. Pull `zfs/lxc/client-b/...` into local target datasets.
3. Update host identity bits (`/etc/hostname`, SSH host keys if required, networking config).
4. Reboot and validate identity/services.

Treat this like a controlled role swap; do not mix two identities in the same active root without a clean receive/boot boundary.

## Automation Pattern

Use `scripts/zfs/shift-sync.sh` plus systemd timer automation.

Files added in this repo:
- `scripts/zfs/shift-sync.sh`
- `scripts/zfs/shift-prune.sh`
- `templates/services/ygg-shift-sync.service.template`
- `templates/timers/ygg-shift-sync.timer.template`
- `templates/services/ygg-shift-sync-pull.service.template`
- `templates/timers/ygg-shift-sync-pull.timer.template`
- `templates/services/ygg-shift-prune.service.template`
- `templates/timers/ygg-shift-prune.timer.template`
- `config/shift-sync/ygg-shift-sync.env.template`
- `config/shift-sync/ygg-shift-sync-pull.env.template`
- `config/shift-sync/ygg-shift-prune.env.template`

### Install and Configure (on a client)

1. Install the units:

```bash
bash scripts/install/install-service.sh
```

Select:
- `ygg-shift-sync.service`
- `ygg-shift-sync.timer`

2. Configure runtime variables:

```bash
sudo cp config/shift-sync/ygg-shift-sync.env.template /etc/default/ygg-shift-sync
sudoedit /etc/default/ygg-shift-sync
```

At minimum, set:
- `SHIFT_IDENTITY` (for example `client-a`),
- `SHIFT_SRC_DATASET`,
- `SHIFT_DST_DATASET`.

Recommended for split-root hosts: set `SHIFT_DATASET_MAPS` instead of single src/dst.
Example:

```bash
SHIFT_DATASET_MAPS=zroot/ROOT/debian=zroot/lxc/{identity}/ROOT/debian,zroot/home/pi=zroot/lxc/{identity}/ROOT/home/pi
```

3. Enable periodic sync:

```bash
sudo systemctl enable --now ygg-shift-sync.timer
```

The service has `ConditionPathExists=/etc/default/ygg-shift-sync`, so it will not run until the env file exists.

### CLI Examples

Push current host into `build-host` replica:

```bash
sudo scripts/zfs/shift-sync.sh push \
  --identity client-a \
  --src rpool/ROOT/debian \
  --dst zfs/lxc/client-a/ROOT/debian \
  --remote root@build-host.example \
  --mode auto
```

Pull from `build-host` replica back to local host:

```bash
sudo scripts/zfs/shift-sync.sh pull \
  --identity client-a \
  --src zfs/lxc/client-a/ROOT/debian \
  --dst rpool/ROOT/debian \
  --remote root@build-host.example \
  --mode auto
```

Dry-run preview:

```bash
sudo scripts/zfs/shift-sync.sh push --dry-run
```

### Periodic Pull Profile

Do **not** run periodic pull into a live booted root dataset. `zfs recv -F` on active system datasets is high risk.
Use pull only in controlled/offline contexts (live USB, rescue environment, or non-mounted target datasets).

1. Install pull units only if your target datasets are offline/non-mounted:

```bash
sudo systemctl enable --now ygg-shift-sync-pull.timer
```

2. Configure pull env file (keep `SHIFT_ALLOW_LIVE_RECV=0` unless you intentionally override):

```bash
sudo cp config/shift-sync/ygg-shift-sync-pull.env.template /etc/default/ygg-shift-sync-pull
sudoedit /etc/default/ygg-shift-sync-pull
```

### Snapshot Pruning

Use `shift-prune.sh` to keep only recent `shift-*` snapshots per mapped dataset.

Manual dry-run:

```bash
sudo bash -lc 'set -a; . /etc/default/ygg-shift-sync; . /etc/default/ygg-shift-prune; set +a; SHIFT_DRY_RUN=1 /home/pi/git/ygg_client/scripts/zfs/shift-prune.sh'
```

Enable timer:

```bash
sudo systemctl enable --now ygg-shift-prune.timer
```

Retention config:

```bash
sudo cp config/shift-sync/ygg-shift-prune.env.template /etc/default/ygg-shift-prune
sudoedit /etc/default/ygg-shift-prune
```

## My Reference (real client-a/build-host workflow)

This section captures concrete commands for the current `client-a` <-> `build-host` setup.

### Current mapping used on client-a

`/etc/default/ygg-shift-sync`:

```bash
SHIFT_ACTION=push
SHIFT_REMOTE=root@build-host.example
SHIFT_IDENTITY=client-a
SHIFT_DATASET_MAPS=zroot/ROOT/debian=zroot/lxc/{identity}/ROOT/debian,zroot/home/pi=zroot/lxc/{identity}/ROOT/home/pi
SHIFT_MODE=auto
SHIFT_SNAPSHOT_PREFIX=shift
SHIFT_SEND_FLAGS=-Rw
SHIFT_RECV_FLAGS=-uF
SHIFT_RECURSIVE=1
SHIFT_DRY_RUN=0
```

### Real Workflow A: normal client-a -> build-host push

```bash
sudo systemctl start ygg-shift-sync.service
```

Or one-off CLI:

```bash
sudo scripts/zfs/shift-sync.sh push \
  --identity client-a \
  --remote root@build-host.example \
  --map zroot/ROOT/debian=zroot/lxc/client-a/ROOT/debian \
  --map zroot/home/pi=zroot/lxc/client-a/ROOT/home/pi \
  --mode auto
```

### Real Workflow B: worked in build-host client-a replica, now pull to client-a

Temporary pull run from `client-a` (safe default blocks live mounted destinations):

```bash
sudo SHIFT_ACTION=pull \
SHIFT_DATASET_MAPS='zroot/lxc/client-a/ROOT/debian=zroot/ROOT/debian,zroot/lxc/client-a/ROOT/home/pi=zroot/home/pi' \
scripts/zfs/shift-sync.sh --mode auto --remote root@build-host.example
```

If you must allow a live receive (not recommended), set `SHIFT_ALLOW_LIVE_RECV=1` explicitly and only for a controlled run.

If this becomes your regular direction for some period, edit `/etc/default/ygg-shift-sync` and set:
- `SHIFT_ACTION=pull`
- `SHIFT_DATASET_MAPS` to the pull mapping above.

### Real Workflow C: prepare fresh machine and become client-a

From live environment on new laptop (after creating `zroot` pool):

```bash
ssh root@build-host.example 'zfs snapshot -r zroot/lxc/client-a/ROOT/debian@shift-bootstrap'
ssh root@build-host.example 'zfs snapshot -r zroot/lxc/client-a/ROOT/home/pi@shift-bootstrap'

ssh root@build-host.example 'zfs send -Rw zroot/lxc/client-a/ROOT/debian@shift-bootstrap' \
  | zfs recv -uF zroot/ROOT/debian
ssh root@build-host.example 'zfs send -Rw zroot/lxc/client-a/ROOT/home/pi@shift-bootstrap' \
  | zfs recv -uF zroot/home/pi
```

Then set bootfs and reboot:

```bash
zpool set bootfs=zroot/ROOT/debian zroot
```

### Real Workflow D: periodic pull timer on client-a

`/etc/default/ygg-shift-sync-pull`:

```bash
SHIFT_ACTION=pull
SHIFT_REMOTE=root@build-host.example
SHIFT_IDENTITY=client-a
SHIFT_DATASET_MAPS=zroot/lxc/{identity}/ROOT/debian=zroot/ROOT/debian,zroot/lxc/{identity}/ROOT/home/pi=zroot/home/pi
SHIFT_MODE=auto
SHIFT_SNAPSHOT_PREFIX=shift
SHIFT_SEND_FLAGS=-Rw
SHIFT_RECV_FLAGS=-uF
SHIFT_SSH_OPTS="-i /home/pi/.ssh/id_ed25519"
SHIFT_RECURSIVE=1
SHIFT_DRY_RUN=0
```

Activate:

```bash
sudo systemctl enable --now ygg-shift-sync-pull.timer
```

### Real Workflow E: prune old shift snapshots

`/etc/default/ygg-shift-prune` example:

```bash
SHIFT_PRUNE_KEEP_LOCAL=48
SHIFT_PRUNE_KEEP_REMOTE=48
SHIFT_PRUNE_LOCAL=1
SHIFT_PRUNE_REMOTE=1
SHIFT_DRY_RUN=0
```

Run once:

```bash
sudo systemctl start ygg-shift-prune.service
```

For conflict avoidance, choose one direction as authoritative per interval:
- Example: daytime work in LXC -> scheduled pull to laptop at end of day.
- Or physical laptop authoritative -> hourly push to `build-host`.

## Safety Checks Before Any Receive

Run every time:

```bash
zpool status
zfs list -o name,mountpoint,canmount,readonly
```

And confirm:
- destination dataset path is correct,
- destination rollback is acceptable (`recv -F`),
- you have a rollback snapshot of destination before receive.

## Boot Reliability (Encrypted Home + GUI Login)

If `/home` is encrypted and the display manager starts before ZFS keys are loaded/mounted, GUI login can fail.

Use:
- `templates/services/ygg-zfs-keys-and-mounts.service.template`

This oneshot service runs:
- `zfs load-key -a`
- `zfs mount -a`

and should be ordered before display manager startup.

## Practical Recommendations

- Keep snapshot names identical across peers when possible; incremental chaining becomes easier.
- Start with one dataset tree (`ROOT/debian`) until process is stable, then include home/user trees.
- Document each host's canonical dataset map in this repo so future restores are deterministic.
- Test a full restore on a spare disk before relying on "shifting heads" in production.
