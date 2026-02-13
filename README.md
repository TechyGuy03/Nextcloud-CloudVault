# Nextcloud CloudVault Deployment

## Overview

This repository documents the deployment of a self hosted Nextcloud instance running on Proxmox.

The objective of this lab was to simulate a production style private cloud storage environment with strong security controls and realistic operational behavior.

### Key Design Goals

- Separation of OS and data storage
- LUKS encrypted data volume
- Controlled encrypted storage lifecycle
- Reverse proxy security model
- Cloudflare Tunnel remote access
- Practical real world architecture decisions

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

Inside fdisk:

```text
n
p
1
<ENTER>
<ENTER>
w
```

### Apply LUKS Encryption

```bash
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 cloudcrypt
```

### Format Mapper Device

```bash
sudo mkfs.ext4 /dev/mapper/cloudcrypt
```

### Mount Volume

```bash
sudo mount /dev/mapper/cloudcrypt /mnt/cloud
```

---

## Persistent Mount Configuration

Retrieve UUID:

```bash
sudo blkid
```

Update fstab:

```bash
sudo nano /etc/fstab
```

Example entry:

```text
UUID=<your-uuid> /mnt/cloud ext4 defaults,nofail 0 2
```

**Design Note**

The `nofail` option prevents emergency boot scenarios when the encrypted volume is intentionally left locked.

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

## Data Directory Migration

Stop Nextcloud:

```bash
sudo snap stop nextcloud
```

Create Nextcloud data marker:

```bash
echo "# Nextcloud data directory" | sudo tee /mnt/cloud/.ncdata
```

Sync data:

```bash
sudo rsync -Aax /var/snap/nextcloud/common/nextcloud/data/ /mnt/cloud/
```

Fix permissions:

```bash
sudo chown -R root:root /mnt/cloud
sudo chmod -R 750 /mnt/cloud
```

Update config:

```bash
sudo snap run nextcloud.occ config:system:set datadirectory --value="/mnt/cloud"
```

Restart:

```bash
sudo snap start nextcloud
```

---

## Encrypted Storage Operational Workflow

The Nextcloud data directory resides on a LUKS encrypted volume requiring manual unlock.

This behavior is intentional and mirrors real world security practices where sensitive storage is not automatically decrypted at boot.

---

## Hardened Unlock Script

Create script:

```bash
sudo nano /usr/local/sbin/unlock.sh
```

Script contents:

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCKFILE="/run/cloudvault.lock"
exec 9>"$LOCKFILE"
flock -n 9 || { echo "[ERROR] Another CloudVault operation is running."; exit 1; }

DEVICE="/dev/sdb1"
MAPPER="cloudcrypt"
MOUNTPOINT="/mnt/cloud"

echo "=== CloudVault UNLOCK ==="

echo "[INFO] Stopping Nextcloud (bounded wait)..."
sudo timeout 60 snap stop nextcloud || true

if sudo cryptsetup status "$MAPPER" >/dev/null 2>&1; then
  echo "[INFO] Mapping already open"
else
  echo "[INFO] Unlocking LUKS volume"
  sudo cryptsetup open "$DEVICE" "$MAPPER"
fi

if mount | grep -q " $MOUNTPOINT "; then
  echo "[INFO] Already mounted"
else
  echo "[INFO] Mounting volume"
  sudo mount "/dev/mapper/$MAPPER" "$MOUNTPOINT"
fi

echo "[INFO] Starting Nextcloud"
sudo snap start nextcloud

echo "[SUCCESS] CloudVault online"
```

Make executable:

```bash
sudo chmod +x /usr/local/sbin/unlock.sh
```

---

## Hardened Lock Script

Create script:

```bash
sudo nano /usr/local/sbin/lock.sh
```

Script contents:

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCKFILE="/run/cloudvault.lock"
exec 9>"$LOCKFILE"
flock -n 9 || { echo "[ERROR] Another CloudVault operation is running."; exit 1; }

MAPPER="cloudcrypt"
MOUNTPOINT="/mnt/cloud"

echo "=== CloudVault LOCK ==="

sudo snap stop --disable nextcloud || true

mount | grep -q " $MOUNTPOINT " && sudo umount "$MOUNTPOINT" || true

sudo cryptsetup status "$MAPPER" >/dev/null 2>&1 && sudo cryptsetup close "$MAPPER"

echo "[SUCCESS] CloudVault locked"
```

Make executable:

```bash
sudo chmod +x /usr/local/sbin/lock.sh
```

---

## Reverse Proxy Architecture

External access is provided via Cloudflare Tunnel.

No inbound ports are exposed to the WAN.

**Traffic Flow**

Client → Cloudflare Edge → Tunnel → Nextcloud Origin

---

## Reverse Proxy Security Configuration

Nextcloud must explicitly trust proxy headers to prevent IP spoofing and protocol confusion.

```bash
sudo snap run nextcloud.occ config:system:set trusted_proxies 0 --value="127.0.0.1"
sudo snap run nextcloud.occ config:system:set trusted_proxies 1 --value="::1"

sudo snap run nextcloud.occ config:system:set forwarded_for_headers 0 --value="HTTP_CF_CONNECTING_IP"

sudo snap run nextcloud.occ config:system:set overwriteprotocol --value="https"
sudo snap run nextcloud.occ config:system:set overwrite.cli.url --value="https://vault.techysec.com"
```

**Security Rationale**

- Prevent forged forwarding headers
- Trust Cloudflare client IP header
- Enforce HTTPS internally

---

## HTTPS Enforcement & HSTS

TLS termination occurs at Cloudflare.

HSTS is enforced at the edge.

Example header policy:

```text
Strict-Transport-Security: max-age=15552000; includeSubDomains; preload
```

**Security Benefits**

- Prevent protocol downgrade attacks
- Enforce HTTPS only client behavior
- Align with production best practices

---

## Cloudflare Tunnel Configuration

Install Cloudflared:

```bash
sudo apt update
sudo apt install cloudflared
```

Install tunnel service:

```bash
sudo cloudflared service install <token>
```

Example ingress policy:

```yaml
ingress:
  - hostname: vault.techysec.com
    service: http://localhost:80
  - service: http_status:404
```

---

## Database Security Characteristics

Nextcloud Snap deployment:

- Credentials automatically generated
- No default credentials
- UNIX socket communication
- No TCP exposure

---

## Security Design Decisions

- Data disk separated from OS disk
- Full LUKS volume encryption
- Manual unlock requirement
- Reverse proxy header hardening
- Remote access via Cloudflare Tunnel
- Minimal external attack surface

---

## Lessons Learned

- Snap deployments differ from package installs
- ncdata file is mandatory
- Encryption and mounts impact boot behavior
- Reverse proxy header trust is critical
- Cloudflare Tunnel simplifies secure exposure

---

## Future Enhancements

- Backup and snapshot strategy
- Health monitoring and alerting
- SIEM and log pipeline integration
- Secret management workflow
- Optional automated unlock with TPM

---

## Disclaimer

This repository represents a personal lab environment intended for educational and research purposes.

No sensitive credentials or production secrets are stored here.
