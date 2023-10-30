---
title: ZeroSSL As A LetsEncrypt Alternative Using Traefik
date: 2020-12-20T12:06:15.000Z
tags: ["traefik","docker","certificates","letsencrypt","howto"]
aliases: ["/get-free-zerossl-certs-using-traefik"]
author: Adam
---

### Introduction
LetsEncrypt is a fantastic service and it has quite literally revolutionised how people use TLS certificates, but having a Single Point Of Failure for these things is always a bad idea. The good news is that other providers of free certificates are starting to emerge and one of the first is ZeroSSL. Unlike LetsEncrypt they don't rate limit, but they *do* require the use of External Account Binding (EAB) which means it's not quite a drop in replacement in your config.

### Getting Started
So first up EAB support is only present in Traefik 2.4, which is still in Release Candidate form as of this post, so you may want to wait a little while if stability is critical for you.

Now Traefik is not (yet, and may never be) a ZeroSSL "Partner ACME Client" which means you have to generate the EAB credentials by hand (rather than using their API) and that means you need a ZeroSSL account. Not a huge barrier to entry and it doesn't cost you anything, but worth bearing in mind.

Once you've got an account, go to the Developer section of your account management and generate some EAB credentials. Make sure you save them somewhere as they aren't stored anywhere on the site.

### Setting Everything Up
In your static config, create a new `certificateResolvers` entry using your EAB kid and hmac.
```yaml
certificatesResolvers:
  zerossl:
    acme:
      caServer: https://acme.zerossl.com/v2/DV90
      email: zerossl@example.com
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
      eab:
        kid: abc123xyz
        hmacEncoded: abc123xzy
```
And then add the resolver to one or more of your containers
```yaml
      - traefik.http.routers.yacht.tls.certresolver=zerossl
```
Recreate the container to update the labels and restart Traefik to load the new config and that's it, you're good to go.

Be aware that if you've previously set up `CAA` records in your DNS for LetsEncrypt you will also need to add records for `sectigo.com` in order for ZeroSSL to be permitted to issue certs for your domain.

### Conclusion
Even if you don't want to use ZeroSSL for any of your certs right now, having an alternative should anything untoward happen with LetsEncrypt is a sensible precaution and having everything rigged up and tested ahead of time just makes your life easier.
