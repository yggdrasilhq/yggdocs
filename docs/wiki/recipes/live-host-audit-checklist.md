# Recipe: Live Host Audit Checklist

Use this checklist whenever you need to understand "what is true right now" on a running Yggdrasil host.

## Goal

Produce a fast, high-signal snapshot of:

- container fleet state
- ZFS topology
- boot orchestration services
- LXC networking defaults

## Runbook

```bash
# Host identity and clock
hostname -f
date -Iseconds

# LXC fleet overview
lxc-ls -f

# ZFS pool and datasets
zpool list
zfs list -o name,mountpoint -d 2

# Service chain status
systemctl status ygg-import-zpool-at-boot --no-pager -n 20
systemctl status ygg-lxc-autostart --no-pager -n 20
systemctl status ygg-infisical-ensure --no-pager -n 20

# LXC path and defaults
cat /etc/lxc/lxc.conf
cat /etc/lxc/default.conf
```

## What to verify

1. `ygg-import-zpool-at-boot` is enabled and successful.
2. `ygg-lxc-autostart` started after pool import.
3. `ygg-infisical-ensure` completed successfully at boot.
4. `/etc/lxc/lxc.conf` points to the intended ZFS-backed path.
5. `/etc/lxc/default.conf` matches expected macvlan/apparmor defaults.

## Failure cues

- Pool import failed or dataset mountpoints drifted.
- Autostart inactive after boot.
- Dependent services healthy in isolation but broken in sequence.

## Documentation habit

After each audit, write one short timestamped note:

- what changed,
- what was unexpected,
- what you fixed,
- what to watch next.

That single habit compounds faster than any dashboard.
