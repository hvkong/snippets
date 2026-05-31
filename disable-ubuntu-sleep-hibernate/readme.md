# Disable Sleep and Hibernate on Ubuntu Linux

Ubuntu (and Linux in general) can suspend or hibernate via multiple pathways — systemd sleep targets, 
desktop environment power settings, or direct `systemctl` calls. The steps below close all of these.

## 📄 Modify `/etc/systemd/sleep.conf`

Open the file with a text editor:

```bash
sudo nano /etc/systemd/sleep.conf
```

Set the following to block all sleep and hibernate states:

```ini
[Sleep]
AllowSuspend=no
AllowHibernation=no
AllowSuspendThenHibernate=no
AllowHybridSleep=no
```

> `AllowSuspendThenHibernate` and `AllowHybridSleep` are sleep variants that remain available
> even when the main `AllowSuspend` and `AllowHibernation` flags are disabled, so it is best
> to explicitly disable them as well.

## 🎯 Mask systemd Sleep Targets

Masking the sleep targets prevents any process — including GNOME power settings or a closing
laptop lid — from triggering sleep:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

To reverse this later:

```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

## 🔄 Apply Changes

Restart the login service to apply the `sleep.conf` changes without rebooting:

```bash
sudo systemctl restart systemd-logind
```

## ✅ Verify

Confirm the targets are masked:

```bash
systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target
```

All four should report `masked`.

## 💡 Manual Sleep

Disabling automatic sleep is for users and servers that require the system to stay on at all times.
If you ever need to sleep or shut down the machine manually, use:

```bash
sudo systemctl suspend    # sleep (if unmasked)
sudo shutdown now         # full shutdown
```

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.
