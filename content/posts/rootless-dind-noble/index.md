---
title: "Running rootless Docker-in-Docker on Ubuntu Noble"
date: 2024-09-04T15:03:00.000Z
tags: ["linux","docker","dind","docker-in-docker","rootless","ubuntu","noble"]
author: "Adam"
---

## Intro

I'm running a local [Gitea](https://about.gitea.com/) instance to, amongst other things, make use of their !Github Actions and build some docker images for my own consumption. To enable this, I run their [rootless DIND actions runner container](https://hub.docker.com/r/gitea/act_runner). After upgrading my host machine from Ubuntu Jammy (22.04) to Noble (24.04) I found that the container was in a fun crash loop and couldn't immediately identify why.

## Bloody Security Improvements

In 23.10, Ubuntu helpfully [restricted unprivileged user namespaces](https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces). Rootlesskit needs unprivileged user namespaces, and indeed bundles an apparmor profile that allows them, but for some reason they weren't working for Docker-in-Docker (DIND). I could see apparmor audit events that seemed to suggest it should be working already:

```shell
audit: type=1400 audit(1725459808.334:4628): apparmor="AUDIT" operation="userns_create" class="namespace" info="Userns create - transitioning profile" profile="unconfined" pid=562953 comm="rootlesskit" requested="userns_create" target="unprivileged_userns"
```

The above article suggests you can just turn off the restriction with `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0` (and persist with a custom file in `/etc/sysctl.d`), but that's not a *good* fix because it exposes your entire host to the risk that this change was designed to mitigate.

A bit more digging turned up [this doc from Docker themselves about apparmor](https://docs.docker.com/engine/security/apparmor/), specifically this section:

> Docker automatically generates and loads a default profile for containers named `docker-default`. The Docker binary generates this profile in tmpfs and then loads it into the kernel.

So, not using the `rootlesskit` profile I had thought it was. The `docker-default` profile doesn't reside in `/etc/apparmor.d` because that would be too easy, and besides granting the permission to every container is almost as bad as doing it to the whole host. Instead, we can leverage the `apparmor` `security_opt` setting for individual containers to apply a specific apparmor profile to them:

```yaml
    security_opt:
      - apparmor=rootlesskit
```

The `rootlesskit` profile seems to be the most apt here, but if you don't have it on the host, then you can either follow the [instructions here](https://docs.docker.com/engine/security/rootless/#distribution-specific-hint) to create it yourself, or make use of the `runc` container runtime profile, which has the same unconfined flags set.

Recreate the container and voilà! Rootless DIND is yours once again, with a minimal increase in security risk for your host. Enjoy.
