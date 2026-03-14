# 02. Configure Build Inputs

This is where the ecosystem split starts to pay off.

You do not hand-edit a mystery blob.
You use `yggcli` to write the native config files that each repo already understands.

```bash
cd ~/gh/yggcli
cargo run
```

`yggcli` currently writes:

- `~/gh/yggdrasil/ygg.local.toml`
- `~/gh/yggclient/yggclient.local.toml`
- `~/gh/yggclient/config/profiles.local.env`
- `~/gh/yggsync/ygg_sync.local.toml`

For the first server build, focus on:

- hostname
- SSH key embedding
- network mode (`dhcp` vs static)
- macvlan / LXC parent interface
- optional APT proxy

Operator note:
keep the public examples generic and let your private infrastructure live only in the local files.
That is how Yggdrasil stays reusable without becoming toothless.
