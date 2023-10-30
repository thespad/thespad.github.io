---
title: Connecting to NFS Shares From Windows (Properly)
date: 2021-02-11T13:27:15.000Z
tags: ["nfs","howto","windows","linux","networking","storage","identity mapping"]
aliases: ["/connecting-to-nfs-shares-from-windows-properly"]
author: Adam
---

### Introduction

If you live in the Windows world you probably haven't had much cause to use NFS because SMB is the done thing, but if you're working with Linux hosts or NAS devices NFS can be simpler to deal with. The problem is that Windows NFS support is a bit...wonky and it doesn't help that almost all the guides on t'internet are giving out bad advice. So to continue my series of "I just figured this out so it seems only fair to share" posts, here's how to setup the NFS client on Windows properly.

### Getting Started

First things first, you need to install the NFS client support; for desktop platforms it's

```powershell
Enable-WindowsOptionalFeature -FeatureName ServicesForNFS-ClientOnly, ClientForNFS-Infrastructure -Online -NoRestart
```

and on servers

```powershell
Install-WindowsFeature NFS-Client
```

Or use the GUI, you do you. Note that right now Windows only supports NFSv3 (unless you're using WSL but then you can't mount to the host so...).

### Identity Mapping

Here's where most guides go off track; they'll tell you to setup Anonymous UID/GID in the registry and then connect to the share with the `anon` option. This is bad because it means anyone on your machine can connect with your privileges. Assuming AUTH_SYS (I'm not going to delve into Kerberos/RPCSEC_GSS here but it's broadly similar) you've got two basic options for identity mapping depending on your situation.

#### Domain Joined

If your machine is domain joined and you have access to modify AD accounts then the simplest option is to run:

```powershell
Set-NfsMappingStore -EnableADLookup $true
```

To enable AD lookups for NFS and then either use the powershell cmdlets:

```powershell
Set-NfsMappedIdentity -UserName someuser -UserIdentifier 1234 -GroupIdentifier 5678

Set-NfsMappedIdentity -GroupName somegroup -GroupIdentifier 5678
```

Or directly edit the AD objects and set the `uidnumber` and `gidnumber` attributes to the correct values.

#### Standalone Machine

If you're not on a domain or don't want to use AD mapping for some reason, then delve into C:\windows\system32\drivers\etc and create yourself `passwd` and `group` files. These are exactly what you'd expect, and serve the same purpose as their Linux counterparts. In short, for `passwd`:

```bash
[domain]\<username>:x:<UnixUID>:<UnixGID>:<Description>:C:\Users\<username>
```

and for `group`:

```bash
[domain]\<groupname>:x:<UnixGID>:<UnixUID>
```

### Mapping Drives

You can map NFS shares using the GUI like any other mapped drive, or using the `mount` command from the CLI. While the classic `host:/share/path` syntax is supported, you can also use the more Windowsy `\\host\share\path` syntax (and Windows will show the mapping in that format anyway). Using `mount` is the only way to specify additional options when connecting, but doesn't offer a means of making the mounts persistent between logins, so you'll need to rig up a login script to do that for you.

Issuing the `mount` command without any arguments will show you all your current mounts including properties such as the UID/GID. You can also see this information from the Properties dialogue of the mapped drive.

### Unmapping Drives

As with any other mapped drive you can just right click > Disconnect, or you can use `umount <drive letter>` from the CLI.

### Conclusion

Connecting to NFS shares in Windows is much easier than it appears when you first start looking at it, it's just that a lot of the information out there is wrong, or at least sub-optimal. Hopefully this post has helped. Perhaps one day we'll get native NFSv4 support in Windows.
