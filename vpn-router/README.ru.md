# VPN-роутер CakesCats своими руками

[English version](README.md)

Что получится: роутер на **OpenWrt**, весь трафик которого (или трафик отдельной приватной Wi-Fi-сети) идёт через **ваш собственный сервер** по протоколу **VLESS + REALITY**. Со стороны такое соединение выглядит как обычный HTTPS к крупному сайту.

Две части:
1. [Сервер](#часть-1-сервер) — арендованный VPS с Xray (≈ 20 минут).
2. [Роутер](#часть-2-роутер) — OpenWrt с sing-box (≈ 30 минут).

> Не хотите настраивать? Готовый роутер с выделенным сервером и поддержкой — на [vpn.cakescats.com](https://vpn.cakescats.com/ru/). Покупка поддерживает наш некоммерческий проект.

Нужно уметь подключаться по SSH и редактировать текстовые файлы. Соблюдайте законы своей страны.

---

## Часть 1. Сервер

### 1.1. VPS

Подойдёт самый простой VPS: 1 vCPU, 1 ГБ RAM, **Ubuntu 24.04 LTS или новее**, публичный IPv4. Выберите страну и провайдера, которые вас устраивают. Подключитесь: `ssh root@IP_СЕРВЕРА`.

### 1.2. Установка Xray

```bash
apt update && apt install -y curl openssl ufw
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

Сгенерируйте ключи и идентификаторы. **Сохраните вывод** — он понадобится для роутера:

```bash
xray uuid          # UUID клиента
xray x25519        # PrivateKey и публичный ключ (в новых версиях он называется Password)
openssl rand -hex 8  # ShortId
```

### 1.3. Конфигурация

Замените `UUID`, `PRIVATE_KEY` и `SHORT_ID` своими значениями:

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

- `"access": "none"` — **журнал подключений не ведётся**.
- `dest`/`serverNames` — сайт, под который маскируется соединение. Нужен крупный сайт с TLS 1.3, доступный в вашей стране. В новых версиях Xray параметр `dest` также называется `target`, оба варианта работают.

### 1.4. Запуск

```bash
xray run -test -c /usr/local/etc/xray/config.json   # проверка конфигурации
systemctl restart xray && systemctl enable xray
ufw allow 22/tcp && ufw allow 443/tcp && ufw --force enable
systemctl status xray --no-pager
```

Статус должен быть `active (running)`.

## Часть 2. Роутер

### 2.1. Какой роутер

- Поддерживается официальным **OpenWrt** ([openwrt.org/toh](https://openwrt.org/toh)), минимум **256 МБ RAM** и **128 МБ флеш-памяти**: sing-box занимает несколько десятков мегабайт.
- Хороший вариант — роутеры GL.iNet: у них прошивка на базе OpenWrt.
- Установите свежую прошивку OpenWrt по инструкции для вашей модели.

### 2.2. Установка sing-box

Подключитесь к роутеру: `ssh root@192.168.1.1`.

```bash
# OpenWrt 24.10:
opkg update && opkg install sing-box kmod-tun
# OpenWrt 25.x (новый менеджер пакетов):
apk update && apk add sing-box kmod-tun

sing-box version   # нужна версия 1.12 или новее
```

### 2.3. Конфигурация

Подставьте `SERVER_IP`, `UUID`, `PUBLIC_KEY` (Public key или Password из `xray x25519`) и `SHORT_ID`:

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
sing-box check -c /etc/sing-box/config.json && echo "Конфигурация в порядке"
```

> Формат конфигурации sing-box иногда меняется между версиями. Если `sing-box check` ругается, сверьтесь с [документацией](https://sing-box.sagernet.org/configuration/).

### 2.4. Запуск

```bash
uci set sing-box.main.enabled='1'
uci set sing-box.main.user='root'
uci commit sing-box
service sing-box enable
service sing-box restart
```

**Проверка:** на любом устройстве в сети роутера откройте [ifconfig.me](https://ifconfig.me) — там должен быть IP вашего сервера.

### 2.5. Отдельная приватная Wi-Fi-сеть (по желанию)

Если через VPN должна идти только отдельная сеть, а основная — напрямую:

1. LuCI → **Network → Interfaces → Add new interface**: имя `private`, протокол *Static address*, адрес `192.168.9.1/24`. На вкладке DHCP Server — **Set up DHCP Server**. На вкладке *Firewall Settings* — создайте зону `private`.
2. **Network → Wireless** → **Add** на нужном радио: имя сети, например `Home-Private`, сеть `private`, шифрование **WPA3-SAE** (или WPA2/WPA3 Mixed).
3. **Network → Firewall** → зона `private`: Input *accept*, Output *accept*, Forward *reject*; разрешите пересылку в `wan`.
4. В `/etc/sing-box/config.json` замените блок `"route"` на:

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

### 2.6. Проверка «аварийного выключателя»

Остановите sing-box (`service sing-box stop`) и проверьте устройство в нужной сети: если интернет всё ещё есть, трафик идёт напрямую без VPN. Надёжный kill switch зависит от модели роутера и версии прошивки. В готовых роутерах мы настраиваем и проверяем его отдельно. Включите обратно: `service sing-box start`.

## Если что-то не работает

| Проблема | Что проверить |
|---|---|
| На сервере `xray` не запускается | `xray run -test -c /usr/local/etc/xray/config.json` покажет ошибку в конфигурации |
| Роутер не подключается | Совпадают ли UUID, публичный ключ, ShortId и `server_name` с сервером; открыт ли порт 443 (`ufw status`) |
| Сайты не открываются, но ping идёт | DNS: в конфигурации роутера должно быть правило `hijack-dns` |
| Медленно | Попробуйте другой сайт в `dest`/`server_name` или другой VPS |

Вопросы — в Telegram [@cakescats](https://t.me/cakescats) или на [mail@cakescats.com](mailto:mail@cakescats.com).
