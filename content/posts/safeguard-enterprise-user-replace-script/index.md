---
title: Sophos/Utimaco Safeguard Enterprise User Addition/Removal Script
author: Adam
date: 2010-12-02T19:20:24+00:00
aliases: ["/angry/37"]
tags:
  - encryption
  - powershell
  - safeguard
  - scripts
  - sophos
  - windows

---
The following script will allow you to add or remove registered user accounts to/from Sophos Safeguard Enterprise clients on a large scale (OU, domain or even org-wide). Needs to be run from a machine with Safeguard Server installed (or have the authentication method changed, of course). Obviously you can’t add Local accounts, but you can remove them – it hasn’t been tested with Workgroups.

Important bits are as follows; Vars to set are:

* $NewUser – This needs to be the “display name” (As seen in the SG management console) of the account you want to Add to machines
* $OldUser – This needs to be the “display name” (As above) of the account you want to Remove from machines
* $SearchPath – This needs to be the ADSPath for the OU or Domain you want to operate in, make it blank to work on the Root
* $Criteria – This is a bit more complex, it can be set to anything from 2.8.1 of the API docs – we’re using “POA” as it seems the most sensible – to filter which machines to make the changes on. If you’re using something that doesn’t return True for what you want then the script will require some further modifications to work properly.

There are two Commented sections further down the script that you can remove if you only want to Add or Remove accounts rather than do both at once - my intent is to swap one account for another, hence the script format - just snip out everything between the marked lines.

Error handling errs on the side of caution, but I’m sure you could still screw up a lot of machines if you’re not careful, so the usual disclaimers apply.

```powershell
#Safeguard User Swap Tool
#Adam Beardwood 30/11/2010
#v1.0 - Initial Release

#Declare useful vars
$NewUser = "New User"
$SearchPath = "dc=somedomain,dc=co,dc=uk"
$OldUser = "Old User"
$Criteria = "POA"

#Load Utimaco API Assembly
[void][System.Reflection.Assembly]::LoadWithPartialName("Utimaco.SafeGuard.AdministrationConsole.Scripting")

#Declare annoying [ref] vars
[ref]$Machines = $null
[ref]$Machine = $null
[ref]$Type = $null
[ref]$Val = $null
[ref]$Prop = $null
[ref]$Users = $null
[ref]$User = $null
[ref]$NUser = $null
[ref]$NType = $null

#Create scripting object, initialize it and Authenticate account (Requires Safeguard Server to be installed)
$Scripting = new-object Utimaco.SafeGuard.AdministrationConsole.Scripting.Base
[void]$Scripting.Initialize()
try{[void]$Scripting.AuthenticateService()}
catch{write-host "Error: This machine doesn't appear to have the Safeguard Server component installed, so it can't authenticate in this way. Quitting...";exit 1}

#Create Directory, Inventory and UMA Instances & Initialize them
$Directory = $Scripting.CreateDirectoryClassInstance()
$Inventory = $Scripting.CreateInventoryClassInstance()
$UMA = $Scripting.CreateUMAClassInstance()

[void]$Directory.Initialize()
[void]$Inventory.Initialize()
[void]$UMA.Initialize()

#Get ADSPath for $NewUser Account
$Result = $Directory.GetOneObject($NewUser,$SearchPath,2,$NUser,$NType)

#Verify we have located it, otherwise bail
if($Result -ne 0){write-host "Error: Could not locate $NUser account. Quitting...";exit 1}

#Initialize a directory search for computer accounts in the $SearchPath domain
[void]$Directory.GetObjectInitialize("*",$SearchPath,1,$Machines)

#For each result returned
for ($i=0; $i -lt $($Machines.value); $i++){

  #Get the object's ADS path and then check the POA status
  [void]$Directory.GetObjectByIndex($i,$Machine,$Type)
  $Result = $Inventory.GetComputerInventory($($Machine.Value),$Criteria,$Val,$Prop)

  #If the reported ADS path doesn't actually exist, skip this machine
  if($Result -ne 0){write-host "Error: Object isn't where it's supposed to be. Skipping...";continue}

  #If POA is enabled
  if($($Prop.Value) -eq $True){

    #Initialize a UMA search for User accounts on the machine
    [void]$UMA.GetUMAOfMachineInitialize($($Machine.value),$Users)

#----- This Section is for Removing Accounts

    #For each result returned
    for ($j=0; $j -lt $($Users.value); $j++){

      #Get the user's ADS path
      [void]$UMA.GetUMAOfMachineByIndex($j,$User)

      #If the given user path matches $OldUser
      if($($User.value) -match $OldUser){
        #Delete the account from the machine
        [void]$UMA.DeleteUMA($($User.value),$($Machine.value))
        write-host "Removed $OldUser Account from $($Machine.value)"
      }
    }

#----- End Removing Accounts Section


#----- This Section is for Adding Accounts

    #Add $ModUser Account
    [void]$UMA.CreateUMA($($NUser.Value),$($Machine.value))
    write-host "Added $NewUser Account to $($Machine.value)"

#----- End Adding Accounts Section

    #Finalize the UMA search in preparation for the next one
    [void]$UMA.GetUMAOfMachineFinalize()
  }

}

#Finalize ScriptingDirectory
[void]$Directory.GetObjectFinalize()

#Free all the resources
[void]$Inventory.FreeResources()
[void]$UMA.FreeResources()
[void]$Directory.FreeResources()
[void]$Scripting.FreeResources()
```
