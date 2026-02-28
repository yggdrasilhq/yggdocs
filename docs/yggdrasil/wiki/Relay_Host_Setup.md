# relay-host.example (VPS) setup

`relay-host.example` is the public VPS that fronts mail services for the home stack. It runs HAProxy in TCP mode and forwards IMAPS/SMTPS/submission to the home HAProxy container on `primary-host`.

## Role
- Public entrypoint for mail protocols.
- Forwards traffic to `primary-host` via HAProxy with PROXY v2.

## HAProxy configuration
- File: `/etc/haproxy/haproxy.cfg`
- Frontends:
  - `*:993` (IMAPS)
  - `*:465` (SMTPS)
  - `*:587` (Submission)
- Backends forward to `10.10.0.108` (home HAProxy) with `send-proxy-v2`.

## Operations
```bash
systemctl status haproxy
systemctl reload haproxy
```

## Notes
- Keep the port map aligned with the home HAProxy container (`primary-host` LXC `haproxy`).
- If the home HAProxy IP changes, update the backend targets here.

