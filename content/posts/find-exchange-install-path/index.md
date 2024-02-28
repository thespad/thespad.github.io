---
title: Find Exchange Install Path
author: Adam
date: 2014-10-13T15:03:50+00:00
aliases: ["/angry/772"]
tags:
  - exchange 2010
  - powershell
  - scripts

---
Quick and easy; Exchange creates an environment variable called "ExchangeInstallPath" which holds the install path for Exchange on a given server, this can be accessed via Powershell using `$env:ExchangeInstallPath`.

This can be useful if you need to call elements such as RemoteExchange.ps1 but aren't sure if Exchange has been installed to the default location.
