# Email hosting (Stalwart)

The mail stack is hosted by the `stalwart` LXC on `primary-host`. Public mail traffic reaches the VPS (`relay-host.example`), then tunnels to the home HAProxy container, which forwards to Stalwart with PROXY v2.

For the higher-level pattern, see [Public Access via VPS: Mail-First Scaffold](recipes/public-access-via-vps-mail-first.md).

## Stalwart container
- Name: `stalwart`
- IP: `10.10.0.235`
- Version: `0.15.5`
- Install root: `/opt/stalwart`
- Config: `/opt/stalwart/etc/config.toml`
- Data: `/opt/stalwart/data`
- Logs: `/opt/stalwart/logs`
- TLS certs: `/opt/stalwart/etc/certs`

## Storage layout
- Stalwart uses local PostgreSQL for `data`, `blob`, `lookup`, `fts`, and internal directory state.
- PostgreSQL runs inside the same `stalwart` container on `127.0.0.1:5432`.
- `/opt/stalwart/data` remains a mounted ZFS dataset for Stalwart-managed files and migration safety, but the primary live mail store is PostgreSQL.

This is an intentional simplicity tradeoff: app and database live in one container, while the VPS remains the public edge.

## Listener ports
Stalwart listens on the standard mail ports:
- SMTP: `25`
- Submission: `587`
- SMTPS: `465`
- IMAP: `143`
- IMAPS: `993`
- POP3: `110`
- POP3S: `995`
- ManageSieve: `4190`
- HTTP: `8080`
- HTTPS: `443`

## Traffic flow
1. Public clients connect to `relay-host.example` on 465/587/993.
2. VPS HAProxy forwards to `10.10.0.108` with PROXY v2.
3. Home HAProxy forwards to `10.10.0.235` (Stalwart) with PROXY v2.
4. Public SMTP on port `25` is handled separately by Postfix on the VPS, which queues and relays mail to Stalwart.

## Outbound relay
Stalwart relays outbound mail via the VPS:
- `remote.vps_relay.address = "relay.example"`
- `remote.vps_relay.port = 25`

Use a hostname for the relay, not a raw IP. If necessary, pin that hostname in `/etc/hosts` inside the mail container.

## Operations
```bash
lxc-attach -n stalwart -- systemctl status stalwart
lxc-attach -n stalwart -- systemctl status postgresql@18-main
lxc-attach -n stalwart -- pg_lsclusters
lxc-attach -n stalwart -- tail -n 200 /opt/stalwart/logs/stalwart.log
```

Do not use `systemctl is-active postgresql` as a health check on Debian. That unit is only a meta-service and can report `active (exited)` while the real cluster is down. Check `postgresql@18-main` or `pg_lsclusters` instead.

## Notes
- PROXY protocol trusted networks are set in `config.toml` for IMAP/POP listeners.
- Keep HAProxy config on the VPS and home gateway in sync with Stalwart ports.
- Outbound relay should stay on a hostname, not a raw IP. If needed, pin it in `/etc/hosts` inside the container.
