# Traccar LXC deployment (primary-host.example)

Steps to recreate/maintain the `traccar` container running Traccar behind nginx.

## Container creation
- Base: copy from `docker` template (zfs snapshot): `lxc-copy -B zfs -s -n docker -N traccar`.
- AppArmor: set `lxc.apparmor.profile = unconfined` in `/zroot/lxc/traccar/config` (required for docker/runc).
- Autostart: uncomment/set `lxc.start.auto = 1`.
- Start container: `lxc-start -n traccar` and confirm IP (macvlan) with `lxc-ls -f` (e.g., 10.10.0.194).

## OS updates (inside container)
```bash
lxc-attach -n traccar -- apt-get update
lxc-attach -n traccar -- apt-get -y dist-upgrade
```

## Traccar runtime (docker)
- Ensure docker present (`docker --version`).
- Prepare persistent dirs:
```bash
lxc-attach -n traccar -- bash -c 'mkdir -p /opt/traccar/{logs,data,conf}'
```
- Seed default config (once): `lxc-attach -n traccar -- docker run --rm --entrypoint /bin/sh traccar/traccar:latest -c "cat /opt/traccar/conf/traccar.xml" > /opt/traccar/conf/traccar.xml`
- Run container:
```bash
lxc-attach -n traccar -- bash -c '
docker rm -f traccar 2>/dev/null || true
docker run -d --name traccar --restart=always \
  -p 8082:8082 \
  -p 5000-5150:5000-5150 \
  -v /opt/traccar/logs:/opt/traccar/logs \
  -v /opt/traccar/data:/opt/traccar/data \
  -v /opt/traccar/conf:/opt/traccar/conf \
  traccar/traccar:latest
'
```
- Check status/logs: `docker ps`, `docker logs traccar`.

## nginx (http/https proxy for UI)
- Site file: `/etc/nginx/sites-enabled/tracker.example.conf` inside the `nginx` container:
  - upstream `traccar` -> `10.10.0.194:8082`
  - redirects 80→443, uses your domain certs, includes `snippets/my-proxy-settings.conf` and `my-ssl-settings.conf`.
- Test/reload in `nginx` container: `nginx -t && nginx -s reload`.

## nginx stream proxy for device port 5055
- Ensure stream module installed: `apt-get install -y libnginx-mod-stream` (nginx container).
- Enable stream block in `/etc/nginx/nginx.conf`:
```
stream {
    include /etc/nginx/streams-enabled/*.conf;
}
```
- Stream config `/etc/nginx/streams-enabled/traccar-5055.conf`:
```
upstream traccar5055 {
    server 10.10.0.194:5055;
}

server {
    listen 5055;
    proxy_connect_timeout 30s;
    proxy_timeout 300s;
    proxy_pass traccar5055;
}
```
- Reload nginx: `nginx -t && nginx -s reload`.
- Device URL for LAN/Internet: `http://tracker.example:5055` (protocol is plain HTTP; 5055 is forwarded by stream proxy).

## Quick checks
- UI: `curl -I https://tracker.example` (expect 200 via nginx).
- Device port: `curl -v http://tracker.example:5055/?id=test123` (expect 400 from Traccar handler).
