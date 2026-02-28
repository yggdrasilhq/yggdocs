# Email hosting (Stalwart)

The mail stack is hosted by the `stalwart` LXC on `primary-host`. Public mail traffic reaches the VPS (`relay-host.example`), then tunnels to the home HAProxy container, which forwards to Stalwart with PROXY v2.

## Stalwart container
- Name: `stalwart`
- IP: `10.10.0.235`
- Install root: `/opt/stalwart`
- Config: `/opt/stalwart/etc/config.toml`
- Data: `/opt/stalwart/data`
- Logs: `/opt/stalwart/logs`
- TLS certs: `/opt/stalwart/etc/certs`

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

## Outbound relay
Stalwart relays outbound mail via the VPS:
- `remote.vps_relay.address = "relay.example"`
- `remote.vps_relay.port = 25`

## Operations
```bash
lxc-attach -n stalwart -- systemctl status stalwart
lxc-attach -n stalwart -- tail -n 200 /opt/stalwart/logs/stalwart.log
```

## Notes
- PROXY protocol trusted networks are set in `config.toml` for IMAP/POP listeners.
- Keep HAProxy config on the VPS and home gateway in sync with Stalwart ports.

