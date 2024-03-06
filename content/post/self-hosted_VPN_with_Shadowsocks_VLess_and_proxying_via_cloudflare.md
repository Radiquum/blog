---
draft: false
title: 'Self-Hosted VPN with Shadowsocks vmess and proxying via CDN'
date: '2024-03-06T02:05:18+05:00'
tags: ["guide"]
categories:
  - Docker
---

This post will be a simple instruction how to install ShadowSocks Proxy via docker and use it with vmess, also proxing it through CDN network via websockets.

## how it works

```md
(Client) <-> [ CDN Service ] <-> [ Upstream Server ] <-> (Internet)
```

## what we will use

- [v2ray docker compose](https://github.com/miladrahimi/v2ray-docker-compose)
- [caddy docker proxy](https://github.com/lucaslorentz/caddy-docker-proxy)
- CDN Service: A Content delivery network like [Cloudflare](https://cloudflare.com/), [ArvanCloud](https://arvancloud.ir/) or [DerakCloud](https://derak.cloud/).

check the CDN [free plans](https://github.com/miladrahimi/v2ray-docker-compose/discussions/89), and choose suitable for you.

This guide assumes you are using CloudFlare as your domain CDN and DNS managment. It will allow to auto provision ssl without any setup and cloudflare have more servers in its infrastructure.

## Requirements

- Linux VPS or any other server with linux and dedicated IP.
- Installed git, docker and docker compose plugin.
- Domain name connected to CDN.
- Python 3

## Preparations

1. In your CDN, create an `A` record pointing to your server IP with the proxy option turned off.

2. Clone v2ray-docker-compose repo to your server.

   ```sh
    git clone https://github.com/lucaslorentz/caddy-docker-proxy
    ```

3. Run `v2ray-docker-compose/utils/bbr.sh` to speed up server network.

4. copy `v2ray` folder and `vmess.py` file to desired folder from `v2ray-docker-compose/v2ray-caddy-cdn/`.

5. Generate a UUID via

    ```sh
    cat /proc/sys/kernel/random/uuid
    ```

6. Replace `<UPSTREAM-UUID>` in `v2ray/config/config.json` with the generated UUID.

7. replace `domain = caddy[:caddy.find(' {')]` in `vmess.py` to `domain = <YOUR-DOMAIN>`.

## ShadowSocks vmess installation

1. create a `docker-compose.yml` file, open with text editor and paste the following:

    ```yml
    version: '3.3'
    networks:
      default:
        name: 'proxy_network'
    services:
      caddy:
        image: "lucaslorentz/caddy-docker-proxy:ci-alpine"
        ports:
          - "80:80"
          - "80:80/udp"
          - "443:443"
          - "443:443/udp"
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - /srv/caddy/:/data
        restart: unless-stopped
        environment:
          - CADDY_INGRESS_NETWORKS=proxy_network
      v2ray:
        image: ghcr.io/getimages/v2fly-core:v4.45.2
        restart: always
        environment:
          - v2ray.vmess.aead.forced=false
        volumes:
          - ./v2ray/config/:/etc/v2ray/
          - ./v2ray/logs:/var/log/v2ray/
        ports:
          - "127.0.0.1:1310:1310"
          - "127.0.0.1:1310:1310/udp"
        labels:
          caddy: http://<YOUR-DOMAIN>
          caddy.reverse_proxy: "http://v2ray:1310"
    ```

2. Run `docker-compose up -d`.

3. In your CDN, turn the proxy option on for the record.

4. Run `python3 ./vmess.py` to generate client configuration (link).

you may want to allow ports 80 (tcp + udp) and 443 (tcp + udp) in your server firewall.

## How to connect

copy the generated link and import it as a configuration url in the client application.

### Client Applications

This is the list of recommended applications to use the VMESS protocol:

- [Nekoray](https://github.com/MatsuriDayo/nekoray/releases) for Windows, and Linux.
- [Nekobox](https://github.com/MatsuriDayo/NekoBoxForAndroid) for Android
- [v2rayNG](https://github.com/2dust/v2rayNG) for Android
- [Nekoray - macOS](https://github.com/abbasnaqdi/nekoray-macos/releases) for MacOS
- [ShadowLink](https://apps.apple.com/us/app/shadowlink-shadowsocks-vpn/id1439686518) for iOS
