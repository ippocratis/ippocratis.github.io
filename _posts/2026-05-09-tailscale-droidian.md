---
title: taiscale on droidian 
description: Tailscale and tsnet micro service on droidian.
image: assets/images/tsnet.png
author: ippo
categories:
    - vpn 
    - droidian
    - proxy
tags:
    - vpn
    - droidian
    - proxy
---

Droidian gives you real Debian on a phone, and Tailscale gives that phone a stable address on your private network wherever you are. I use the combo to SSH home, and to expose a couple of local services without touching my router.

This is the exact setup I run on a redmi note 9s running Droidian. It has two parts: first the normal Tailscale client, then a tiny Go app that uses tsnet to publish a service over Tailscale Funnel.

## Part 1: Install Tailscale on Droidian

Droidian tracks Debian bookworm, so we can use Tailscale's official bookworm repo. You need sudo access.

```bash
# Add Tailscale's GPG key
sudo curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg \
  | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

# Add the repository
sudo curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.tailscale-keyring.list \
  | sudo tee /etc/apt/sources.list.d/tailscale.list
```

Install and bring it up:

```bash
sudo apt update
sudo apt install tailscale -y

sudo tailscale up
```

Check you are connected:

```bash
tailscale status
```

That is enough if you just want the phone on your tailnet. For a public HTTPS endpoint without port forwarding, keep going.

## Part 2: Tailscale tsnet Funnel service

tsnet lets an app bring its own Tailscale node. No system daemon needed, and with Funnel you get a `https://yourname.ts.net` address that proxies to a local port. I use it for Vaultwarden, but the pattern works for any HTTP service.

You will need Go:

```bash
sudo apt install golang -y
```

### 1. Set your variables

Edit these before you run anything.

```bash
SERVICE_NAME="vaultwarden"
HOSTNAME="vaultwarden"
LOCAL_PORT=8080
USER=$(whoami)
SERVICE_DIR="/home/droidian/tsnet/vaultwarden"
```

### 2. Create the workspace

```bash
mkdir -p "$SERVICE_DIR/state"
cd "$SERVICE_DIR"
```

### 3. Add your auth key

Create a `.env` file with a reusable auth key from your Tailscale admin console. 

```bash
if [ ! -f ".env" ]; then
  echo 'TS_AUTHKEY=tskey-auth-YOURKEYHERE' > .env
  echo ".env created, edit it with your key"
  exit 0
fi
```

Generate the key at https://login.tailscale.com/admin/settings/keys with "Reusable" and "Ephemeral" checked if you want the node to disappear when the service stops.

### 4. Write the Go proxy

This is a minimal reverse proxy that listens on tsnet's Funnel and forwards to localhost.

```bash
cat > main.go << EOF
package main

import (
    "log"
    "net/http"
    "net/http/httputil"
    "os"
    "tailscale.com/tsnet"
)

func main() {
    srv := &tsnet.Server{
        Hostname: "$HOSTNAME",
        AuthKey:  os.Getenv("TS_AUTHKEY"),
        Dir:      "./state",
    }
    defer srv.Close()

    ln, err := srv.ListenFunnel("tcp", ":443")
    if err != nil {
        log.Fatal(err)
    }

    proxy := &httputil.ReverseProxy{
        Director: func(r *http.Request) {
            r.URL.Host = "localhost:$LOCAL_PORT"
            r.URL.Scheme = "http"
        },
    }

    log.Println("Starting reverse proxy for $SERVICE_NAME...")
    log.Fatal(http.Serve(ln, proxy))
}
EOF
```

### 5. Initialize the module

```bash
go mod init tsnet/$SERVICE_NAME
go mod tidy
go get tailscale.com/tsnet
```

### 6. Build

```bash
go build -o app
```

### 7. Create a systemd service

```bash
sudo tee /etc/systemd/system/${SERVICE_NAME}-funnel.service > /dev/null <<EOF
[Unit]
Description=Tailscale Funnel Proxy for $HOSTNAME
After=network.target

[Service]
EnvironmentFile=$SERVICE_DIR/.env
WorkingDirectory=$SERVICE_DIR
ExecStart=$SERVICE_DIR/app
Restart=always
User=$USER
Group=$USER

[Install]
WantedBy=multi-user.target
EOF
```

### 8. Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable ${SERVICE_NAME}-funnel.service
sudo systemctl start ${SERVICE_NAME}-funnel.service
sudo systemctl status ${SERVICE_NAME}-funnel.service
```

### 9. Fix permissions if needed

```bash
sudo chown -R $USER:$USER "$SERVICE_DIR"
```

You should now see `https://vaultwarden.your-tailnet.ts.net` live in your Tailscale admin, proxying to port 8080 on the phone. Funnel handles TLS automatically.

## Notes from running this on a phone

- State: the `./state` folder holds your node keys. Back it up if you want the same hostname to survive a reinstall.
- Security: Funnel is public internet. Put auth in front of anything sensitive. Vaultwarden already has it, for a raw dashboard add Tailscale serve ACLs or basic auth in the proxy.
- Multiple services: duplicate the folder, change HOSTNAME and LOCAL_PORT, build a second binary. Each gets its own tsnet node.

That is the whole setup. One apt repo for the client, one 30-line Go program for Funnel, and your Droidian device behaves like a tiny home server.