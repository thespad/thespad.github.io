---
title: Getting Last Logon Times For Members of A Group – ADSI Edition
author: Adam
date: 2012-08-20T09:16:54+00:00
aliases: ["/angry/441"]
tags:
  - active directory
  - ADSI
  - microsoft
  - powershell
  - scripts

---
This handy little script will pull all of the users from the specified AD group and then grab the LastLogon time from each specified DC (or you could use`[DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers` to get all of them in the current domain) as well as grabbing the LastLogonTimeStamp for good measure. You can also specify which attribute you want to sort the results on; I recommend `samaccountname` because it's usually the most useful.

Obviously it's much quicker and simpler to do this with the ActiveDirectory cmdlets, but sometimes you're stuck working with a bunch of 2003 DCs and have to make do with ADSI.

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

#AD Group Last Logon Checker (ADSI Edition)
#Adam Beardwood 20/08/2012
#v1.0 - Initial Release
#v1.5 - Added Datagrid & CSV output

$dcs = ("DC1","DC2","DC3")
$group = "CN=Domain Admins,CN=Users,DC=domain,dc=local"
$basedn = "DC=domain,DC=local"
$sorton = "samaccountname"
$useroutput = @()

foreach($dc in $dcs){
  $Search = New-Object DirectoryServices.DirectorySearcher([ADSI]"LDAP://$dc/$basedn")
  $Search.Filter = "(&(memberof=$group)(objectclass=user))"
  $Search.Sort = New-Object System.DirectoryServices.SortOption($sorton,"Ascending")
  $SearchRes1 = $Search.Findall()

  foreach($result in $searchres1){

    $user = $result.Properties.samaccountname
    $Search.Filter = "(&(objectclass=user)(samaccountname=$user))"
    $SearchRes2 = $Search.Findall()

    New-Variable -Name "$user" -value (New-Object System.Object) -ErrorAction Silentlycontinue
    (Get-Variable "$user").value | Add-Member -type NoteProperty -name Name -value $user[0] -ErrorAction Silentlycontinue
    foreach($adc in $dcs){
      (Get-Variable "$user").value | Add-Member -type NoteProperty -name $adc -value "" -ErrorAction Silentlycontinue
    }
    (Get-Variable "$user").value| Add-Member -type NoteProperty -name LLTS -value "" -ErrorAction Silentlycontinue

    if($SearchRes2[0].Properties.lastlogon -ne $null){
      (Get-Variable "$user").value.$dc = ([DateTime]::FromFileTime([Int64]::Parse($SearchRes2[0].Properties.lastlogon)))
    }

    if($SearchRes2[0].Properties.lastlogontimestamp -ne $null){
      (Get-Variable "$user").value.LLTS = ([DateTime]::FromFileTime([Int64]::Parse($SearchRes2[0].Properties.lastlogontimestamp)))
    }
  }
}

foreach($result in $searchres1){
  $user = $result.Properties.samaccountname
  $useroutput += (Get-Variable "$user").value
}

$useroutput | Sort-Object Name | Out-GridView -Title "Last Logon Times"
#$useroutput | Sort-Object Name | Export-CSV C:\logons.csv
```
