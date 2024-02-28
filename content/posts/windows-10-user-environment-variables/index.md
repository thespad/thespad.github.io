---
title: Windows 10 User Environment Variables
author: Adam
date: 2017-09-06T08:19:55+00:00
aliases: ["/angry/936"]
tags:
  - environment variables
  - microsoft
  - windows 10

---
Windows 10 has made a lot of changes from previous versions, one of which is that you can no longer view System Properties as a non-admin user. This means you can no longer view/edit your user environment variables via System Properties.

There are 3 ways around this:

* Use another method such as `set` or Powershell or direct registry editing
* Go to Control Panel->User Accounts->Change My Environment Variables
* Run `"C:\WINDOWS\System32\rundll32.exe" sysdm.cpl,EditEnvironmentVariables` to envoke the Environment Variables window directly
