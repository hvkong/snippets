## Remove Windows 11 Right Click

Windows 11 "refreshed" the right click context menu to be consistent with it's design and
hid the full options under "Show More Options" menu item. It has also made the right click
pop-up window slower to appear.

### To restore the old Windows 10 right click behavior, start a terminal session and enter the below command.

`
reg.exe add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /f /ve
`

### To undo this change use the below command.

`
reg.exe delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f
`

Restart your PC or restart the Windows Explorer process to have the changes take effect.
