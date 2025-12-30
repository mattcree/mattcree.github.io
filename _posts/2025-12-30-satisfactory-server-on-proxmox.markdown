---
layout: post
title:  "Running a Satisfactory Dedicated Server on Proxmox"
date:   2025-12-30
categories: homelab
tags: proxmox lxc docker gaming satisfactory
---

I've been playing [Satisfactory][satisfactory] with friends lately, and like any self-respecting homelab enthusiast, I couldn't resist the urge to host my own dedicated server. There's something satisfying about running your own infrastructure rather than paying for a hosted solution.

What follows is my journey setting up a Satisfactory server using LXC containers and Docker on Proxmox. I wanted something lightweight (no full VM overhead), easy to manage, and with some basic network separation since the server would be exposed to the internet.

**A quick note:** I'm not a security expert - this is just how I've set things up based on my own tinkering. If you spot something I've missed or have suggestions, I'd love to hear from you at [mattcree@proton.me](mailto:mattcree@proton.me).

## Why LXC + Docker?

I could have run the game server in a full VM, but that felt like overkill. VMs are great, but they're heavy - you're dedicating RAM to an entire operating system just to run one application.

LXC containers are much lighter. And since there's already a well-maintained [Docker image][dockerhub] for Satisfactory, running Docker inside LXC seemed like a nice middle ground: lightweight, easy to update, and reasonably contained.

## The Architecture

Here's what I ended up with:

```
Internet
    │ (only ports 7777, 8888)
    ▼
Router (192.168.174.1)
    │
    ▼
Proxmox Host
    ├── vmbr0 (Home LAN - 192.168.174.0/24)
    │       ▲
    │       │ BLOCKED (iptables DROP)
    │       │
    └── vmbr1 (Game DMZ - 10.10.10.0/24)
            │
            ▼
        LXC Container (10.10.10.10)
            │
            ▼
        Docker → Satisfactory Server
```

The key insight is that **network isolation happens at the Proxmox host level**, not inside the container. Even if an attacker completely owns the container, they can't reach my home network because the iptables rules on Proxmox block it.

## Prerequisites

Before we start, you'll need:

- Proxmox VE 7.x or 8.x
- At least 12GB RAM available (8GB for the server + overhead)
- 60GB free disk space
- SSH access to your Proxmox host

I've put all the scripts on [GitHub][repo] if you want to follow along.

## Step 1: Create the DMZ Network

First, we create an isolated bridge network. This is where our game server will live, completely separated from the home LAN.

SSH into your Proxmox host and run:

```bash
cd /root/satisfactory-server
chmod +x *.sh
./setup-dmz-bridge.sh --dry-run  # Preview first
./setup-dmz-bridge.sh            # Apply when ready
```

This script does several things:

1. Creates `vmbr1` bridge with IP `10.10.10.1/24`
2. Enables IP forwarding
3. Adds the critical firewall rule: **DROP all traffic from DMZ to home LAN**
4. Sets up NAT so the container can reach the internet

The firewall rule order is crucial. The `ESTABLISHED,RELATED` rule must come before the `DROP` rule, otherwise return traffic from the game server gets blocked:

```
CORRECT ORDER:
1. ACCEPT ESTABLISHED,RELATED    ← Allows return traffic
2. DROP state NEW to home LAN    ← Only blocks NEW connections
```

## Step 2: Configure Port Forwarding

Next, we set up the firewall rules to forward game traffic to our container:

```bash
./proxmox-firewall.sh --dry-run
./proxmox-firewall.sh
```

This configures:
- Port 7777 (TCP/UDP) for game traffic
- Port 8888 (TCP) for server messaging
- DNAT rules to redirect traffic to the container
- Hairpin NAT so LAN clients can connect too

## Step 3: Create the LXC Container

Now for the container itself:

```bash
./create-lxc.sh --dry-run
./create-lxc.sh
```

You'll be prompted for a root password. The script automatically:
- Finds the next available container ID
- Downloads the latest Debian 12 template
- Creates an unprivileged container with Docker compatibility settings
- Places it on the isolated `vmbr1` network

### The Docker-in-LXC Trade-off

Here's where things get interesting. Running Docker inside LXC requires relaxing some container security settings:

```
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: proc:rw sys:rw
```

This looks scary, and it is a trade-off. These settings weaken isolation between the LXC container and the Proxmox host. But remember: **the network isolation is enforced by iptables on Proxmox**, not by LXC. Even with these relaxed settings, an attacker in the container still can't reach my home network.

The alternative would be a full VM, which uses significantly more resources. For a game server, this trade-off is acceptable.

## Step 4: Install Docker

Start the container and install Docker:

```bash
pct start <CTID>
pct push <CTID> setup-docker.sh /root/setup-docker.sh
pct exec <CTID> -- chmod +x /root/setup-docker.sh
pct exec <CTID> -- /root/setup-docker.sh
```

The setup script installs Docker with some security hardening:
- `no-new-privileges: true` in the daemon config
- Log rotation to prevent disk exhaustion
- A dedicated non-root user for the game server

## Step 5: Deploy the Server

Finally, deploy the Satisfactory server:

```bash
pct push <CTID> docker-compose.yml /opt/satisfactory/docker-compose.yml
pct exec <CTID> -- bash -c "cd /opt/satisfactory && docker compose up -d"
```

The first startup downloads about 8GB of game files. Monitor progress with:

```bash
pct exec <CTID> -- docker compose -f /opt/satisfactory/docker-compose.yml logs -f
```

## Step 6: Router Configuration

Don't forget to forward ports on your router to your Proxmox host:

| Port | Protocol | Description |
|------|----------|-------------|
| 7777 | TCP + UDP | Game traffic |
| 8888 | TCP | Server messaging |

## Connecting to Your Server

Once everything is running:

1. Launch Satisfactory
2. Go to **Server Manager**
3. Click **Add Server**
4. Enter your public IP (find it at [whatismyip.com][whatismyip])
5. Claim the server and set an admin password

## Verifying It Works

You can verify the network setup is working:

```bash
# From inside the container - this should timeout (blocked by firewall)
pct exec <CTID> -- ping -c 3 192.168.174.1

# This should work (internet access for updates)
pct exec <CTID> -- ping -c 3 8.8.8.8
```

If the first ping times out and the second succeeds, the DMZ is working as intended.

## Quick Reference

Once everything is set up, here are the commands I use most:

```bash
# View server logs
ssh root@192.168.174.6 "pct exec 102 -- docker compose -f /opt/satisfactory/docker-compose.yml logs --tail 50"

# Restart server
ssh root@192.168.174.6 "pct exec 102 -- docker compose -f /opt/satisfactory/docker-compose.yml restart"

# Update server
ssh root@192.168.174.6 "pct exec 102 -- bash -c 'cd /opt/satisfactory && docker compose pull && docker compose up -d'"
```

## Was It Worth It?

For me? Yes. The setup took an afternoon, but now I have:

- A game server that starts automatically on boot
- Some level of network isolation (hopefully meaningful!)
- Easy updates with `docker compose pull`
- Resource efficiency (no VM overhead)

The friends I play with don't notice any difference from a commercial hosted server. And I feel better knowing there's at least some separation between the game server and my home network.

Is it perfect? Almost certainly not. Is it better than just running the server directly on my network with no isolation? I think so.

---

All the scripts are available on [GitHub][repo]. Feel free to adapt them for your own setup - just remember to change the IP addresses to match your network.

If you have suggestions, improvements, or spot any glaring security issues, please do get in touch at [mattcree@proton.me](mailto:mattcree@proton.me). This is very much a learning exercise for me, and I'd love to hear from folks who know more about this stuff than I do.

[satisfactory]: https://www.satisfactorygame.com/
[dockerhub]: https://hub.docker.com/r/wolveix/satisfactory-server
[repo]: https://github.com/mattcree/satisfactory-server
[whatismyip]: https://whatismyip.com
