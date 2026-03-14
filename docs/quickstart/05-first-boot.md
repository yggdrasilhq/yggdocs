# 05. Boot and Validate

After writing the ISO to Ventoy and booting, do not rely on vibes.
Walk the machine.

```bash
systemctl status ygg-import-zpool-at-boot
systemctl status ygg-lxc-autostart
systemctl status ygg-infisical-ensure
cat /etc/lxc/lxc.conf
cat /etc/lxc/default.conf
```

On the live reference host, the invariant looks like this:

- `/etc/lxc/lxc.conf` points at `/zroot/lxc`
- `/etc/lxc/default.conf` carries the `macvlan` template on `eno1`
- `ygg-import-zpool-at-boot` runs before `ygg-lxc-autostart`
- `ygg-infisical-ensure` runs after container autostart, not before

For KDE, add these checks:

- desktop login works
- Wi-Fi radios are visible
- `zpool status` and `zfs list` work from the installed userland
- any laptop-specific firmware expectations are verified on the actual hardware

Use `wiki/smoke-tests.md` for the complete post-boot checklist.
