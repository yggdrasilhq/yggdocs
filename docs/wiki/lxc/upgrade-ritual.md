# LXC fleet upgrade ritual

Documenting the recurring steps to update and clean the LXC fleet on `primary-host.example`, the Arch guests, and the `relay-host.example` host.

## Relay Host (Debian host)
- Upgrade and clean:
  ```bash
  ssh root@relay-host.example '
    export DEBIAN_FRONTEND=noninteractive
    apt update && apt full-upgrade -y
    apt autoremove --purge -y
    apt autoclean && apt clean
  '
  ```
- Reboot to pick up the new kernel:
  ```bash
  ssh root@relay-host.example 'reboot'
  ```

## Primary Host (Debian live host)
- Do **not** apt-upgrade the live host.
- List LXCs to see what is running:
  ```bash
  ssh root@primary-host.example 'lxc-ls -f'
  ```

### Active Debian-based LXCs on primary-host
- Upgrade and clean every running container (all current ones are apt-based):
  ```bash
  ssh root@primary-host.example '
    for c in $(lxc-ls --active); do
      echo "=== $c ==="
      lxc-attach -n "$c" -- bash -lc "
        export DEBIAN_FRONTEND=noninteractive
        apt-get update &&
        apt-get -y full-upgrade &&
        apt-get autoremove --purge -y &&
        apt-get autoclean &&
        apt-get clean
      "
    done
  '
  ```
- Verify nothing is left running:
  ```bash
  ssh root@primary-host.example "pgrep -fa 'apt(-get)? full-upgrade'"
  ```

### Arch-based LXCs on primary-host (`arch`, `zig`)
- Start, update with pacman, and stop each:
  ```bash
  ssh root@primary-host.example '
    for c in arch zig; do
      lxc-start -n "$c"
      lxc-attach -n "$c" -- bash -lc "pacman -Syyu --noconfirm"
      lxc-stop -n "$c"
    done
  '
  ```
- If mirrors fail to resolve, re-run the pacman command inside the container once DNS/mirror reachability is back.
- Optional: remove pacman orphans inside Arch guests:
  ```bash
  lxc-attach -n arch -- pacman -Qdt
  lxc-attach -n arch -- pacman -Rns --noconfirm <orphans>
  lxc-attach -n zig -- pacman -Qdt
  lxc-attach -n zig -- pacman -Rns --noconfirm <orphans>
  ```

## Post-checks
- Spot-check key services in heavier LXCs after upgrades (e.g., `infisical`, `immich`, `peertube*`, `vaultwarden`, `haproxy`).
- Ensure `apt-cacher-ng` in the `apt` LXC is healthy if other LXCs use it (service: `apt-cacher-ng`).
- Reboot only `relay host` unless a container-specific restart is required.
