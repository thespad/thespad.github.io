---
title: Find Unlinked GPOs
author: Adam
date: 2014-05-30T08:31:46+00:00
aliases: ["/wordpress/676"]
tags:
  - active directory
  - GPO
  - group policy
  - powershell
  - scripts
  - windows
  - XML

---
You know how it is, you don't pay attention to the management of your domain for just 5 or 6 years and suddenly you have hundreds of GPOs with no idea what half of them do or even if they're actually linked somewhere. For some reason, the Powershell GPO module doesn't have a simple cmdlet or property that lets you tell if a GPO is linked or not, because that would be far too helpful, but it's not too hard to do if you don't mind parsing some XML.

This code is based on a much more complicated script from [here][1], designed to let you search for individual settings within a GPO. It will accept a number of arguments, but run without any it will simply output to the console a list of all of the unlinked GPOs in the current domain.

```powershell
<#
Copyright (c) 2014, Adam Beardwood
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR
ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#>

#Find Unlinked Linked GPOs in a domain
#Adam Beardwood 20/05/2014
#v1.0 - Initial Release

param (
[Parameter(Mandatory=$false)]
[boolean] $outfile=$false,
[Parameter(Mandatory=$false)]
[string] $filename="UnlinkedGPO-$(get-date -f HHmmss).txt",
[Parameter(Mandatory=$false)]
[string] $DomainName = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
)

Import-Module GroupPolicy;

[string] $Extension="Enabled"

$allGposInDomain = Get-GPO -All -Domain $DomainName | Sort DisplayName;

$xmlnsGpSettings = "http://www.microsoft.com/GroupPolicy/Settings";
$xmlnsSchemaInstance = "http://www.w3.org/2001/XMLSchema-instance";
$xmlnsSchema = "http://www.w3.org/2001/XMLSchema";

$QueryString = "gp:LinksTo";

$host.UI.WriteLine();

foreach ($Gpo in $allGposInDomain)
{
  $xmlDoc = [xml] (Get-GPOReport -Guid $Gpo.Id -ReportType xml -Domain $Gpo.DomainName);
  $xmlNameSpaceMgr = New-Object System.Xml.XmlNamespaceManager($xmlDoc.NameTable);

  $xmlNameSpaceMgr.AddNamespace("", $xmlnsGpSettings);
  $xmlNameSpaceMgr.AddNamespace("gp", $xmlnsGpSettings);
  $xmlNameSpaceMgr.AddNamespace("xsi", $xmlnsSchemaInstance);
  $xmlNameSpaceMgr.AddNamespace("xsd", $xmlnsSchema);

  $extensionNodes = $xmlDoc.DocumentElement.SelectNodes($QueryString, $XmlNameSpaceMgr);

  $stringToPrint = $($Gpo.DisplayName) + " is not linked in this domain";

  if($extensionNodes[0] -eq $null){
    if($outfile -eq $true){
      $stringToPrint | Out-File $filename -Append
    }else{
      write-host $stringToPrint -foregroundcolor red
    }
  }
}
```

 [1]: http://activedirectory.ncsu.edu/advanced-topics/scripting-center/gpo-setting-search-powershell-example/
