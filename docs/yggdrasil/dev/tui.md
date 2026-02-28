# mkconfig TUI (Developer Notes)

`scripts/mkconfig-tui.sh` now has two UX layers:

- `simple`: constrained, low-cognitive-load prompts.
- `advanced`: full dial visibility for operators.

The file emitted by the TUI is consumed by both:

- `mkconfig.sh --config <envfile>`
- `scripts/build-profile.sh --config <envfile>`

Important emitted vars:

- `YGG_BUILD_PROFILE`
- `YGG_ENABLE_QEMU_SMOKE`
- `YGG_SETUP_MODE`
- `YGG_*` networking fields
- `YGG_APT_*` proxy fields
