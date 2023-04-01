---
title: "将 forward_proxy 与 sing-box 结合实现远程分流"
date: 2023-04-01T14:00:00+08:00
tags: ['network']
---

无感知的远程分流。

<!--more-->

由于 New Bing 和 ChatGPT 将越来越多的 IDC IP 屏蔽，我们很难通过其本身 IP 正常访问。而 WARP 提供的 IP 大多是可以正常访问的。但是 WARP 全局又会导致速度较慢和另一些 IP 歧视。因此我萌生了使用 forward_proxy 与 sing-box 结合，实现使用 naive 进行无感远程分流的想法。

## 原理

**naive** -> **caddy** (forward_proxy) -> socks5h://127.0.0.1:10000 (inbound) **sing-box** (outbound) **WARP** (socks5h://127.0.0.1:40000) OR **Direct**

其中，需要使用官方工具或者脚本使用 WARP 在 socks5h://127.0.0.1:40000 开端口。例如：

- [官方工具](https://developers.cloudflare.com/warp-client/get-started/linux/)
- [fscarmen/warp](https://github.com/fscarmen/warp)
- [P3TERX/warp.sh](https://github.com/P3TERX/warp.sh)

​	**请注意：使用官方工具 + warp（默认 wireguard）会导致小鸡失联，且无法挽回。**

## 配置文件

Caddyfile

```Caddtfile
{
  order forward_proxy before file_server
}
:443, example.com {
  tls me@example.com
  forward_proxy {
    basic_auth user pass
    hide_ip
    hide_via
    probe_resistance
    upstream socks5h://127.0.0.1:10000
  }
  file_server {
    root /var/www/html
  }
}
```

sing-box

```json
{
  "log": {
    "level": "info"
  },
  "dns": {
    "servers": [
      {
        "address": "tls://1.1.1.1"
      }
    ]
  },
  "inbounds": [
    {
      "type": "shadowsocks",
      "tag": "ss-in",
      "listen": "::",
      "listen_port": 8080,
      "sniff": true,
      "method": "2022-blake3-aes-128-gcm",
      "password": "8JCsPssfgS8tiRwiMlhARg=="
    },
    {
      "type": "socks",
      "tag": "socks-in",
      "listen": "127.0.0.1",
      "listen_port": 10000,
      "sniff": true
    }
  ],
  "outbounds": [
    {
      "type": "direct"
    },
    {
      "type": "socks",
      "tag": "cf_out",
      "server": "127.0.0.1",
      "server_port": 40000
    },
    {
      "type": "dns",
      "tag": "dns-out"
    }
  ],
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "domain_suffix": [
          "bing.com",
          "openai.com"
        ],
        "outbound": "cf_out"
      }
    ]
  }
}
```

Enjoy!
