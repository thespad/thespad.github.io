---
title: Send Skype For Business (Lync) IMs via Powershell
author: Adam
date: 2016-02-02T12:12:13+00:00
aliases: ["/wordpress/865"]
tags:
  - IM
  - instant messaging
  - lync
  - microsoft
  - powershell
  - S4B
  - scripts
  - skype for business
  - UCWA

---
This script uses [UCWA][1] to send IMs via Skype For Business. Unlike the commonly documented methods using Microsoft.Lync.Model from the Lync SDK which require the Lync/S4B client to be installed, running and logged in to work, this will run from any machine on your internal network (and in theory could be used externally with some tweaking too). It still needs some cleaning up because none of the code examples I could find were for Powershell so there was a bit of trial and error and I'm sure things can be done more efficiently, but at a basic level it does what it's supposed to.

The help should cover most of the arguments, it's all fairly self-explanatory. You'll probably want to update the following line in the script with your own Application name and GUID:

```powershell
$postparams = @{UserAgent="My UCWA Application";EndpointId="75d0449f-aa09-4f5d-add5-eeefc518c48a";Culture="en-US"} | ConvertTo-JSON
```

You can use `([guid]::NewGuid()).guid` to generate a new GUID for your application.

You can also edit the default $messageheader, $messagefooter, $messagebody & $messagesubject values. The Subject and Header must be plaintext, the Body and Footer support HTML.

```powershell
#Requires -version 4

<#
.SYNOPSIS
  Sends IM via S4B
.DESCRIPTION
    Sends IM via S4B using UCWA
.PARAMETER username
  Username of account used to send messages in UPN format
.PARAMETER password
  Password of account as plaintext (exclusive of pwd)
.PARAMETER pwd
  Password of account as securestring (exclusive of password)
.PARAMETER recipients
  List of recipient SIP addresses, comma separated
.PARAMETER messagesubject
  Message subject (Plain text only)
.PARAMETER messagebody
  Message body (HTML allowed)
.PARAMETER messageheader
  Message header (Plain text only), sent as part of the invitation
.PARAMETER messagefooter
  Message footer (HTML allowed), sent after the message body
.EXAMPLE
    C:\>Send-S4BMessage.ps1
.NOTES
    Author: Adam Beardwood
    Date: January 13, 2016
    v1.0: Initial Release
    v1.1: Bugfix for multiple recipients
#>

[CmdletBinding(DefaultParameterSetName="secure")]
param(
  [Parameter(Mandatory=$true)][string]$username,
  [Parameter(Mandatory=$true,ParameterSetName='secure')][System.Security.SecureString]$pwd,
  [Parameter(Mandatory=$true,ParameterSetName='plain')][string]$password,
  [Parameter(Mandatory=$true)][array]$recipients,
  [Parameter(Mandatory=$false)][string]$messagesubject,
  [Parameter(Mandatory=$false)][string]$messagebody,
  [Parameter(Mandatory=$false)][string]$messageheader,
  [Parameter(Mandatory=$false)][string]$messagefooter
)

function sendmessage ($operationID, $rootappurl, $appid, $authcwt, $recipient, $messagesubject, $messagebody, $messagefooter, $ackid) {

  $statecount = 0
  $ackid = $ackid.tostring()

  write-verbose "#Send Message Invite"

  write-verbose "$rootappurl/$appid/communication/"

  try{
    $postparams = @{"importance"="Normal";"sessionContext"="$(([guid]::NewGuid()).guid)";"subject"="$messagesubject";"telemetryId"=$null;"to"=$recipient;"operationId"=$operationID;"_links"=@{"message"=@{"href"="data:text/plain,$messageheader"}}} | convertto-json

    $data = Invoke-WebRequest -Uri "$rootappurl/$appid/communication/messagingInvitations" -Method POST -Body "$postparams" -Headers @{"Authorization"="Bearer $authcwt"} -ContentType "application/json" -UseBasicParsing
  }catch{
    write-verbose "Unable to send message invite"
    return $false
  }

  write-verbose "#Check state & get Conversation ID"

  do{

    $data = Invoke-WebRequest -Uri "$rootappurl/$appid/events?ack=$ackid" -Method GET -Headers @{"Authorization"="Bearer $authcwt"} -ContentType "application/JSON" -UseBasicParsing

    $state = (($data.content | ConvertFrom-JSON).sender.events._embedded.messaging.state)

    if($state.gettype().name -eq "string"){

    }else{
      $state = $state[-1]
    }

    $statecount++

    if(($statecount -ge 25) -or ($state -eq "Disconnected")){
      write-verbose "No response from endpoint or conversation declined"
      return $false
    }

    start-sleep 1

  }while($state -notcontains "Connected")

  $JSONdata = $data.content | ConvertFrom-JSON

  $conversationID = ($JSONdata.sender.events.link | ?{$_.rel -eq "conversation"}).href.split("/")[-1]

  $conversationID

  write-verbose "#Send Messages"

  try{
    $data = Invoke-WebRequest -Uri "$rootappurl/$appid/communication/conversations/$conversationID/messaging/messages" -Method POST -Body $messagebody -Headers @{"Authorization"="Bearer $authcwt"} -ContentType "text/HTML" -UseBasicParsing
    $data = Invoke-WebRequest -Uri "$rootappurl/$appid/communication/conversations/$conversationID/messaging/messages" -Method POST -Body $messagefooter -Headers @{"Authorization"="Bearer $authcwt"} -ContentType "text/HTML" -UseBasicParsing
  }catch{
    write-verbose "Unable to send message"
    $data = Invoke-WebRequest -Uri "$rootappurl/$appid/communication/conversations/$conversationID/messaging/terminate" -Method POST -Headers @{"Authorization"="Bearer $authcwt"} -UseBasicParsing
    return $false
  }
  write-verbose "#Terminate Conversation"

  try{
    $data = Invoke-WebRequest -Uri "$rootappurl/$appid/communication/conversations/$conversationID/messaging/terminate" -Method POST -Headers @{"Authorization"="Bearer $authcwt"} -UseBasicParsing
  }catch{
    write-verbose "Failed to terminate conversation"
    return $false
  }

  rv conversationID, recipient, ackid, state

  return $true

}

if($pwd){[string]$password = (New-Object System.Management.Automation.PSCredential('dummy',$pwd)).getnetworkcredential().password}
if(!$messagesubject){$messagesubject = "S4B Automated Message"}
if(!$messagebody){$messagebody = "<font face='calibri'>Someone forgot to set a message so you're seeing this default instead</font>"}
if(!$messageheader){$messageheader = "This is an automated alert from <system>"}
if(!$messagefooter){$messagefooter = "<font face='calibri'><i>This message was sent by a bot, there's no point in replying to it.</i></font>"}

write-verbose "#Get Autodiscover Information"

try{
  $data = Invoke-WebRequest -Uri "https://lyncdiscoverinternal.$($env:userdnsdomain)" -Method GET -ContentType "application/json" -UseBasicParsing

  $baseurl = (($data.content | ConvertFrom-JSON)._links.user.href).split("/")[0..2] -join "/"

  $oauthurl = ($data.content | convertfrom-json)._links.user.href
}catch{
  write-output "Unable to get autodiscover information"
  exit 1
}
write-verbose "#Authenticate to server"

try{
  $postParams = @{grant_type="password";username=$username;password=$password}

  $data = Invoke-WebRequest -Uri "$baseurl/WebTicket/oauthtoken" -Method POST -Body $postParams -UseBasicParsing

  $authcwt = ($data.content | ConvertFrom-JSON).access_token
}catch{
  write-output "Unable to authenticate, verify credentials and try again"
  exit 1
}
write-verbose "#Get application URLs"

try{
  $data = Invoke-WebRequest -Uri "$oauthurl" -Method GET -Headers @{"Authorization"="Bearer $authcwt"} -UseBasicParsing

  $rootappurl = ($data.content | ConvertFrom-JSON)._links.applications.href
}catch{
  write-output "Unable to get Application URLs"
  exit 1
}

write-verbose "#Create App Instance"

try{
  $postparams = @{UserAgent="My UCWA Application";EndpointId="75d0449f-aa09-4f5d-add5-eeefc518c48a";Culture="en-US"} | ConvertTo-JSON

  $data = Invoke-WebRequest -Uri "$rootappurl" -Method POST -Body "$postparams" -Headers @{"Authorization"="Bearer $authcwt"} -ContentType "application/json" -UseBasicParsing

  $appurl = $(($data.content | ConvertFrom-JSON)._links.self.href)

  $appurl = "$($rootappurl.split("/")[0..2] -join "/")$(($data.content | ConvertFrom-JSON)._links.self.href)"

  $appid = $appurl.split("/")[-1]

  $operationID = (($data.content | ConvertFrom-JSON)._embedded.communication | GM -Type Noteproperty)[0].name
}catch{
  write-output "Unable to create application instance"
  exit 1
}

write-verbose "#Allow HTML messages to be sent"

try{
  $postparams = @{"supportedMessageFormats"="Plain","Html"} | ConvertTo-JSON

  $data = Invoke-WebRequest -Uri "$rootappurl/$appid/communication/makeMeAvailable" -Method POST -Body $postparams -Headers @{"Authorization"="Bearer $authcwt"} -ContentType "application/json" -UseBasicParsing
}catch{
  write-verbose "HTML Messaging Already Configured"
}

write-verbose "#Send messages"

$i = 0

foreach($recipient in $recipients){

  if($recipient -notmatch "^sip:\S+"){
    $recipient = "sip:$recipient"
  }
  $i++
  $msgresult = sendmessage $operationID $rootappurl $appid $authcwt $recipient $messagesubject $messagebody $messagefooter $i
  if($msgresult){
    write-verbose "Message sent to $recipient"
  }else{
    write-verbose "Message not sent to $recipient"
  }
  start-sleep 1
}

write-verbose "#Delete App Instance"
$deleteapp = Invoke-WebRequest -Uri "$rootappurl/$appid" -Method DELETE -Headers @{"Authorization"="Bearer $authcwt"} -UseBasicParsing
```

 [1]: https://ucwa.skype.com/
