---
title: Sophos Safeguard Enterprise Auto-Sync Script
author: Adam
date: 2010-12-05T20:29:10+00:00
aliases: ["/angry/64"]
tags:
  - encryption
  - powershell
  - safeguard
  - scripts
  - sophos

---
Those of you who have used Safeguard will know that for reasons known only to the Germans, Utimaco decided not to provide any way to automatically sync Safeguard with your AD domain(s) without resorting to a rather buggy API. They provide some example code for VBScript and Sophos are rumoured to be adding an automatic sync function in 5.60, but if you'd rather do it in Powershell and have some decent error handling and reporting then look no further.

As per usual with these things, this needs to be run on a machine that has Safeguard Server installed; set your DSN, Log path & sync options at the top of the script and then add your email details below if you want email alerting (It'll only alert on failures, but the local logs will be created regardless). For those of you who may have used my previous sync script(s), the key change in this one is the use of the previously unknown (to me) `CanSynchronizeDirectory()` function to check that nobody else is currently syncing the database before attempting it.

```powershell
#Safeguard Directory Synchronisation Tool
#Adam Beardwood 04/02/2010
#v1.0 - Initial Release
#v2.0 - Cleanup re-write with better error and email handling

#Load Safeguard .NET Assembly for use
[void][System.Reflection.Assembly]::LoadWithPartialName("Utimaco.SafeGuard.AdministrationConsole.Scripting")

#---Declare Variables---
#DateTime Stamp
$DTS = date -format yyyy-MM-dd--hh-mm
#Root DSN to bind connection to
$dsn = "DC=somedomain,DC=co,DC=uk"
#Location for Log File
$logFileName = "C:\Logs\SGSync."+$DTS+".log"
#Sync Group Membership [0|1]
$membership = 1
#Sync Account State [0|1]
$accountState = 1
#Relocate Move Objects if they have been relocated to another sync'd OU [0|1]
$takeCareOfMovedObjects = 1
#Reference Vars for Functions
[ref] $CanSync = $null
#---End Variables---

#---Define Functions---

#Function to send email alert
Function SendEmail ($Errs){

#-- Set variables for email notification --
$smtpServer = "smtp.somedomain.co.uk"
$mailto = "<someuser@somedomain.co.uk>"
$mailfrom = "SGE AD Sync<SGSync@somedomain.co.uk>"
$mailsubject = "SGE Sync Process"

$bodytext=@"
The Safeguard Enterprise Automated Sync process ran. The following errors occurred:

$(foreach($item in $Errs){$item;"`r"})
"@

$att = $logFileName

Send-MailMessage -To $mailto -From $mailfrom -Subject $mailsubject -Body $bodytext -Smtpserver $smtpserver -Attachments $att

write-host "Email sent, also," $Errs
}

#Function to actually sync Safeguard
Function SGSync ($OU){

$adsStartContainer = $OU+","+$dsn
write-host "Syncing:" $adsStartContainer

[ref] $Outcome = $Directory.SynchronizeDirectory($dsn, $adsStartContainer, 1, $logFileName, $membership, $accountState, $takeCareOfMovedObjects)

$Result = $Scripting.GetLastError($Outcome)
write-host "GetLastError returns:" $Result
}

#---End Functions---

#Create scripting objects, authenticate to directory and then initialise the sync process
write-host "Synchronization of Users & Computers ... Started"

try{$Scripting = new-object Utimaco.SafeGuard.AdministrationConsole.Scripting.Base}
catch{write-host "An Error Occurred While Attempting To Load Safeguard Directory Synchronisation Libraries. Quitting...";exit 0}

[void]$Scripting.Initialize()

try{[void]$Scripting.AuthenticateService()}
catch{write-host "Error: This machine doesn't appear to have the Safeguard Server component installed, so it can't authenticate in this way. Quitting...";exit 1}

$Directory = $Scripting.CreateDirectoryClassInstance()
[void]$Directory.Initialize()

#Check if we can sync
[void]$Directory.CanSynchronizeDirectory($CanSync)

if($($CanSync.Value) -eq 1){

  #---Sync the following OUs---
  SGSync("OU=SomeOU")
  SGSync("OU=SubOU,OU=SomeOU")

}else{
  write-host "Unable to Sync - Another synchronisation is already in progress"
  }

#Free up resources
[void]$Directory.FreeResources()
[void]$Scripting.FreeResources()

#Get errors from the generated log file
$Errs = select-string -pattern "Failure" -path $logFileName

#Send email alert
if($Errs -ne $null){SendEmail $Errs}

write-host "Synchronization of Users & Computers...End"
```
