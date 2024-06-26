---
layout: post
title:  "Owntracks location tracker"
author: ippo
categories: [ map, location ]
tags: [ map, location, mtls ]
image: assets/images/owntracks.jpg
---

Tracking you own and family/friends location and have a history of locations on map like a google maps history open source alternative

- Owntracks is a location history tracking app
- Mosquitto is a mqtt protocol,  publish/subscribe message  brocker that stores data that recieves from clients
- Recorder is a lightweight program for storing and accessing location data published via MQTT (or HTTP) and displays them in a web ui on a map as tracks,  points etc
- The android app tracks location and sends the location data to mosquitto, then owntracks recorder get the data from mosquitto and graphically displays them in a webui
- mTLS is used to authenticate clients, whereas "normal" TLS just authenticates the server. The authentication is now mutual!
- mTLS is one of the puzzle pieces of building a Zero Trust Network as it strictly controls which clients are allowed to connect to a service regardless of where a user or device is connecting from 
- in the below setup the android app connects with mtls with mosquitto and the browser connects with mtls with the webui through caddy
- Caddy 2 is a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go
- Recorder also supports tls but I suffered trying to make it work without success as it talks with mosquitto only inside our lan and is basic auth protected I'm still fine with the current setup.

**Lets dive in**

**create directories**

`mkdir {config,mosquitto,store,store/last,mosquitto/config}`


**create the certificates for mosquitto and owntracks android app**

- script

`nano certs.sh`

Paste

```
#!/bin/bash

IP="your_lan_ip"
SUBJECT_CA="/C=SE/ST=Athens/L=Athens/O=ippo/OU=CA/CN=$IP"
SUBJECT_SERVER="/C=SE/ST=Athens/L=Athens/O=ippo/OU=Server/CN=$IP"
SUBJECT_CLIENT="/C=SE/ST=Athens/L=Athens/O=ippo/OU=Client/CN=$IP"

function generate_CA () {
   echo "$SUBJECT_CA"
   openssl req -x509 -nodes -sha256 -newkey rsa:2048 -subj "$SUBJECT_CA"  -days 3650 -keyout ca.key -out ca.crt
}

function generate_server () {
   echo "$SUBJECT_SERVER"
   openssl req -nodes -sha256 -new -subj "$SUBJECT_SERVER" -keyout server.key -out server.csr
   openssl x509 -req -sha256 -in server.csr -CA ca.crt -extfile v3.ext -CAkey ca.key -CAcreateserial -out server.crt -days 3650
}

function generate_client () {
   echo "$SUBJECT_CLIENT"
   openssl req -new -nodes -sha256 -subj "$SUBJECT_CLIENT" -out client.csr -keyout client.key 
   openssl x509 -req -sha256 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 3650
}

generate_CA
generate_server
generate_client
```

- Change IP and ST,L,O for ca,server,client crt's also pick an appropriate -days for them
- place ext file on the same dir with the script
- copy ca.crt,server.crt,server.key to mosquitto/config (read next)

**ext file for filling S.A.N. field**

`nano v3.ext`

Paste

```
subjectKeyIdentifier   = hash

authorityKeyIdentifier = keyid:always,issuer:always

basicConstraints       = CA:TRUE

keyUsage               = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment, keyAgreement, keyCertSign

subjectAltName         = DNS:mqtt.example.org

issuerAltName          = issuer:copy


```

- Note subjectAltName use your dynamic dns address
This is mantatory these will be the allowed domain for this certificate

 **make the script executable and run it** 

`chmod +x certs.sh`

`bash certs.sh`

**generate a pkcs12 bundle out of the client cert**

`openssl pkcs12 -export -out cert.p12 -inkey client.key -in client.crt -legacy`

- Transfer ca.crt and cert.p12 on android device

`cp ca.crt server.crt server.key  mosquitto/config`

- Install ca.crt on android>settings>security>encryption>install a certificate>ca certificate
- Select cert.p12 under owntracks>preferences>connection>security
- Android can't  handle modern pkcs encryption algorythms (PBES2, PBKDF2, AES-256-CBC, Iteration 2048, PRF hmacWithSHA256) that is used on openssl v3 . You can omit the -legacy flag if you are creating the pkcs cert with older openssl versions

**create the recorder configuration file**

`nano config/recorder.conf`

Paste

```
OTR_TOPICS = "owntracks/#"
OTR_HTTPHOST = "0.0.0.0"
OTR_HOST = "your lan ip"
OTR_USER = "user"
OTR_PASS = "pass"
```

**create the mosquitto's configuration file**

`nano mosquitto/config/mosquitto.conf`

Paste

```
persistence true
persistence_location /mosquitto/data/
listener 1883
password_file /mosquitto/passwd/pass

listener 8883

cafile /mosquitto/config/ca.crt
certfile /mosquitto/config/server.crt
keyfile /mosquitto/config/server.key
require_certificate true
use_identity_as_username true
protocol websockets
```
**create the compose file**

`nano docker-compose.yml`

Paste

```
version: '3'
services:
    mosquitto:
        image: eclipse-mosquitto:openssl
        container_name: mosquitto
        restart: unless-stopped
        ports:
            - "1883:1883"
            - "8883:8883"
        volumes:
            - "./mosquitto/config:/mosquitto/config"
            - "./mosquitto/data:/mosquitto/data"
            - "./mosquitto/config/passwd:/mosquitto/passwd"
        environment:
            - TZ=Europe/Athens
        user: "1000:1000"
    otrecorder:
        image: otr-arm:latest
        ports:
            - 8083:8083
        volumes:
            - ./config:/config
            - ./store:/store
        restart: unless-stopped
```
**create and build the recorder image**

### The files:

Dockerfile:

```
FROM alpine:3.16 AS builder

ARG RECORDER_VERSION=0.9.3
# ARG RECORDER_VERSION=master

RUN apk add --no-cache \
        make \
        gcc \
        git \
        shadow \
        musl-dev \
        curl-dev \
        libconfig-dev \
        mosquitto-dev \
        lmdb-dev \
        libsodium-dev \
        lua5.2-dev \
 util-linux-dev

RUN git clone --branch=${RECORDER_VERSION} https://github.com/owntracks/recorder /src/recorder
WORKDIR /src/recorder

COPY config.mk .
RUN make -j $(nprocs)
RUN make install DESTDIR=/app

FROM alpine:3.16

VOLUME ["/store", "/config"]

RUN apk add --no-cache \
 curl \
 jq \
 libcurl \
 libconfig \
 mosquitto \
 lmdb \
 libsodium \
 lua5.2 \
 util-linux

COPY recorder.conf /config/recorder.conf
COPY JSON.lua /config/JSON.lua
COPY --from=builder /app /

COPY recorder-health.sh /usr/sbin/recorder-health.sh
COPY entrypoint.sh /usr/sbin/entrypoint.sh

RUN chmod +x /usr/sbin/*.sh
RUN chmod +r /config/recorder.conf

# If you absolutely need health-checking, enable the option below.  Keep in
# mind that until https://github.com/systemd/systemd/issues/6432 is resolved,
# using the HEALTHCHECK feature will cause systemd to generate a significant
# amount of spam in the system logs.
# HEALTHCHECK CMD /usr/sbin/recorder-health.sh

EXPOSE 8083

# ENV OTR_CAFILE=/etc/ssl/cert.pem
ENV OTR_STORAGEDIR=/store
ENV OTR_TOPIC="owntracks/#"

ENTRYPOINT ["/usr/sbin/entrypoint.sh"]
```

config.mk:

```
CFLAGS += -g -DNS_ENABLE_IPV6

INSTALLDIR = /usr

# Do you want support for MQTT?
WITH_MQTT ?= yes

# Do you want recorder's built-in HTTP REST API?
WITH_HTTP ?= yes

# Do you want recorder support for shared views? Requires WITH_HTTP
WITH_TOURS ?= yes

# Do you want to use reverse-geo caching? (Highly recommended)
WITH_LMDB ?= yes

# Do you have Lua libraries installed and want the Lua hook integration?
WITH_LUA ?= yes

# Do you want support for the `pingping' monitoring feature?
WITH_PING ?= yes

# Do you want support for removing data via the API? (Dangerous)
WITH_KILL ?= no

# Do you want support for payload encryption with libsodium?
# This requires WITH_LMDB to be configured.
WITH_ENCRYPT ?= yes

# Do you want R_only support? (Probably not; this is for Hosted)
# If you set this to `yes', WITH_LMDB will be set to yes
WITH_RONLY ?= no

# Do you require support for OwnTracks Greenwich firmware?
WITH_GREENWICH ?= no

# Where should the recorder store its data? This directory must
# exist and be writeable by recorder (and readable by ocat)
STORAGEDEFAULT = /store

# Where should the recorder find its document root (HTTP)?
DOCROOT = /htdocs

# Define the precision for reverse-geo lookups. The higher
# the number, the more granular reverse-geo will be:
#
# 1	=> 5,009.4km x 4,992.6km
# 2	=> 1,252.3km x 624.1km
# 3	=> 156.5km x 156km
# 4	=> 39.1km x 19.5km
# 5	=> 4.9km x 4.9km
# 6	=> 1.2km x 609.4m
# 7	=> 152.9m x 152.4m
# 8	=> 38.2m x 19m
# 9	=> 4.8m x 4.8m
# 10	=> 1.2m x 59.5cm

GHASHPREC = 7

GEOCODE_TIMEOUT = 4000

# Should the JSON emitted by recorder be indented? (i.e. beautified)
# yes or no
JSON_INDENT ?= no

# Location of optional default configuration file
CONFIGFILE = /config/recorder.conf

# Optionally specify the path to the Mosquitto libs, include here
MOSQUITTO_CFLAGS = `$(PKG_CONFIG) --cflags libmosquitto`
MOSQUITTO_LIBS   = `$(PKG_CONFIG) --libs libmosquitto`

# Debian requires uuid-dev
# RHEL/CentOS needs libuuid-devel
MORELIBS += -luuid # -lssl


# If WITH_LUA is configured, specify compilation and linkage flags
# for Lua either manually or using pkg-config. This may require tweaking,
# and in particular could require you to add the lua+version (e.g lua-5.2)
# to both pkg-config invocations

LUA_CFLAGS = `pkg-config --cflags lua5.2`
LUA_LIBS   = `pkg-config --libs lua5.2`

SODIUM_CFLAGS = `pkg-config --cflags libsodium`
SODIUM_LIBS = `pkg-config --libs libsodium`

# Where will the recorder find the TZ data file?
TZDATADB = /usr/share/owntracks/recorder/timezone16.bin
```

- You have to add `# Where will the recorder find the TZ data file?
TZDATADB = /usr/share/owntracks/recorder/timezone16.bin` to config.mk [link](https://github.com/owntracks/docker-recorder/issues/73#issuecomment-1917752943)

### Build the Dockerfile (arm only,  remember I run my services on a RaspberryPi)

clone the owntracks docker git

`git clone https://github.com/owntracks/docker-recorder.git`

replace recorder and mosquitto config files with the ones we created:

`rm docker-recorder/recorder.conf docker-recorder/mosquitto.conf`

and

`cp config/recorder.conf docker-recorder/recorder.conf`

`mosquitto/config/mosquitto.conf docker-recorder/mosquitto.conf`

name and build the dockerfile:

`docker build -t otr-arm -f docker-recorder/Dockerfile .`

- owntracks recorder publishes x86 images on dockerhub but there are no official ARM images
- mosquitto publishes images for all aarch's

**run the compose file**

`docker compose up`

- Comment password_file on mosquitto/config/mosquitto.conf on first run then run mosquitto_passwd command as user root on the mosquitto container and finaly uncomment password_file and re-run docker compose up

**create user/pass for recorder in mosquitto**

`docker exec -it --user root mosquitto mosquitto_passwd -c /mosquitto/passwd/pass username`

- now you can run the container detached

`docker compose up -d`

**android app settings**

> Connection:

- mode mqtt

- host mqtt.example.org

- Port 8883 (open port on router)
Client ID random name

- Websockets toggle enabled

> Identification:

- Username random name (will be displayed on recorder ui)

- Password empty

- Device ID random name

- Tracker ID random name

> Security:

- TLS enabled

- select client cert under preferences>connection>security

- CA cert empty (installed the ca.crt in the device user store)

**access the webui**

- Configure caddy with reverse proxy and  mutual TLS (mtls) for the proxied service and install the client certificate on android

**Create certs**

- Request a new key and crt

`openssl req -x509 -newkey rsa:4096 -keyout cert_name.key -out cert_name.crt -days 365`

- Request a new certificate signing request

`openssl req -new -key cert_name.key -out cert_name.CSR`

- Request a new certificate authority

`openssl x509 -req -days 365 -in cert_name.csr -signkey cert_name.key -out cert_name-CA.crt`

- Create a pem certificate

`cat cert_name.crt cert_name.key > cert_name.pem`

- Create a pkcs12 certificate

`openssl pkcs12 -export -out cert_name.p12 -inkey cert_name.key -in cert_name.pem -legacy`

- On android you have to install the cert_name.p12 cert as vpn & app user certificate under settings > security > more > credentials  > install > VPN & app user cert
- copy cert_name-CA.crt and cert_name.crt to /var/lib/caddy/cert/

**Add the certificate directive to caddyfile and reverse proxy the recorder webui subdomain to the running port it is running on localhost**

`nano /etc/caddy/Caddyfile`

```
#cert directive
(fancy_name) {
  tls {
    client_auth {
      mode require_and_verify
      trusted_ca_cert_file /var/lib/caddy/cert/cert_name-CA.crt
      trusted_leaf_cert_file /var/lib/caddy/cert/cert_name.crt
    }
  }
}

#owntracks
owntracks.example.org {
import fancy_name
reverse_proxy localhost:8083
}
```

- You can view your location history visiting owntracks.example.org

**domains**

- We used two domains
One for publishing mqtt location messages from android to mosquitto (mqtt.example.org)
And one for accessing the recorder webui with our browser (owntracks.e.ample.org)
- You can use any dynamic dns provider to create your free subdomains
You'll also need a cron scrpt to regullary update that domain to point to you actually public ip

**certificates**

- We installed two certificates on the android certificate store
- the certificate authority for the mosquitto client cert ca.crt 
- the caddy client certificate cert_name.p12
- we selected the mosquitto client cert from within the owntracks app cert.p12
- we created two certificate authorities. The caddy directive "fancy_name" can be imported for other services that you reverse proxy with caddy also

**Folder structure**

```
ls -R
.:
config  docker-compose.yml  mosquitto  store

./config:
recorder.conf

./mosquitto:
config  data

./mosquitto/config:
ca.crt  mosquitto.conf  passwd  server.crt  server.key

./mosquitto/config/passwd:
pass

./mosquitto/data:

./store:
ghash  last  monitor  rec

./store/ghash:
data.mdb  lock.mdb
```

### Additional config

- reverse geocode

reverse geocoding is getting street/places names from coordinates
Owntracks supports opencage as provider

To set it up

Add your opencage api key to entrypoint.sh

```
#!/bin/sh

if ! [ -f ${OTR_STORAGEDIR}/ghash/data.mdb ]; then
    ot-recorder --initialize
fi

ot-recorder ${OTR_TOPIC}  --geokey "opencage:xxxxxxxxxx"
```


In config/recorder.conf add:

`OTR_GEOKEY = "opencage:xxxxxxccccccccccc"`

And in the recoders service in compose add:

```
environment:
            - OTR_GEOKEY = "opencage:xxxxxxxxxxxxxxxxx"
```

rebuild your Dockerfile and rerun compose file

> Owntracks is caching the results of reverse geocoding and is feeding back the last (possibly unsuccessful) cached entry instead of going out a fresh to OpenCage

So

> Use a mock lockation provider to feed device false GPS data in order to start seeing reverse geocode in action in the recorder

- Add an image to a friend/family icon

image2card.sh script:

`https://github.com/owntracks/recorder/blob/master/contrib/faces/image2card.sh`

Run:

`image2card.sh image.jpg username > mosquitto/config/user.json`

and

```
docker exec -it mosquitto mosquitto_pub  -d -h mqtt.example.org -p 8883 --cafile /mosquitto/config/ca.crt --cert /mosquitto/config/server.crt --key /mosquitto/config/server.key -t owntracks/username/device_id/info -f user.json -r
```

- Restore conflicting Frieds faces

```
docker exec -it mosquitto mosquitto_pub  -d -h mqtt.example.org -p 8883 --cafile /mosquitto/config/ca.crt --cert /mosquitto/config/server.crt --key /mosquitto/config/server.key -t owntracks/user/device_id/info -r -n
```

and

`rm -rf store/cards`

then recreate cards


**sources**

this is just a summ up with countless trial and error's from varius sources 

[one](https://owntracks.org/booklet/features/tls/) [two](https://owntracks.org/booklet/features/tlscert/) [three](http://www.steves-internet-guide.com/mosquitto-tls/) [four](https://medium.com/himinds/mqtt-broker-with-secure-tls-and-docker-compose-708a6f483c92) [five](https://www.reddit.com/r/selfhosted/comments/shvkkb/reverse_proxy_client_certificates_for_dummies/)
