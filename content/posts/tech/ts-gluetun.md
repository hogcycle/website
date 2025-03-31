---
date: '2025-03-06T20:18:57-05:00'
draft: true
title: 'Commercial VPN exit nodes within tailnet'
---

So far my "new" [remote access solution]("https://gysli.ng/posts/tech/ts-caddy") has been pretty solid. Other than hiccups with Eduroam wireless and an idiot admin who didn't set the container to start on boot leading to an outage after a power loss, it's been pretty solid. Expanding upon this, I would like to add an exit node of an appliance VPN while still retaining in-house DNS and subnet routing. This implementation is done in Docker as the routing is all done in Docker's tables and it makes things a little easier.

# TODO LOCAL DNS DOESN'T WORK??? WHY???
```
services:
  gluetun-tailscale:
      image: tailscale/tailscale
      container_name: gluetun-tailscale
      hostname: gluetun
      volumes:
        - ./tailscale:/var/lib/tailscale
        - /dev/net/tun:/dev/net/tun
      cap_add:
        - NET_ADMIN
        - SYS_MODULE
      ports:
        - 8080:8080/tcp 
        - 8081:8081/tcp 
      environment:
        - TS_AUTH_KEY=<key>
        - TS_EXTRA_ARGS=--advertise-exit-node
        - TS_STATE_DIR=./tailscale
  gluetun:
      image: qmcgaw/gluetun:v3
      container_name: gluetun
      network_mode: service:gluetun-tailscale
      cap_add:
        - NET_ADMIN
      devices:
        - /dev/net/tun:/dev/net/tun
      environment:
        - VPN_SERVICE_PROVIDER=airvpn
        - VPN_TYPE=wireguard
        - WIREGUARD_ADDRESSES=10.160.220.238/32
        - WIREGUARD_PRIVATE_KEY=<key>
        - WIREGUARD_PRESHARED_KEY=<key>
        - SERVER_COUNTRIES="United States"
      depends_on:
        - gluetun-tailscale
      restart: unless-stopped
```




