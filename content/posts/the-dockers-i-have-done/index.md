---
title: The Dockers I Have Done
date: 2021-09-07T10:56:15.000Z
lastmod: 2024-03-13T13:23:00.000Z
tags: ["LSIO","docker","containers","linux","linuxsever.io"]
aliases: ["/the-dockers-ive-done"]
author: Adam
---

As you may be aware, I'm part of [linuxserver.io](https://linuxserver.io/) where I maintain a number of Docker containers such as [grav](https://github.com/linuxserver/docker-grav/) and [syslog-ng](https://github.com/linuxserver/docker-syslog-ng) but there are times when I need a container that isn't a suitable linuxserver candidate for any number of reasons so I just publish it myself. It occurred to me that I should probably make an effort to promote them given how difficult docker discovery is on places like Docker Hub where there are hundreds of containers for any given thing, almost all of which had one image push 3 years ago and haven't been touched since. So, without further ado...

### [apcupsd_exporter](https://github.com/thespad/docker-apcupsd_exporter)

`ghcr.io/thespad/apcupsd_exporter`

[apcupsd_exporter](https://github.com/mdlayher/apcupsd_exporter) provides a Prometheus exporter for the apcupsd Network Information Server (NIS).

### [arr-in-one](https://github.com/thespad/docker-arr-in-one)

`ghcr.io/thespad/arr-in-one`

arr-in-one is a really dumb proof of concept that bundles the nightly branch builds (develop for Sonarr as it lacks a nightly branch) of all of the *arr applications into a single container, built daily.

### [dive](https://github.com/thespad/docker-dive)

`ghcr.io/thespad/dive`

[dive](https://github.com/wagoodman/dive) is a tool for exploring a docker image, layer contents, and discovering ways to shrink the size of your Docker/OCI image. These are armhf and aarch64 ARM builds because the official builds are amd64 only.

### [get_iplayer](https://github.com/thespad/docker-get_iplayer)

`ghcr.io/thespad/get_iplayer`

[get_iplayer](https://github.com/get-iplayer/get_iplayer/) is a BBC iPlayer/BBC Sounds Indexing Tool and PVR. My container includes [SonarrAutoImport](https://github.com/Webreaper/SonarrAutoImport/) for automation.

### [docker-kopia-server](https://github.com/thespad/docker-kopia-server)

`ghcr.io/thespad/kopia-server`

[Kopia](https://github.com/kopia/kopia) is a fast and secure open-source backup/restore tool that allows you to create encrypted snapshots of your data and save the snapshots to remote or cloud storage of your choice, to network-attached storage or server, or locally on your machine. I built my image to add the docker CLI for use with automating pre/post backup processes with other containers.

### [planka](https://github.com/thespad/docker-planka)

`ghcr.io/thespad/planka`

[Planka](https://github.com/plankanban/planka/) is an elegant open source project tracking tool.

### [py-kms](https://github.com/TheSpad/docker-py-kms)

`ghcr.io/thespad/py-kms`

[py-kms](https://github.com/SystemRage/py-kms) is a port of node-kms created by cyrozap, and is derived from the reverse-engineered code of Microsoft's official KMS. My builds simply add support for Windows Server 2022 as the official builds are currently broken due to the discontinuation of Docker Hub's free image build service.

### [whisparr](https://github.com/thespad/docker-whisparr)

`ghcr.io/thespad/whisparr`

[whisparr](https://github.com/whisparr/whisparr) is an adult movie collection manager for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new movies and will interface with clients and indexers to grab, sort, and rename them.

All of my containers support amd64 and aarch64 and are published to both Docker Hub and GHCR. Hopefully you'll find something useful in there.
