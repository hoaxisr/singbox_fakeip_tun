# singbox_fakeip_tun

Инструкция по установке и настройке **Sing-box** на Debian/Ubuntu с использованием его в качестве шлюза (gateway) для клиентов локальной сети.


ВАЖНО!: NB! Предполгается, что у устройства на котором будет развернут sing-box задан статический IP и настройки DNS не получаются по DHCP. 
---

## 1. Установка Sing-box

```bash
sudo nano /etc/sysctl.conf
# Уберите символ # перед следующей строкой:
net.ipv4.ip_forward=1

sudo sysctl -p
sudo apt update
sudo apt install curl
curl -fsSL https://sing-box.app/install.sh | sh
```

---

## 2. Создание конфигурации

Откройте файл конфигурации:

```bash
sudo nano /etc/sing-box/config.json
```

Используйте следующий пример (актуально для версии **1.12.8**).  
⚠️ Перед использованием замените необходимые данные для VLESS-подключений или удалите лишние секции.  
Если у вас более старая версия Sing-box, прочитайте [migration guide](https://sing-box.sagernet.org/migration/).

<details>
<summary>📂 Пример config.json</summary>

```jsonc
{
  "log": { "level": "info" },
  "dns": {
    "servers": [
      {
        "tag": "google",
        "type": "tls",
        "server": "8.8.8.8"
      },
      {
        "tag": "local-server",
        "type": "local"
      },
      {
        "tag": "fakeip-server",
        "type": "fakeip",
        "inet4_range": "198.18.0.0/15"
      }
    ],
    "rules": [
      { "query_type": ["A"], "server": "fakeip-server" }
    ],
    "final": "local-server",
    "independent_cache": true,
    "disable_cache": false,
    "disable_expire": false
  },
  "inbounds": [
    {
      "type": "mixed",
      "listen": "::",
      "listen_port": 1080,
      "tag": "socks5",
      "tcp_fast_open": true
    },
    {
      "tag": "dns-in",
      "type": "direct",
      "listen": "127.0.0.1",
      "listen_port": 5353
    },
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": ["172.18.0.1/30"],
      "auto_route": true,
      "auto_redirect": true,
      "strict_route": false,
      "stack": "system",
      "sniff": true
    }
  ],
  "outbounds": [
    {
      "type": "selector",
      "tag": "select",
      "outbounds": ["auto", "vless1", "vless2", "vless3"],
      "default": "vless1",
      "interrupt_exist_connections": false
    },
    {
      "type": "urltest",
      "tag": "auto",
      "outbounds": ["vless1", "vless2", "vless3"],
      "url": "https://www.gstatic.com/generate_204",
      "interval": "5m",
      "tolerance": 30,
      "idle_timeout": "30m",
      "interrupt_exist_connections": false
    },
    {
      "type": "vless",
      "tag": "vless1",
      "server": "IP_server_VLESS",
      "server_port": 443,
      "uuid": "your_uuid",
      "flow": "xtls-rprx-vision",
      "packet_encoding": "xudp",
      "tls": {
        "enabled": true,
        "server_name": "your_masking_SNI",
        "utls": { "enabled": true, "fingerprint": "your_fingerprint" },
        "reality": {
          "enabled": true,
          "public_key": "your_pbk",
          "short_id": "your_short_id"
        }
      }
    }
    // аналогично vless2 и vless3
  ],
  "route": {
    "rule_set": [
      {
        "tag": "mydomains",
        "type": "local",
        "format": "source",
        "path": "/etc/sing-box/mydomains.json"
      },
      {
        "tag": "geoip-ru",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-ru.srs",
        "download_detour": "auto"
      }
      // дополнительные rulesets см. оригинальный файл
    ],
    "rules": [
      { "inbound": ["socks5"], "outbound": "vless1" },
      { "inbound": ["tun-in", "dns-in"], "action": "sniff" },
      { "protocol": "dns", "action": "hijack-dns" },
      { "ip_is_private": true, "outbound": "direct" },
      { "rule_set": "mydomains", "outbound": "auto" }
    ],
    "default_domain_resolver": "local-server",
    "final": "direct",
    "auto_detect_interface": true
  },
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_fakeip": true
    },
    "clash_api": {
      "external_controller": "0.0.0.0:9090",
      "external_ui": "dashboard"
    }
  }
}
```

</details>

---

## 3. Список собственных доменов

Создайте файл:

```bash
sudo nano /etc/sing-box/mydomains.json
```

Пример содержимого:

```json
{
  "version": 3,
  "rules": [
    { "domain_suffix": ".io" },
    { "domain_suffix": "selfh.st" }
  ]
}
```

---

## 4. Настройка роутера

1. Задайте **статический маршрут** для сети `198.18.0.0/15` до IP вашего Sing-box.
   
2. В настройках **DHCP** укажите DNS сервер для клиентов в локальной сети IP-адрес Sing-box.

---

## 5. Запуск Sing-box

```bash
sudo systemctl enable sing-box
sudo systemctl start sing-box
```

Переподключитесь к локальной сети.

- Дашборд доступен по адресу:  
  [http://IP_singbox:9090/ui](http://IP_singbox:9090/ui)

---

## 6. Troubleshooting

Просмотр логов:

```bash
sudo journalctl -u sing-box --output cat -e
```

---

✦ Теперь вы можете использовать **Sing-box** как gateway для всей локальной сети!








Спасибо https://github.com/vernette за собранные srs и очень качественную инструкцию
Спасибо zipdots https://forummikrotik.ru/viewtopic.php?t=16938
По большому счету эта инструкция просто для себя, чтобы не забыть :)
