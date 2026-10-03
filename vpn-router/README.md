# CakesCats VPN Router, do it yourself

[Русская версия](README.ru.md)

What you get: an **OpenWrt** router whose traffic (or the traffic of a separate private Wi-Fi network) goes through **your own server** using **VLESS + REALITY**. From the outside the connection looks like ordinary HTTPS to a large website.

Two parts:
1. [Server](#part-1-server) — a rented VPS with Xray (≈ 20 minutes).
2. [Router](#part-2-router) — OpenWrt with sing-box (≈ 30 minutes).

> Don't want to configure it yourself? A ready router with a dedicated server and support is at [vpn.cakescats.com](https://vpn.cakescats.com/). Buying it supports our non-profit project.

You'll need to connect over SSH and edit text files. Follow the laws of your country.

---

## Part 1. Server

### 1.1. VPS

The simplest VPS works: 1 vCPU, 1 GB RAM, **Ubuntu 24.04 LTS or newer**, a public IPv4 address. Pick a country and provider you're comfortable with. Connect: `ssh root@SERVER_IP`.

### 1.2. Install Xray

```bash
apt update && apt install -y curl openssl ufw
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

Generate keys and IDs. **Save the output** — you'll need it for the router:

```bash
xray uuid            # client UUID
xray x25519          # PrivateKey and the public key (called "Password" in newer versions)
openssl rand -hex 8  # ShortId
```

### 1.3. Configuration

Replace `UUID`, `PRIVATE_KEY` and `SHORT_ID` with your values:

```bash
cat > /usr/local/etc/xray/config.json <<'EOF'
{
  "log": { "loglevel": "warning", "access": "none" },
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{ "id": "UUID", "flow": "xtls-rprx-vision" }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "dest": "www.microsoft.com:443",
        "serverNames": ["www.microsoft.com"],
        "privateKey": "PRIVATE_KEY",
        "shortIds": ["SHORT_ID"]
      }
    },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
  }],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct" },
    { "protocol": "blackhole", "tag": "block" }
  ]
}
EOF
```

- `"access": "none"` — **no connection log is kept**.
- `dest`/`serverNames` — the site the connection imitates: a large TLS 1.3 site reachable in your country. In newer Xray versions `dest` is also called `target`; both work.

### 1.4. Start

```bash
xray run -test -c /usr/local/etc/xray/config.json   # validate the config
systemctl restart xray && systemctl enable xray
ufw allow 22/tcp && ufw allow 443/tcp && ufw --force enable
systemctl status xray --no-pager
```

The status should be `active (running)`.

## Part 2. Router

### 2.1. Which router

- Supported by official **OpenWrt** ([openwrt.org/toh](https://openwrt.org/toh)), at least **256 MB RAM** and **128 MB flash** — sing-box takes a few dozen megabytes.
- GL.iNet routers are a good choice: their firmware is OpenWrt-based.
- Install a current OpenWrt release following the guide for your model.

### 2.2. Install sing-box

Connect to the router: `ssh root@192.168.1.1`.

```bash
# OpenWrt 24.10:
opkg update && opkg install sing-box kmod-tun
# OpenWrt 25.x (new package manager):
apk update && apk add sing-box kmod-tun

sing-box version   # 1.12 or newer is required
```

### 2.3. Configuration

Fill in `SERVER_IP`, `UUID`, `PUBLIC_KEY` (Public key / Password from `xray x25519`) and `SHORT_ID`:

```bash
mkdir -p /etc/sing-box
cat > /etc/sing-box/config.json <<'EOF'
{
  "log": { "level": "warn" },
  "dns": {
    "servers": [
      { "type": "tls", "tag": "dns-remote", "server": "1.1.1.1", "detour": "proxy" }
    ],
    "final": "dns-remote"
  },
  "inbounds": [{
    "type": "tun",
    "tag": "tun-in",
    "interface_name": "singtun",
    "address": ["172.19.0.1/30"],
    "mtu": 1400,
    "auto_route": true,
    "auto_redirect": true,
    "strict_route": true
  }],
  "outbounds": [
    {
      "type": "vless",
      "tag": "proxy",
      "server": "SERVER_IP",
      "server_port": 443,
      "uuid": "UUID",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": { "enabled": true, "fingerprint": "chrome" },
        "reality": { "enabled": true, "public_key": "PUBLIC_KEY", "short_id": "SHORT_ID" }
      }
    },
    { "type": "direct", "tag": "direct" }
  ],
  "route": {
    "rules": [
      { "action": "sniff" },
      { "protocol": "dns", "action": "hijack-dns" },
      { "ip_is_private": true, "outbound": "direct" }
    ],
    "auto_detect_interface": true,
    "final": "proxy"
  }
}
EOF
sing-box check -c /etc/sing-box/config.json && echo "Config OK"
```

> The sing-box config format sometimes changes between versions. If `sing-box check` complains, see the [documentation](https://sing-box.sagernet.org/configuration/).

### 2.4. Start

```bash
uci set sing-box.main.enabled='1'
uci set sing-box.main.user='root'
uci commit sing-box
service sing-box enable
service sing-box restart
```

**Check:** on any device on the router's network open [ifconfig.me](https://ifconfig.me) — it should show your server's IP.

### 2.5. A separate private Wi-Fi network (optional)

If only a separate network should use the VPN and the main one should go direct:

1. LuCI → **Network → Interfaces → Add new interface**: name `private`, protocol *Static address*, address `192.168.9.1/24`. On the DHCP Server tab — **Set up DHCP Server**. On *Firewall Settings* — create zone `private`.
2. **Network → Wireless** → **Add** on the radio you want: SSID e.g. `Home-Private`, network `private`, encryption **WPA3-SAE** (or WPA2/WPA3 mixed).
3. **Network → Firewall** → zone `private`: Input *accept*, Output *accept*, Forward *reject*; allow forwarding to `wan`.
4. In `/etc/sing-box/config.json` replace the `"route"` block with:

```json
"route": {
  "rules": [
    { "action": "sniff" },
    { "protocol": "dns", "action": "hijack-dns" },
    { "ip_is_private": true, "outbound": "direct" },
    { "source_ip_cidr": ["192.168.9.0/24"], "outbound": "proxy" }
  ],
  "auto_detect_interface": true,
  "final": "direct"
}
```

5. `sing-box check -c /etc/sing-box/config.json && service sing-box restart`.

### 2.6. Test the kill switch

Stop sing-box (`service sing-box stop`) and check a device on the network in question: if it still has internet, traffic is going direct without the VPN. A reliable kill switch depends on the router model and firmware version — in our ready-made routers we configure and test it separately. Start it again: `service sing-box start`.

## Troubleshooting

| Problem | Check |
|---|---|
| `xray` won't start on the server | `xray run -test -c /usr/local/etc/xray/config.json` shows config errors |
| Router doesn't connect | UUID, public key, ShortId and `server_name` match the server; port 443 is open (`ufw status`) |
| Sites don't load but ping works | DNS — the router config needs the `hijack-dns` rule |
| Slow | Try another site in `dest`/`server_name`, or another VPS |

Questions — Telegram [@cakescats](https://t.me/cakescats) or [mail@cakescats.com](mailto:mail@cakescats.com).
