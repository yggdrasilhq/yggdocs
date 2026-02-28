# Yggdrasil docs server (MkDocs)

This container hosts the MkDocs manual for Yggdrasil at `https://docs.example`.

## Container
- Name: `yggdrasil` (cloned from the `deb` base).
- Rootfs: `zroot/lxc/yggdrasil`.
- Autostart: enabled (`lxc.start.auto = 1`).
- Service port: `8000` (MkDocs dev server).

### Create the container
```bash
lxc-copy -B zfs -s -n deb -N yggdrasil
# edit /zroot/lxc/yggdrasil/config
#   lxc.start.auto = 1
#   remove INFISICAL_* env lines
lxc-start -n yggdrasil
```

## MkDocs service
Inside the container:

```bash
apt-get update
apt-get install -y git python3 python3-venv

mkdir -p /opt
cd /opt
git clone https://<your-git-remote>/yggdrasil.git
cd /opt/yggdrasil
python3 -m venv .venv-docs
./.venv-docs/bin/python -m pip install -r docs/requirements.txt
```

Systemd units:
- `/etc/systemd/system/yggdrasil-docs.service` runs MkDocs on `0.0.0.0:8000`.
- `/usr/local/sbin/yggdrasil-docs-update` pulls git changes and restarts the service.
- `/etc/systemd/system/yggdrasil-docs-update.timer` pulls every 5 minutes.

```bash
systemctl daemon-reload
systemctl enable --now yggdrasil-docs.service
systemctl enable --now yggdrasil-docs-update.timer
```

## Nginx frontend
The nginx LXC proxies to the container:
- Config: `/etc/nginx/sites-enabled/docs.example.conf`
- Upstream: `10.10.0.x:8000` (container IP)

Reload nginx after changes:
```bash
nginx -t && nginx -s reload
```

## Notes
- The update timer assumes `origin/main` is reachable. Ensure `/root/.git-credentials` is present inside the container.
- MkDocs is running in server mode (not a static build) to keep updates fast.

