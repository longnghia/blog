---
date: "2026-02-17T09:20:00+07:00"
draft: false
title: "Adding and Passing Through Disks on Proxmox"
summary: "Identify, format, and mount an extra drive on the host, then bind it into an LXC container and on into Docker — plus health checks and spin-down."
categories:
  - Guides
tags:
  - proxmox
  - lxc
  - disk
  - fstab
  - nas
  - server
---

Adding a drive to a Proxmox box is four steps that each have their own way of going wrong: identify it, format it, mount it on the host, then hand it down through the LXC container to whatever needs it. This is the sequence I follow, and the checks that tell me each step actually worked.

Part of a set — see also [Proxmox Notes: LXC Containers, Templates, and First-Boot Setup](../proxmox-lxc-basics/) and [Self-Hosting Services in Proxmox LXCs](../proxmox-services/).

## 1. Identify the drive

Don't guess at device names. `/dev/sdb` today can be `/dev/sdc` after a reboot, which is exactly why the fstab step later uses UUIDs.

```sh
lsblk                                              # all block devices
lsblk -o NAME,LABEL,FSTYPE,SIZE,MOUNTPOINT         # the useful columns
blkid                                              # UUIDs and filesystem types
fdisk -l | grep '^Disk'                            # sizes, at a glance
```

To see what filesystem something already has:

```sh
df -T
df -T /dev/sda
```

Confirm the size and label match the drive you physically installed before you format anything.

## 2. Format

XFS or ext4 both work fine. XFS is what I use for bulk media storage.

```sh
mkfs.xfs /dev/sdb -f          # -f forces over an existing filesystem
mkfs.ext4 -F /dev/sdb         # note: ext4's force flag is -F, not -f
```

That flag difference has bitten me. `mkfs.xfs` takes `-f`; `mkfs.ext4` (really `mke2fs`) takes `-F`. Passing the wrong one gets you a usage error, which is at least better than the alternative.

For exFAT — worth it only if the drive also needs to be readable on Windows and macOS:

```sh
apt install exfat-fuse exfatprogs
mkfs.exfat -n ugreen /dev/sdc
fsck.exfat /dev/sdc
```

## 3. Mount on the host

```sh
mkdir /mnt/nas/
mount /dev/sdb /mnt/nas/
```

Verify before moving on:

```sh
df -h /mnt/nas
```

If the output shows the drive rather than the root filesystem, the mount took.

### Make it survive a reboot

Use the UUID from `blkid`, never the device name:

```sh
vi /etc/fstab
```

```conf
UUID=d50040a8-41f1-4624-bf0b-aeb81f7752e7  /mnt/nas  xfs  defaults  0 0
```

Then — and this is the step worth not skipping — check the syntax *before* rebooting. A bad fstab line can drop the host into emergency mode at boot:

```sh
findmnt --verify
mount /mnt/nas
```

An alternative line, for a drive that regular users should be able to mount:

```conf
UUID=31f39d50-16fa-4248-b396-0cba7cd6eff2  /media/Data  auto  rw,user,auto  0 0
```

## 4. Pass it into an LXC container

The container can't see host mounts unless you bind them in. Either use `pct set`:

```sh
pct set 102 -mp0 /mnt/nas,mp=/media
```

Or edit the container config directly:

```sh
nano /etc/pve/lxc/102.conf
```

```conf
mp0: /mnt/nas,mp=/media
```

`mp0` is the host path; `mp=` is where it appears inside the container. Restart the container to pick it up.

## 5. And on into Docker

If the service is a Docker container inside the LXC, there's one more hop. In `docker-compose.yml`:

```yaml
volumes:
  - /media:/media:ro,z
```

The `z` suffix relabels for SELinux. `ro` for anything that only reads — a media server has no business writing to your library.

For a directory the container *does* write to, permissions have to line up with the container's user, which is usually UID 1000:

```sh
mkdir -m 755 /mnt/nas/appdata
chown 1000:1000 /mnt/nas/appdata
```

Then bind it in the compose file the same way. Getting this wrong produces permission errors inside the container that look like application bugs.

## Health checks

Full SMART report:

```sh
smartctl -a /dev/sda | less
```

Surface scan for bad blocks — this is slow, and read-only in this form:

```sh
badblocks -s /dev/sdb
```

## Spin-down

A drive holding cold archive data doesn't need to spin all day.

```sh
hdparm -y /dev/sdb      # spin down now
hdparm -C /dev/sdb      # check state
```

```text
/dev/sdb:
 drive state is: standby
```

Getting it to *stay* down is the hard part — `smartd`, media scanners and Samba all wake disks constantly. In my case Samba was the culprit. [This thread on unmounted disks still spinning up](https://askubuntu.com/questions/25543/unmounted-disk-still-spins-up-regularly) covers the general problem, and I wrote up the full configuration in [Spinning Down Idle Disks on a Linux Home Server](../idle-server-disk/).

## Reference

[This video](https://www.youtube.com/watch?v=9H7PNcrdF8s) walks through hard drive passthrough end to end, and is what I originally worked from.
