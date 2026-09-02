---
date: "2026-01-08T08:31:21+07:00"
draft: false
title: "Spinning Down Idle Disks on a Linux Home Server"
summary: "hdparm spindown_time, udev rules, and the services (smartd, ZFS, media scanners) that quietly keep your drives awake."
categories:
  - Guides
tags:
  - linux
  - server
  - hdparm
  - nas
  - power
---

{{< gpt >}}

A home server that idles 22 hours a day doesn't need its spinning disks turning the whole time. Getting them to actually sleep takes two things: telling the disk when to stop, and finding whatever keeps waking it up. The second part is where most of the work is.

## Test it by hand first

Before configuring anything persistent, confirm the drive will spin down at all. `hdparm` is the standard tool:

```bash
sudo hdparm -y /dev/sdX
```

That spins the disk down immediately. It's temporary — the next access, or a reboot, undoes it. Check the result:

```bash
sudo hdparm -C /dev/sdX
```

You want to see `standby`. If it says `active/idle` a second after you issued `-y`, something is already hammering the disk and no amount of configuration will help until you find it. Skip ahead to [what prevents spin-down](#what-prevents-spin-down).

## Make it automatic

Install `hdparm` if it isn't already there:

```bash
sudo apt install hdparm        # Debian/Ubuntu
sudo dnf install hdparm        # Fedora/RHEL
sudo pacman -S hdparm          # Arch
```

Then set an idle timeout in `/etc/hdparm.conf`:

```conf
/dev/sdX {
    spindown_time = 240
}
```

The units are the awkward part. `spindown_time` is **not** seconds or minutes:

| Value   | Meaning                        |
| ------- | ------------------------------ |
| 1–240   | value × 5 seconds              |
| 241–251 | 30 minutes to 5.5 hours        |
| 0       | spin-down disabled             |

So:

- 10 minutes → `120`
- 20 minutes → `240`
- 1 hour → `252`

Apply it without rebooting:

```bash
sudo systemctl restart hdparm
```

### Or via udev

Some distros prefer a udev rule, which also survives the disk being re-plugged. Create `/etc/udev/rules.d/69-hdparm.rules`:

```conf
ACTION=="add|change", KERNEL=="sdX", RUN+="/sbin/hdparm -S 240 /dev/sdX"
```

Reload:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Then verify the same way as before:

```bash
sudo hdparm -C /dev/sdX
```

`active/idle` means spinning, `standby` means you've won.

## What prevents spin-down

This is the real problem. A disk with a perfectly good 20-minute timeout will never sleep if anything touches it every 19 minutes. The usual suspects:

- **`smartd`** — SMART polling, by default wakes the disk to read attributes
- **Media servers** — Plex and Jellyfin library scans
- **File indexing** — Tracker, `updatedb`/`mlocate`
- **Docker containers** — logs and volumes on the data disk
- **ZFS** — writes metadata frequently and effectively never idles
- **Btrfs** — background metadata writes

`smartd` is the most common one, and it's also the easiest to fix. Add `-n standby` to the drive's line in `/etc/smartd.conf`:

```conf
/dev/sdX -a -n standby
```

That tells `smartd` to skip the check entirely if the disk is already asleep, rather than waking it to ask how it's doing.

```bash
sudo systemctl restart smartd
```

## Filesystem matters more than you'd expect

**ZFS** does not spin down reliably. If power saving is the priority, don't put your archive pool on ZFS — you're choosing between the two.

**mdadm RAID** works, with a caveat: any access to one member wakes *all* of them, and array metadata writes can block sleep outright.

**ext4 and XFS** on single disks give the most reliable behaviour. If a drive exists only to hold cold data, this is the combination that actually sleeps.

## Timeouts worth using

- **Home NAS** — 20–30 minutes
- **Backup-only NAS** — 10–15 minutes
- **Media server** — often not achievable; scanning wins

Avoid anything under 5 minutes. Spin-up cycles cause more wear than the idle spinning does, and a disk that parks and unparks every few minutes will die sooner than one you left alone.
