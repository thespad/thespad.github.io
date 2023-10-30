---
title: Wireguard as a VPN client in Docker using PIA
date: 2020-09-26T21:30:15.000Z
tags: ["wireguard","docker","vpn","howto","PIA","LSIO"]
aliases: ["/wireguard-as-a-vpn-client-in-docker-using-pia"]
author: Adam
---

### Update
Since posting this the scripts have changed slightly so the line numbers are no longer correct, that said the functional elements are still the same so it shouldn't be too hard to figure out where to make the changes. Also `get_region_and_token.sh` is now `get_token.sh` and `get_region.sh` so you'll need to run the two of them in your init script (`get_token.sh` first).

### Introduction
Compared to a lot of VPN providers PIA have been pretty slow off the mark in supporting DIY Wireguard connections; they've had Wireguard support in their client for a while but that doesn't help if you want to use something like the `linuxserver/wireguard` container as your client. However, as of last week they have published a [Github repo](https://github.com/pia-foss/manual-connections) with scripts and instructions for rigging things up by hand.

It's still not really designed for the docker use-case, however, so I spent the afternoon playing around to get it working the way I wanted and I thought I'd share in case it helps anyone.

### Wireguard
First up we need a client container; that's the easy part
```yaml
version: "2.4"
services:
  wireguard_client:
    image: linuxserver/wireguard:latest
    container_name: wireguard_client
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - PIA_USER=${PIA_USER}
      - PIA_PASS=${PIA_PASS}
      - VPN_PROTOCOL=wireguard
    volumes:
      - ./data:/config
      - /lib/modules:/lib/modules
    restart: always
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=1
      - net.ipv6.conf.default.disable_ipv6=1
```
Simple. We also need an `.env` file (or docker secrets) for our login details. We'll leave it down for the moment.

#### PIA

> You need a dummy wg0.conf to get started otherwise the Wireguard container won't get to the point of executing the PIA token/conf scripts.

We can grab the `get_region_and_token.sh` script and use it more or less as-is. You might want to make it a bit less "noisy" as you're going to be running it headless but it won't hurt to have that extra information while we're playing around.

We'll also need `ca.rsa.4096.crt` so the container trusts the endpoint, and `connect_to_wireguard_with_token.sh` which we'll modify a bit later. Copy them all into your `/config` folder and make the scripts executable with `chmod +x <filename>`.

At this point if you want to test out the basics fire up the container, exec in and run `./connect_to_wireguard_with_token.sh` from the `/config` directory. It should output information about the best endpoint to connect to and an auth token to use for generating your client config.

Now we need to modify the connect script to do our bidding. We want to remove everything after line 112, as that's when it starts trying to bring the connection up and that's something the container will handle for us. Then we need to change where it's dumping the config to
```shell
" > /etc/wireguard/pia.conf || exit 1
```
Becomes
```shell
" > /config/wg0.conf || exit 1
```
At this point we could just feed the output of the first script into this one, generate the .conf and be done with it, but we want to be dynamic and exciting and make sure we're not reusing a dead endpoint on container start.

First we're going to add an extra environment variable to our compose
```yaml
    environment:
      - TZ=Europe/London
      - PIA_USER=${PIA_USER}
      - PIA_PASS=${PIA_PASS}
      - VPN_PROTOCOL=wireguard
      - AUTOCONNECT=true
```
This tells the get script to try and launch the connect script when it finishes. Then we set everything to run on startup, thankfully Linuxserver containers have an inbuilt mechanism to achieve it. Create a `custom-cont-init.d` directory in your `/config` folder and in it create a new file, I called mine `00-setup-wireguard` but it doesn't matter hugely here. The contents are very simple
```shell
#!/bin/bash

cd /config
./get_region_and_token.sh
```
Then make it executable with `chmod +x 00-setup-wireguard`. That's it. Now when the container starts it will run `get_region_and_token.sh` which will in turn run our modified `connect_to_wireguard_with_token.sh` and generate a wg0.conf. Then the container will start its services and establish a connection for us.

#### Extending The VPN Network
A VPN connection is great, but it's not much use if you don't have anything to send down it. Let's add a qBittorrent container to our compose file and seed some Linux ISOs.
```yaml
  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - WEBUI_PORT=8080
    volumes:
      - ./data:/config
      - /mnt/downloads:/downloads
    restart: always
    network_mode: service:wireguard_client
```
Points of interest include
```yaml
    network_mode: service:wireguard_client
```
This causes the qBittorrent container to use the `wireguard_client` container's network. Note that this behaves as if all services are running on the same host, so you need to watch out for things like port conflicts.

We don't want the qBittorrent container running if Wireguard isn't, but sharing an interface with `network_mode: service:` requires the owner of that interface to be running before the qBittorrent one can be started. Note that this *doesn't* require the Wireguard connection to be up and running, just the container, but we'll get to that.

Same as with Wireguard, we're going to create a `custom-cont-init.d` directory for qBittorrent and add a script to its startup. Unfortunately PIA don't provide a nice "Am I connected" test endpoint like Mullvad so we need to get creative. There are a few different options depending on your situation; the easiest is if you've got a domain or dynamic DNS service pointing at your WAN IP.
```shell
#!/bin/bash

while [ $(curl -s https://icanhazip.com/) = $(curl -v --silent https://example.com/ 2>&1 | grep Connected | tr "()" " " | awk '{print $5}') ]
do
  echo "VPN not up, waiting a bit"
  sleep 5
done
```
This looks horrendous, but that's only because we're working within the limits of the tools available inside the container. All this does is get the public IP address of the container (via `icanhazip.com`) and compares it to your WAN IP address. If the IPs are the same it waits 5 seconds and tries again, once they're different it means the VPN is up and it allows the container to continue starting.

If you have a static IP (or don't have any way to dynamically query it) you can always hard-code things.
```shell
#!/bin/bash

while [ $(curl -s https://icanhazip.com/) = 127.0.0.1 ]
do
  echo "VPN not up, waiting a bit"
  sleep 5
done
```
Don't forget to make the script executable.
#### But Wait...
Cool, that's everything sorted then, right? Not quite. You may have noticed that while everything is working nicely you can't actually connect to the WebUI for qBittorrent. This is because by default Wireguard routes all traffic out the VPN interface and blocks anything from leaking to/from the LAN interface.


Wireguard uses IPTables to control where traffic can flow and supports modifying those rules as part of your connection config. We need to create PostUp and PreDown rules to allow us to connect to the containers from our LAN. These are general purpose examples so you're going to have to adapt them for your use.
```shell
PostUp = DROUTE=$(ip route | grep default | awk '{print $3}'); HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route add $HOMENET3 via $DROUTE;ip route add $HOMENET2 via $DROUTE; ip route add $HOMENET via $DROUTE;iptables -I OUTPUT -d $HOMENET -j ACCEPT;iptables -A OUTPUT -d $HOMENET2 -j ACCEPT; iptables -A OUTPUT -d $HOMENET3 -j ACCEPT;  iptables -A OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
```
```shell
PreDown = HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route delete $HOMENET; ip route delete $HOMENET2; ip route delete $HOMENET3; iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT; iptables -D OUTPUT -d $HOMENET -j ACCEPT; iptables -D OUTPUT -d $HOMENET2 -j ACCEPT; iptables -D OUTPUT -d $HOMENET3 -j ACCEPT
```
If, for exmple, your VPN provider hands out addresses in the 10.32.157.0/24 range to clients then you don't want to be trying to route `10.0.0.0/8` to your LAN as it'll break things rather badly. If you want to make it easier to read, just insert a line break at every `;` but note that for the Wireguard config it needs to all be on a single line.

Normally these would just go into the `[Interface]` section of the wg0.conf but because we're regenerating ours on container startup we need to get the PostUp/PreDown rules added in there too.

So back to `connect_to_wireguard_with_token.sh` and add them into the conf generation section under `[Interface]` *but* you need to escape all the `$` signs with a `\` otherwise it'll try and evalute them in the script, rather than at connect-time. So here's what it'd look like with the examples above.
```shell
echo -n "Trying to write /config/wg0.conf... "
echo "
[Interface]
Address = $(echo "$wireguard_json" | jq -r '.peer_ip')
PrivateKey = $privKey
DNS = $(echo "$wireguard_json" | jq -r '.dns_servers[0]')
PostUp = DROUTE=\$(ip route | grep default | awk '{print \$3}'); HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route add \$HOMENET3 via \$DROUTE;ip route add \$HOMENET2 via \$DROUTE; ip route add \$HOMENET via \$DROUTE;iptables -I OUTPUT -d \$HOMENET -j ACCEPT;iptables -A OUTPUT -d \$HOMENET2 -j ACCEPT; iptables -A OUTPUT -d \$HOMENET3 -j ACCEPT;  iptables -A OUTPUT ! -o %i -m mark ! --mark \$(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route delete \$HOMENET; ip route delete \$HOMENET2; ip route delete \$HOMENET3; iptables -D OUTPUT ! -o %i -m mark ! --mark \$(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT; iptables -D OUTPUT -d \$HOMENET -j ACCEPT; iptables -D OUTPUT -d \$HOMENET2 -j ACCEPT; iptables -D OUTPUT -d \$HOMENET3 -j ACCEPT
[Peer]
PublicKey = $(echo "$wireguard_json" | jq -r '.server_key')
AllowedIPs = 0.0.0.0/0
Endpoint = ${WG_SERVER_IP}:$(echo "$wireguard_json" | jq -r '.server_port')
" > /config/wg0.conf || exit 1
echo OK!
```
That's it, you should now be able to get to the qBittorrent WebUI from your LAN while the VPN is up. [Pick one](https://torrent.ubuntu.com/tracker_index) and get cracking.
#### Going Further
At this point you can add other containers to the VPN service network as well. You'll probably want to give them similar startup checks to make sure the VPN is running and maybe think about ongoing monitoring so you know if the connection goes down. qBittorrent lets you bind to a specific interface, so you can protect against it leaking traffic out from your public address but not all apps will behave the same way.

Don't forget that this method means that all containers are effectively sharing the network interface of the Wireguard container so you need to use unique ports and if you're tring to connect between containers, use `localhost` rather than the container name.
### Conclusions
It's still very much a fiddly experience and the lack of a nice curl-able endpoint from PIA to check your connection status is a pain, but at least it's now possible to run Wireguard manually with their service and that means it's possible to do it with Docker. In the end, isn't that what we all want?
