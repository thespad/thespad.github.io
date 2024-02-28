---
title: Update Window Settings After Scripted Changes
author: Adam
date: 2015-12-03T14:13:55+00:00
aliases: ["/angry/855"]
tags:
  - explorer
  - microsoft
  - powershell
  - refresh
  - registry
  - windows
  - WM_SETTINGCHANGE

---
Let's say you make a change to your locale settings by directly editing the registry, modifying the `HKCU\Control Panel\International\sTimeFormat` key. Problem is that Windows doesn't pick up these changes until you log off and back on again or restart explorer.exe. Now if you make the changes via Control Panel you don't have to do this, so why do you if you modify the registry?

Well, when you use the UI to make changes to the locale or any other policy or environment settings, Windows sends a [WM_SETTINGCHANGE][1] broadcast to all Windows notifying them of the change to settings so they can refresh their config and _you can do it too!_

```powershell
if (-not ("win32.nativemethods" -as [type])) {
    add-type -Namespace Win32 -Name NativeMethods -MemberDefinition @"
[DllImport("user32.dll", SetLastError = true, CharSet = CharSet.Auto)]
public static extern IntPtr SendMessageTimeout(
    IntPtr hWnd, uint Msg, UIntPtr wParam, string lParam,
    uint fuFlags, uint uTimeout, out UIntPtr lpdwResult);
"@
}

$HWND_BROADCAST = [intptr]0xffff;
$WM_SETTINGCHANGE = 0x1a;
$result = [uintptr]::zero

[win32.nativemethods]::SendMessageTimeout($HWND_BROADCAST, $WM_SETTINGCHANGE,[uintptr]::Zero, "Environment", 2, 5000, [ref]$result);
```

From the docs in reference to the lParam parameter of [SendMessageTimeout][2], set to "Environment" in the above example:

> This string can be the name of a registry key or the name of a section in the Win.ini file. When the string is a registry name, it typically indicates only the leaf node in the registry, not the full path.
>
> When the system sends this message as a result of a change in policy settings, this parameter points to the string "Policy".
>
> When the system sends this message as a result of a change in locale settings, this parameter points to the string "intl".
>
> To effect a change in the environment variables for the system or the user, broadcast this message with lParam set to the string "Environment".

 [1]: https://msdn.microsoft.com/en-us/library/ms725497%28VS.85%29.aspx
 [2]: https://msdn.microsoft.com/en-us/library/ms644952%28v=vs.85%29.aspx
