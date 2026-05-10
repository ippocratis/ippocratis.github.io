---
layout: post
title:  "Droidian compatible phone as a home server"
author: ippo
categories: [ server, droidian ]
tags: [ server, droidian ]
image: assets/images/droidian.png
---

**Strip the UI, kill the sleep, keep the wake-lock.**

Droidian is just Debian with a phone kernel. That makes a $40 Pixel 3a or Redmi Note 9s in my case a perfect 2-4W ARM64 server â€” until Phosh, ModemManager, and Android-style deep sleep keep putting it to bed. This is the minimal setup I use to turn Droidian into a reliable, always-on box you can SSH into .

> **Heads-up:** Do this over SSH with a static IP or USB-ethernet first. Once Phosh is masked, the screen goes permanently black.

---

## 1) Kill the UI completely

```bash
# Stop it now
sudo systemctl stop phosh

# Prevent auto-start
sudo systemctl disable phosh

# Make it impossible to start
sudo systemctl mask phosh.service
```

On first reboot you'll see 1-2 bootloops, then a black screen. That's normal â€” SSH should come up after ~30 seconds.

## 2) Strip phone hardware you don't need

```bash
# Disable cellular modem
sudo systemctl disable --now ModemManager

# Disable the telephony stack
sudo systemctl disable --now ofono
```

## 3) Memory tuning

```bash
# check current swappiness
cat /proc/sys/vm/swappiness
# set to 10
sudo sh -c 'echo "vm.swappiness=10" >> /etc/sysctl.conf'
sudo sysctl -p
```

## 4) Murder all sleep targets

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Edit logind:
```bash
sudo nano /etc/systemd/logind.conf
```
Set:
```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
IdleAction=ignore
```
Then:
```bash
sudo systemctl restart systemd-logind
```

## 5) The permanent "never sleep" service

Android kernels use /sys/power/wake_lock. If you don't hold one, the phone enters deep sleep after 10-15 minutes even with systemd masked.

Create:
```bash
sudo nano /etc/systemd/system/keep-awake.service
```

Paste:

```
[Unit]
Description=Server Hardware Stability Master
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c "\
  sleep 10 && \
  iw dev wlan0 set power_save off && \
  echo -n 'server_master_lock' > /sys/power/wake_lock && \
  (echo 0 > /sys/class/backlight/panel0-backlight/brightness || true)"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

## 6) Verify it's actually awake

```bash
sudo cat /sys/kernel/debug/wakeup_sources | grep server
cat /sys/power/wake_lock
```

You should see `server_master_lock`.

## 7) Clean up the last GNOME power bits

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.desktop.session idle-delay 0
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now keep-awake.service
```

What this does:
1.  iw dev wlan0 set power_save off — prevents the Wi-Fi to go idle.
2.  echo 'server_master_lock' > /sys/power/wake_lock — tells the Android kernel "stay awake"
3.  brightness 0 — turns the panel off in case backlight wants to kick in. 

6) Verify it's actually awake

```bash
# You should see your lock counting active time
sudo cat /sys/kernel/debug/wakeup_sources | grep server

# Should return "server_master_lock"
cat /sys/power/wake_lock
```

If you see microseconds increasing, you're golden. If not, check your wlan interface name (ip a — some phones use wlp1s0).

7) Clean up the last power bits
Even headless, gsettings can still fire:

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.desktop.session idle-delay 0
```

check if gps is active 

```bash
gsettings get org.gnome.system.location enabled
```

and disable it

```bash
gsettings set org.gnome.system.location enabled false
```


8) The "Hard Limit" (Systemd Cgroups)

We can tell the Linux kernel to physically prevent apps e.g. Jellyfin from taking more than a certain amount of RAM. If it tries to cross the line, the kernel will force it to clean up.

Run: 

```bash
sudo systemctl edit jellyfin
```

paste this in 

```bash
[Service]
# Limit Jellyfin to 600MB of RAM
MemoryMax=600M
MemoryHigh=500M
# Lower its priority so it doesn't lag the phone's UI
CPUWeight=50
IOWeight=50
```



---

## Result

upower gives us the power state of our server and battery related stats .
ps filtered out gives us the ram used by the top 12 ram consumption process.

```bash
upower -i $(upower -e | grep 'battery')

ps -eo size,pid,user,command --sort -size | awk '{ hr=$1/1024 ; printf("%13.2f Mb ",hr) } { for ( x=4 ; x<=NF ; x++ ) { printf("%s ",$x) } print "" }' | head -n 12
```

My curtana now idles at:
•  0,3W , 900MB RAM used
•  67 days uptime (before I updated)
•  Running tailscale , vaultwarden, jellyfin and a amall go tsnet service .

Halium/Android layer stuff like Netmgrd,Qcrild and Cnd/Qseecomd occupy like 200mb of ram and cant be disabled without risking a brick .


