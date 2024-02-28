---
title: A Failure of Comprehension
author: Adam
date: 2010-11-23T21:29:58+00:00
aliases: ["/angry/27"]
tags:
  - group policy
  - helpdesk
  - rant
  - shares
  - support
  - troubleshooting
  - windows

---
There are three things that almost everyone that I meet in IT seems incapable of understanding; Share Permissions vs NTFS Permissions, NTFS Full Control vs Modify permissions and Group Policy vs Local Permissions.

For those who don't know, Windows folders presented over a network via CIFS share have two levels of permissions: **Share Permissions**, which are mostly a lingering reminder of the pre-NTFS days, when they were only way to control access to network resources and are pretty basic with only Read, Change & Full Control available to you. Then there are NTFS permissions, which are the normal Windows file system permissions, and have a dizzying array of permission settings available, but usually come down to List, Read, Modify & Full Control.

Best practice is to either leave the share permissions as the default (Everyone: Read) or, if some or all users need to modify files in the share, change them to Everyone: Change. That's it. Everything else should be handled through NTFS permissions (or other local file system permissions where available) because they're more granular, can be used to set different permissions on files and subfolders of a share (unlike share permissions which apply to the share as a whole) and, importantly, apply equally to users who log onto the machine locally as well as those connecting via the share. Yet somehow, people are obsessed with screwing around with Share permissions, adding random users or groups to them when they've already set Everyone: Change or leaving them as Everyone: Read and then complaining that users can't write to the share, but can write to the same location locally on the server if they connect via RDP or console.

Seguing nicely into **NTFS permissions** we find the inability of people to understand the difference between Modify and Full Control. Ever deal with a 3rd party when troubleshooting a permissions issue with one of their apps and they will almost always tell you "The users need Full Control on that folder/file, not just Modify" but will almost never be able to tell you why, or what the difference is. I'll tell you what the difference is; "Change Permissions" and "Delete subfolders and files". They are the only two things that Full Control gives you over Modify, namely the ability to alter the permissions on the object and to [delete subfolders and files, even if the Delete permission has not been granted on the subfolder or file][1]. In other words, for 99% of use cases, no difference whatsoever.

Finally, **Group Policy vs Local Permissions**, not NTFS permissions mind, but user account permissions. Specifically, this refers to the way that people are unable to grasp that having local Administrator rights on a machine does not magically allow you to do things that are restricted by Group Policy. If your user account or PC has a GPO applied to it that prevents access to the Control Panel, then adding your account to the local Administrators groups isn't going to make the slightest bit of difference to your ability to access said Control Panel. What it will do, of course, is allow you to work around Group Policy settings in certain circumstances by modifying registry permissions so that the system can't apply the GPOs on startup/login, but most people aren't aware that it's even possible, let alone how to go about it so my point remains valid.

If I could only make people understand these three things, I'd probably eliminate 20% of the support issues I have to deal with on a daily basis. Oh well, a man can dream...

 [1]: http://technet.microsoft.com/en-us/library/cc787794(WS.10).aspx
