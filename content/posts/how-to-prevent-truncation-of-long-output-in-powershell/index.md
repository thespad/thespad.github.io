---
title: How to Prevent Truncation of Long Output In Powershell
author: Adam
date: 2017-04-04T16:05:55+00:00
aliases: ["/angry/925"]
tags:
  - powershell
  - scripts
  - truncate
  - truncation
  - windows

---
You know how annoying it is when you return some information in Powershell that includes a list of items and the console helpfully truncates it with a ...

[![Capture.png](Capture.png)](Capture.png)

Whereas what you really want is for it to just show the whole thing like:

[![Capture-1.png](Capture-1.png)](Capture-1.png)

Well you can. The truncation is controlled by `$FormatEnumerationLimit` and if you set it to `-1` it won't truncate output at all. The default for a standard Powershell instance is `4`, the Exchange Management Shell ups this to `16` and other console files may make their own modifications.

Simple.
