# Prevent Windows 11 Sign-in on Wake Up
For some reason, Windows 11 forces a login for security every time the system resumes and totally disrespects the power options setting. Use the following ```powercfg``` commands in a Terminal (Admin) to toggle the ```CONSOLELOCK``` setting.


## 🔌When Plugged In (AC)

The below command will modify the behavior when plugged-in.
```
powercfg /SETACVALUEINDEX SCHEME_CURRENT SUB_NONE CONSOLELOCK 0​
```
## 🔋On Battery (DC)
Execute the below command to modify the behavior when on battery (applicable for devices like laptops).
```​
powercfg /SETDCVALUEINDEX SCHEME_CURRENT SUB_NONE CONSOLELOCK 0​
```

## 💡Manual Lock
Disabling the automatic sign-in behavior is for power users who want to be the one in control of locking their PC achieved using keyboard shortcut ⊞ Win + L.
