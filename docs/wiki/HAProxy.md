# HAProxy (home mail gateway)

The `haproxy` LXC on `primary-host` terminates the VPS mail tunnels and forwards them to the Stalwart mail server. HAProxy runs in TCP mode with PROXY protocol enabled to preserve client IPs.

## Container
- Name: `haproxy`
- IP: `10.10.0.108`
- Config: `/etc/haproxy/haproxy.cfg`

## Frontends
- `*:993` (IMAPS) with `accept-proxy`
- `*:465` (SMTPS) with `accept-proxy`
- `*:587` (Submission) with `accept-proxy`

## Backends
- IMAPS -> `10.10.0.235:993`
- SMTPS -> `10.10.0.235:465`
- Submission -> `10.10.0.235:587`
- Each backend uses `send-proxy-v2`.

## Stalwart integration
Stalwart must trust HAProxy as a proxy source. Ensure the following are present in `/opt/stalwart/etc/config.toml`:

```
server.listener.imap.proxy.trusted-networks = ["10.10.0.107/32", "10.10.0.108/32", "10.0.0.1/32"]
server.listener.imaptls.proxy.trusted-networks = ["10.10.0.107/32", "10.10.0.108/32", "10.0.0.1/32"]
server.listener.pop3.proxy.trusted-networks = ["10.10.0.107/32", "10.10.0.108/32", "10.0.0.1/32"]
server.listener.pop3s.proxy.trusted-networks = ["10.10.0.107/32", "10.10.0.108/32", "10.0.0.1/32"]
```

## Operations
```bash
lxc-attach -n haproxy -- systemctl status haproxy
lxc-attach -n haproxy -- systemctl reload haproxy
```

