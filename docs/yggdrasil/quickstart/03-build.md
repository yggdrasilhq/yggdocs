# 03. Build Server + KDE ISOs

Build both profiles:

```bash
./mkconfig.sh --profile both --config ./config/includes.chroot/etc/yggdrasil/user-config.env
```

Expected outputs:

- `yggdrasil-<timestamp>-amd64.hybrid.iso`
- `yggdrasil-<timestamp>-kde-amd64.hybrid.iso`
- `artifacts/server-latest.iso`
- `artifacts/kde-latest.iso`
