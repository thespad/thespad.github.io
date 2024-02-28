---
title: DHCP-Multitool
author: Adam
date: 2012-10-16T09:18:15+00:00
aliases: ["/wordpress/471"]
tags:
  - DHCP
  - microsoft
  - powershell
  - scripts

---
The following script is an amalgamation of several tools I've built up over the last couple of years to work with multiple DHCP scopes on servers. It has 6 basic modes of operation, as detailed below, which allow you to list all scopes on a server, enable all scopes on a server, disable all scopes on a server, change the lease time for all scopes on a server, change the offer delay for all scopes on a server or add reservations to one or more servers. You can use `help .\DHCP-Tool.ps1` to get more information on the various switches, though it's worth pointing out that leasetimes are in seconds while delaytimes are in milliseconds.

Hopefully the new Server 2012 powershell modules for DHCP solve a lot of these problems, but I haven't had a chance to dig into them yet.

```powershell
    .\DHCP-Tool.ps1 -Server <String> [-show]
    .\DHCP-Tool.ps1 -Server <String> [-enable]
    .\DHCP-Tool.ps1 -Server <String> [-disable]
    .\DHCP-Tool.ps1 -Server <String> [-lease] -leasetime <String>
    .\DHCP-Tool.ps1 -Server <String> [-delay] -delaytime <Int32>
    .\DHCP-Tool.ps1 -Server <String> [-reserve] -ipscope <String> -ipaddr <String> -mac <String> [-altservers <String>] [-name <String>]
```

```powershell
################################################################################################
#This software is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;    #
#without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.     #
#THERE IS NO WARRANTY FOR THE SOFTWARE, TO THE EXTENT PERMITTED BY APPLICABLE LAW. EXCEPT      #
#WHEN OTHERWISE STATED IN WRITING THE COPYRIGHT HOLDERS PROVIDE THE SOFTWARE "AS IS" WITHOUT   #
#WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, THE         #
#IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE. THE ENTIRE RISK   #
#AS TO THE QUALITY AND PERFORMANCE OF THE SOFTWARE IS WITH YOU. SHOULD THE SOFTWARE PROVE      #
#DEFECTIVE, YOU ASSUME THE COST OF ALL NECESSARY SERVICING, REPAIR OR CORRECTION. IN NO EVENT  #
#UNLESS REQUIRED BY APPLICABLE LAW OR AGREED TO IN WRITING WILL ANY COPYRIGHT HOLDER, BE       #
#LIABLE TO YOU FOR DAMAGES, INCLUDING ANY GENERAL, SPECIAL, INCIDENTAL OR CONSEQUENTIAL DAMAGES#
#ARISING OUT OF THE USE OR INABILITY TO USE THE SOFTWARE (INCLUDING BUT NOT LIMITED TO LOSS    #
#OF DATA OR DATA BEING RENDERED INACCURATE OR LOSSES SUSTAINED BY YOU OR THIRD PARTIES OR A    #
#FAILURE OF THE SOFTWARE TO OPERATE WITH ANY OTHER PROGRAMS), EVEN IF SUCH HOLDER HAS BEEN     #
#ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.                                                   #
################################################################################################

<#
.SYNOPSIS
    DHCP Multitool
.DESCRIPTION
    Provides a variety of functions for making changes to multiple DHCP scopes and/or multiple DHCP servers
.PARAMETER Server
    The hostname or IP address of the DHCP server to work with
.PARAMETER Show
    Action: List current scope configuration
.PARAMETER Enable
  Action: Enable all scopes on server
.PARAMETER Disable
  Action: Disable all scopes on server
.PARAMETER Lease
  Action: Alters lease time on all scopes on server
.PARAMETER LeaseTime
  DHCP Lease time in seconds
.PARAMETER Delay
  Action: Alters offer delay on all scopes on server
.PARAMETER DelayTime
  Offer Delay time in miliseconds
.PARAMETER Reserve
  Action: Add reservation to server
.PARAMETER AltServer
  Additional servers to create reservation on
.PARAMETER IPScope
  Name of scope to work with
.PARAMETER IPAddr
  IP Address to reserve
.PARAMETER MAC
  MAC Address to associated reservation with
.PARAMETER Name
  Comment to add to reservation
.EXAMPLE
    C:\>dhcp-tool.ps1 -server 127.0.0.1 -enable
  Enables all scopes on server 127.0.0.1
.EXAMPLE
    C:\>dhcp-tool.ps1 -server 127.0.0.1 -delay -delaytime 1000
  Sets an offer delay of 1000ms on all scopes on server 127.0.0.1
.NOTES
    Author: Adam Beardwood
    Date: October 12, 2012
  v1.0: Initial Release
  v1.1: Added Better Error Handling and Command Feedback
#>

[CmdletBinding(DefaultParametersetName="showDHCP")]
Param(
        [Parameter(Mandatory=$true,ValueFromPipeline=$true)][String]$Server,

    [Parameter(ParameterSetName='showDHCP')][switch]$show,

    [Parameter(ParameterSetName='enableDHCP')][switch]$enable,

    [Parameter(ParameterSetName='disableDHCP')][switch]$disable,

    [Parameter(ParameterSetName='leaseDHCP')][switch]$lease,
    [Parameter(ParameterSetName='leaseDHCP',Mandatory=$true,ValueFromPipeline=$true)][string]$leasetime,

    [Parameter(ParameterSetName='delayDHCP')][switch]$delay,
    [Parameter(ParameterSetName='delayDHCP',Mandatory=$true,ValueFromPipeline=$true)][ValidateRange(0,1000)][int]$delaytime,

    [Parameter(ParameterSetName='reserveDHCP')][switch]$reserve,
    [Parameter(ParameterSetName='reserveDHCP',Mandatory=$true)][string]$ipscope,
    [Parameter(ParameterSetName='reserveDHCP',Mandatory=$true)][string]$ipaddr,
    [Parameter(ParameterSetName='reserveDHCP',Mandatory=$true)][string]$mac,
    [Parameter(ParameterSetName='reserveDHCP',Mandatory=$false)][string]$altservers = $null,
    [Parameter(ParameterSetName='reserveDHCP',Mandatory=$false)][string]$name
)

function getDHCPScope($servername, $type) {

  $ips = @()

  $scopes = netsh dhcp server $servername show scope

  if($type -eq "show"){
    foreach($scope in $scopes){
      write-host $scope -foregroundcolor green
    }
  }else{

    $regex = [regex]"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"

    foreach($scope in $scopes){

      $addr = $scope | foreach {$regex.Matches($_) | foreach {$_.Captures[0].Value}}
      if($addr -ne $null){
        $ips += $addr[0]
      }
    }

    return $ips
  }
}

function enableDHCPScope($servername, $ip){
  netsh dhcp server $servername scope $ip set state 1 | out-null
}

function disableDHCPScope($servername, $ip){
  netsh dhcp server $servername scope $ip set state 0 | out-null
}

function leaseDHCPScope($servername, $ip, $leasetime){
  netsh dhcp server $servername scope $ip set optionvalue 51 DWORD $leasetime | out-null
  $leaseinfo = netsh dhcp server $servername scope $ip show optionvalue
  foreach($info in $leaseinfo){if($info -match "OptionId : 51"){write-host "Scope $ip lease length: $($($leaseinfo[$([array]::IndexOf($leaseinfo, $info))+4]).split('=')[1]) seconds" -foregroundcolor green}}
}

function delayDHCPScope($servername, $ip, $leasetime){
  netsh dhcp server $server scope $ip set delayoffer $delaytime | out-null
  $delayinfo = netsh dhcp server $servername scope $ip show delayoffer
  foreach($info in $delayinfo){if($info -match "DelayOffer"){write-host $info -foregroundcolor green}}
}

function reserveDHCPScope($servername, $altservers, $ipscope, $ipaddr, $mac, $name){

  $servers = @($servername)

  foreach($alt in $altservers){
    if(($alt -ne "") -and ($alt -ne $null)){$servers += "\\$alt"}
  }

  write-host "Servers: $servers"

  foreach($server in $servers){

    $result = netsh dhcp server $server scope $ipscope
    if($result -match "The command needs a valid Scope IP Address"){
      write-host "Scope Does Not Exist!"
      exit 1
    }

    $result = netsh dhcp server $server scope $ipscope show clients
    if($result -match $ipaddr){
      write-host "IP Address Already Reserved!"
      exit 1
    }

    $regex = [regex]"([0-9A-Fa-f]{2}[:-]*){5}([0-9A-Fa-f]{2})"
    $result = $mac | foreach {$regex.Matches($_) | foreach {$_.Captures[0].Value}}
    if($result -eq $null){
      write-host "Invalid MAC Address!"
      exit 1
    }

    netsh dhcp server $server scope $ipscope add reservedip $ipaddr $mac $name
  }

}

if($(Get-Service -ComputerName $server -Name DHCPServer -ErrorAction SilentlyContinue) -eq $null){
  write-host "The specified server does not appear to be running DHCP"
  exit 1
}

$servername = "\\$server"

switch ($PsCmdlet.ParameterSetName){
  "showDHCP"{getDHCPScope $servername "show" }
  "enableDHCP"{foreach($ip in $(getDHCPScope $servername)){enableDHCPScope $servername $ip};getDHCPScope $servername "show"}
  "disableDHCP"{foreach($ip in $(getDHCPScope $servername)){disableDHCPScope $servername $ip};getDHCPScope $servername "show"}
  "leaseDHCP"{foreach($ip in $(getDHCPScope $servername)){leaseDHCPScope $servername $ip $leasetime}}
  "delayDHCP"{foreach($ip in $(getDHCPScope $servername)){delayDHCPScope $servername $ip $delaytime}}
  "reserveDHCP"{reserveDHCPScope $servername $altservers $ipscope $ipaddr $mac $name}
}
```
