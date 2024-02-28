---
title: Am I an Administrator?
author: Adam
date: 2012-02-16T16:24:21+00:00
aliases: ["/wordpress/380"]
tags:
  - powershell
  - scripts
  - admin

---
Powershell. Returns True or False. Simple.

```powershell
(New-Object System.Security.Principal.WindowsPrincipal([System.Security.Principal.WindowsIdentity]::GetCurrent())).IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator)
```
