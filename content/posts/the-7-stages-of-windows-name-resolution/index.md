---
title: The 7 Stages of Windows Name Resolution
author: Adam
date: 2012-02-15T15:33:17+00:00
aliases: ["/angry/365"]
tags:
  - DNS
  - name resolution
  - networking
  - windows

---
You'd be surprised how many people don't know this and how rarely that matters (OK, probably not that surprised). This is the process Windows uses to resolve names to IP addresses, it can be useful when troubleshooting name resolution issues or in the few edge cases where you need the same hostname to resolve to different addresses depending on where you are:

  1. Windows checks whether the host name is the same as the local host name.
  2. DNS cache (This includes the Hosts file)
  3. DNS servers (If configured)
  4. NetBIOS cache
  5. WINS servers (If configured)
  6. NetBIOS broadcast (If enabled)
  7. Lmhosts file (If present)

Note: If the host name is 16 characters or longer or an FQDN, Windows does not try to resolve the host name using NetBIOS/WINS/LmHosts.
