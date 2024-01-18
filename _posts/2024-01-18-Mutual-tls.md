---
layout: post
title:  "Mutual TLS directive for caddy webserver"
author: ippo
categories: [ mtls, server, security ]
tags: [ mtls, server, security]
image: assets/images/mtls.jpg
---

Configure caddy with reverse proxy and  mutual TLS (mtls) for the proxied service and install the client certificate on android

Mutual TLS, or mTLS for short, is a method for mutual authentication. mTLS ensures that the parties at each end of a network connection are who they claim to be by verifying that they both have the correct private key. 
The information within their respective TLS certificates provides additional verification. 
If the client do not provide a valid cert, the rerver is not respondind to the clients requests.

I use caddy for reverse proxy on my running services.

Caddy is configured by a file without extention named Caddyfile.

Normal TLS is added by default to the reverse_proxy directive without further configuration.

For mutual TLS we'll need to add an new directive:

- Create certs:

Request a new key and crt

`openssl req -x509 -newkey rsa:4096 -keyout cert_name.key -out cert_name.crt -days 3650`

Request a new certificate signing request

`openssl req -new -key cert_name.key -out cert_name.CSR`

Request a new certificate authority

`openssl x509 -req -days 365 -in cert_name.csr -signkey cert_name.key -out cert_name-CA.CRT`

Create a pem certificate

`cat cert_name.crt cert_name.key > cert_name.pem`

Create a pkcs12 certificate

`openssl pkcs12 -export -out cert_name.p12 -inkey cert_name.key -in cert_name.pem`

A the mule directive at the beginning of the Caddyfile

Caddyfile:

```
(fancy_name) {
  tls {
    client_auth {
      mode require_and_verify
      trusted_ca_cert_file /cert_path/cert_name-CA.crt
      trusted_leaf_cert_file /cert_path/cert_name.crt
    }
  }
}
```

Now you can import client cert to any proxied service

Service directive with mtls,  reverse proxy and https
```
your_url.com {
  import fancy_name
  reverse_proxy localhost:port
}
```

> On android you have to install the cert_name.p12 cert as vpn & app user certificate under settings > security > more > credentials  > install > VPN & app user cert
