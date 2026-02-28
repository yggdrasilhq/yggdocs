# Apt caching LXC (apt-proxy.local)

This container runs `apt-cacher-ng` and is fronted by nginx at `apt-proxy.local`. DNS for `apt-proxy.local` should resolve to `10.10.0.131` (the cache host).

## Service overview
- Package: `apt-cacher-ng`.
- Service: `apt-cacher-ng.service`.
- Listen: `0.0.0.0:3142`.
- Config: `/etc/apt-cacher-ng/acng.conf` and `/etc/apt-cacher-ng/backends_debian`.
- Cache/logs: `/var/cache/apt-cacher-ng`, `/var/log/apt-cacher-ng`.
- Report page: `http://127.0.0.1:3142/acng-report.html`.

## Nginx frontend (nginx LXC)
- Upstream: `server 10.10.0.131:3142;` (keepalive 4).
- 80 -> 443 redirect.
- HTTPS vhost `apt-proxy.local` proxies `/` to the upstream.
- Clients can use `https://apt-proxy.local/debian` in sources, or proxy via `http://apt-proxy.local:3142`.

## Client configuration
Use the global apt proxy (preferred) or point sources at the mirror path.

### Proxy mode
Drop `/etc/apt/apt.conf.d/02proxy`:

```
Acquire::http::Proxy "http://apt-proxy.local:3142";
Acquire::https::Proxy "http://apt-proxy.local:3142";
# avoid proxy loops on the cache host itself
Acquire::http::Proxy::apt-proxy.local "DIRECT";
Acquire::https::Proxy::apt-proxy.local "DIRECT";
```

Optional tuning (`/etc/apt/apt.conf.d/03cache-tuning`):

```
Acquire::Retries "5";
Acquire::http::Pipeline-Depth "0";
```

### Mirror mode
Keep apt proxy unset and point sources to:

```
deb https://apt-proxy.local/debian sid main
```

## Validation
- Socket: `ss -lnpt '( sport = 3142 )'`.
- Report page: `curl -s http://127.0.0.1:3142/acng-report.html | head`.
- Cache hits: inspect `/var/log/apt-cacher-ng/*` after `apt-get update` from a client.

## Maintenance
- Restart after config changes: `systemctl restart apt-cacher-ng`.
- If exposing beyond LAN, restrict via firewall or nginx allowlists (apt-cacher-ng has no `AllowedHosts`).

