# LXC Fleet Inventory Template

This page is intentionally generic. Use it as the operational template for your own fleet inventory.

## Purpose

Track each container's role, autostart policy, networking model, storage root, and operational notes in one place.
This becomes the single source of truth for rebuild and incident response.

## Recommended Table

| Container | Role | State | Autostart | Network | Rootfs | Runtime | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `docker` | Docker base/template | RUNNING/STOPPED | 0/1 | macvlan/bridge | `zfs:<pool>/lxc/docker` | docker | apparmor profile, mounts |
| `nginx` | Reverse proxy + TLS | RUNNING/STOPPED | 1 | macvlan | `zfs:<pool>/lxc/nginx` | native | cert paths, upstream map |
| `smb0` | Samba node | RUNNING/STOPPED | 1 | macvlan | `zfs:<pool>/lxc/smb0` | native | mounted datasets |
| `vaultwarden` | Secrets service | RUNNING/STOPPED | 1 | macvlan | `zfs:<pool>/lxc/vaultwarden` | docker | backup + restore notes |
| `uptime` | Monitoring UI | RUNNING/STOPPED | 1 | macvlan | `zfs:<pool>/lxc/uptime` | docker | alert route mapping |

## Per-Container Detail Block

Use this block for each production container:

```markdown
### <container-name>

- Role: <one-line purpose>
- State: RUNNING/STOPPED
- Autostart: 0/1
- IP/Network: <CIDR or DHCP + interface>
- Rootfs: zfs:<pool>/lxc/<container-name>
- Runtime: native/docker/k8s
- AppArmor: generated/unconfined/custom
- Mounts:
  - <host path> -> <container path>
- Secrets/Env: <where managed>
- Recovery:
  - <restore command or runbook link>
```

## Minimum Operational Guarantees

- Keep this page updated on every topology change.
- Record all production mount points.
- Record autostart ordering dependencies.
- Link every service to its dedicated runbook.

