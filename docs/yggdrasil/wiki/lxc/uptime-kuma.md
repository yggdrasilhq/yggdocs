# Uptime Kuma (status.example)

This container runs Uptime Kuma for service health dashboards. The UI is served at `https://status.example` via the nginx LXC.

## Container
- Name: `uptime_kuma`
- Docker container: `uptime-kuma` (image `louislam/uptime-kuma:1`)
- Data volume: `/var/lib/docker/volumes/uptime-kuma/_data`

## Nginx frontend
- Config: `/etc/nginx/sites-enabled/status.example.conf` (in the `nginx` LXC)
- Upstream: `10.10.0.153:3001` (Uptime Kuma container)

Reload nginx after edits:
```bash
lxc-attach -n nginx -- nginx -t
lxc-attach -n nginx -- nginx -s reload
```

## Populate monitors from primary-host
Monitors were seeded directly into the SQLite DB so they appear for the primary user (`dada`).

```bash
# count monitors
lxc-attach -n uptime_kuma -- sqlite3 /var/lib/docker/volumes/uptime-kuma/_data/kuma.db \
  "SELECT COUNT(*) FROM monitor;"

# assign orphaned monitors to the primary user (id=1)
lxc-attach -n uptime_kuma -- sqlite3 /var/lib/docker/volumes/uptime-kuma/_data/kuma.db \
  "UPDATE monitor SET user_id=1 WHERE user_id IS NULL;"

# restart the app container
lxc-attach -n uptime_kuma -- docker restart uptime-kuma
```

## Add more monitors later
- Use the web UI to add/edit monitors.
- If you insert via SQL, ensure `user_id` is set so the UI can see them.

## Quick checks
```bash
# container status
lxc-attach -n uptime_kuma -- docker ps | rg -n uptime-kuma

# DB location
lxc-attach -n uptime_kuma -- ls -la /var/lib/docker/volumes/uptime-kuma/_data
```

