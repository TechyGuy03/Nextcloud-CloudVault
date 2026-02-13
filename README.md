# Nextcloud CloudVault Deployment

## Overview

This repository documents the deployment of a self-hosted Nextcloud instance running on Proxmox.

The objective of this lab was to simulate a production-style private cloud storage environment with strong security controls.

### Key Design Goals

- Separation of OS and data storage
- LUKS encrypted data volume
- Persistent mount configuration
- Cloudflare Tunnel remote access
- Practical, real world architecture decisions

---

## Environment Architecture

**Hypervisor:** Proxmox VE  
**Guest VM:** Ubuntu Server 24.04 LTS  
**Application:** Nextcloud (Snap Deployment)

### Storage Design

- scsi0 = OS Disk (50GB NVMe)
- scsi1 = Data Disk (1TB)
- LUKS encryption applied to the data disk

---

## Disk & Encryption Configuration

### Identify Disks

```bash
lsblk
```

### Partition Disk

```bash
sudo fdisk /dev/sdb
```

Inside `fdisk`:

```text
n
p
1
<ENTER>
<ENTER>
w
```

### Format Disk

```bash
sudo mkfs.ext4 /dev/sdb1
```

### Apply LUKS Encryption

```bash
sudo cryptsetup luksFormat /dev/sdb1
```

Open encrypted volume:

```bash
sudo cryptsetup open /dev/sdb1 cloudcrypt
```

Format decrypted mapper device:

```bash
sudo mkfs.ext4 /dev/mapper/cloudcrypt
```

Mount volume:

```bash
sudo mount /dev/mapper/cloudcrypt /mnt/cloud
```

---

## Persistent Mount Configuration

Retrieve UUID:

```bash
sudo blkid
```

Update `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Example entry:

```text
UUID=<your-uuid> /mnt/cloud ext4 defaults 0 2
```

Reload systemd and mount:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

> Note: If the encrypted volume is not unlocked at boot, this mount can fail.
> (See “Encrypted Storage Operational Workflow” for recovery + automation.)

---

## Nextcloud Deployment

Install Nextcloud via Snap:

```bash
sudo snap install nextcloud
```

Verify services:

```bash
sudo snap services nextcloud
```

---

## Web-Based Initial Configuration

After installing Nextcloud, complete the initial setup using the web interface.

Access Nextcloud:

```text
http://<VM-IP>
```

Example:

```text
http://192.168.x.x
```

During setup:

- Create the admin user
- Set a strong admin password
- Continue using Snap-managed defaults (Apache/MySQL/Redis)

You may be prompted to install recommended apps:
- Safe to skip for labs
- Can install later from the App Store

---

## Data Directory Migration

Stop Nextcloud:

```bash
sudo snap stop nextcloud
```

Create Nextcloud data marker:

```bash
echo "# Nextcloud data directory" | sudo tee /mnt/cloud/.ncdata
```

Sync data to the encrypted volume:

```bash
sudo rsync -Aax /var/snap/nextcloud/common/nextcloud/data/ /mnt/cloud/
```

Fix permissions:

```bash
sudo chown -R root:root /mnt/cloud
sudo chmod -R 750 /mnt/cloud
```

Update Nextcloud config:

```bash
sudo nextcloud.occ config:system:set datadirectory --value="/mnt/cloud"
```

Restart services:

```bash
sudo snap start nextcloud
```

---

## Encrypted Storage Operational Workflow

The Nextcloud data directory resides on a LUKS-encrypted volume requiring manual unlock.

This behavior is intentional and mirrors real-world security practices where sensitive storage is not automatically decrypted at boot.

### Unlock Workflow (Post-Reboot / Maintenance)

```bash
sudo cryptsetup open /dev/sdb1 cloudcrypt
sudo mount /dev/mapper/cloudcrypt /mnt/cloud
sudo snap start nextcloud
```

Verify mount:

```bash
df -h | grep /mnt/cloud
```

### Lock Workflow (Shutdown / Security)

```bash
sudo snap stop nextcloud
sudo umount /mnt/cloud
sudo cryptsetup close cloudcrypt
```

### Failure Scenario Behavior

If the encrypted disk is not unlocked at boot:

- `/mnt/cloud` mount fails
- Nextcloud may error or show missing files
- System may enter emergency mode depending on fstab configuration

Recovery:

```bash
sudo cryptsetup open /dev/sdb1 cloudcrypt
sudo mount /dev/mapper/cloudcrypt /mnt/cloud
sudo snap start nextcloud
```

---

## Unlock Script Automation

Create script:

```bash
sudo nano /usr/local/sbin/unlock-cloudvault.sh
```

Script contents:

```bash
#!/bin/bash

echo "----------------------------------------"
echo "CloudVault Unlock Workflow"
echo "----------------------------------------"

DEVICE="/dev/sdb1"
MAPPER="cloudcrypt"
MOUNTPOINT="/mnt/cloud"

if [ ! -b "$DEVICE" ]; then
  echo "[ERROR] LUKS device not found: $DEVICE"
  exit 1
fi

if [ -e "/dev/mapper/$MAPPER" ]; then
  echo "[INFO] Volume already unlocked"
else
  echo "[INFO] Unlocking LUKS volume..."
  sudo cryptsetup open "$DEVICE" "$MAPPER" || exit 1
fi

if mount | grep -q " $MOUNTPOINT "; then
  echo "[INFO] Volume already mounted"
else
  echo "[INFO] Mounting volume..."
  sudo mount "/dev/mapper/$MAPPER" "$MOUNTPOINT" || exit 1
fi

if [ ! -f "$MOUNTPOINT/.ncdata" ]; then
  echo "[WARNING] .ncdata file missing - Nextcloud may reject data directory"
else
  echo "[INFO] Nextcloud data marker verified"
fi

echo "[INFO] Starting Nextcloud services..."
sudo snap start nextcloud

echo "[SUCCESS] CloudVault online"
df -h | grep "$MOUNTPOINT"
```

Make executable:

```bash
sudo chmod +x /usr/local/sbin/unlock-cloudvault.sh
```

Run:

```bash
sudo /usr/local/sbin/unlock-cloudvault.sh
```

---

## Lock Script Automation

Create script:

```bash
sudo nano /usr/local/sbin/lock-cloudvault.sh
```

Script contents:

```bash
#!/bin/bash

echo "----------------------------------------"
echo "CloudVault Lock Workflow"
echo "----------------------------------------"

MAPPER="cloudcrypt"
MOUNTPOINT="/mnt/cloud"

echo "[INFO] Stopping Nextcloud services..."
sudo snap stop nextcloud

if mount | grep -q " $MOUNTPOINT "; then
  echo "[INFO] Unmounting $MOUNTPOINT..."
  sudo umount "$MOUNTPOINT"

  if mount | grep -q " $MOUNTPOINT "; then
    echo "[ERROR] Failed to unmount $MOUNTPOINT (still mounted)."
    echo "[TIP] Check what's using it: sudo lsof +f -- $MOUNTPOINT"
    exit 1
  fi
else
  echo "[INFO] $MOUNTPOINT is not mounted (skipping unmount)."
fi

if [ -e "/dev/mapper/$MAPPER" ]; then
  echo "[INFO] Closing LUKS mapping $MAPPER..."
  sudo cryptsetup close "$MAPPER" || exit 1
else
  echo "[INFO] LUKS mapping $MAPPER is not open (skipping close)."
fi

echo "[SUCCESS] CloudVault locked (Nextcloud stopped, volume unmounted, mapping closed)."
```

Make executable:

```bash
sudo chmod +x /usr/local/sbin/lock-cloudvault.sh
```

Run:

```bash
sudo /usr/local/sbin/lock-cloudvault.sh
```

---

## Cloudflare Tunnel Configuration

Install Cloudflared:

```bash
sudo apt update
sudo apt install cloudflared
```

Install tunnel service (token-based):

```bash
sudo cloudflared service install <token>
```

Verify service:

```bash
sudo systemctl status cloudflared --no-pager -l
```

Tunnel routes should point to the local Nextcloud service:

```yaml
ingress:
  - hostname: vault.techysec.com
    service: http://localhost:80
  - service: http_status:404
```

If Nextcloud reports a trusted domains error, add the hostname:

```bash
sudo nextcloud.occ config:system:set trusted_domains 1 --value="vault.techysec.com"
```

---

## Security Design Decisions

- Data disk separated from OS disk
- Full LUKS volume encryption
- Manual unlock requirement (no auto-unlock)
- Remote access via Cloudflare Tunnel (no inbound port forwarding)
- Minimal external attack surface

---

## Lessons Learned

- Snap-based Nextcloud behaves differently than package deployments
- .ncdata file is mandatory for data directory recognition
- Encryption + mounts can trigger emergency boot scenarios
- systemd requires daemon reload after fstab modification
- Cloudflare Tunnel simplifies secure remote exposure

---

## Future Enhancements

- Backup and snapshot strategy
- Multi-user onboarding workflow
- Network segmentation via VLANs
- SIEM and log monitoring integration
- Optional: tighter automation + health checks

---

## Disclaimer

This repository represents a personal lab environment intended for educational and research purposes.

No sensitive credentials or production secrets are stored here.
