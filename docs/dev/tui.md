# yggcli (Developer Notes)

The shell TUI experiment is over.
`yggcli` is now the intended operator-facing terminal UX for the ecosystem.

## Why it exists

The old private stack worked, but it was too easy for configuration to become “whatever Avikalpa remembers this week.”
That is not a public product surface.

`yggcli` fixes this by doing three things:

1. guiding the operator through the main decisions,
2. writing the native config files that each repo already understands,
3. leaving those files human-editable after generation.

## Current output

`yggcli` writes:

- `yggdrasil/ygg.local.toml`
- `yggclient/yggclient.local.toml`
- `yggclient/config/profiles.local.env`
- `yggsync/ygg_sync.local.toml`

## Design rule

The TUI is the front end, not the prison.

If an operator wants to bypass the TUI and edit TOML directly, the system should still make sense.
That is how you keep trust with advanced users while still giving newcomers a good first run.
