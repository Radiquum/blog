---
draft: false
title: 'Делаем свой VPN с помощью Shadowsocks vmess и проксируем его через Сеть Доставки Контента (CDN).'
date: '2024-03-06T02:05:18+05:00'
tags: ["руководство"]
categories:
  - Docker
---

Этот пост будет инструкцией как установить ShadowSocks vmess прокси через докер и использовать его через CDN.

## Как это работает

```md
(Клиент) <-> [ CDN Сервис ] <-> [ Сервер ] <-> (Интернет)
```

## Что будет использоваться

- [v2ray docker compose](https://github.com/miladrahimi/v2ray-docker-compose)
- [caddy docker proxy](https://github.com/lucaslorentz/caddy-docker-proxy)
- CDN сервис: [Cloudflare](https://cloudflare.com/), [ArvanCloud](https://arvancloud.ir/) или [DerakCloud](https://derak.cloud/).

проверьте [бесплатные планы](https://github.com/miladrahimi/v2ray-docker-compose/discussions/89), и выберите подходящий.

Это руководство подразумевает, что вы используете CloudFlare как CDN и управление DNS для вашего домена.

## Требования

- Linux VPS или любой другой сервер с линукс и выделенным IP.
- Установленные git, docker и docker compose плагин.
- Доменное имя подключённое к CDN.
- Python 3

Для доменов можно использовать .ru и .su, так как это самые дешёвые доменные имена на данный момент.

Для выбора провайдера VPS можно использовать статью на хабре: [Какого провайдера VPS выбрать для собственного VPN в 2023 году. Платим за всё российской картой](https://habr.com/ru/articles/729750/)

## Подготовка

1. В панели управления CDN, создайте `A` запись которая указывает на IP сервера с выключенной функцией проксирования.

2. Склонируйте репозиторий v2ray-docker-compose на ваш сервер.

   ```sh
    git clone https://github.com/lucaslorentz/caddy-docker-proxy
    ```

3. Запустите `v2ray-docker-compose/utils/bbr.sh` для улучшений работы сети на сервере.

4. Скопируйте папку `v2ray` и файл `vmess.py` в любую папку из `v2ray-docker-compose/v2ray-caddy-cdn/`.

5. Создайте UUID с помощью выполнения команды

    ```sh
    cat /proc/sys/kernel/random/uuid
    ```

6. Замените `<UPSTREAM-UUID>` в `v2ray/config/config.json` на созданный UUID.

7. Замените `domain = caddy[:caddy.find(' {')]` в `vmess.py` на `domain = <ваш домен>`.

## Установка ShadowSocks vmess

1. создайте файл `docker-compose.yml`, откройте его с помощью текстового редактора и вставьте следующее:

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
          caddy: http://<ваш домен>
          caddy.reverse_proxy: "http://v2ray:1310"
    ```

    не забудьте заменить `<YOUR-DOMAIN>` на ваше доменное имя.

2. Выполните команду `docker compose up -d`.

3. В панели управления CDN, включите функцию проксирования.

4. Запустите `python3 ./vmess.py` что-бы создать ссылку конфигурации.

вам может потребоваться открыть порты 80 (tcp + udp) и 443 (tcp + udp) на сервере.

## Как подключится

скопируйте созданную ссылку конфигурации и импортируйте её в приложение для подключения.

### приложения

Это список рекомендованных приложений для использования с протоколом VMESS:

- [Nekoray](https://github.com/MatsuriDayo/nekoray/releases) для Windows, и Linux.
- [Nekobox](https://github.com/MatsuriDayo/NekoBoxForAndroid) для Android
- [v2rayNG](https://github.com/2dust/v2rayNG) для Android
- [Nekoray - macOS](https://github.com/abbasnaqdi/nekoray-macos/releases) для MacOS
- [ShadowLink](https://apps.apple.com/us/app/shadowlink-shadowsocks-vpn/id1439686518) для iOS
