---
title: Get GC Status From All DCs
author: Adam
date: 2014-07-10T12:21:12+00:00
aliases: ["/wordpress/733"]
tags:
  - active directory
  - AD
  - DC
  - GC
  - microsoft
  - powershell
  - scripts

---
Quick and easy one-liner - if you need to know at a glance which DCs in a domain are GCs and which aren't:

```powershell
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers | %{"$($_.Name) : $($_.isglobalcatalog())"}
```
