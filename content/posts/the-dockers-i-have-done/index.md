---
title: The Dockers I Have Done
date: 2021-09-07T10:56:15.000Z
tags: ["LSIO","docker","containers","linux","linuxsever.io"]
aliases: ["/the-dockers-ive-done"]
author: Adam
---

As you may be aware, I'm part of [linuxserver.io](https://linuxserver.io/) where I maintain a number of Docker containers such as [grav](https://github.com/linuxserver/docker-grav/) and [syslog-ng](https://github.com/linuxserver/docker-syslog-ng) but there are times when I need a container that isn't a suitable linuxserver candidate for any number of reasons so I just publish it myself. It occurred to me that I should probably make an effort to promote them given how difficult docker discovery is on places like Docker Hub where there are hundreds of containers for any given thing, almost all of which had one image push 3 years ago and haven't been touched since. So, without further ado...

### [changedetection.io](https://github.com/TheSpad/docker-changedetection.io)
`ghcr.io/thespad/changedetection.io`
[changedetection.io](https://github.com/dgtlmoon/changedetection.io) is a tool for self-hosted change monitoring of web pages. I made this one for size reasons, the official image is just shy of 500Mb, mine is 150Mb.
### [dive](https://github.com/TheSpad/docker-dive)
`ghcr.io/thespad/dive`
[dive](https://github.com/wagoodman/dive) is a tool for exploring a docker image, layer contents, and discovering ways to shrink the size of your Docker/OCI image. These are armhf and aarch64 ARM builds because the official builds are amd64 only.
### [get_iplayer](https://github.com/TheSpad/docker-get_iplayer)
`ghcr.io/thespad/get_iplayer`
[get_iplayer](https://github.com/get-iplayer/get_iplayer/) is a BBC iPlayer/BBC Sounds Indexing Tool and PVR. My container includes [SonarrAutoImport](https://github.com/Webreaper/SonarrAutoImport/) for automation.
### [Matomo](https://github.com/TheSpad/docker-matomo)
`ghcr.io/thespad/matomo`
[Matomo](https://github.com/matomo-org/matomo/) is a powerful web analytics platform that gives you 100% data ownership. It requires an external MariaDB/MySQL database (why not use the [linuxserver container](https://github.com/linuxserver/docker-mariadb)?). I made this one because the official container options are either Apache-based (and over 500Mb) or fpm-only (and therefore require another webserver to front it anyway and still somehow over 500Mb). Mine is under 150Mb and includes nginx.
### [Monit](https://github.com/TheSpad/docker-monit)
`ghcr.io/thespad/monit`
[Monit](https://mmonit.com/monit/) is a small Open Source utility for managing and monitoring Unix systems. My container includes [Apprise](https://github.com/caronc/apprise) for alerting. I originally built it because there weren't any good Monit containers available for ARM.
### [py-kms](https://github.com/TheSpad/docker-py-kms)
`ghcr.io/thespad/py-kms`
[py-kms](https://github.com/SystemRage/py-kms) is a port of node-kms created by cyrozap, and is derived from the reverse-engineered code of Microsoft's official KMS. My builds simply add support for Windows Server 2022 as the official builds are currently broken due to the discontinuation of Docker Hub's free image build service.

All of my containers support amd64, armhf, and aarch64 (Except dive which doesn't have amd64 images) and are published to both Docker Hub and GHCR. Hopefully you'll find something useful in there.
