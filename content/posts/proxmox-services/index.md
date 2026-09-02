---
date: "2026-02-17T09:25:00+07:00"
draft: true
title: "Self-Hosting Services in Proxmox LXCs"
summary: "Install notes for Docker and Portainer, Pi-hole, Samba with Cockpit, Jellyfin, Immich, Gitea and JDownloader — plus the gotchas that cost me an evening each."
categories:
  - Guides
tags:
  - proxmox
  - lxc
  - docker
  - selfhosted
  - pihole
  - samba
  - jellyfin
---

Install notes for the services I actually run, one container each. For what's running and why, see [My Self-hosted Services](../my-self-hosted/). For the container and disk plumbing underneath, see [Proxmox Notes: LXC Containers, Templates, and First-Boot Setup](../proxmox-lxc-basics/) and [Adding and Passing Through Disks on Proxmox](../proxmox-disk-passthrough/).

## Docker in an LXC

The base for most of the rest. In a fresh container (created with `nesting=1` — see the [container notes](../proxmox-lxc-basics/)):

```sh
apt update && apt upgrade -y
apt install curl -y

curl -sSL https://get.docker.com/ | sh
```

Then Portainer, so you get a web UI rather than SSHing in to check whether something is up:

```sh
docker run --restart always -d \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

[This walkthrough](https://www.wundertech.net/how-to-set-up-docker-containers-in-proxmox/) is the one I followed originally.

## Pi-hole

```sh
apt update && apt upgrade -y
apt install curl -y
curl -sSL https://install.pi-hole.net | bash
```

Set the admin password after install:

```sh
sudo pihole -a -p
```

Tail the query log when you want to see what's actually being blocked:

```sh
pihole -t
```

**The gotcha:** Pi-hole works everywhere except your Android phone, which keeps resolving ads. That's **Private DNS** — Android is routing DNS over TLS to Google and skipping your resolver entirely. Turn it off in the network settings and the phone starts showing up in the query log. [Discussion](https://www.reddit.com/r/pihole/comments/15ig7ev/pihole_working_but_not_working_on_my_phone/), and the [install guide](https://www.wundertech.net/how-to-install-pi-hole-on-proxmox/) I used.

## Samba, and the NAS question

There's a real architectural decision here, laid out well in [this thread](https://www.reddit.com/r/Proxmox/comments/mco03f/comment/gspy1go/):

1. **Install a NAS OS** (OpenMediaVault, TrueNAS) in a VM and pass through the whole drive controller. The VM owns the physical storage and the shares.
2. **Let Proxmox manage the storage** and run only the file-server part in an LXC. ZFS datasets get bind-mounted into the container, which sees them as ordinary directories.

I went with option 2. It keeps disk management in one place — the host — and means the file server is a container I can rebuild in ten minutes rather than a VM holding my data hostage.

### Cockpit for the web UI

```sh
apt install --no-install-recommends cockpit -y
```

`--no-install-recommends` matters; the full install drags in a lot you don't want in a container. If it doesn't come up:

```sh
journalctl -b -u cockpit
systemctl edit cockpit.service
```

References: [apalrd's ultimate NAS writeup](https://www.apalrd.net/posts/2023/ultimate_nas/#cockpit-setup) and [this video](https://www.youtube.com/watch?v=Hu3t8pcq8O0).

### Samba itself

```sh
apt install samba
nano /etc/samba/smb.conf
smbpasswd -a username
systemctl restart smbd
```

The [Alpine wiki page on setting up Samba](https://wiki.alpinelinux.org/wiki/Setting_up_a_Samba_server) is a clearer reference than most of the Debian ones.

One consequence worth knowing: a running Samba server will keep your disks awake. If spin-down matters, see the [disk notes](../proxmox-disk-passthrough/#spin-down).

## Jellyfin

Easiest path is the community script, which builds the container for you:

```sh
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/jellyfin.sh)"
```

If you'd rather run it as Docker inside an LXC, [this comment](https://github.com/jellyfin/jellyfin/issues/1601#issuecomment-767027450) covers the permissions and hardware-acceleration setup that trips people up.

[This video](https://www.youtube.com/watch?v=9H7PNcrdF8s) covers Jellyfin alongside the drive passthrough it needs.

## Immich

For photos. Two maintenance things I keep having to look up.

Clean up duplicates with [immich-go](https://github.com/simulot/immich-go):

```sh
immich-go -server=http://192.168.1.104:2283 -key=$IMMICH_API_KEY duplicate -yes
```

Generate the API key in the Immich web UI under account settings, and keep it in an environment variable rather than your shell history.

If the Postgres volume has grown out of control and you're rebuilding anyway, deleting Immich's `pgdata` directory resets it — destructive, so only with backups in hand.

Related reading on backup strategy generally: [How and why do you backup your photos](https://www.reddit.com/r/DataHoarder/comments/15xdaee/how_and_why_do_you_backup_your_photos/). For iPhone specifically, mounting the phone and `cp -a` once a month is unglamorous but it works.

## Gitea

Paths worth knowing, because the config isn't where you'd guess:

```text
/data/git/repositories        # the bare repos themselves
/data/gitea/conf/app.ini      # all configuration
```

Health and version info is at `<host>:3000/admin/system_status`.

Set up the bind mounts before first run:

```sh
mkdir -p gitea/{data,config}
```

## JDownloader 2

Runs headless with a web UI on port 5800. The workflow that isn't obvious: it watches the clipboard, and links you copy appear in a side popup for you to accept into the LinkGrabber — they don't just start downloading. [Docs on adding links](https://support.jdownloader.org/en/knowledgebase/article/linkgrabber-how-to-add-links).

## IP cameras

Cheap cameras all speak RTSP, which is what you want — it means you can point `ffmpeg` or Jellyfin at them instead of using the vendor's app.

To find a camera's address, check the router's DHCP client list. The web UI on these tends to work in Chrome and fail in Firefox, which is worth knowing before you conclude the camera is dead.

Stream URLs differ by vendor:

```text
# Imou
rtsp://<camera-ip>/cam/realmonitor?channel=1&subtype=0

# Yoosee
rtsp://<camera-ip>:554/onvif1
```

Both want credentials. **Change the defaults** — these ship with `admin` and a password printed on the box, and they are on your LAN.

## All-in-one options I looked at

If per-service containers sound like too much work, these bundle the whole media stack:

- [DockSTARTer](https://dockstarter.com/) — interactive setup for the \*arr stack

  ```sh
  apt-get install curl git -y
  bash -c "$(curl -fsSL https://get.dockstarter.com)"
  reboot
  ```

- [Saltbox](https://github.com/saltyorg/Saltbox) — Ansible-based, more opinionated
- [Ansible-NAS](https://github.com/davestephens/ansible-nas) — same idea, broader service list

[This thread](https://www.reddit.com/r/selfhosted/comments/1bagrm8/whats_the_modern_oneinall_program_for_media/) compares them. I ended up not using any of them — when something breaks I'd rather debug one compose file than someone else's Ansible role — but they're a fast way to get running.
