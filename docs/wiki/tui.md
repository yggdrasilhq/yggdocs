# mkconfig TUI

Run:

```bash
./scripts/mkconfig-tui.sh
```

The TUI is designed in two layers:

- `simple`: modern minimal flow for most users.
- `advanced`: exposes additional build dials.

Security modes:

- `recommended`: SSH keys + secure defaults.
- `quick-try`: evaluation mode with explicit reminders.

Saved output is an env file consumed by `mkconfig.sh --config`.

Generated dials include:

- Build profile selection (`server`, `kde`, `both`).
- Optional QEMU smoke gate (`YGG_ENABLE_QEMU_SMOKE`).
- SSH key embedding behavior.
- Hostname override.
- LXC parent interface + macvlan CIDR/route.
- DHCP or static networking (IP/gateway/DNS).
- Optional APT proxy values (advanced mode).
