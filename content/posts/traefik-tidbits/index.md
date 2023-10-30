---
title: Traefik Titbits
date: 2020-10-05T22:05:15.000Z
tags: ["traefik","docker"]
author: Adam
summary: This is just a quick collection of random bits I've learned about Traefik since writing my original How To.
---

### Introduction
This is just a quick collection of random bits I've learned about Traefik since writing my original How To. The example container I'm using here is [Yacht](https://github.com/SelfhostedPro/Yacht), an up and coming Docker managment interface which I've been playing with and you might want to take a look at.
### Traefik
#### TLS Again
So this one is just me overlooking something obvious in the (bad) docs. My original How To said to use this label:
```yaml
     - traefik.http.routers.yacht-https.tls=true
```
For your https connections but you don't actually need to because you can just put it into your static config (`traefik.yml`):
```yaml
entryPoints:
  https:
    address: ":443"
    http:
      tls: {}
```
And that will set it as the default. You can still override it per container if you find yourself in the weird situation of wanting to make a non-TLS connection over port 443.
#### Services and Ports
So normally if you want to setup Traefik via Docker labels you'd do something like this:
```yaml
    labels:
     - traefik.enable=true
     - traefik.http.routers.yacht-https.rule=Host(`yacht.example.com`)
     - traefik.http.routers.yacht-https.entrypoints=https
     - traefik.http.routers.yacht-https.service=yacht-svc
     - traefik.http.services.yacht-svc.loadbalancer.server.port=8000
```
Right, but what if you do this instead:
```yaml
    labels:
     - traefik.enable=true
     - traefik.http.routers.yacht-https.rule=Host(`yacht.example.com`)
     - traefik.http.routers.yacht-https.entrypoints=https
     - traefik.http.services.yacht-https.loadbalancer.server.port=8000
```
If you name the service the same as the router, you don't have to explicitly tell Traefik to use it with the `traefik.http.routers.yacht-https.service` label.

Now if you want to go even more minimalist you could just do:
```yaml
    labels:
     - traefik.enable=true
     - traefik.http.routers.yacht-https.rule=Host(`yacht.example.com`)
     - traefik.http.routers.yacht-https.entrypoints=https
```
"But wait!" you say, how does it know which port to connect to the backend service on? Well, it uses the EXPOSE statement in the Dockerfile to determine which port to use. This isn't always a good idea, of course, because it relies on the developer a) including an EXPOSE statement and b) putting the port you want first if there's more than one. That said if it's your image and/or you know it's setup properly, it may be of use.
#### Entrypoints
When dealing with multiple entrypoints, you can simplify your labels a little by squishing everything down, rather than duplicating router config.
```yaml
      - traefik.enable=true
      - traefik.http.routers.yacht-all.rule=Host(`yacht.example.com`)
      - traefik.http.routers.yacht-all.entrypoints=https,http
      - traefik.http.routers.yacht-all.service=yacht-svc
      - traefik.http.routers.yacht-all.middlewares=middleware-https-redirect
      - traefik.http.services.yacht-svc.loadbalancer.server.port=8000
```
By listing both http and https entrypoints we treat the two the same, and as long as you've set your https entrypoint to always do TLS and don't need different middlewares, it'll all work nicely. You *can* omit the entrypoints statement entirely at which point Traefik will listen on all of them, which is fine if you've only got http/https, but if you're working with strange ports you probably don't want every service accepting connections on them.

Note that the redirect middleware is smart enough that it won't try and redirect https to https and cause a weird loop or anything like that.
### Conclusions
I keep learning new bits and pieces about Traefik that I wish I'd known upfront because they'd have really helped me understand things a lot better. They may not always be useful depending on your use case but it's always good to know this stuff for when you do need it.
