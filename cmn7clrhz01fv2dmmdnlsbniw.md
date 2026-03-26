---
title: "Fix Kernel Boot Errors After GCP to AWS Migration"
seoTitle: "Fix GCP to AWS Kernel Boot Errors (EC2 Guide)"
seoDescription: "Fix EC2 boot failures after GCP migration by replacing kernels and enabling ENA/NVMe drivers with this step-by-step guide.
"
datePublished: 2026-03-26T10:47:17.883Z
cuid: cmn7clrhz01fv2dmmdnlsbniw
slug: fix-gcp-to-aws-kernel-boot-errors
cover: https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/262300be-c6a9-496c-b8f3-84ce4ad06c8e.png
ogImage: https://cdn.hashnode.com/uploads/og-images/66f590c9120870cbbe3a72f4/89905991-3686-471a-8612-5787568c281d.png
tags: ubuntu, ec2, linux, aws, cloud-computing, devops, gcp, cloud-migration

---

> So your VM import completed successfully, you launched the EC2 instance — and it won't boot. This is one of the most common issues when doing a disk-level migration from GCP to AWS. This guide explains **why it happens** and walks you through every command to fix it.

* * *

## Why Does the Kernel Break?

GCP Compute Engine VMs run a **GCP-optimized kernel** (`linux-image-*-gcp`). This kernel is compiled with:

*   GCP-specific virtio drivers
    
*   GCP's virtual network and disk hardware abstractions
    
*   Google's custom boot parameters and GRUB configuration
    
*   No support for AWS Nitro/Xen hypervisor drivers (ENA, NVMe, xen-blkfront)
    

When you import the disk image into AWS, the EC2 hypervisor is fundamentally different. AWS Nitro instances use **ENA** (Elastic Network Adapter) for networking and **NVMe** for storage. The GCP kernel has none of these drivers, so the instance can't attach its root disk or network interface — and it hangs or panics at boot.

**The fix:** Replace the GCP kernel with the standard **Ubuntu generic kernel**, inject AWS drivers into initramfs, and update GRUB.

* * *

## Overview of What We're Doing

1.  Mount the RAW disk image on the converter EC2
    
2.  `chroot` into the mounted disk
    
3.  Fix DNS inside the chroot environment
    
4.  Install the generic Ubuntu kernel
    
5.  Add AWS ENA + NVMe drivers to initramfs
    
6.  Set the generic kernel as the default in GRUB
    
7.  Remove all GCP kernels
    
8.  Update initramfs and GRUB
    
9.  Exit, unmount, and re-upload
    

* * *

## Prerequisites

*   The `analytics-server.raw` file on your EC2 converter instance
    
*   The converter EC2 has internet access (for `apt-get`)
    
*   Ubuntu 22.04 (Jammy) on the source disk — adjust repo URLs if using a different version
    

* * *

## Step 1 — Set Up the Loop Device and Mount the RAW Disk

The RAW file is a full disk image. We use a **loop device** to attach it as a block device, then mount its partition.

```bash
# Create the mount point
sudo mkdir -p /mnt/gcp-disk

# Attach the RAW file as a loop device (auto-detect partitions with -P)
sudo losetup -fP analytics-server.raw

# Find the loop device that was assigned
sudo losetup -a
# Example output: /dev/loop3: []: (/home/ubuntu/analytics-server.raw)

# Inspect the partition table
sudo fdisk -l /dev/loop3
# Look for the partition — typically /dev/loop3p1
```

* * *

## Step 2 — Mount the Root Partition

```bash
sudo mount /dev/loop3p1 /mnt/gcp-disk
```

> If you see `wrong fs type` or `bad superblock`, run `sudo fdisk -l /dev/loop3` and confirm the correct partition number. Use `p2` if `p1` is a boot/EFI partition.

* * *

## Step 3 — Mount Virtual Filesystems for chroot

A basic `mount` isn't enough for `apt-get` to work inside the chroot. You need to bind-mount the host's virtual filesystems (`/proc`, `/sys`, `/dev`) into the chroot environment.

```bash
sudo mount --bind /proc  /mnt/gcp-disk/proc
sudo mount --bind /sys   /mnt/gcp-disk/sys
sudo mount --bind /dev   /mnt/gcp-disk/dev
sudo mount --bind /dev/pts /mnt/gcp-disk/dev/pts
sudo mount --bind /run   /mnt/gcp-disk/run
```

> **Why?** `apt-get` calls scripts that need `/proc` for process info and `/dev` to interact with devices. Without these mounts, package installation will fail or produce broken kernel images.

* * *

## Step 4 — Fix DNS Inside the chroot Environment

GCP's resolv.conf likely points to GCP's internal DNS (`169.254.169.254` or a GCP-private IP). This will not resolve inside your AWS converter EC2's chroot. Copy the host's DNS config before entering.

```bash
# Copy the host's working DNS config into the mounted disk
sudo cp /etc/resolv.conf /mnt/gcp-disk/etc/resolv.conf
```

* * *

## Step 5 — Enter the chroot

```bash
sudo chroot /mnt/gcp-disk /bin/bash
```

You are now "inside" the GCP disk as if it were your root filesystem.

**Verify DNS works:**

```bash
cat /etc/resolv.conf
# Should show 8.8.8.8 or your host's DNS

ping -c 2 archive.ubuntu.com
# Should get replies
```

If `ping` doesn't work, manually override the DNS:

```bash
echo "nameserver 8.8.8.8"   >  /etc/resolv.conf
echo "nameserver 1.1.1.1"   >> /etc/resolv.conf
```

* * *

## Step 6 — Fix APT Sources (Override GCP Repos)

GCP VMs often have Google-specific APT sources or restricted mirrors. Replace with standard Ubuntu repos.

```bash
cat > /etc/apt/sources.list <<EOF
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu jammy-security main restricted universe multiverse
EOF
```

> Replace `jammy` with your Ubuntu codename if different (e.g., `focal` for 20.04, `noble` for 24.04). Check with `lsb_release -cs`.

Update the package index:

```bash
apt-get update
```

* * *

## Step 7 — Install the Generic Ubuntu Kernel

This installs the standard, hardware-agnostic Ubuntu kernel that works on any hypervisor including AWS.

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  linux-image-generic \
  linux-headers-generic \
  linux-modules-extra-generic
```

> **Note:** `DEBIAN_FRONTEND=noninteractive` prevents `apt-get` from asking interactive questions that would hang the terminal inside a chroot. If `linux-modules-extra-generic` fails, try without it:

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  linux-image-generic \
  linux-headers-generic
```

Verify the installed kernel version:

```bash
ls /boot/vmlinuz-*
# e.g., /boot/vmlinuz-5.15.0-173-generic
```

Note the exact version — you'll need it for GRUB configuration.

* * *

## Step 8 — Add AWS ENA and NVMe Drivers to initramfs

**This is critical.** Without these drivers baked into the initial RAM disk, the kernel won't be able to find the root disk or bring up the network interface on EC2.

```bash
echo -e "ena\nnvme\nxen-blkfront\nxen-netfront" >> /etc/initramfs-tools/modules
```

Verify the modules were added:

```bash
cat /etc/initramfs-tools/modules
# Should contain: ena, nvme, xen-blkfront, xen-netfront
```

> **Theory:**
> 
> *   `ena` — AWS Elastic Network Adapter driver (required for networking on Nitro instances)
>     
> *   `nvme` — NVMe storage driver (required for EBS volumes on Nitro instances)
>     
> *   `xen-blkfront` / `xen-netfront` — Xen paravirtual drivers (required for older instance types like t2, m4)
>     

* * *

## Step 9 — Update initramfs for the Generic Kernel

Rebuild the initial RAM disk to include the new drivers:

```bash
update-initramfs -u -k 5.15.0-173-generic
```

Replace `5.15.0-173-generic` with the exact version from Step 7.

* * *

## Step 10 — Set the Generic Kernel as the Default Boot Entry

Tell GRUB to boot the generic kernel by default instead of the GCP kernel.

```bash
cat > /etc/default/grub <<EOF
GRUB_DEFAULT="gnulinux-5.15.0-173-generic-advanced-b9bb90b2-7129-4d59-b97b-04f07739a8ac"
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=$(lsb_release -i -s 2>/dev/null || echo Debian)
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX="console=ttyS0,115200 console=tty1"
GRUB_TERMINAL="console serial"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
EOF
```

> **Why** `console=ttyS0`**?** AWS EC2 console output (visible in EC2 → Actions → Monitor → Get System Log) is routed through the serial console. Without this, you won't see boot logs if something goes wrong.

> **Finding your GRUB\_DEFAULT ID:** Run `grep -i "menuentry\|linux\|initrd" /boot/grub/grub.cfg | head -30` and locate the `gnulinux-...-generic-advanced-...` entry for your kernel version. Copy it exactly.

* * *

## Step 11 — Remove All GCP Kernels

The GCP kernel must be fully removed so GRUB doesn't accidentally fall back to it.

```bash
DEBIAN_FRONTEND=noninteractive apt-get remove --purge -y \
  linux-image-6.8.0-1052-gcp \
  linux-image-6.8.0-1048-gcp \
  linux-headers-6.8.0-1052-gcp \
  linux-headers-6.8.0-1048-gcp \
  linux-modules-6.8.0-1052-gcp \
  linux-modules-6.8.0-1048-gcp \
  linux-gcp \
  linux-headers-gcp \
  linux-image-gcp

apt-get autoremove -y
```

> List your exact GCP kernel versions first with: `dpkg -l | grep gcp`

You can also remove GCP-specific agents that serve no purpose on AWS:

```bash
DEBIAN_FRONTEND=noninteractive apt-get remove --purge -y \
  google-cloud-cli \
  google-osconfig-agent \
  google-compute-engine

apt-get autoremove -y
```

* * *

## Step 12 — Update GRUB

Regenerate the GRUB config to reflect all the changes:

```bash
update-grub
```

**Verify only the generic kernel is listed:**

```bash
grep "menuentry\|vmlinuz" /boot/grub/grub.cfg | grep -v "#"
```

You should see only `5.15.0-*-generic` entries — no `gcp` entries.

* * *

## Step 13 — Exit chroot and Unmount Everything

```bash
# Exit the chroot
exit

# Unmount all virtual filesystems in reverse order
sudo umount /mnt/gcp-disk/proc
sudo umount /mnt/gcp-disk/sys
sudo umount /mnt/gcp-disk/dev/pts
sudo umount /mnt/gcp-disk/dev
sudo umount /mnt/gcp-disk/run

# Unmount the root partition
sudo umount /mnt/gcp-disk

# Detach the loop device
sudo losetup -d /dev/loop3
```

> **Order matters.** Always unmount child mounts before the parent (`dev/pts` before `dev`). If unmount fails with "target is busy", run `sudo fuser -km /mnt/gcp-disk` to kill processes using it.

* * *

## Step 14 — Upload the Fixed RAW to S3

```bash
# Verify the file is still intact
ls -lh ~/analytics-server.raw

# Upload to S3 with a new key to distinguish from the original
aws s3 cp ~/analytics-server.raw \
  s3://YOUR_S3_BUCKET/migrations/analytics-server-fixed.raw \
  --no-progress
```

* * *

## Step 15 — Re-Import as AMI

```bash
aws ec2 import-image \
  --description "analytics-server-new migration v2" \
  --disk-containers '[
    {
      "Description": "analytics-server-fixed",
      "Format": "RAW",
      "UserBucket": {
        "S3Bucket": "YOUR_S3_BUCKET",
        "S3Key": "migrations/analytics-server-fixed.raw"
      }
    }
  ]' \
  --profile=company
```

Track progress:

```bash
aws ec2 describe-import-image-tasks --import-task-ids import-ami-XXXXXXXX
```

Once complete, you'll receive an AMI ID. Launch your instance from it — it should now boot cleanly on EC2.

* * *

## Troubleshooting

### `apt-get update` fails inside chroot

*   DNS isn't working. Re-run: `echo "nameserver 8.8.8.8" > /etc/resolv.conf`
    
*   Virtual filesystems not mounted. Check: `mount | grep gcp-disk`
    

### `update-initramfs` says no kernel found

*   The generic kernel wasn't installed correctly. Re-run `apt-get install linux-image-generic` and check `ls /boot/vmlinuz-*`
    

### Instance still won't boot after re-import

*   Check the EC2 System Log: EC2 Console → Instance → Actions → Monitor and troubleshoot → Get system log
    
*   Look for lines like `dracut-initqueue timeout` or `VFS: Cannot open root device` — these point to missing NVMe/ENA drivers
    
*   Verify modules were added: the `cat /etc/initramfs-tools/modules` output should include `ena` and `nvme`
    

### GRUB\_DEFAULT not matching

*   The GRUB entry ID must match **exactly** what's in `/boot/grub/grub.cfg`. Grab it with:
    
    ```bash
    grep "gnulinux.*generic.*advanced" /boot/grub/grub.cfg | head -3
    ```
    

* * *

## Summary

| Step | Action |
| --- | --- |
| Mount loop device | Attach RAW file as block device |
| Bind-mount virtual FS | Enable apt-get inside chroot |
| Fix DNS | Point to public resolvers |
| Fix APT sources | Use standard Ubuntu mirrors |
| Install generic kernel | `linux-image-generic` + headers |
| Add AWS drivers | ENA, NVMe, Xen into initramfs modules |
| Update initramfs | Rebuild initrd with new drivers |
| Set GRUB default | Point GRUB to generic kernel entry |
| Remove GCP kernels | Purge all `*-gcp` packages |
| Update GRUB | Regenerate GRUB config |
| Unmount cleanly | Detach loop device |
| Re-upload & re-import | Push fixed RAW back to AWS |

Once your instance is running on AWS with the generic kernel, you can proceed to install the SSM Agent, AWS CloudWatch agent, or any other AWS-native tooling to replace the GCP equivalents you removed.