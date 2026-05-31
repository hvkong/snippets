# Restore Windows 10 Right-Click Menu on Windows 11

Windows 11 replaced the classic right-click context menu with a simplified version, hiding
most options behind a **"Show More Options"** submenu. This also noticeably slowed down the
time it takes for the menu to appear. Use the `reg.exe` commands below in a Terminal to restore
the full Windows 10 right-click behavior.

## ✅ Restore Classic Right-Click Menu

```
reg.exe add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /f /ve
```

## ↩️ Undo / Revert to Windows 11 Menu

```
reg.exe delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f
```

## 🔄 Apply Changes

Restart your PC, or restart the Windows Explorer process without rebooting via Task Manager
or the below command in Terminal:

```
taskkill /f /im explorer.exe & start explorer.exe
```

## 💡 Notes

- This change is **per-user** (stored under `HKCU`) and does not affect other accounts on the same machine.
- No admin privileges required — the registry key is in the current user's hive.
- Applies to **Windows 11** only. Windows 10 already uses the classic menu by default.
