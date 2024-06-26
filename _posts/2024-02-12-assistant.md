---
title: Home Assistant under Docker
description: Home Assistant  under Docker
author: ippo
image: assets/images/assistant.jpeg
categories:
    - alternative
    - assistant 
tags:
    - alternative
    - assistant 
---

Home assistant is an Open source home automation that puts local control and privacy first. Powered by a worldwide community of tinkerers and DIY enthusiasts.

## Docker compose 
```
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
#      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      DISABLE_JEMALLOC: true
```


Your instance is live at http://your_lan_ip:8123

D-Bus is optional but required if you plan to use the Bluetooth integration.

The HA Container is using jemalloc lib for Python runtime speedup.

It can cause issues on certain hardware.

[integrations](https://www.home-assistant.io/integrations/#all)

## Add the community store

`mkdir config/custom_components/hacs`

And cd to it

`wget https://github.com/hacs/integration/releases/latest/download/hacs.zip`

`unzip hacs.zip`

restart container

## On the android app

Settings>devices>add integration>hacs

Acknowledge all and submit

Activate device on the URL from the activation prompt and provide the code it displays

Under settings>devices>hacs

Enable AppDaemon apps discovery & tracking and submit

From the left sidebar open hacs

From the top right menu remove all filters and categories to view the full list or use search

## Local speech to text

Speech to text for home assistant "assist"

Docker compose

```
version: '3'
services:
  wyoming-whisper:
    image: rhasspy/wyoming-whisper
    ports:
      - "10300:10300"
    volumes:
      - "./data:/data"
    command: ["--model", "tiny-int8", "--language", "en"]
```

On HS

Settings>devices>add integration>whisper>ok

Host is your raspberry pi IP e.g. 192.168.1.1

Port is 10300 (can map any available port on host o.c.)

Then under Settings>devices>Wyoming protocol>entity>fast whisper>settings>enable

And finaly

Settings>voice assistant>assist>select whisper under speech to text

[source](https://www.home-assistant.io/installation/linux#docker-compose)
