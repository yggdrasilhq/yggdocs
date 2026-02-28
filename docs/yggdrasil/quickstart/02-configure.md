# 02. Configure Build Inputs

Run the guided TUI:

```bash
./scripts/mkconfig-tui.sh
```

This produces a user config env file with:

- SSH key embedding behavior.
- Host/network defaults.
- Macvlan/LXC networking values.
- Optional APT proxy settings.

Use `recommended` mode for production-bound builds. Use `quick-try` only for exploratory test images.
