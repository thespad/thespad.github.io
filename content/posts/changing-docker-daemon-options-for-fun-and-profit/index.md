---
title: "Changing Docker Daemon Options For Fun and Profit"
date: 2022-03-31T18:00:00.000Z
tags: ["docker","howto","daemon","network"]
aliases: ["/changing-docker-daemon-options-for-fun-and-profit"]
author: "Adam"
---

## Introduction
Did you know there are all kinds of interesting options that Docker supports but doesn't necessarily expose, or document, very well? Most of them are very simple to configure and can have substantial benefits so it's well worth investigating.

## Daemons
All of these options are configured via the Docker daemon. You can pass arguments to dockerd via the systemd service file or, preferably, use a config file, which defaults to `/etc/docker/daemon.json`. Depending on the settings changed, editing the file will require a Reload or a Restart of the daemon to take effect.

There's a full list of options available [here](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file) and some basic information about what some of them do.

Let's have a look at some of your more interesting choices.

### Metrics
Want lots of fancy Grafana graphs about your docker environment?
```json
{
  "metrics-addr" : "0.0.0.0:9323",
  "experimental" : true
}
```
Then just point Prometheus at your new endpoint.

### Live Restore
This one is neat; it lets your containers stay running when the daemon stops, which means you can now patch your Docker install without restarting all your containers.
```json
{
  "live-restore" : true
}
```
There are some caveats, however. It only supports jumping between patch releases not major versions, it doesn't work if you change certain network and runtime daemon options as part of the restart, you obviously can't control the containers without the daemon running, and it's not designed to keep containers running long-term without access to the daemon.

### Default Address Pools
Out of the box when you create a new network Docker assigns a `/16` (65,536 addresses) address block to it, which is almost always extreme overkill. You can define the address range when you create the network, but you can also change the default behaviour.
```json
{
  "default-address-pools":
  [
    {"scope":"local","base":"172.20.0.0/16","size":24}
  ]
}
```
Here we're telling Docker to use `/24` network blocks from the `172.20.0.0/16` range, which gives us 256 to work with. Obviously this won't affect any existing networks you've created. You can also use the `global` scope for swarm/overlay networks.

### Runtimes
Let's say you're bored of `runc` and you want to try out a fancy new runtime like [crun](https://github.com/containers/crun), but only for a few containers, you can do that:
```json
{
  "default-runtime": "runc",
  "runtimes": {
    "crun": {
      "path": "/usr/local/bin/crun"
    }
  }
}
```
Then just use `--runtime=crun` or `runtime: crun` when creating a container. If you use GPU acceleration with an Nvidia card you may already have made use of this.

### Default Bridge Network
Normally when you create a container using `docker run` it attaches it to the default Bridge network, but it doesn't have to:
```json
{
  "bridge": "myfancynet"
}
```
You can still use `--net=` to override the network you want to attach to.

### Control The Default Gateway
Have multiple interfaces on your host on different networks and want to control which one your containers use?
```json
{
  "default-gateway": "192.168.0.1"
}
```

### Download Options
Want your pulls and pushes to be faster? You can increase the number of concurrent down/uploads from the following defaults:
```json
{
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5
}
```
Go on, see if you can max out your connection. Probably won't help if you're running a Pi off an SD card.

### Conclusion
Don't forget that this is a JSON file and needs to be properly formatted or the daemon will fail to reload/restart. Run it through a linter if you need to to make sure you're not about to knock all your containers offline. Set all of them if you're feeling brave:
```json
{
  "metrics-addr" : "0.0.0.0:9323",
  "experimental" : true,
  "live-restore" : true,
  "default-address-pools":
  [
    {"base":"172.20.0.0/16","size":24}
  ],
  "default-runtime": "runc",
  "runtimes": {
    "crun": {
      "path": "/usr/local/bin/crun"
    }
  },
  "bridge": "myfancynet",
  "default-gateway": "192.168.0.1",
  "max-concurrent-downloads": 5,
  "max-concurrent-uploads": 10
}
```

Enjoy your newfound power.
