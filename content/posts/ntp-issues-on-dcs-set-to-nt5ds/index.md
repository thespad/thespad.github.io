---
title: NTP issues on DCs set to NT5DS
author: Adam
date: 2012-01-24T15:41:05+00:00
aliases: ["/angry/349"]
tags:
  - microsoft
  - NT5DS
  - NTP
  - w32tm
  - windows

---
I ran into an issue this morning with a pair of Windows 2003 Server DCs that were set to sync their clocks using NT5DS (which is the default and means they should sync from the domain hierarchy, which for DCs is often the PDC emulator). They kept logging the following error in the System Log:

> The time provider NtpClient was unable to find a domain controller to use as a time source. NtpClient will try again in 15 minutes.

After much prodding, swearing and Googling, it became apparent that with 2003 if a DC has ever held the PDC Emulator role then it will still think it is the authoritative time source for the domain when that role is moved off it. This meant that we had 3 DCs all thinking that they were the One True Time Source and all being out of sync with each other by 2 or 3 minutes.

This issue can be resolved by running the following command on the former PDC Emulator(s): `w32tm /config /syncfromflags:domhier /reliable:no /update` which will tell the DC that it is no longer a reliable time source and so it should check for updates from a source that is (i.e. the PDC). You can speed things up a bit by issuing a `w32tm /resync` command to force the Windows Time service to update.

You can use `w32tm /stripchart /computer:<PDC Name>` to see in real-time how the local clock differs from the time on the PDC Emulator (The o: value) as the time service attempts to bring it back into sync.
