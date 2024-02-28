---
title: Disconnect RDP Console Session Without Locking Server
author: Adam
date: 2013-07-03T10:51:58+00:00
aliases: ["/wordpress/585"]
tags:
  - console
  - RDP
  - remote desktop
  - server
  - windows

---
Sometimes you have badly written apps that need to run interactively on servers, which means you have to connect to the console session to manage them over RDP. If you need to be able to leave the console session unlocked it causes issues as disconnecting an RDP session will lock it by default.

This can be changed by running the following command from within the RDP session:
`tscon 0 /dest:console`

Where "0" is the ID of the session you're connected to - on 2K3 boxes this will be 0 if you're connected to the console, but on 2K8+ it will vary depending on how many other users are connected. You can find out the session ID by running `query session`.
