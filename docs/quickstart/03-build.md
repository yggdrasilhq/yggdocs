# 03. Build Server + KDE ISOs

Build both profiles:

```bash
cd ~/gh/yggdrasil
./mkconfig.sh --config ./ygg.local.toml --profile both
```

Expected outputs:

- `yggdrasil-<timestamp>-amd64.hybrid.iso`
- `yggdrasil-<timestamp>-kde-amd64.hybrid.iso`
- `artifacts/server-latest.iso`
- `artifacts/kde-latest.iso`

If you only want the conservative host image:

```bash
./mkconfig.sh --config ./ygg.local.toml --profile server
```

If you are building the KDE image, treat it as a separate artifact with its own release gate.
That is one of the hard lessons behind this public refactor: “mostly works” is not enough when desktop regressions only show up after boot.
