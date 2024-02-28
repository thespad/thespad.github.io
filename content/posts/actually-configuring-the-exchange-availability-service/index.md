---
title: Actually Configuring The Exchange Availability Service In A Cross-Forest Environment
author: Adam
date: 2017-06-09T12:02:55+00:00
aliases: ["/wordpress/930"]
tags:
  - autodiscover
  - availability service
  - exchange
  - free/busy
  - microsoft
  - rant
  - troubleshooting

---
If you've ever looked at configuring the Exchange Availability Service to allow cross-forest free/busy lookups you've probably realised that the documentation surrounding it is _awful_.

```powershell
Get-MailboxServer | Add-ADPermission -Accessrights Extendedright -Extendedrights "ms-Exch-EPI-Token-Serialization" -User "<Remote Forest Domain>\Exchange servers"
```

From [here][1], doesn't even work for a start, because Get-MailboxServer doesn't return the correct identity objects for Add-ADPermission. Once you've worked out how to get that sorted and done your

```powershell
Add-AvailabilityAddressSpace -Forestname ContosoForest.com -AccessMethod PerUserFB -UseServiceAccount:$true
```

You're probably thinking that you're done, but it usually isn't that simple. For a start if the namespaces in either forest are even a little bit complicated or you're using a custom target address space then you're in for some autodiscover fun - just because autodiscover works in its home forest doesn't mean it will from your trusted partner. Before you even start, make sure your Exchange certificates are trusted by the target servers; don't assume that just because they're from a public CA that they will be, you never know what weird stuff has been done to the servers if you didn't build them and they're not under your control. Obviously if you can you should export the SCP to the partner forest with

```powershell
Export-AutodiscoverConfig -TargetForestDomainController "dc.contoso.com" -TargetForestCredential (Get-Credential) -MultipleExchangeDeployments $true
```

But even then there are a few things that aren't clearly documented and might trip you up; for example, [you need Outlook Anywhere enabled][2] for the Availability Service to function, which isn't a given if you're still running Exchange 2010 or 2007 in one or both forests. Furthermore, if one or both parties are running Exchange 2013 or 2016 and you're _not_ using the SCP for autodiscover then you'll probably find the free/busy lookups fail [because][3]:

> the availability service sends an Autodiscover request by using an automatically generated SMTP address for the anchor mailbox. This SMTP address that is used is 01B62C6D-4324-448f-9884-5FEC6D18A7E2@**Availability_Address_Space_domain**.
>
> However, the Exchange Server 2013 Client Access server in the attendee forest cannot locate a mailbox for this email address and responds with a 404 status.

That's right, Exchange uses an essentially random (and as far as I can tell only documented in that KB article) SMTP address for it's autodiscover query which is rejected by the target server because, obviously, it doesn't exist. The &#8220;fix"? Slap that SMTP address onto any old mailbox so the server returns a valid autodiscover response.

Hopefully this post will be of some help to anyone struggling to get the availability service working in their environment, I spent 2 weeks dicking about with Microsoft support trying to understand how it operates so that with any luck you won't have to.

 [1]: https://technet.microsoft.com/en-us/library/bb125182(v=exchg.150).aspx
 [2]: https://support.microsoft.com/en-gb/help/2734791/cross-forest-or-hybrid-free-busy-availability-lookups-fail-in-exchange-server
 [3]: https://support.microsoft.com/en-gb/help/3010570/cross-forest-free-busy-lookup-fails-when-target-forest-is-exchange-server-2013-or-exchange-server-2016