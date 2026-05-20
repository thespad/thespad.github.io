---
title: "Rootlessless"
date: 2026-05-20T13:45:00.000Z
tags: ["linux","docker","dind","docker-in-docker","rootless","ubuntu"]
author: "Adam"
---

You've got to love Docker's approach to updates; 29.5.0 [changed the default rootlesskit driver](https://github.com/moby/moby/releases/tag/docker-v29.5.0) from `slirp4netns` to `gvisor-tap-vsock`, which would be fine, except they have also removed `slirp4netns` from their repos for 29.5.0 onwards, so it removes the package (or marks it for autoremoval) when you upgrade Docker. You probably won't notice this until the next time you reboot.

Now, rootlesskit falls back to `gvisor-tap-vsock` for its network driver so that's fine for an OOTB setup, but if you care about silly things like source IP propagation (so your containers don't see all IP addresses as the Docker gateway) you would have been using `slirp4netns` as the port driver (via `DOCKERD_ROOTLESS_ROOTLESSKIT_PORT_DRIVER`) as well, and guess what happens when it doesn't exist any more?

Even better, if you remove the custom port driver config so it falls back to the builtin one (because rootlesskit 3+, which Docker is now bundling, supports IP propagation with the builtin port driver), you'll find it isn't compatible with the Docker userland proxy, so it doesn't actually do IP propagation. This is mentioned in the rootlesskit documentation, but *not* in the Docker rootless documentation, unless you go to the very end of their separate [rootless troubleshooting page](https://docs.docker.com/engine/security/rootless/troubleshoot/#networking-errors).

So you disable the userland proxy (add `"userland-proxy": false` to your Docker `daemon.json`), and then the Docker service won't even start because it needs the `br_netfilter` kernel module loaded to be able to expose ports when the userland proxy is disabled. So you load that:

```bash
sudo tee /etc/modules-load.d/docker.conf <<EOF >/dev/null
br_netfilter
EOF

sudo systemctl restart systemd-modules-load.service
```

and then the Docker service will start again, except all your Docker networks are now broken and need deleting and recreating from scratch.

So you do that and then it all works again.
