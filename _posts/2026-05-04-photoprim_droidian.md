---
title: photoprism on droidian 
description: install PhotoPrism image server on a droidian compatible phone.
image: assets/images/photoprism.png
categories:
    - gallery 
    - droidian
tags:
    - gallery
    - droidian
---

# PhotoPrism on Droidian , All on Host

Droidian has limited container support, so we wil install everything natively. This guide builds a low-power PhotoPrism server on a Redmi Note 9S with an external USB drive.

The stack is a dependency chain: **Toshiba USB Drive >MariaDB > PhotoPrism**. If the drive sleeps, the services wait. If you touch the drive, the whole stack wakes.

Why: 
It covers three critical things:
• Prevents "Internal Suicide": If the drive doesn't have power during boot, the chain blocks MariaDB and PhotoPrism from starting. This stops them from creating "fake" empty databases that would fill up your phone’s tiny internal storage and brick the OS.
• Automates Recovery: Instead of you manually fixing things, the chain waits for the drive to "wake up." Once the power stabilizes and the drive appears, the apps see it and start themselves automatically.
• Protects Data Integrity: It ensures the "Librarian" (MariaDB) never tries to write to a "Missing Bookshelf" (the drive), which prevents database corruption during power dips.
In short: It turns a potential system crash into a temporary pause, keeping your phone fast and your photos safe.


---

## Phase 1: The Foundation (MariaDB and Hardware)

### 1. Install MariaDB natively
Lighter on RAM than Docker.

```bash
sudo apt update
sudo apt install mariadb-server -y
```

### 2. Migrate existing database (optional)
If you are moving from a Raspberry Pi, if you are not using the same drive that holds the files , copy your old database folder to the drive first  then fix ownership.

```bash
sudo chown -R mysql:mysql /mnt/drive/photoprism/database
sudo chmod -R 750 /mnt/drive/photoprism/database
```

### 3. Fix AppArmor for /mnt access (Droidian/Ubuntu)
Droidian blocks MariaDB from /mnt by default. Create a local override if you hit permission denied.

```bash
sudo nano /etc/apparmor.d/local/usr.sbin.mariadbd
```
Add:
```
/mnt/drive/photoprism/database/ r,
/mnt/drive/photoprism/database/** rwk,
```
Reload:
```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mariadbd
```

### 4. Ensure base directory exists
MariaDB expects /var/lib/mysql even if datadir is elsewhere.

```bash
sudo mkdir -p /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
```

### 5. Restart MariaDB
```bash
sudo systemctl stop mariadb
sudo systemctl start mariadb
```

### 6. Mount the external drive
Find your drive UUID.

```bash
lsblk -f
# Example UUID: d0289834-0b0d-47ba-bfc4-ebb6fa6f5827
```

Create mount point and edit fstab.

```bash
sudo mkdir -p /mnt/drive
sudo nano /etc/fstab
```

Add this line. The `noauto,x-systemd.automount` trick prevents boot failures when the USB port is underpowered. The drive mounts only when first accessed.

```
UUID=d0289834-0b0d-47ba-bfc4-ebb6fa6f5827 /mnt/drive ext4 defaults,noauto,x-systemd.automount,x-systemd.idle-timeout=600,nofail 0 2
```

Apply:
```bash
sudo systemctl daemon-reload
```

---

## Phase 2: Connecting Hardware to Software

### 1. Make MariaDB wait for the drive
Create a systemd drop-in.

```bash
sudo mkdir -p /etc/systemd/system/mariadb.service.d/
sudo nano /etc/systemd/system/mariadb.service.d/override.conf
```

Content:
```ini
[Unit]
RequiresMountsFor=/mnt/drive
```

This forces MariaDB to stay off until /mnt/drive is confirmed mounted. It solves the crash when the drive is in standby.

### 2. Point MariaDB to the external datadir
Edit the server config.

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Use these low-memory settings for a 4GB phone:

```ini
[mysqld]
user = mysql
pid-file = /run/mysqld/mysqld.pid
basedir = /usr
datadir = /mnt/drive/photoprism/database

# PhotoPrism required settings
transaction-isolation = READ-COMMITTED
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Mobile tuning
innodb_buffer_pool_size = 128M
max_connections = 50
```

Reload MariaDB:
```bash
sudo systemctl restart mariadb
```

### 3. Verify databases
```bash
sudo mariadb -u root -p -e "SHOW DATABASES;"
```

If you migrate from an old database use you old credential

---

## Phase 3: Installing PhotoPrism Natively

### 1. Install dependencies
These let PhotoPrism read HEIC, RAW, and video.

```bash
sudo apt update
sudo apt install -y curl libheif-examples libvips42 libexif12 libexif-dev ffmpeg
```

### 2. Install the ARM64 deb
PhotoPrism provides static builds.

```bash
curl -sLO https://dl.photoprism.app/pkg/linux/deb/photoprism_latest_arm64.deb
sudo dpkg -i photoprism_latest_arm64.deb
sudo ln -sf /opt/photoprism/bin/photoprism /usr/local/bin/photoprism
photoprism --version
```

### 3. Create the systemd service

```bash
sudo nano /etc/systemd/system/photoprism.service
```

```ini
[Unit]
Description=PhotoPrism AI-Powered Photos App
After=mariadb.service
RequiresMountsFor=/mnt/drive

[Service]
Type=simple
User=photoprism
Group=photoprism
WorkingDirectory=/opt/photoprism
ExecStart=/usr/local/bin/photoprism start
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable it later after config.

### 4. Configure paths
All data lives on the Toshiba drive.

```bash
sudo nano /etc/photoprism/defaults.yml
```

Example config:

```yaml
ConfigPath: "~/.config/photoprism"
StoragePath: "/mnt/drive/photoprism/storage"
OriginalsPath: "/mnt/drive/photos"
ImportPath: "/mnt/drive/photoimport"

AdminUser: "ippo"
AdminPassword: "CHANGE-ME-STRONG-PASSWORD"
AuthMode: "password"

HttpHost: "0.0.0.0"
HttpPort: 2342
HttpCompression: "gzip"

DatabaseDriver: "mysql"
DatabaseServer: "127.0.0.1:3306"
DatabaseName: "photoprism"
DatabaseUser: "photoprism"
DatabasePassword: "CHANGE-ME"

# Mobile optimization
Workers: 1
IndexWorkers: 1
LowMem: true
DatabaseConns: 5
DatabaseConnsIdle: 2

DisableTLS: false
DefaultTLS: true
JpegQuality: 85
DetectNSFW: false
UploadNSFW: true
```

**Critical fix:** `HttpHost: "0.0.0.0"` not `127.0.0.1`. This exposes the UI on your LAN at 192.168.1.198:2342.

Fix ownership:
```bash
sudo chown -R photoprism:photoprism /mnt/drive/photoprism/storage
```

### 5. Test first run
```bash
photoprism start
```

Check logs in another terminal:
```bash
sudo journalctl -u photoprism -f
```

Then enable:
```bash
sudo systemctl enable --now photoprism
```

---

## Phase 4: Resource Optimization

1. **Workers = 1**  Prevents using all 8 cores and overheating the Redmi Note 9S
2. **innodb_buffer_pool_size = 128M** Keeps MariaDB under 200MB RAM
3. **Swap 2GB** Acts as emergency RAM for face recognition
4. **x-systemd.idle-timeout=600**  Lets the drive sleep after 10 minutes idle

---

## The Boot Flow

1. Droidian boots, /mnt/drive is not mounted yet
2. systemd tries MariaDB, sees RequiresMountsFor, waits
3. You access PhotoPrism, automount triggers, drive spins up
4. MariaDB starts using /mnt/drive/photoprism/database
5. PhotoPrism starts and serves UI on port 2342

---

## Key Commands

**Health check the whole chain:**
```bash
systemctl status mnt-drive.mount mariadb photoprism
```

**Check hardware:**
```bash
lsblk -f
```

**Check memory:**
```bash
free -h
```

**Live logs:**
```bash
sudo journalctl -u photoprism -f
sudo journalctl -u mariadb -f
```

**Monitor heavy lifting:**
```bash
top -b -n 1 | grep photoprism
```

**Emergency RAM clear:**
```bash
sudo sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
clear:**
```


### Verdict , initial impressions

The PhotoPrism user experience I had is better on the droidian micro server than it was on my raspberry pi 400 .

Browsing my assets is snappier from the web ui and from the iOS app prismatic.

Indexing also looks snappier. Even though we set 1 worker against 2 that I had on raspberry pi importing 100 photos took like 10 - 15 mins tensorflow enabled .
Keep in mind without the 1 worker parameter droidian crashed. 

There is extra hardware required:
Plugging in an external hard drive on a phone is not gonna just work . Power from the usb c port is not sufficient to spin the drive .
I used an externally powered usb hub to power the drive. 
Also the dual purpose on a single port problem has to be addressed :
You need to power the phone and use otg at the same time as the phone battery will deplete at some point . There are some option for Y splitters on the market . I'm waiting my 10$ splitter from Amazon in the next days .


That's all , if you have a droidian compatible around don't hesitate to give it a go .

