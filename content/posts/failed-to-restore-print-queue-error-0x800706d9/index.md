---
title: Failed To Restore Print Queue…Error 0x800706d9
author: Adam
date: 2012-02-22T11:54:07+00:00
aliases: ["/wordpress/386"]
tags:
  - errors
  - microsoft
  - printers
  - scripts
  - windows

---
**Situation**
You're using PrintBRM.exe to migrate printers between servers (or to backup and later restore them on the same server) but when it tries to restore the print queues it fails with "Failed to restore print queue \<Printer name\>. Error 0x800706d9"

**Cause**
PrintBRM tries to query the Windows Firewall when it creates the shares for the print queues. If the Windows Firewall service is disabled, this step fails and the print queue is not created.

**Solution**
Start the Windows Firewall service (and make sure you either have physical access to the box, iLO/DRAC access, have already configured the firewall not to protect your network interfaces by default or have put an exception in for inbound RDP traffic, otherwise starting the service will lock you out of the box) and run the PrintBRM restore again.
