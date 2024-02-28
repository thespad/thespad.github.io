---
title: Decode UserAccountControl Values With Powershell
author: Adam
date: 2012-01-25T11:03:48+00:00
aliases: ["/wordpress/358"]
tags:
  - active directory
  - microsoft
  - powershell
  - scripts
  - windows

---
One of the most annoying things when working with Powershell and AD accounts is the UserAccountControl value. This value is what determines settings such as whether or not the account is locked out, disabled, requires a smartcard for authentication, uses reversible encryption for its password, etc. The default is 512 (NORMAL_ACCOUNT) but there are all kinds of weird and wonderful combinations that can turn up depending on how the account is configured and when you're trying to (for example) find all the accounts that are set to USE_DES_KEY_ONLY then having so many different possible values (any number that could have 2097152 as part of its makeup) makes it a pain to work out.

Thankfully, someone has solved this problem in a relatively simple manner; I first came across it here: [http://bsonposh.com/archives/288](http://bsonposh.com/archives/288) and it is listed in all its glory below. It takes a single argument, which is the UAC value for an account and will return a list of all the flags that apply to that value. So, for example, plugging in a value of 66048 will give an output of:

```text
NORMAL_ACCOUNT
DONT_EXPIRE_PASSWORD
```

```powershell
param
([int]$value)
$flags = @("","ACCOUNTDISABLE","", "HOMEDIR_REQUIRED",
"LOCKOUT", "PASSWD_NOTREQD","PASSWD_CANT_CHANGE", "ENCRYPTED_TEXT_PWD_ALLOWED",
"TEMP_DUPLICATE_ACCOUNT", "NORMAL_ACCOUNT", "","INTERDOMAIN_TRUST_ACCOUNT", "WORKSTATION_TRUST_ACCOUNT",
"SERVER_TRUST_ACCOUNT", "", "", "DONT_EXPIRE_PASSWORD", "MNS_LOGON_ACCOUNT", "SMARTCARD_REQUIRED",
"TRUSTED_FOR_DELEGATION", "NOT_DELEGATED","USE_DES_KEY_ONLY", "DONT_REQ_PREAUTH",
"PASSWORD_EXPIRED", "TRUSTED_TO_AUTH_FOR_DELEGATION")
1..($flags.length) | ? {$value -band [math]::Pow(2,$_)} | % { $flags[$_] }
```
