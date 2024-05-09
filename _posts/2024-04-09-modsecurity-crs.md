---
title: Modsecurity with CRS，開源的 WAF 防火牆
author: Cooper
date: 2024-04-09 00:30:00 +0800
categories: [Server]
tags: [WAF, DevOps]
---

## 前言

`ModSecurity`、又稱 `Modsec`，原本是作為 `Apache HTTP Server` 的其中一個模組，後來自己發展為獨立形式的防火牆，結合 `OWASP` 定期會更新的漏洞規則 `CRS`，可以達成 Web 應用的基本防務，但要注意的是 `ModSecurity` 的性能不算好，所以建議要做其他進階設定，可以考慮透過外部 WAF 來處理

## 部屬

首先簡單說明一下部屬的要求

1. 了解 `nginx` 或是 `openresty` 的設置
2. `docker` 基礎知識

沒錯，就是這麼簡單，如果搭配 `openresty` 也可以撰寫 `lua` 腳本配合 redis 去處理快取等功能

這邊使用 `owasp/modsecurity-crs:openresty-alpine-fat` 這個鏡像，以下是 `docker-compose.yml`

backend 這個服務可以是任何後端服務，例如 `go`、`php`、`java` 等

```yaml
services:
  backend:
    build: .
    restart: unless-stopped
    volumes:
      - ./:/app

  openresty:
    image: owasp/modsecurity-crs:openresty-alpine-fat
    restart: unless-stopped
    volumes:
      - .docker/openresty/main.conf:/usr/local/openresty/nginx/conf/conf.d/main.conf
      - ./:/app
    sysctls:
      - net.ipv4.tcp_max_tw_buckets=20000
      - net.core.somaxconn=65535
      - net.ipv4.tcp_max_syn_backlog=262144
    ulimits:
      nofile:
        soft: 1024000
        hard: 1024000
    ports:
      - "80:80"
```

然後是 `main.conf`

```conf
upstream backend {
  server backend:80;
}

server {
  listen 80;
  server_name 127.0.0.1;

  access_log off;

  root /app/public;

  client_max_body_size 100M;

  location ~ /\.ht {
    deny all;
  }

  location ~ /\.git {
    deny all;
  }

  location ^~ / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    if (!-f $request_filename) {
      proxy_pass http://backend;
    }
  }
}
```

> 事實上這個鏡像一開始就是啟用 `modsecurity` 模組的，所以當作一般的 nginx conf 撰寫即可
> {: .prompt-warning }

到這一步其實你的網站就有基本的 WAF 防護功能，但如果你對 `ModSecurity` 還有更多細項要調整，如 `detection_paranoia_level`，偵測敏感等級等
，可以參考[**官方文檔**][1]

最後測試幾個簡單的項目

```bash
# 命令注入攻擊 sh、bash
curl --location 'http://127.0.0.1/?id=%2Fbin%2Fbash'
```

```bash
# SQL injection id=1 AND 1=1
curl --location 'http://127.0.0.1/?id=1%20AND%201%3D1'
```

```bash
# XSS，id='<script>alert(/xss/)</script>'
curl --location 'http://127.0.0.1/?id=%27%3Cscript%3Ealert(%2Fxss%2F)%3C%2Fscript%3E%27'
```

不意外的就算在預設最低的 `detection_paranoia_level` 下，全部被擋下來，回傳 `403`

```html
<html>
  <head>
    <title>403 Forbidden</title>
  </head>
  <body>
    <center><h1>403 Forbidden</h1></center>
    <hr />
    <center>openresty</center>
  </body>
</html>
```

搭配上 `Cloudflare` 的機器人對抗模式、進階安全設定，就可以最大的保証應用安全啦

[1]: https://github.com/coreruleset/modsecurity-crs-docker
