---
layout: post
title:  "Caddy as a webdav and file server"
author: ippo
categories: [ server ]
tags: [ server ]
image: assets/images/caddy_dav.png
---

Caddy as a WebDAV and file server with basic auth , allowed DNS, basic auth and logs.

<br>

- Caddy does not include webdav module by default so you have to build the caddy executable including the webdav handler with xcaddy tool.
  link to the unofficial module is [here](https://github.com/mholt/caddy-webdav)

## Dockerfile
```
FROM caddy:builder AS builder

# Add WebDAV plugin
RUN xcaddy build \
    --with github.com/mholt/caddy-webdav

FROM caddy:latest

# Copy the custom Caddy binary
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

Caddyfile
```
:8080 {
    root * /data

    # Replace 'basicauth' with 'basic_auth'
    basic_auth / {
        user password_hash
    }

    # Use a route block for WebDAV
    route {
        webdav
    }

    # Host-based access control
    @allowedHosts host example.org
    respond @allowedHosts "Access Denied" 403

log {
        output file /var/log/caddy/access.log
        format json
        level info
    }
}
```

docker-compose.yml
```
version: "3.9"
services:
  caddy:
    image: caddy-webdav
    user: 1000:1000
    ports:
      - "8043:8080"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - /run/media/ippo/TOSHIBA:/data
      - ./logs:/var/log/caddy  # Mount logs directory
    restart: unless-stopped
```
    
## Password hash

`docker exec -it caddy-webdav caddy hash-password --plaintext "your_password_here"`

## Tips

- Where caddy-webdav the container name or Id

- Host-based access control : your ddns goes here

- If you need http directory browsing and static file serving you can use the file_server  directive 

In this case you need matchers if you route the same path for file server and webdav and if plan to use others directives in your Caddyfile like reverse proxy.

- The order also matters , WebDAV should go first.

- In my usecase I use caddy in host for reverse proxy and this container for file_serving and webdav

 
Caddyfile
```
:8080 {
    root * /data

    route /* {

# order matters handle webdav first

# Match WebDAV clients first
        @webdavClients not header User-Agent *Mozilla*
        webdav @webdavClients

        # Match browser requests and serve a directory listing
        @browserRequests header User-Agent *Mozilla*
        file_server browse
}

# Authentication
    basicauth / {
        user password_hash
    }

# Host-based access control
    @allowedHosts host example.com
    respond @allowedHosts "Access Denied" 403

    # Logging
    log {
        output file /var/log/caddy/access.log
        format json
        level info
    }
}
```

Diferent routes example

```
# WebDAV access
    route /some_dir* {
        @webdavClients not header User-Agent *Mozilla* webdav @webdavClients
    }

    # Directory browsing
    route /some_dir* {
        @browserRequests header User-Agent *Mozilla* file_server browse
    }
```
