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

LXC containers are much lighter. They share the host's kernel, so you're not running a whole separate OS. And since there's already a well-maintained [Docker image][dockerhub] for Satisfactory, running Docker inside LXC seemed like a nice middle ground: lightweight, easy to update, and reasonably contained.

## The Setup

Here's what we're building:

- A separate network (10.10.10.0/24) just for the game server
- An LXC container running Docker
- Firewall rules to keep the game server isolated from the rest of the home network
- Port forwarding so players can connect from the internet

The idea is that even if something goes wrong with the game server, it can't reach the rest of my home network.

## Prerequisites

Before we start, you'll need:

- Proxmox VE 7.x or 8.x
- At least 12GB RAM available (8GB for the server + overhead)
- 60GB free disk space
- SSH access to your Proxmox host

I'll be using these example values throughout - adjust them to match your network:

| Setting | Value |
|---------|-------|
| Proxmox Host IP | 192.168.174.6 |
| Home Network | 192.168.174.0/24 |
| Game Server Network | 10.10.10.0/24 |
| Game Server IP | 10.10.10.10 |

## Step 1: Create an Isolated Network Bridge

First, we need to create a separate network for the game server. In Proxmox (and Linux networking generally), a "bridge" is like a virtual network switch - it lets you connect virtual machines and containers together.

SSH into your Proxmox host and let's set this up.

### Edit the network configuration

Open the network interfaces file:

```bash
nano /etc/network/interfaces
```

Add the following at the end of the file:

```
# DMZ Bridge for Game Server
# This creates an isolated network separate from your home LAN
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1
    netmask 255.255.255.0
    bridge_ports none
    bridge_stp off
    bridge_fd 0
```

A few things to note:
- `address 10.10.10.1` - This is the gateway IP for our game network. The Proxmox host will be the router for this network.
- `bridge_ports none` - This bridge isn't connected to any physical network interface. It's completely virtual.
- We're using 10.10.10.0/24, which is a private IP range that won't conflict with most home networks.

### Enable IP forwarding

For Proxmox to route traffic between networks, we need to enable IP forwarding. This tells Linux "yes, you're allowed to pass network packets between different interfaces."

```bash
# Enable immediately
sysctl -w net.ipv4.ip_forward=1

# Make it permanent (survives reboot)
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```

### Bring up the bridge

```bash
ifup vmbr1
```

Verify it's working:

```bash
ip addr show vmbr1
```

You should see the bridge with IP 10.10.10.1.

## Step 2: Set Up Firewall Rules

Now for the important part - the firewall rules. We're using `iptables`, which is the standard Linux firewall. If you haven't used it before, here's the basic idea:

- **iptables** looks at every network packet and decides what to do with it based on rules you define
- Rules are organised into "chains" - FORWARD (packets passing through), INPUT (packets coming in), OUTPUT (packets going out)
- For NAT (Network Address Translation), there are additional chains like PREROUTING and POSTROUTING

### Install iptables-persistent

First, let's install a package that will save our firewall rules across reboots:

```bash
apt-get update
apt-get install -y iptables-persistent
```

### The firewall rules

Now let's add the rules. I'll explain each one:

```bash
# Rule 1: Allow established connections
# This is crucial - it lets response traffic flow back.
# Without this, the game server could send packets out but never receive replies.
iptables -I FORWARD 1 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Rule 2: Block the game server from reaching our home network
# This is the isolation - the container can't initiate connections to 192.168.174.0/24
# The "state NEW" part means it only blocks NEW connections, not responses to existing ones
iptables -I FORWARD 2 -i vmbr1 -d 192.168.174.0/24 -m state --state NEW -j DROP

# Rule 3: Allow the game server to reach the internet
# It needs this for Steam updates and player connections
iptables -A FORWARD -i vmbr1 -o vmbr0 -j ACCEPT

# Rule 4-6: Allow incoming game traffic
# These let players connect to the game server
iptables -A FORWARD -p tcp -d 10.10.10.10 --dport 7777 -m state --state NEW -j ACCEPT
iptables -A FORWARD -p udp -d 10.10.10.10 --dport 7777 -m state --state NEW -j ACCEPT
iptables -A FORWARD -p tcp -d 10.10.10.10 --dport 8888 -m state --state NEW -j ACCEPT
```

**Why the order matters:** Rule 1 must come before Rule 2. Otherwise, Rule 2 would block ALL traffic to the home network, including response packets from legitimate connections.

### NAT rules

NAT (Network Address Translation) lets our game server share the Proxmox host's IP address when talking to the internet:

```bash
# Masquerade: Replace the game server's private IP with the Proxmox IP for outgoing traffic
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE

# Hairpin NAT: Lets LAN clients connect to the game server via the Proxmox IP
iptables -t nat -A POSTROUTING -s 192.168.174.0/24 -d 10.10.10.10 -j MASQUERADE
```

### Port forwarding

When traffic arrives at Proxmox on the game ports, we need to redirect it to the container:

```bash
# Forward game port (TCP and UDP)
iptables -t nat -A PREROUTING -d 192.168.174.6 -p tcp --dport 7777 -j DNAT --to-destination 10.10.10.10:7777
iptables -t nat -A PREROUTING -d 192.168.174.6 -p udp --dport 7777 -j DNAT --to-destination 10.10.10.10:7777

# Forward messaging port
iptables -t nat -A PREROUTING -d 192.168.174.6 -p tcp --dport 8888 -j DNAT --to-destination 10.10.10.10:8888

# Also catch traffic coming from the router (for external connections)
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 7777 -j DNAT --to-destination 10.10.10.10:7777
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 7777 -j DNAT --to-destination 10.10.10.10:7777
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 8888 -j DNAT --to-destination 10.10.10.10:8888
```

### Save the rules

```bash
mkdir -p /etc/iptables
iptables-save > /etc/iptables/rules.v4
```

This ensures the rules persist after a reboot.

## Step 3: Create the LXC Container

Now let's create the container that will run our game server.

### Download a template

First, we need a container template. Debian 12 works well:

```bash
pveam update
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
```

### Create the container

```bash
pct create 404 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
    --hostname satisfactory \
    --memory 8192 \
    --swap 512 \
    --cores 4 \
    --rootfs local-lvm:50 \
    --net0 name=eth0,bridge=vmbr1,ip=10.10.10.10/24,gw=10.10.10.1,firewall=1 \
    --nameserver "8.8.8.8 8.8.4.4" \
    --features nesting=1,keyctl=1 \
    --unprivileged 1 \
    --onboot 1 \
    --start 0 \
    --protection 1 \
    --password
```

Let me break this down:
- `404` - Container ID (use whatever's free on your system)
- `--memory 8192` - 8GB RAM for the game server
- `--cores 4` - 4 CPU cores
- `--rootfs local-lvm:50` - 50GB disk
- `--net0 ... bridge=vmbr1,ip=10.10.10.10/24` - Connect to our isolated network
- `--nameserver "8.8.8.8 8.8.4.4"` - Use Google's DNS (since we blocked access to the home router)
- `--features nesting=1,keyctl=1` - Required for Docker to work
- `--unprivileged 1` - Run as unprivileged container (safer)
- `--onboot 1` - Start automatically when Proxmox boots
- `--start 0` - Don't start yet (we need to configure it first)
- `--protection 1` - Prevent accidental deletion

You'll be prompted to set a root password for the container.

### Configure for Docker compatibility

Docker inside LXC needs some extra settings. Edit the container config:

```bash
nano /etc/pve/lxc/404.conf
```

Add these lines at the end:

```
# Docker compatibility settings
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: proc:rw sys:rw
```

These settings relax some of the container isolation to let Docker work. It's a trade-off - we're relying on the network isolation rather than container isolation for our security boundary.

## Step 4: Install Docker in the Container

Start the container and get inside:

```bash
pct start 404
pct enter 404
```

Now, inside the container, let's install Docker:

```bash
# Update packages
apt-get update
apt-get upgrade -y

# Install prerequisites
apt-get install -y ca-certificates curl gnupg

# Add Docker's GPG key
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable Docker to start on boot
systemctl enable docker
```

### Configure Docker daemon

Create a configuration file with some sensible defaults:

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    },
    "storage-driver": "overlay2",
    "live-restore": true,
    "userland-proxy": false,
    "no-new-privileges": true,
    "icc": false,
    "default-ulimits": {
        "nofile": {
            "Name": "nofile",
            "Hard": 65535,
            "Soft": 65535
        }
    }
}
EOF
```

This sets up log rotation, enables live restore (so containers keep running during Docker daemon restarts), and applies some security defaults.

Restart Docker to apply:

```bash
systemctl restart docker
```

Verify it's working:

```bash
docker --version
docker compose version
```

### Create a user for the game server

The Satisfactory Docker image runs as a non-root user. Let's create the user it expects:

```bash
groupadd -g 1000 satisfactory
useradd -u 1000 -g 1000 -m satisfactory
```

### Create the directory structure

```bash
mkdir -p /opt/satisfactory/config
chown -R 1000:1000 /opt/satisfactory
```

## Step 5: Deploy the Satisfactory Server

Still inside the container, let's create the Docker Compose file:

```bash
nano /opt/satisfactory/docker-compose.yml
```

Paste this configuration:

```yaml
services:
  satisfactory-server:
    container_name: satisfactory-server
    hostname: satisfactory-server
    image: wolveix/satisfactory-server:latest
    restart: unless-stopped

    ports:
      # Game port (TCP + UDP required)
      - "7777:7777/tcp"
      - "7777:7777/udp"
      # Server messaging/beacon port
      - "8888:8888/tcp"

    volumes:
      - ./config:/config

    environment:
      # Server settings
      - MAXPLAYERS=4
      - SERVERGAMEPORT=7777
      - SERVERMESSAGINGPORT=8888

      # Performance tuning
      - MAXOBJECTS=2162688
      - MAXTICKRATE=30

      # Update behavior
      - SKIPUPDATE=false
      - STEAMBETA=false

      # Run as non-root user
      - PUID=1000
      - PGID=1000

      # Timezone
      - TZ=UTC

    # Limit resources so a runaway server doesn't kill the host
    deploy:
      resources:
        limits:
          memory: 8G
        reservations:
          memory: 4G

    # Limit log size to prevent disk filling up
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"
        compress: "true"

    # Health check
    healthcheck:
      test: ["CMD-SHELL", "pgrep -f FactoryServer || exit 1"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 300s

    # Give server time to save on shutdown
    stop_grace_period: 120s

    security_opt:
      - no-new-privileges:true
```

Now start it up:

```bash
cd /opt/satisfactory
docker compose up -d
```

The first startup will download about 8GB of game files. This takes a while. Monitor progress with:

```bash
docker compose logs -f
```

Press Ctrl+C to stop following logs. The server is ready when you see "Server started" in the output.

## Step 6: Router Configuration

Don't forget to forward ports on your home router to your Proxmox host:

| Port | Protocol | Forward To |
|------|----------|------------|
| 7777 | TCP | 192.168.174.6:7777 |
| 7777 | UDP | 192.168.174.6:7777 |
| 8888 | TCP | 192.168.174.6:8888 |

The exact steps depend on your router. Look for "Port Forwarding" or "NAT" in your router's admin interface.

## Connecting to Your Server

Once everything is running:

1. Launch Satisfactory
2. Go to **Server Manager**
3. Click **Add Server**
4. Enter your public IP (find it at [whatismyip.com][whatismyip])
5. Claim the server and set an admin password

## Verifying It Works

You can verify the network setup is working. From the Proxmox host:

```bash
# This should timeout (container can't reach home network)
pct exec 404 -- ping -c 3 192.168.174.1

# This should work (container can reach internet)
pct exec 404 -- ping -c 3 8.8.8.8
```

If the first ping times out and the second succeeds, the isolation is working.

## Quick Reference

Here are the commands I use most often, run from the Proxmox host:

```bash
# View server logs
pct exec 404 -- docker compose -f /opt/satisfactory/docker-compose.yml logs --tail 50

# Follow logs live
pct exec 404 -- docker compose -f /opt/satisfactory/docker-compose.yml logs -f

# Check server status
pct exec 404 -- docker compose -f /opt/satisfactory/docker-compose.yml ps

# Restart server
pct exec 404 -- docker compose -f /opt/satisfactory/docker-compose.yml restart

# Update server
pct exec 404 -- bash -c "cd /opt/satisfactory && docker compose pull && docker compose up -d"

# Stop server
pct exec 404 -- docker compose -f /opt/satisfactory/docker-compose.yml down

# Start server
pct exec 404 -- docker compose -f /opt/satisfactory/docker-compose.yml up -d
```

## Was It Worth It?

For me? Yes. The setup took an afternoon, but now I have:

- A game server that starts automatically on boot
- Easy updates with `docker compose pull`
- Resource efficiency (no full VM overhead)
- Some peace of mind from the network isolation

The friends I play with don't notice any difference from a commercial hosted server. And I learned a fair bit about Linux networking along the way.

Is it perfect? Probably not. Is it better than just running the server directly on my main network? I think so.

---

If you have suggestions, improvements, or spot any issues, please get in touch at [mattcree@proton.me](mailto:mattcree@proton.me). This was very much a learning exercise for me, and I'd love to hear from folks with more experience.

[satisfactory]: https://www.satisfactorygame.com/
[dockerhub]: https://hub.docker.com/r/wolveix/satisfactory-server
[whatismyip]: https://whatismyip.com
