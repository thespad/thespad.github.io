---
title: Proxy Authentication With Powershell
author: Adam
date: 2016-09-09T11:44:04+00:00
aliases: ["/wordpress/900"]
tags:
  - .NET
  - authentication
  - microsoft
  - powershell
  - proxies
  - proxy
  - windows

---
We all know how annoying it is working somewhere with a proxy server that requires authentication, especially as Microsoft increasingly don't support the scenario with many of their Azure-related tools. However, it is quite possible to use authenticated proxies with .NET applications including Powershell.

For the former, edit the application .config file and add

```xml
<system.net>
<defaultProxy useDefaultCredentials="true" />
</system.net>
```

And for Powershell, add the following to your scripts or $profile

```powershell
$proxyString = "http://proxy:8080"
$proxyUri = new-object System.Uri($proxyString)

[System.Net.WebRequest]::DefaultWebProxy = new-object System.Net.WebProxy ($proxyUri, $true)
[System.Net.WebRequest]::DefaultWebProxy.Credentials = [System.Net.CredentialCache]::DefaultCredentials
```
