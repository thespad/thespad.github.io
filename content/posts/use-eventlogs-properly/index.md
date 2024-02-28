---
title: Use Eventlogs Properly!
author: Adam
date: 2010-11-08T10:02:24+00:00
aliases: ["/wordpress/18"]
tags:
  - errors
  - logging
  - microsoft
  - rant
  - windows

---
Windows has a centralised logging facility for applications; the Windows Event Log. If you're writing applications for Windows then for the love of God please use it properly.

**DO** [create your own event message DLL(s)][1] where appropriate to avoid your events [looking like this][2]

**DO** log important errors and warnings. Application failures, communication issues, invalid configuration data and the like. Things that will help administrators to troubleshoot issues that may occur.

**DO** make your logs intelligible to someone other than you. Not having developed the application myself, I have no way of knowing if "Invalid foo in bar. More cheese needed at 0x8003387" means that someone's made a typo in a config file somewhere, a firewall rule needs changing or that the application doesn't support running during the vernal equinox.

**DO** throttle your logging. Don't log the same error every second, it's pointless, generates a lot of "noise" and - much worse - forces other, potentially useful events out of the log's retention.

**DO** make your logging level easily configurable by the user and **DO** set a sensible default.

**DON'T** log every single informational or debug event that your application generates. Nobody gives a shit that you successfully checked a message queue and found it was empty; either use a [Custom Event Log][3] or a log file in the application directory if you want to record that kind of information.

 [1]: http://www.eventlogblog.com/blog/2010/11/creating-your-very-own-event-m.html
 [2]: http://www.eventlogblog.com/blog/2008/04/event-log-message-files-the-de.html
 [3]: http://www.jasonsamuel.com/2010/01/08/creating-a-custom-event-log-under-event-viewer-to-log-server-events/
