# Ubuntu Server 1TB NVMe LVM Partitioning Setup Guide

**Complete Guide for LVM Setup with File-Based Swap**

---

## Table of Contents

- [System Specifications](#system-specifications)
- [Partition Layout Overview](#partition-layout-overview)
- [Detailed Partition Information](#detailed-partition-information)
- [Installation Steps](#installation-steps)
- [Post-Installation Configuration](#post-installation-configuration)
- [LVM Management](#lvm-management)
- [Swap Management](#swap-management)
- [Verification Commands](#verification-commands)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Quick Reference](#quick-reference)

---

## System Specifications

| Component | Specification |
|-----------|--------------|
| **Storage Device** | `/dev/nvme0n1` |
| **Capacity** | 1TB NVMe SSD |
| **RAM** | 64GB |
| **GPU VRAM** | 16GB (NVIDIA RTX 5060 Ti) |
| **OS** | Ubuntu Server (22.04 LTS or newer) |
| **Partition Table** | GPT |
| **Volume Management** | LVM2 |
| **Swap Type** | File-based (8GB) |

---

## Partition Layout Overview

```
/dev/nvme0n1 (1TB)
├── nvme0n1p1    1GB      /boot/efi  (ESP, FAT32)
├── nvme0n1p2    2GB      /boot      (ext4)
└── nvme0n1p3    997GB    (LVM PV)
    └── vg0      997GB    (Volume Group)
        ├── root   40GB   /          (ext4)
        ├── var    200GB  /var       (ext4)
        ├── home   100GB  /home      (ext4)
        └── free   657GB  (unallocated for future use)

Swap: 8GB swap file at /swapfile (on root partition)
```

---

## Detailed Partition Information

### Physical Partitions

#### Partition 1: EFI System Partition (ESP)
- **Device**: `/dev/nvme0n1p1`
- **Size**: 1GB
- **Filesystem**: FAT32
- **Mount Point**: `/boot/efi`
- **Flags**: esp, boot
- **Purpose**: UEFI bootloader files

#### Partition 2: Boot Partition
- **Device**: `/dev/nvme0n1p2`
- **Size**: 2GB
- **Filesystem**: ext4
- **Mount Point**: `/boot`
- **Purpose**: Kernel, initramfs, GRUB configuration files

#### Partition 3: LVM Physical Volume
- **Device**: `/dev/nvme0n1p3`
- **Size**: ~997GB
- **Type**: Linux LVM
- **Purpose**: Contains all logical volumes

### Logical Volumes (LVM)

| LV Name | Size | Mount Point | Filesystem | Device Path | Purpose |
|---------|------|-------------|------------|-------------|---------|
| root | 40GB | / | ext4 | /dev/vg0/root | Root filesystem - OS, applications, system files |
| var | 200GB | /var | ext4 | /dev/vg0/var | Variable data - logs, databases, Docker, cache |
| home | 100GB | /home | ext4 | /dev/vg0/home | User home directories and personal data |
| (unallocated) | 657GB | - | - | - | Free space for future expansion |

### Swap Configuration

- **Type**: Swap File (recommended for flexibility)
- **Size**: 8GB
- **Location**: `/swapfile` (on root partition)
- **Swappiness**: 10 (minimal swap usage with 64GB RAM)

**Reasoning for 8GB Swap:**
- With 64GB RAM, 8GB provides adequate safety buffer
- File-based swap offers maximum flexibility for resizing
- No performance penalty on NVMe drives
- Can easily adjust size without LVM operations
- GPU VRAM (16GB) is managed separately by NVIDIA driver

---

## Installation Steps

### Prerequisites

Before beginning, ensure you have:
- Ubuntu Server installation media (USB or CD)
- Backup of any existing data
- Verified device name is `/dev/nvme0n1` (use `lsblk` command)
- Network connectivity for package installation

⚠️ **WARNING**: Double-check device names before proceeding! All data on the target drive will be erased.

---

### Step 1: Identify and Verify the Drive

```bash
# List all block devices
lsblk

# View detailed information about the NVMe drive
sudo fdisk -l /dev/nvme0n1

# Verify this is the correct drive
# WARNING: Replace /dev/nvme0n1 with your actual device if different
```

---

### Step 2: Install Required Tools

```bash
# Update package list
sudo apt update

# Install partitioning and LVM tools
sudo apt install parted gdisk lvm2 -y
```

---

### Step 3: Create Partitions

#### Method 1: Using gdisk (Recommended)

```bash
# Start gdisk partitioning tool
sudo gdisk /dev/nvme0n1
```

**Inside gdisk interactive prompt:**

```
# Create new GPT partition table
Command: o
Proceed? Y

# Partition 1: EFI System Partition (1GB)
Command: n
Partition number: 1
First sector: (press Enter for default)
Last sector: +1G
Hex code: EF00

# Partition 2: /boot (2GB)
Command: n
Partition number: 2
First sector: (press Enter for default)
Last sector: +2G
Hex code: 8300

# Partition 3: LVM Physical Volume (remaining space)
Command: n
Partition number: 3
First sector: (press Enter for default)
Last sector: (press Enter for default - uses all remaining space)
Hex code: 8E00

# Verify partition table
Command: p

# Write changes to disk
Command: w
Do you want to proceed? Y
```

#### Method 2: Using parted (Alternative)

```bash
sudo parted /dev/nvme0n1

# Inside parted:
(parted) mklabel gpt
(parted) mkpart primary fat32 1MiB 1025MiB
(parted) set 1 esp on
(parted) mkpart primary ext4 1025MiB 3073MiB
(parted) mkpart primary ext4 3073MiB 100%
(parted) set 3 lvm on
(parted) print
(parted) quit
```

---

### Step 4: Verify Partition Creation

```bash
# View partition layout
lsblk /dev/nvme0n1

# Detailed partition information
sudo fdisk -l /dev/nvme0n1
```

**Expected output:**
```
nvme0n1           1TB
├─nvme0n1p1       1G   (EFI)
├─nvme0n1p2       2G   (boot)
└─nvme0n1p3       ~997G (LVM)
```

---

### Step 5: Format EFI and Boot Partitions

```bash
# Format EFI System Partition (FAT32)
sudo mkfs.fat -F32 /dev/nvme0n1p1

# Format boot partition (ext4 with label)
sudo mkfs.ext4 -L boot /dev/nvme0n1p2
```

---

### Step 6: Set Up LVM Physical Volume

```bash
# Initialize partition as LVM Physical Volume
sudo pvcreate /dev/nvme0n1p3

# Verify Physical Volume creation
sudo pvdisplay
sudo pvs
```

---

### Step 7: Create LVM Volume Group

```bash
# Create Volume Group named 'vg0'
sudo vgcreate vg0 /dev/nvme0n1p3

# Verify Volume Group
sudo vgdisplay vg0
sudo vgs
```

---

### Step 8: Create Logical Volumes

**Note**: We create NO swap LV - swap will be file-based instead!

```bash
# Create root LV (40GB)
sudo lvcreate -L 40G -n root vg0

# Create var LV (200GB)
sudo lvcreate -L 200G -n var vg0

# Create home LV (100GB)
sudo lvcreate -L 100G -n home vg0

# Verify Logical Volumes
sudo lvdisplay
sudo lvs

# Check free space (should show ~657GB free)
sudo vgdisplay vg0 | grep Free
```

---

### Step 9: Format Logical Volumes

```bash
# Format root LV with ext4
sudo mkfs.ext4 -L root /dev/vg0/root

# Format var LV with ext4
sudo mkfs.ext4 -L var /dev/vg0/var

# Format home LV with ext4
sudo mkfs.ext4 -L home /dev/vg0/home
```

---

### Step 10: Mount Filesystems for Installation

```bash
# Mount root
sudo mount /dev/vg0/root /mnt

# Create mount point directories
sudo mkdir -p /mnt/boot
sudo mkdir -p /mnt/boot/efi
sudo mkdir -p /mnt/var
sudo mkdir -p /mnt/home

# Mount boot
sudo mount /dev/nvme0n1p2 /mnt/boot

# Mount EFI
sudo mount /dev/nvme0n1p1 /mnt/boot/efi

# Mount var
sudo mount /dev/vg0/var /mnt/var

# Mount home
sudo mount /dev/vg0/home /mnt/home

# Verify all mounts
df -h
lsblk
```

---

### Step 11: Get UUIDs for fstab Configuration

```bash
# Display all UUIDs
sudo blkid

# Or get specific UUIDs:
sudo blkid /dev/nvme0n1p1  # EFI
sudo blkid /dev/nvme0n1p2  # boot
sudo blkid /dev/vg0/root   # root
sudo blkid /dev/vg0/var    # var
sudo blkid /dev/vg0/home   # home

# Copy these UUIDs for the next step
```

---

### Step 12: Configure /etc/fstab

Edit `/mnt/etc/fstab` during installation or `/etc/fstab` on a running system:

```bash
sudo nano /mnt/etc/fstab
```

#### fstab Content (Using UUIDs - Recommended)

```fstab
# /etc/fstab: static file system information
# <file system>                           <mount point>  <type>  <options>              <dump>  <pass>

# Root filesystem
UUID=<root-uuid>                          /              ext4    errors=remount-ro      0       1

# Boot partition
UUID=<boot-uuid>                          /boot          ext4    defaults               0       2

# EFI System Partition
UUID=<efi-uuid>                           /boot/efi      vfat    umask=0077             0       1

# Var partition
UUID=<var-uuid>                           /var           ext4    defaults               0       2

# Home partition
UUID=<home-uuid>                          /home          ext4    defaults               0       2

# Swap file (will be created in step 13)
/swapfile                                 none           swap    sw                     0       0
```

#### Alternative: Using Device Mapper Paths

```fstab
# Alternative: Using device mapper paths instead of UUIDs
/dev/mapper/vg0-root    /          ext4    errors=remount-ro    0    1
/dev/nvme0n1p2          /boot      ext4    defaults             0    2
/dev/nvme0n1p1          /boot/efi  vfat    umask=0077           0    1
/dev/mapper/vg0-var     /var       ext4    defaults             0    2
/dev/mapper/vg0-home    /home      ext4    defaults             0    2
/swapfile               none       swap    sw                   0    0
```

**Note**: Replace `<uuid>` placeholders with actual UUIDs from `blkid` command.

---

### Step 13: Create Swap File

**Perform this after system installation is complete and system is booted.**

```bash
# Create 8GB swap file using fallocate
sudo fallocate -l 8G /swapfile

# Set correct permissions (important for security!)
sudo chmod 600 /swapfile

# Format as swap
sudo mkswap /swapfile

# Enable swap
sudo swapon /swapfile

# Verify swap is active
sudo swapon --show
free -h

# Ensure swap file entry is in /etc/fstab (from step 12)
grep swapfile /etc/fstab
```

---

### Step 14: Configure Swappiness

For a system with 64GB RAM, reduce swappiness to minimize swap usage:

```bash
# Check current swappiness (default is usually 60)
cat /proc/sys/vm/swappiness

# Set swappiness to 10 (only swap when really necessary)
sudo sysctl vm.swappiness=10

# Make it permanent
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Verify the change
cat /proc/sys/vm/swappiness
```

---

### Step 15: Install Ubuntu Base System

If performing manual installation:

```bash
# Use Ubuntu installer with 'Manual partitioning' option
# Point installer to use the mounted filesystems

# Or use debootstrap for minimal installation:
sudo debootstrap jammy /mnt http://archive.ubuntu.com/ubuntu

# Continue with standard Ubuntu installation procedures
```

---

### Step 16: Final Verification After Reboot

```bash
# Check all mounts
df -h
lsblk
mount | grep -E '(nvme|mapper)'

# Verify LVM setup
sudo pvs
sudo vgs
sudo lvs

# Check swap
sudo swapon --show
free -h

# Verify fstab entries are valid
sudo findmnt --verify

# Update boot configuration
sudo update-grub
sudo update-initramfs -u
```

---

## Post-Installation Configuration

### Enable TRIM for NVMe SSD Longevity

```bash
# Enable fstrim timer (recommended for SSDs/NVMe)
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer

# Verify timer is active
sudo systemctl status fstrim.timer

# Alternative: Add 'discard' option to fstab mount options for continuous TRIM
# Example: UUID=xxx / ext4 errors=remount-ro,discard 0 1
```

### Set Up System Monitoring

```bash
# Install monitoring tools
sudo apt install htop iotop ncdu -y

# Monitor disk usage
df -h
sudo lvs

# Monitor swap usage
free -h
vmstat 5

# Check I/O statistics (install sysstat if needed)
sudo apt install sysstat -y
iostat -x 5
```

---

## LVM Management

### Extending Logical Volumes

#### Example: Extend /var by 50GB

```bash
# Check available free space
sudo vgdisplay vg0 | grep Free

# Extend the logical volume
sudo lvextend -L +50G /dev/vg0/var

# Resize the filesystem
sudo resize2fs /dev/vg0/var

# Verify new size
df -h /var
sudo lvs
```

#### Example: Extend /home to Use All Remaining Free Space

```bash
# Extend using all free space in VG
sudo lvextend -l +100%FREE /dev/vg0/home

# Resize filesystem
sudo resize2fs /dev/vg0/home

# Verify
df -h /home
sudo lvs
```

---

### Creating New Logical Volumes

#### Example: Create 100GB Volume for Docker Data

```bash
# Create new LV
sudo lvcreate -L 100G -n docker vg0

# Format it
sudo mkfs.ext4 -L docker /dev/vg0/docker

# Create mount point and mount
sudo mkdir -p /var/lib/docker
sudo mount /dev/vg0/docker /var/lib/docker

# Add to fstab
echo '/dev/mapper/vg0-docker  /var/lib/docker  ext4  defaults  0  2' | sudo tee -a /etc/fstab

# Verify
df -h /var/lib/docker
sudo lvs
```

---

### Shrinking Logical Volumes

⚠️ **WARNING: DANGEROUS OPERATION - Always backup before shrinking!**

```bash
# Example: Shrink /home by 20GB
# 1. BACKUP ALL DATA FIRST!

# 2. Unmount the filesystem
sudo umount /home

# 3. Check filesystem for errors
sudo e2fsck -f /dev/vg0/home

# 4. Resize filesystem first (make it smaller)
sudo resize2fs /dev/vg0/home 80G

# 5. Reduce LV size
sudo lvreduce -L 80G /dev/vg0/home

# 6. Remount
sudo mount /home

# 7. Verify
df -h /home
```

---

### LVM Snapshots

```bash
# Create snapshot of root before system upgrade (10GB snapshot size)
sudo lvcreate -L 10G -s -n root_snapshot /dev/vg0/root

# List snapshots
sudo lvs

# Mount snapshot to view/recover files
sudo mkdir /mnt/snapshot
sudo mount /dev/vg0/root_snapshot /mnt/snapshot -o ro

# Browse snapshot
ls /mnt/snapshot

# Restore from snapshot if needed (CAREFUL!)
sudo lvconvert --merge /dev/vg0/root_snapshot
# System needs reboot to complete merge

# Remove snapshot when done
sudo umount /mnt/snapshot
sudo lvremove /dev/vg0/root_snapshot
```

---

## Swap Management

### Resize Swap File

#### Increase Swap to 16GB

```bash
# Turn off current swap
sudo swapoff /swapfile

# Remove old swap file
sudo rm /swapfile

# Create new 16GB swap file
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile

# Format and enable
sudo mkswap /swapfile
sudo swapon /swapfile

# Verify
free -h
sudo swapon --show
```

#### Decrease Swap to 4GB

```bash
sudo swapoff /swapfile
sudo rm /swapfile
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h
```

---

### Add Temporary Additional Swap

Useful during memory-intensive operations:

```bash
# Create temporary 16GB swap in /tmp
sudo fallocate -l 16G /tmp/emergency-swap
sudo chmod 600 /tmp/emergency-swap
sudo mkswap /tmp/emergency-swap
sudo swapon /tmp/emergency-swap

# Verify total swap
sudo swapon --show
free -h

# Remove when workload completes
sudo swapoff /tmp/emergency-swap
sudo rm /tmp/emergency-swap
```

---

### Monitor Swap Usage

```bash
# Current swap usage
free -h
cat /proc/swaps

# Real-time monitoring
watch -n 1 free -h

# Check swap activity
vmstat 5

# Historical swap usage (if sysstat installed)
sar -S
```

---

## Verification Commands

### Quick Health Check

```bash
# Overall disk usage
df -h

# LVM status summary
sudo pvs && sudo vgs && sudo lvs

# Memory and swap status
free -h

# All active mounts
mount | grep -E '(nvme|mapper|swap)'

# Verify fstab entries are valid
sudo findmnt --verify
```

---

### Detailed System Information

```bash
# Detailed LVM information
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay

# Block device tree with filesystems
lsblk -f

# Disk I/O statistics
iostat -x

# Filesystem usage with types
df -hT

# Inode usage (to check if running out of inodes)
df -i
```

---

## Troubleshooting

### LVM Volumes Not Found at Boot

**Problem**: System doesn't boot - LVM volumes not accessible

**Solution**:
```bash
# Boot from live USB
sudo vgscan
sudo vgchange -ay

# Mount and chroot
sudo mount /dev/vg0/root /mnt
sudo mount /dev/nvme0n1p2 /mnt/boot
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt

# Rebuild initramfs and GRUB
update-initramfs -u
update-grub

# Exit and reboot
exit
sudo reboot
```

---

### Swap File Not Activated at Boot

**Problem**: Swap file exists but not activated

**Solution**:
```bash
# Check fstab entry
grep swapfile /etc/fstab

# Verify file exists and has correct permissions
ls -lh /swapfile
# Should show: -rw------- (600 permissions)

# Fix permissions if needed
sudo chmod 600 /swapfile

# Manually activate
sudo swapon /swapfile

# Verify
sudo swapon --show
free -h
```

---

### Disk Space Full on /var or /

**Problem**: Partition is full

**Solution**:
```bash
# Find largest directories
sudo du -sh /var/* | sort -rh | head -10

# Clean package cache
sudo apt clean
sudo apt autoremove

# Clean old journal logs (keep last 7 days)
sudo journalctl --vacuum-time=7d

# For Docker cleanup (if Docker installed)
docker system prune -a

# If still full, extend the LV
sudo lvextend -L +50G /dev/vg0/var
sudo resize2fs /dev/vg0/var

# Verify
df -h /var
```

---

### Root Partition Full But VG Has Free Space

**Problem**: Root (/) is full but volume group has unallocated space

**Solution**:
```bash
# Check free space in VG
sudo vgdisplay vg0 | grep Free

# Extend root LV by 20GB
sudo lvextend -L +20G /dev/vg0/root

# Resize filesystem
sudo resize2fs /dev/vg0/root

# Verify
df -h /
sudo lvs
```

---

## Best Practices

1. **Leave 15-20% of VG space unallocated** for flexibility and future needs
2. **Monitor /var regularly** on servers (Docker, logs can grow quickly)
3. **Take LVM snapshots** before major system changes or upgrades
4. **Enable automatic TRIM** for SSD/NVMe longevity (fstrim.timer)
5. **Keep swap file size modest** with 64GB RAM (8-16GB is sufficient)
6. **Use UUIDs in fstab** for reliability across reboots and hardware changes
7. **Regular backups** of /home and critical /var data
8. **Monitor swap usage** - if consistently high, you need more RAM, not more swap
9. **Use lvs, vgs, pvs commands regularly** to track LVM space usage
10. **Document any changes** to LVM layout for future reference

---

## Quick Reference

### Essential Commands

```bash
# LVM overview
sudo lvs && sudo vgs && sudo pvs

# Disk usage
df -h && sudo lvs

# Memory and swap status
free -h && sudo swapon --show

# Block devices with filesystems
lsblk -f

# Formatted mount points
mount | column -t
```

### When to Extend Volumes

| Volume | Extend When | Typical Extension |
|--------|-------------|-------------------|
| root (/) | Reaches 80% capacity | +10-20GB |
| /var | Logs/Docker > 160GB (80% of 200GB) | +50-100GB |
| /home | User data > 80GB (80% of 100GB) | +50GB or as needed |
| New LV | For dedicated services | Create new LV for Docker, databases, etc. |

---

## Swap File vs Partition Summary

### Swap File Advantages ✓
- Easily resizable (seconds vs minutes)
- Can create multiple swap files temporarily
- No LVM space commitment required
- Modern Ubuntu default approach
- Hibernation fully supported (kernel 5.0+)
- Maximum flexibility for dynamic workloads

### Swap Partition Advantages
- Slightly more traditional approach
- Visible in LVM tools (lvs output)
- Consistent with rest of LVM setup

### Performance
**Negligible difference on NVMe drives** - modern kernels have optimized swap file performance to match partitions.

### Recommendation
**Use swap file for maximum flexibility**, especially with 64GB RAM where swap is rarely used.

---

## GPU and NVIDIA Notes

### NVIDIA RTX 5060 Ti - 16GB VRAM

- **VRAM Management**: Completely separate from system RAM
- **Swap Impact**: GPU VRAM does NOT affect system swap sizing decisions
- **Driver Installation**: Install `nvidia-driver-XXX` package after system setup
- **CUDA Support**: Install `nvidia-cuda-toolkit` if needed for ML/AI workloads
- **Monitoring**: Use `nvidia-smi` to monitor GPU memory usage

```bash
# Install NVIDIA drivers (example)
sudo apt update
sudo ubuntu-drivers list
sudo ubuntu-drivers install nvidia:535  # or latest stable version

# Monitor GPU
nvidia-smi

# Continuous monitoring
watch -n 1 nvidia-smi
```

---

## Final Setup Checklist

- [ ] Verified device name (`/dev/nvme0n1`)
- [ ] Created GPT partition table
- [ ] Created EFI, boot, and LVM partitions
- [ ] Initialized LVM PV, VG, and LVs
- [ ] Formatted all filesystems (FAT32, ext4)
- [ ] Configured `/etc/fstab` with correct UUIDs
- [ ] Created and activated 8GB swap file
- [ ] Set swappiness to 10
- [ ] Enabled `fstrim.timer` for SSD maintenance
- [ ] Verified all mounts after reboot
- [ ] Tested LVM commands (`lvs`, `vgs`, `pvs`)
- [ ] Documented setup for future reference
- [ ] Created this documentation for reference

---

## Advantages of This Setup

✓ **Maximum flexibility** with LVM for future growth  
✓ **Swap file** allows easy resizing without LVM complexity  
✓ **Separate /var** prevents logs from filling root partition  
✓ **Separate /home** allows OS reinstallation without data loss  
✓ **657GB unallocated** space ready for future expansion  
✓ **Boot partitions** outside LVM for maximum compatibility  
✓ **UEFI-compatible** with proper ESP setup  
✓ **Optimized for NVMe** with TRIM support  
✓ **Production-ready** configuration suitable for server workloads

---

## Additional Resources

### Official Documentation
- Ubuntu LVM Guide: https://ubuntu.com/server/docs/lvm
- LVM2 Documentation: `man lvm`
- Swap Management: `man swapon`, `man mkswap`
- fstab Documentation: `man fstab`

### Man Pages
```bash
man lvm
man pvcreate
man vgcreate
man lvcreate
man lvextend
man resize2fs
man mkswap
man swapon
man fstab
```

---

**Document Version**: 1.0  
**Created**: December 07, 2025  
**For**: Ubuntu Server 1TB NVMe LVM Setup with File-Based Swap  
**System**: 64GB RAM, NVIDIA RTX 5060 Ti (16GB VRAM)

---

*This guide provides a complete, production-ready configuration for Ubuntu Server with maximum flexibility for future expansion.*
