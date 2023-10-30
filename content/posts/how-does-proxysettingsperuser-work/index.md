---
title: How Exactly Does ProxySettingsPerUser Work?
date: 2021-04-14T11:02:15.000Z
tags: ["windows","powershell","proxy","proxysettingsperuser","DefaultConnectionSettings","PAC"]
aliases: ["/how-does-proxysettingsperuser-work"]
author: Adam
---

### Update

A nice man at Microsoft advised me that in addition to setting DefaultConnectionSettings you should also set SavedLegacySettings to the same value under the same key. This value holds, and I quote, "configuration used by network connections other than the default connection". I have been unable to divine exactly what this means but it sounds like it might be important.

### Introduction

You may occasionally in your career have had a want or need to set Windows proxy settings for every account on a machine regardless of who is logged in. There are a few common ways to do this.

* Group Policy Loopback
* netsh winhttp set proxy
* Horrendous fudging with scripts

The problem with loopback policy is that it doesn't affect local accounts and the problem with winhttp proxy settings is they don't affect anything that uses wininet, which is most user-facing applications.

There is another option, ProxySettingsPerUser, which is also known as "Make proxy settings per-machine (rather than per-user)" in Group Policy. This setting is reasonably clear and if you enable it (or set the registry value to 0) then it will

> Applies proxy settings to all users of the same computer.
If you enable this policy, users cannot set user-specific proxy settings. They must use the zones created for all users of the computer.
This policy is intended to ensure that proxy settings apply uniformly to the same computer and do not vary from user to user.

What it doesn't mention is that when you set this option, any services that were using winhttp will also pick up these settings.

Sounds great, but once you've turned this setting on, how do you actually configure the proxy for everyone?

### A Long Journey

It turns out that Microsoft's documentation of this is *appalling*. Like really bad, even by their inconsistent standards. There's just nothing on how to do this anywhere. I spent several days digging and this is what I was able to uncover.

So, everyone knows that wininet proxy settings are kept in the registry at `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings`, under a couple of values like ProxyEnable, ProxyServer, and AutoConfigURL right? Well almost. Those values are stored there, but that's not what wininet actually uses. It uses the DefaultConnectionSettings binary value under `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\Connections`. Whenever wininet is invoked by launching a browser or going into the Internet Settings control panel applet, some background process takes the settings from those registry values and bundles them up into the DefaultConnectionSettings object.

Herein lies our first problem: this doesn't work on a machine level. There's no background process to convert those registry values into a DefaultConnectionSettings object so you can't just push them via Group Policy Preferences. Now obviously you could just copy it from HKCU into the same location under HKLM, but while that's viable on a single machine it's not exactly practical across a business, nor is it manageable going forward. You can also use the Internet Settings control panel applet (inetcpl.cpl) but again it's not exactly a scalable solution.

OK, well surely the format of DefaultConnectionSettings is properly documented right? Wrong. The closest I can find to any kind of documentation is [this post](https://docs.microsoft.com/en-gb/archive/blogs/askie/what-is-defaultconnectionsettings-key) from the now defunct askie blog which tells you what the options are for *just the 9th byte*. It's something at least.

It's worth noting at this point that we had logged an actual support call with Microsoft about this because of the problems we were having and the total lack of available documentation and they weren't able to provide anything that could help.

After some more digging I came across a couple of (3rd party) PowerShell and VBScripts written to help you set these values, but they were all focused on setting just the ProxyServer value, which is fine if you use that, but if you're using a PAC file for example then it's not a lot of help.

So, drawing deep on my reserves of bloody-mindedness, I spent several hours reverse-engineering the DefaultConnectionSettings and I'm pleased to be able to present you with `Set-ProxyBytes.ps1`.

This lovely result of stubbornness will allow you to generate and set DefaultConnectionSettings values for any combination of Auto-detection, PAC files, and Proxy Servers that you like. It will write to HKLM or HKCU. It will output to a custom registry value if you're testing and don't want to overwrite your live settings. It will output to the console in a format you can just copy and paste into a Group Policy Preferences registry setting if you want to deploy it to machines.

Please note that it will do exactly what you tell it to. If you set `-EnableProxy` and then don't provide a value for `-Proxy` it will enable the proxy server with an empty server hostname and port. Obviously to write to HKLM you need to be running with local admin rights.

```powershell
<#
.SYNOPSIS
Generate DefaultConnectionSettings binary data

.DESCRIPTION
Generate DefaultConnectionSettings binary data for setting proxy configuration. Can be used to output directly to the registry in HKCU or HKLM, or can output to the console in a Group Policy Preferences Registry setting-friendly format.

.PARAMETER EnableAuto
Enable "Automatically Detect Settings" option

.PARAMETER EnablePAC
Enable "User Automatic Configuration Script" option

.PARAMETER EnableProxy
Enable "User a proxy server" option

.PARAMETER EnableLocal
Enable the "Bypass Proxy For Local Addresses" option

.PARAMETER PAC
PAC file URL

.PARAMETER Proxy
Proxy server and port in <server>:<port> format

.PARAMETER Bypass
Semi-colon-separated list of IPs, hosts, or domains, to bypass the proxy

.PARAMETER RegHive
Registry hive to write to; HKCU or HKLM. Defaults to HKCU.

.PARAMETER RegValue
Registry value to write to. Defaults to DefaultConnectionSettings

.PARAMETER IncludeWOW64
Also set WOW6432Node reg value for 32-bit applications on 64-bit Windows (HKLM only)

.PARAMETER OutReg
Output result directly to the registry. Can be used with OutConsole.

.PARAMETER OutConsole
Output result to the console in GPP-friendly format. Can be used with OutReg.

.EXAMPLE
PS> Set-ProxyBytes.ps1 -EnablePAC -PAC "https://wpad.contoso.com/proxy.pac" -OutConsole

.NOTES
    Author: Adam Beardwood
    Date: 2021-04-10
    Version History:
        v1.0 - Initial Release
#>

[cmdletbinding()]

Param(
    [Parameter(Mandatory=$false)][switch]$EnableAuto,
    [Parameter(Mandatory=$false)][switch]$EnablePAC,
    [Parameter(Mandatory=$false)][switch]$EnableProxy,
    [Parameter(Mandatory=$false)][switch]$EnableLocal,
    [Parameter(Mandatory=$false)][string]$PAC="",
    [Parameter(Mandatory=$false)][string]$Proxy="",
    [Parameter(Mandatory=$false)][string]$Bypass="",
    [Parameter(Mandatory=$false)][string]$RegHive="HKCU",
    [Parameter(Mandatory=$false)][string]$RegValue="DefaultConnectionSettings",
    [Parameter(Mandatory=$false)][switch]$IncludeWOW64,
    [Parameter(Mandatory=$false)][switch]$OutReg,
    [Parameter(Mandatory=$false)][switch]$OutConsole
)

if(!$OutConsole -and !$OutReg){
    write-output "ERROR: No output type specified. Please use -OutReg and/or -OutConsole"
    exit 1
}

#Static vars
$RegistryPath = "$($RegHive):\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
$WOW64RegistryPath = "$($RegHive):\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Internet Settings"
$Revision = "02"
$LocalBypass = "<local>"

if((!$EnableProxy) -and (!$EnablePAC) -and (!$EnableAuto)){
    #Nothing
    $ProxyOptions = "01"
    $PAC = ""
    $Proxy = ""
    $Bypass = ""
    $LocalBypass = $false
}elseif(($EnableProxy) -and (!$EnablePAC) -and (!$EnableAuto)){
    #Proxy
    $ProxyOptions = "03"
    $PAC = ""
}elseif((!$EnableProxy) -and ($EnablePAC) -and (!$EnableAuto)){
    #PAC
    $ProxyOptions = "05"
    $Proxy = ""
    $Bypass = ""
    $LocalBypass = $false
}elseif(($EnableProxy) -and ($EnablePAC) -and (!$EnableAuto)){
    #Proxy + PAC
    $ProxyOptions = "07"
}elseif((!$EnableProxy) -and (!$EnablePAC) -and ($EnableAuto)){
    #Auto
    $ProxyOptions = "09"
    $PAC = ""
    $Proxy = ""
    $Bypass = ""
    $LocalBypass = $false
}elseif(($EnableProxy) -and (!$EnablePAC) -and ($EnableAuto)){
    #Proxy + Auto
    $ProxyOptions = "11"
    $PAC = ""
}elseif((!$EnableProxy) -and ($EnablePAC) -and ($EnableAuto)){
    #PAC + Auto
    $ProxyOptions = "13"
    $Proxy = ""
    $Bypass = ""
    $LocalBypass = $false
}elseif(($EnableProxy) -and ($EnablePAC) -and ($EnableAuto)){
    #All
    $ProxyOptions = "15"
}else{
    #Fallback
    write-output "Invalid options provided, aborting"
    exit 1
}

write-debug "Setting`nProxy Options: $ProxyOptions`nPAC: $PAC`nProxy: $Proxy`nBypass: $Bypass`nLocalBypass: $EnableLocal"

if($EnableLocal){
    $Bypass = "$LocalBypass;$Bypass"
}

$PacBytes = [system.Text.Encoding]::ASCII.GetBytes($PAC)
$ProxyBytes = [system.Text.Encoding]::ASCII.GetBytes($Proxy)
$BypassBytes = [system.Text.Encoding]::ASCII.GetBytes($Bypass)

$DefaultConnectionSettings = [byte[]]@(@(70, 0, 0, 0) + @($Revision, 0, 0, 0) + @($ProxyOptions, 0, 0, 0) + @($ProxyBytes.Length, 0, 0, 0) + $ProxyBytes + @($BypassBytes.Length, 0, 0, 0) + $BypassBytes + @($PacBytes.Length, 0, 0, 0) + $PacBytes + @(1..32 | % { 0 }))

if($OutReg){
    Set-ItemProperty -Path "$RegistryPath\Connections" -Name $RegValue -Value $DefaultConnectionSettings
}

if($OutReg -and $IncludeWOW64 -and ($RegHive -eq "HKLM")){
    Set-ItemProperty -Path "$WOW64RegistryPath\Connections" -Name $RegValue -Value $DefaultConnectionSettings
}

if($OutConsole){
    [System.BitConverter]::ToString($DefaultConnectionSettings) -replace "-"
}
```

### Conclusion

Ugh.
