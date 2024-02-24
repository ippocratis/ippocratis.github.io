---
title: Simple site to site tunnel with Wireguard
description: Connect to you Rpi remotely over wirehuard
image: assets/images/wireguard.png
categories:
    - vpn
    - ssh
    - security
tags:
    - vpn
    - ssh
    - security
---

WireGuard is an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography.

# preperation

The docker setup needs the wireguard kernel module , an open udp port in router, a free  public subdomain ,  port forword the WG port to RPI lan IP in router settings

load kernel module

`modprobe wireguard`

load it on boot

create the config file:

`sudo nano /etc/modules-load.d/wireguard.conf`

add the module name

`wireguard`

check status of modules 

`lsmod |grep wireguard`

Check if systemd service loaded the module:

`systemctl status systemd-modules-load.service`

grab the proccess id
e.g.

`Main PID: 210 (code=exited, status=0/SUCCESS)
PID = 210`

status=0/SUCCESS means systemd succesfully loaded the module

double verify

`journalctl -b _PID=210`

systemd-modules-load[210]: Inserted module 'wireguard'

if lsmod is not confirming the journal  look for 

blacklist wireguard at configs

Forword ipv4

`sudo nano /etc/sysctl.d/ipv4_forwarding.conf `

add

`net.ipv4.ip_forward=1`

Verify

`sysctl net.ipv4.ip_forward`


## setup

Docker & docker-compose

[linuxserver Wireguard Docker image](https://github.com/linuxserver/docker-wireguard)

Documentation is pretty straightforword there but lets clarify some things

Client config is under path/to/appdata/config as was set in the volume env var as PNG and text

Docker run

```
docker run -d   --name=wireguard   --cap-add=NET_ADMIN   --cap-add=SYS_MODULE   -e PUID=1000   -e PGID=1000   -e TZ=Europe/London   -e SERVERURL=your.ddns.domain.name.com `#optional`   -e SERVERPORT=51820 `#optional`   -e PEERS=1 `#optional`   -e PEERDNS=auto `#optional`   -e INTERNAL_SUBNET=10.13.13.0 `#optional`   -e SERVER_ALLOWEDIPS_PEER_phone="192.168.1.x/24, 10.13.13.1"   -e LOG_CONFS=true `#optional`   -p 51820:51820/udp   -v path/to/appdata/config:/config   -v /lib/modules:/lib/modules   --sysctl="net.ipv4.conf.all.src_valid_mark=1"   --restart unless-stopped   linuxserver/wireguard
```
docker-compose

```
version: '3'

services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - SERVERURL=your.ddns.domain.name.com  # optional
      - SERVERPORT=51820  # optional
      - PEERS=1  # optional
      - PEERDNS=auto  # optional
      - INTERNAL_SUBNET=10.13.13.0  # optional
      - SERVER_ALLOWEDIPS_PEER_phone=192.168.1.x/24,10.13.13.1
      - LOG_CONFS=true  # optional
    ports:
      - "51820:51820/udp"
    volumes:
      - path/to/appdata/config:/config
      - /lib/modules:/lib/modules
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

This will create
path/to/appdata/config/wg0.conf
which is your server config

And 
path/to/appdata/config/peer1/peer1.conf
Which is your client config

You'll add this to the android wiregurad app by either import the file or by scanning the path/to/appdata/config/peer1/peer1.png
You can display the qr code in terminal with
`docker exec -it wireguard /app/show-peer 1`


Consider the 
`SERVER_ALLOWEDIPS_PEER_phone=`
And
`INTERNAL_SUBNET=`
enviromental variables
If you only need access to your RPI for remote ssh all you need is to add your its local lan IP e.g. 192.168.1.24 together with the servers  wireguard subnet address e.g. 10.13.13.1

Also consider the
`SERVERURL=`
Env var 
Which is your domain through which you access your raspbery on its running port
e.g.
If you own the ippo subdomain in duckdns
`SERVERURL=https://ippo.duckdns.org:51820`

The wireguard config logic:

Wireguard needs a set of configs in the form of:

client:

```
[Interface]
Address = 10.13.13.2
PrivateKey = blahblahblah1=
ListenPort = 51000
DNS = 10.13.13.1

[Peer]
PublicKey = blahblahblah2=
PresharedKey = blahblahblah3=
Endpoint = my.ddns.domain.com:51000
AllowedIPs = 0.0.0.0/0, ::/0
```

server:

```
[Interface]
Address = 10.13.13.1
ListenPort = 51000
PrivateKey = blahblahblah4
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth+ -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth+ -j MASQUERADE

[Peer]
# peer1
PublicKey = blahblahblah5
PresharedKey = blahblahblah6
AllowedIPs = 10.13.13.2/32
```

In the public key field of the peer section of the server goes the public key of the client

In the public key field of the client's peer section goes the server's public key

In the PresharedKey field of both the server and the client goes the PresharedKey from the client's folder. This key is optional and provides additional security

In the Endpoint field of the peer section of the client goes the public ip of the server and the port it is listening on e.g. if we have the dynamic dns domain wire-test.duckdns.org we will put this in this field wire-test.duckdns.org:5000

In the DNS field of the interface section of the client we put the wireguard subnet address of the server e.g. 10.13.13.1 as shown in the address field of the interface section of the server

The AllowedIPs fiend of the peer section of the client is the most interesting field because from this field we can configure whether the client can connect only to the specific device on which the server is running e.g. for remote ssh or to the whole subnet to all devices of the local LAN i.e. to be connected to the wifi of the house

Connecting to the whole subnet:
`AllowedIPs = 0.0.0.0/0, ::/0`

Connection only to the server:
Enter the local lan IP of the device running server e.g. 192.168.1.24 along with the wireguard subnet address of the server e.g. 10.13.13.1 as shown in the Address field of the interface section of the server
`AllowedIPs = 192.168.1.24/32 , 10.13.13.1/32`

In the AllowedIPs fiend of the peer section of the server goes the wireguard subnet address of the client as shown in the Address field of the interface section of the client

The ListenPort field of the interface section of the server is the udp port on which the wireguard listens. We define it as env var to docker run. It has to be an open router port and  port forworded to the device IP that is running the server

The address field of the interface section of both the server and the client are their ip address in the wireguard subnet (the wireguard internal addresses not the LAN's)

In the PrivateKey fields on the interface section on both server and client are as shown in privatekey-peer1 and privatekey-server

The PostUp and PostDown fields of the server interface section are the iptables as created by the docker run.
