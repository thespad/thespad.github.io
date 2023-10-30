---
title: "Really Simple Network Bridging With qemu"
date: 2022-09-04T11:55:00.000Z
tags: ["linux","networking","qemu","howto","bridging","systemd"]
author: "Adam"
---

## Intro
I've not really used qemu much before but I found myself needing it last week for a particular project and discovered - to my shock and amazement - that the documentation (official and 3rd party) surrounding it is almost universally terrible. A mixture of overly-complex, out of date, incredibly niche, and just straight up poorly-written.

I'll be honest, I wasn't after much, I had a single qemu VM and I wanted it to be routable on my LAN. For various reasons, PCI pass-through wasn't an option (not that the docs there are any better) but I did have a NIC that wasn't being used for anything else that I was hoping could be put to use here.

## As Simple As Possible

It took an afternoon of screwing around but I got everything I wanted working and ultimately it was extremely simple - far simpler than all the man pages and blogs I read would suggest.

First some prep, edit `/etc/qemu/bridge.conf` and add the line `allow virtbr0`. `virtbr0` is an arbitrary name for the virtual bridge you're going to use. Call it whatever you want.

Then `chmod +s /usr/lib/qemu/qemu-bridge-helper`. This allows a non-admin user to make use of the qemu bridge so we can run our VM unprivileged.

You'll also want to disable DHCP on the NIC you're going to use here - or unassign the IP if it's static - because you'll be using the bridge IP instead. You don't *have* to, but it's a waste of an address if you don't.

Now the network bridge. First install `bridge-utils` (Debian/Ubuntu), and then:

```shell
sudo brctl addbr virtbr0
sudo brctl addif virtbr0 enp3s0
sudo ip addr add 192.168.0.20/24 dev virtbr0
sudo ip link set virtbr0 up
```

In short: create a bridge called `virtbr0`, add your spare NIC to it (`enp3s0` in my case), give the bridge an IP address on your LAN, then bring up the bridge interface.

Next, configure iptables/nftables to forward traffic across the bridge network:

```shell
sudo iptables -I FORWARD -m physdev --physdev-is-bridged -j ACCEPT
```

And that's pretty much it. Configure your VM with `-net nic,model=virtio,macaddr=52:54:00:00:00:01 -net bridge,br=virtbr0` and away you go.

The MAC here is arbitrary, `52:54:00` is the qemu prefix and the rest can be whatever as long as it doesn't conflict with an existing virtual NIC on your network. The `br` should obviously match whatever you've called your network bridge.

## Persistence Is The Key

This is great but a host reboot will wipe it all out again. To persist the config with systemd, you can create a service in `/lib/systemd/system/`, I called mine `qemu-startup.service`, and do something like:

```ini
[Unit]
Description=Setup qemu network bridging
After=network-online.target

[Service]
Type=oneshot
Restart=on-failure
ExecStart=brctl addbr virtbr0
ExecStart=brctl addif virtbr0 enp3s0
ExecStart=ip addr add 192.168.0.20/24 dev virtbr0
ExecStart=ip link set virtbr0 up
ExecStart=iptables -I FORWARD -m physdev --physdev-is-bridged -j ACCEPT

[Install]
WantedBy=multi-user.target
```

You can add whichever services and targets you like to the `After` statement to control when it tries to do this, in case you've got other things you want to start first. If you're also auto-starting your VMs via systemd just add `qemu-startup.service` to *its* `After` statement so it doesn't try and start before your network config is in place.

Simple (I hope).
