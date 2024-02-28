---
title: Changing Printer Drivers En Masse
author: Adam
date: 2012-03-01T10:53:36+00:00
aliases: ["/wordpress/400"]
tags:
  - automation
  - drivers
  - powershell
  - printers
  - scripts
  - windows

---
If you need to change the drivers for a large number of printers, such as on a print server, then you can use the following Powershell to do it. Set $driver to the name of the driver you wish to set and $pattern to match for the printers you wish to affect (so you don't change the driver on printers you don't want to).

Note: This script will run pretty quickly, but depending on the number of printers it may take upwards of 10 minutes for Windows to do all the background processing associated with the driver changes. Keep an eye out for a bunch of rundll32.exe processes which will spawn; once they close themselves down the changes should be complete.

```powershell
$driver = "HP Universal Printing PCL 6 (v5.4)"
$pattern = "HP"

$printers = gwmi win32_printer

foreach($printer in $printers){
        $name = $printer.name
        if($name -match $pattern){
                & rundll32 printui.dll PrintUIEntry /Xs /n $name DriverName $driver
        }
}
```
