---
title: Multi Tailscale tsnet.Server Funnels
description: Automating the exposure of multiple services using Tailscale Funnel with tsnet.Server, including Go service generation, Docker containers, and systemd integration.
image: assets/images/tailscale.png
categories:
    - vpn
    - ssh
    - security
tags:
    - vpn
    - ssh
    - security
---

**Expose Multiple Services on Tailscale Funnel with tsnet and Python Automation**

## Tailscale

Tailscale is a secure, zero-config, modern site-to-site mesh network built on WireGuard. It allows you to create a private e2e encrypted network between your devices. Devices connect to each other directly using NAT traversal (like STUN + hole punching) or via encrypted relays (derp).

## Funnel

Tailscale Funnel is a feature that allows you to expose a local service (like a web app running on your Raspberry Pi) to the public internet via a Tailscale-assigned HTTPS URL (e.g., https://your-device-name.ts.net). **It's ideal for sharing services without configuring port forwarding or exposing your whole network.**

Funnel  must be explicitly enabled.

It supports HTTPS automatically with TLS handled by Tailscale.


## tsnet.Server

tsnet.Server is a Go library provided by Tailscale that lets you embed Tailscale networking directly into your Go programs—no need to run the tailscaled daemon separately.

Key features:

Acts as a lightweight embedded Tailscale node.

You can assign it a custom Hostname.

Call srv.ListenFunnel("tcp", ":443") in its configuration to publicly expose services.

Supports custom state directory (Dir) and authentication via TS_AUTHKEY.

It’s perfect for exposing microservices securely and publicly with minimal setup.



## The problem

By default, Tailscale allows Funnels only on these 3 ports: 443,80,8080 . You can only bind to one of the allowed ports per instance . **"Each tailnet node can only expose one service per allowed port using Funnel .**

## The easy way:

To get around the single-port limitation, one common approach is to use subpaths with a reverse proxy

It listens for incoming HTTP(S) requests and forwards them to the correct internal service based on the path.

e.g.

https://myproxy.ts.net/notepad   → localhost:8081

https://myproxy.ts.net/webdav    → localhost:8082

https://myproxy.ts.net/vault     → localhost:8083

Creating funnels on subpaths is easy using tailscale CLI:

eg:

`sudo tailscale funnel --bg --set-path /radicale http://localhost:5232`

bg flag runs the funnel in the background and starts it on reboot

The --set-path flag is self-explanatory

5232 is the port your service exposes to the host.

## the subpath problem

But subpaths aren't always ideal. 

Many applications are designed to run at the root of the domain.

Additionally apps may use relative URLs that break when accessed under a subpath. For example, links within the app may point to "/file" but expect the base domain (https://mydevice.ts.net/) instead of the subpath (https://mydevice.ts.net/notepad/), causing them to fail

That said, some popular apps support setting up subpath 

For example nextcloud offer the 'overwritewebroot' flag in config.php and photoprism the PHOTOPRISM_SITE_URL environmental variable 

## Solution

Multiple tsnet.server instances.

Each tsnet.Server instance acts like a separate lightweight Tailscale device with its own hostname and Funnel. This bypasses the single-Funnel-per-port limitation by simulating separate devices.

Spin up separate virtual devices via tsnet.Server with unique Hostname exposed via Funnel.

Example:

tsnet.Server{
  Hostname: "notes",
  Dir: "/state/notes",
}
With srv.ListenFunnel("tcp", ":443")

And

tsnet.Server{
  Hostname: "vault",
  Dir: "/state/vault",
}
With srv.ListenFunnel("tcp", ":443")

This way, you'll get:

https://vault.yourtail.ts.net

https://notes.yourtail.ts.net


Each service is exposed securely with automatic TLS, without needing a public IP, and without exposing the entire network. 

That’s a huge plus.

For users behind a cgnat without a router public ip . 

For Users that want to limit their setup attack surface and not expose any open ports on their router to the public internet.


## Prerequisites

- Go
-  Python
-  Git
-  Systemd
-  Tailscale

## Automation

We will use a Python script that automates the deployment of tsnet-based services for Tailscale Funnel:

1. Configuration: It reads services.yml to get the service names, hostnames, and ports.


2. Go Binary Creation: It generates a Go binary (app) for each service that listens on a Tailscale Funnel port and proxies traffic to the service.


3. Systemd Service: It sets up systemd services to manage the Go binaries, ensuring they start on boot and restart on failure.


4. Environment Handling: It uses the .env file to pass the TS_AUTHKEY to the Go binaries.


5. Automated Deployment: The script automates creating directories, fixing permissions, building the application, and installing systemd services.


## Systemd.py
{% raw %}
```
import yaml
import os
import subprocess
from pathlib import Path

# Read configuration
with open("services.yml") as f:
    config = yaml.safe_load(f)

# Get original user
USER = os.getenv("SUDO_USER") or os.getenv("USER")

# Validate .env
env_path = Path(".env").absolute()
if not env_path.exists():
    raise SystemExit("❌ .env file not found at current directory")

# Parse .env
env_vars = {}
with open(env_path) as f:
    for line in f:
        if "=" in line and not line.strip().startswith("#"):
            key, val = line.split("=", 1)
            env_vars[key.strip()] = val.strip()

if "TS_AUTHKEY" not in env_vars:
    raise SystemExit("❌ TS_AUTHKEY missing in .env")

SYSTEMD_DIR = Path("/etc/systemd/system")

def run_cmd(cmd, cwd=None):
    result = subprocess.run(cmd, cwd=cwd, capture_output=True, text=True)
    if result.returncode != 0:
        raise RuntimeError(f"Command failed: {result.stderr}")
    return result

for name, info in config["services"].items():
    service_dir = Path(name).absolute()
    hostname = info["hostname"]
    port = info["port"]
    binary_path = service_dir / "app"
    systemd_unit_path = SYSTEMD_DIR / f"{name}-funnel.service"

    print(f"\n🚀 Processing {name}")

    (service_dir / "state").mkdir(parents=True, exist_ok=True)

    try:
        run_cmd(["chown", "-R", f"{USER}:{USER}", str(service_dir)])
    except Exception as e:
        print(f"⚠️ Permission fix error: {e}")

    main_go = service_dir / "main.go"

    if not binary_path.exists():
        print("📝 Writing Go source and building binary...")
        main_go.write_text(f'''package main

import (
    "log"
    "net/http"
    "net/http/httputil"
    "os"
    "tailscale.com/tsnet"
)

func main() {{
    srv := &tsnet.Server{{
        Hostname: "{hostname}",
        AuthKey:  os.Getenv("TS_AUTHKEY"),
        Dir:      "./state",
    }}
    defer srv.Close()

    ln, err := srv.ListenFunnel("tcp", ":443")
    if err != nil {{
        log.Fatal(err)
    }}

    proxy := &httputil.ReverseProxy{{
        Director: func(r *http.Request) {{
            r.URL.Host = "localhost:{port}"
            r.URL.Scheme = "http"
        }},
    }}

    log.Println("Starting reverse proxy for {name}...")
    log.Fatal(http.Serve(ln, proxy))
}}
''')

        try:
            print("🔨 Building binary...")
            run_cmd(["go", "mod", "init", f"tsnet/{name}"], cwd=service_dir)
            run_cmd(["go", "mod", "tidy"], cwd=service_dir)
            run_cmd(["go", "get", "tailscale.com/tsnet"], cwd=service_dir)
            run_cmd(["go", "build", "-o", "app"], cwd=service_dir)
            print("✅ Build successful")
        except Exception as e:
            print(f"❌ Build failed: {str(e)}")
            continue
    else:
        print("✅ Binary already exists, skipping build")

    if not systemd_unit_path.exists():
        print("🛠️ Creating and installing systemd unit...")
        service_file = service_dir / f"{name}-funnel.service"
        service_content = f"""
[Unit]
Description=Tailscale Funnel Proxy for {hostname}
After=network.target

[Service]
EnvironmentFile={env_path}
WorkingDirectory={service_dir}
ExecStart={service_dir}/app
Restart=always
User={USER}
Group={USER}

[Install]
WantedBy=multi-user.target
"""
        service_file.write_text(service_content.strip())

        try:
            run_cmd(["mv", str(service_file), str(systemd_unit_path)])
            run_cmd(["systemctl", "daemon-reload"])
            run_cmd(["systemctl", "enable", f"{name}-funnel.service"])
            run_cmd(["systemctl", "start", f"{name}-funnel.service"])
            print(f"✅ {name} systemd service installed and started")
        except Exception as e:
            print(f"❌ Failed to install/start service: {str(e)}")
    else:
        print("✅ Systemd service already exists, skipping installation")

print("\n🎉 All services checked and deployed!")
```
{% endraw %}

## brief explanation

1. Read Configuration from YAML

The script reads the configuration file services.yml to extract information about each service that will be deployed. Each service must have a hostname and port defined in the configuration.


`with open("services.yml") as f:
    config = yaml.safe_load(f)`

File structure

**services.yml**

```
services:
  photoprism:
    port: 2342
    hostname: photoprism
  caddydav:
    port: 8043
    hostname: caddydav
  vault:
    port: 8066
    hostname: vault
  nginxdav:
    port: 32080
    hostname: nginxdav
  nextcloud:
    port: 8080
    hostname: nextcloud
  wallabag:
    port: 8106
    hostname: wallabag
  radicale:
    port: 5233
    hostname: radicale
  baikal:
    port: 8456
    hostname: baikal
```


2. Validate .env File

Validates that the .env file exists in the current directory. The .env file should contain a TS_AUTHKEY key (Tailscale authentication key).

To generate an auth key:

Open the Keys page of the admin console.
https://login.tailscale.com/admin/settings/keys

Select Generate auth key.

Fill out the form 

Select Pre-approved

Select Generate key.

Copy the key to .env

.env file format

`TS_AUTHKEY=your_key_goes_here`

3. Checks if go binaries and systemd units exits if so skips yhem else generates them

4. Systemd Setup

The script defines a path to the systemd directory (/etc/systemd/system), where service files will be installed.


SYSTEMD_DIR = Path("/etc/systemd/system")

The generated unit file is in the format:

## service-funnel.service

```
[Unit]
Description=Tailscale Funnel Proxy for wallabag
After=network.target

[Service]
EnvironmentFile=/run/media/ippo/TOSHIBA/tsnet-funnel/stack/.env
WorkingDirectory=/run/media/ippo/TOSHIBA/tsnet-funnel/stack/wallabag
ExecStart=/run/media/ippo/TOSHIBA/tsnet-funnel/stack/wallabag/app
Restart=always
User=ippo
Group=ippo

[Install]
WantedBy=multi-user.target
```

4. Helper Function: run_cmd

The run_cmd function is used to run shell commands (subprocess.run). It includes a cwd argument to specify the working directory for commands. If a command fails, it raises an error with the command's stderr.


5. Iterate Over Each Service

The script loops through each service defined in services.yml, processing each one by:

Creating Directories: It creates a state directory for storing service state.

Fixing Permissions: It attempts to set the ownership of the service directory to the current user.

Generating main.go: It writes a Go file (main.go) that sets up a tsnet.Server to listen on Tailscale's Funnel port (:443) and reverse proxy traffic to the specified service on localhost:{port}.

Building the Go Binary: It uses go commands to build a binary for the service.

That generated main.go app is in the format:

## main.go
{% raw %}
```
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
        Hostname: "vault",
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
            r.URL.Host = "localhost:8066"
            r.URL.Scheme = "http"
        },
    }

    log.Println("Starting reverse proxy for vault...")
    log.Fatal(http.Serve(ln, proxy))
}
```
{% endraw %}


6. Build Go Binary

The script runs several go commands to initialize the module, fetch dependencies (including Tailscale), and build the Go binary (app) for each service.

7. Create and Install systemd Service

For each service, a systemd service file is generated. This service file:

Sets the environment file to .env.

Defines the service to execute the Go binary (app).

Configures the service to restart on failure.

Installs the service by moving the file to the systemd directory and enabling and starting the service with systemctl.

The service file is written to disk, moved to /etc/systemd/system, and then installed using systemctl.

9. Final Output

Once all services are processed and installed, a success message is printed.


print("\n🎉 All services deployed!")

## summary

You need 3 files 

- The systemd.py script
- The services.yml
- the .env

You run the script `sudo python3 systemd.py`

That's all

Easy..

Your funnels are live at their subdomains at ` hostname.tailscale_host.ts.net`

You can check the subdomain of each service at the admin panel on your tailscale account page or from the CLI with

`tailscale status --json`

and get a list of all subdomains with:

`grep 'hostname:' ./services.yml | awk '{print $2}' | xargs -I{} sh -c 'echo -n "{}: "; tailscale status --json | jq -r --arg hostname "{}" ".Peer[] | select(.HostName == \$hostname) | .DNSName"'`

## Auth key expiration

Tailscale auth keys expire .
Max duration ATM is 90 days .