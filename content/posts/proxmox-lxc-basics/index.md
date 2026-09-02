---
date: "2026-02-17T09:18:33+07:00"
draft: false
title: "Proxmox Notes: LXC Containers, Templates, and First-Boot Setup"
summary: "Why I run services in LXC instead of VMs, the pct commands worth memorizing, and a repeatable script for standing up a fresh container."
categories:
  - Guides
tags:
  - proxmox
  - lxc
  - server
  - selfhosted
  - docker
---

Working notes from running a Proxmox box at home. This part covers containers and host setup; disks are in [Adding and Passing Through Disks on Proxmox](../proxmox-disk-passthrough/), and the actual services are in [Self-Hosting Services in Proxmox LXCs](../proxmox-services/).

## Terminology, because Proxmox assumes you know it

The docs and forum posts throw these around freely:

| Term          | Meaning                                                                   |
| ------------- | ------------------------------------------------------------------------- |
| **PV**        | Physical volume — a block device (or part of one) where data really lives |
| **VG**        | Volume group — one or more PVs grouped together                           |
| **LV**        | Logical volume — a volume stored on a VG                                  |
| **thin pool** | Volumes that can hold LVs without allocating full size upfront            |
| **lxc**       | Linux container                                                           |
| **pct**       | Proxmox Container Toolkit — the CLI for LXC containers                    |
| **pve**       | Proxmox Virtual Environment — the host itself                             |

## LXC or VM?

For running Docker containers, both work. The difference is overhead.

A **VM** gets a fixed memory allocation whether it's using it or not. An **LXC container** shares the host kernel and only consumes what it actually needs, so total system resource use is meaningfully lower.

I run Docker inside LXC. [This walkthrough](https://www.wundertech.net/how-to-set-up-docker-containers-in-proxmox/) covers both approaches if you want to compare.

The catch: LXC needs `nesting=1` to run Docker, and unprivileged containers occasionally trip over permissions that a VM wouldn't. For a home server that trade is worth it. For anything where isolation is a security boundary, use a VM.

## Host post-install

First thing on a fresh install, run the community post-install script. It disables the enterprise repo, stops the subscription nag, and applies sensible defaults:

```sh
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"
```

Then the shell, because the default root `bash` on Proxmox is bare. [This `.bashrc`](https://gist.github.com/pythoninthegrass/509a7769ab690bcce3f606dd460a13ed) is a good starting point.

```sh
# zoxide, for jumping around
curl -sS https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | bash
echo eval "$(zoxide init bash)" >> .bashrc

# stop the UTF-8 warnings
echo export LC_CTYPE=en_US.UTF-8 >> .bashrc
echo export LC_ALL=en_US.UTF-8 >> .bashrc

apt-get install -y powertop samba mc tmux

curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o ~/.local/bin/yt-dlp
chmod a+rx ~/.local/bin/yt-dlp
```

### Power draw

If the box runs 24/7, it's worth knowing what it costs you. `powertop` will write a full report:

```sh
powertop --html=report.html
```

[This homelab thread](https://www.reddit.com/r/homelab/comments/14nqjmu/help_needed_low_power_home_server_idle/) is a good reference for what idle figures are actually achievable.

## Creating a container

`pveam` manages templates. Grab a Debian 12 template:

```sh
pveam update
LX=`pveam available | grep debian-12-standard | sed 's/system\s*//g'`
pveam download local $LX
```

Templates land in `/var/lib/vz/template/iso`.

Then create and start the container. Note `nesting=1` — without it Docker won't run inside:

```sh
ID=103

pct create $ID "local:vztmpl/$LX" \
  --unprivileged 1 \
  -features nesting=1 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=dhcp,type=veth \
  --storage local-lvm

nano /etc/pve/lxc/$ID.conf   # any extra config, e.g. mount points

pct start $ID
sleep 10
pct enter $ID
```

Inside the container, the first-run ritual:

```sh
passwd root
passwd -u root       # the template ships with root locked

apt update
apt dist-upgrade -y
apt install curl -y

echo export LC_CTYPE=en_US.UTF-8 >> .bashrc
echo export LC_ALL=en_US.UTF-8 >> .bashrc
```

## pct commands worth remembering

```sh
pct enter 102                          # shell into a container
pct exec 102 -- bash -c 'apk update && apk upgrade'   # run a command (Alpine here)
pct pull 102 /root/site.tar.gz /root/site.tar.gz      # copy a file out
pct resize 102 rootfs 20G              # grow the root disk
pct set 105 -mp0 /media/seagate/,mp=/data              # add a bind mount
```

Anything set with `pct set` also lands in `/etc/pve/lxc/<ID>.conf`, so you can edit it there instead:

```conf
mp0: /media/seagate/,mp=/data
```

### Debugging a container that won't start

`pct start` failing tells you almost nothing. Start it in the foreground with debug logging instead:

```sh
lxc-start -n 101 -F -lDEBUG -o lxc-101.log
```

## Finding container IPs

Three ways, in increasing order of effort:

```sh
lxc-info -i -n 110              # from the host, per container
```

Or from the router: **DHCP → DHCP client list** for leases, **IP & MAC → ARP list** for what's currently reachable.

Check a port is actually bound before blaming the network:

```sh
ss -tulpen | grep 9090
```

## Errors I hit

### `Temporary failure in name resolution`

Container has no DNS. Fix `/etc/resolv.conf`:

```conf
search .
nameserver 1.1.1.1
```

[Forum thread](https://forum.proxmox.com/threads/temporary-failure-in-name-resolution.133176/) with the longer discussion.

### SSH to the host

Nothing to configure — SSH is already running on a Proxmox host, log in with root credentials. What you usually want is to stop typing the password:

```sh
ssh-keygen -R 192.168.1.100     # clear a stale host key after a reinstall
ssh-copy-id -i ~/.ssh/paul.pub root@192.168.1.100
```

## Other things I've bookmarked

- [Home Assistant OS on Proxmox 8](https://community.home-assistant.io/t/installing-home-assistant-os-using-proxmox-8/201835)
- [OpenVPN in LXC](https://pve.proxmox.com/wiki/OpenVPN_in_LXC) — the official wiki page, and [DigitalOcean's OpenVPN guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-20-04) for the server config itself
- [TrueNAS on Proxmox: should you?](https://www.reddit.com/r/Proxmox/comments/1617z4f/should_i_install_truenas_on_proxmox_or_not/)
