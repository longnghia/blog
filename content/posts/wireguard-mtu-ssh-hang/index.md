---
date: "2026-09-01T23:51:27+07:00"
draft: false
title: "SSH Hangs Forever: A WireGuard MTU Black Hole"
summary: "Small SSH commands worked, big ones hung forever. The culprit wasn't the server, the disk, or SSH — it was 28 bytes of MTU."
categories:
  - Code
tags:
  - ssh
  - wireguard
  - networking
  - mtu
  - linux
  - server
  - troubleshooting
---

{{< gpt >}}

## The symptom

I ran a perfectly ordinary command against my home Docker host over SSH:

```bash
ssh docker 'id; pwd; ls -ld ~/docker; ls -la ~/docker | head -40'
```

And got back this:

```text
uid=1000(paul) gid=1000(paul) groups=1000(paul),27(sudo),100(users),996(docker)
HOME=/home/paul
/home/paul
drwxr-xr-x 67 paul paul 4096 Jun  1 09:40 /home/paul/docker
```

Then nothing. The `ls -la` never printed. The session didn't error, didn't time out, didn't drop. It just sat there. I killed it after two minutes.

The maddening part: every command _before_ the `ls -la` worked fine. Reconnecting and running `ls -1 ~/docker` returned all 68 entries instantly. So the server was up, SSH was fine, the directory was readable. Only _some_ commands hung.

## Chasing the wrong things first

When `ls -1` is fast and `ls -la` hangs, there are some classic explanations. I worked through them in order, and every one was wrong.

**Hypothesis 1: slow disk / cold cache.** The host is a Proxmox VM on thin-provisioned LVM. `ls -la` has to `stat()` all 68 entries; `ls -1` doesn't stat anything. Cold metadata reads could be slow.

```bash
for d in *; do timeout 3 stat -c "%n" "$d" >/dev/null 2>&1 || echo "SLOW: $d"; done
```

Nothing. Every entry stat'd instantly.

**Hypothesis 2: a stale network mount.** A hung NFS or SSHFS mount underneath one of those directories would block `stat()` forever while a plain readdir sailed through.

```bash
findmnt -t nfs,nfs4,cifs,fuse.sshfs -o TARGET,SOURCE,FSTYPE
```

Empty. No network mounts at all.

**Hypothesis 3: NSS lookups.** This one is a genuinely great fit for the symptom. `ls -1` prints names only. `ls -la` calls `getpwuid()` / `getgrgid()` on every entry to turn UIDs into names. If `/etc/nsswitch.conf` points at LDAP or SSSD and that server is unreachable, `ls -l` hangs on lookup timeouts while `ls -1` is instant. Containers writing files as unmapped UIDs make this even more likely.

The decisive test is `ls -lan` — same `stat()` calls, but numeric IDs, so no NSS at all:

```bash
t(){ local s=$(date +%s%3N); timeout 30 "$@" >/dev/null 2>&1; local e=$(date +%s%3N); echo "$((e-s)) ms :: $*"; }
t ls -1
t ls -lan
t ls -la
```

```text
5 ms  :: ls -1
7 ms  :: ls -lan
7 ms  :: ls -la
```

All fast. And `nsswitch.conf` was just `files systemd` — no network source to hang on anyway.

At this point the bug had stopped reproducing, which is the worst possible outcome. I'd have written it off as a transient blip.

## The clue that reframed everything

Before giving up, I checked whether the original hung process had ever actually died:

```bash
ps -o pid,etime,stat,command -ax | grep "[s]sh"
```

```text
4416  21:06  S  ssh -o BatchMode=yes docker id; echo ...
```

**Twenty-one minutes**, still alive, state `S` — sleeping on a socket. And the socket was still there:

```bash
netstat -an | grep "192.168.0.104.22"
```

```text
tcp4  0  0  10.7.0.2.49645  192.168.0.104.22  ESTABLISHED
```

That changed the whole framing. This was never a filesystem problem. The command on the server had almost certainly finished long ago. The _bytes_ never made it back.

And look at the local address: `10.7.0.2`. Not my LAN address. That's a VPN tunnel.

```bash
route -n get 192.168.0.104 | grep interface
#   interface: utun4

ifconfig utun4
#   utun4: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1420
#   inet 10.7.0.2 --> 10.7.0.2 netmask 0xffffff00
```

My laptop was on a different subnet from the server, reaching it over WireGuard. Now the symptom pattern made sense in a completely different way:

- `id`, `pwd`, `ls -ld` → a few hundred bytes → small packets → **fine**
- `ls -la | head -40` → ~3 KB → SSH fills **full-size** TCP segments → **gone**

That's not a filesystem signature. That's a **path MTU black hole**.

## What a PMTU black hole actually is

Every link has a maximum packet size. When a packet is too big for the next hop, the router is supposed to drop it _and_ send back an ICMP "Fragmentation Needed" message so the sender learns to use smaller packets. That feedback loop is Path MTU Discovery.

A **black hole** happens when the packet gets dropped but the ICMP message never comes back — because a firewall filters ICMP, or a middlebox is misconfigured. The sender has no idea anything is wrong. It just keeps retransmitting the same oversized packet forever.

This produces a very distinctive failure:

- Small transfers work perfectly. Pings work. Handshakes complete. DNS works.
- Anything that fills a full-size packet hangs **forever**, not slowly.

SSH is a perfect trap for this, because the login and small commands all fit in small packets. The connection looks completely healthy right up until something produces real output.

## Finding the real MTU

You can measure the ceiling directly with do-not-fragment pings, bisecting on payload size. On macOS, `-D` sets the DF bit; total packet size is payload + 28 bytes of IP and ICMP headers:

```bash
for s in 1392 1372 1332 1272; do
  printf "payload %4d (MTU %4d): " "$s" "$((s+28))"
  ping -D -c 2 -W 1500 -s "$s" 192.168.0.104 >/dev/null 2>&1 && echo OK || echo FAIL
done
```

```text
payload 1392 (MTU 1420): FAIL
payload 1372 (MTU 1400): FAIL
payload 1332 (MTU 1360): OK
payload 1272 (MTU 1300): OK
```

There it is. Narrowing further pinned the exact boundary:

```text
payload 1364 (MTU 1392): OK
payload 1368 (MTU 1396): FAIL
```

**The tunnel was configured for 1420. The path could only carry 1392.** Every packet in that 28-byte window vanished silently.

The arithmetic explains where 1392 comes from. WireGuard over IPv4 adds 60 bytes of overhead: 20 (IP) + 8 (UDP) + 16 (WireGuard header) + 16 (Poly1305 tag).

```text
1392 inner + 60 overhead = 1452 outer
```

So the underlying internet path to my VPN endpoint tops out at **1452**, not the 1500 that everyone assumes. Meanwhile `wg-quick` picks its default MTU as `1500 - 80 = 1420`. That default is simply wrong on any path that isn't a clean 1500 — and mine wasn't.

## The fix

Set the tunnel MTU explicitly. In `wg0.conf`, or the **MTU** field in the WireGuard macOS/iOS app:

```ini
[Interface]
MTU = 1280
```

Toggle the tunnel off and on. The hang disappeared immediately — a full `ls -laR ~/docker | head -500` that used to wedge now returns instantly.

**Why 1280 and not 1392?** Because 1392 is only correct for the network I happened to be on when I measured it. This is a laptop; it moves between home, the office, and hotel Wi-Fi, and each of those has a different path MTU. 1280 is the minimum MTU that IPv6 guarantees every link must support, so it's deliverable everywhere and never needs retuning. The throughput cost versus 1392 is about 8%, which is meaningless for SSH and admin work.

Pick 1392 if you want maximum throughput on one fixed network. Pick 1280 if you want it to never break again. I picked 1280.

## Defense in depth: make SSH fail loudly

The MTU was the root cause, but there's a second bug here worth fixing: **the hang lasted 21 minutes and produced no error.** That's because my SSH config had no keepalives, so `ssh` will happily wait forever on a wedged socket.

Add this to the **end** of `~/.ssh/config` — SSH is first-match-wins, so a `Host *` block must come after your specific hosts:

```ssh-config
Host *
	ServerAliveInterval 15
	ServerAliveCountMax 3
	TCPKeepAlive yes
	ConnectTimeout 10
```

Now a dead session dies in about 45 seconds with `Timeout, server not responding` instead of hanging indefinitely. This doesn't fix anything — but it converts a silent, mysterious stall into a visible, googleable error, which is worth a lot at 2am.

## Takeaways

**Symptom shape tells you the layer.** "Small commands work, large ones hang forever" is an MTU signature. Not slow — _forever_. If it were bandwidth or disk, big transfers would be slow but would finish. A hard, permanent stall on size alone means packets are being discarded, not delayed.

**Check whether the process actually died.** I nearly wrote this off as transient when it stopped reproducing. `ps` showing a 21-minute-old process in state `S`, plus `netstat` showing `ESTABLISHED`, was what proved the connection was wedged rather than the server being slow. A "transient" bug that leaves a zombie process behind is not transient.

**Know which flags trigger which syscalls.** `ls -1` vs `ls -lan` vs `ls -la` differ precisely in whether they `stat()` and whether they hit NSS. Those three commands cleanly separate I/O problems from name-resolution problems. That triage was wrong here, but it was wrong _fast_, which is the point.

**Distrust default MTUs on tunnels.** `wg-quick`'s 1420 assumes a 1500-byte path. PPPoE, CGNAT, mobile networks, and nested tunnels all break that assumption, and the failure mode is silent. If you run WireGuard and see unexplained hangs, measure before you theorize — it's four lines of `ping`.
