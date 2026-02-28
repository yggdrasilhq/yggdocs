# OpenWrt access (tailscale + passwordless SSH)

Notes on how this OpenWrt box is set up for key-only SSH and Tailscale access, and the commands to repeat it if needed.

## Passwordless SSH (Dropbear)
- Use the workstation key at `~/.ssh/id_ed25519.pub` (or whichever public key you prefer).
- Copy the key into Dropbear’s path, not `/root/.ssh`:
  ```sh
  ssh root@10.20.0.1 "printf '%s\n' 'ssh-ed25519 <your-public-key> <comment>' > /etc/dropbear/authorized_keys && chmod 600 /etc/dropbear/authorized_keys"
  ```
  (Replace the key string if you’re using a different key.)
- Test locally: `ssh root@10.20.0.1`.
- Test over Tailscale once it’s up: `ssh root@100.71.117.29`.

## Tailscale setup on OpenWrt
- Install: `opkg update && opkg install tailscale tailscaled`.
- Bring it up and log in (once): `tailscale up` (follow the auth URL).
- Ensure the Tailscale interface exists:
  - `/etc/config/network` has `config interface 'tailscale'` with `proto 'none'` and `device 'tailscale0'` (created by the package).

### Firewall (critical rules)
- Allow inbound from Tailscale: `uci set firewall.ts_zone.input='ACCEPT'`.
- Allow Tailscale to LAN (already present by default): `firewall.ts_lan_forwarding.src='tailscale'` / `dest='lan'`.
- Allow Tailscale to WAN (needed for exit-node or general Internet egress):
  ```sh
  uci add firewall forwarding >/dev/null
  uci set firewall.@forwarding[-1].src='tailscale'
  uci set firewall.@forwarding[-1].dest='wan'
  uci commit firewall
  /etc/init.d/firewall restart
  ```
- Reverse path filter is already permissive: `net.ipv4.conf.all.rp_filter=0`, `net.ipv4.conf.tailscale0.rp_filter=0`.

### Exit node and subnet routes
- Exit node usage (recommended for Android when cellular links are finicky):
  - On the phone, select `openwrt` as exit node and toggle “Allow LAN access”.
  - Keep Tailscale DNS to “Automatic” and Android Private DNS to Off/Automatic.
- Subnet route advertising (optional; previously caused Wi‑Fi issues here):
  - To advertise: `tailscale set --advertise-routes=10.10.0.0/24`.
  - Approve the route in the admin console, then enable “Use Tailscale subnets” on the client.
  - If LAN breaks, remove/disable the advertised route and rely on the exit node instead.

### Quick diagnostics
- Status: `tailscale status` (or `tailscale status --json`).
- Tailnet reachability: `tailscale ping <peer>`.
- Router’s Tailscale IPs: `tailscale ip -4`, `tailscale ip -6`.
- Firewall sanity: `nft list ruleset | grep tailscale` (input accept + forwarding to lan/wan should be present).

## KDE Connect over Tailscale
- Discovery/broadcast doesn’t cross Tailscale; use direct IP.
- Host firewall (host-a): tailscale0 is in firewalld `trusted` zone with TCP/UDP 1714-1764 open. If recreating:
  ```sh
  sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0
  sudo firewall-cmd --permanent --zone=trusted --add-port=1714-1764/tcp
  sudo firewall-cmd --permanent --zone=trusted --add-port=1714-1764/udp
  sudo firewall-cmd --reload
  ```
- Pairing/connecting from Android when using OpenWrt as exit node:
  - Make sure Tailscale is connected, exit node = openwrt, “Allow LAN access” on.
  - In KDE Connect on the phone, add the PC by tailnet IP (e.g., `100.86.127.89`). Discovery alone won’t find it over the tunnel.
