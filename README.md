# Holoneon OpenSimulator Region Self-Hosting Guide

by Fiona Sweet <fiona@pobox.holoneon.com> - updated 2026-06-04

## How to Host Your Own OpenSim Region at Home Using a Small VPS

You do not need to host your region on someone else's grid server. A practical alternative is to run OpenSim on your own home machine and use a small, inexpensive VPS as a public internet gateway.

This approach keeps your region server under your control while avoiding common home networking problems such as router port forwarding complications, dynamic home IP addresses, and CGNAT.

---

## How the Traffic Flows

```
Internet / Viewer
      |
      v
your-region.example.com  (DNS A record -> VPS public IP)
      |
      v
Small public VPS  (forwards TCP + UDP region port)
      |
      v
WireGuard VPN tunnel
      |
      v
OpenSim running on your home machine
```

### Example Addresses

| Role                  | Address              |
|-----------------------|----------------------|
| VPS public IP         | 203.0.113.50         |
| VPS WireGuard IP      | 10.99.0.1            |
| Home WireGuard IP     | 10.99.0.2            |
| OpenSim region port   | 8050                 |
| Public region name    | region.example.com   |

The viewer connects to `region.example.com:8050`. The VPS forwards that traffic through the WireGuard tunnel to `10.99.0.2:8050`. This works for both TCP and UDP, including the UDP traffic OpenSim regions require.

---

## What You Need

1. A small VPS with a public IPv4 address (a $5/month plan is usually sufficient)
2. A domain name or subdomain pointing to the VPS
3. WireGuard installed on both the VPS and your home machine
4. OpenSim running on your home machine
5. nginx stream proxy on the VPS to forward traffic
6. TCP and UDP forwarded for each OpenSim region port

The VPS only acts as a relay. The actual OpenSim workload runs at home, so a minimal VPS is fine.

---

## DNS

Create an A record pointing your subdomain at the VPS public IP:

```
region.example.com  ->  203.0.113.50
```

Do not point DNS at your home IP. The VPS is the public-facing address.

### IPv4 Notes

Use IPv4 for the public region address. Do not rely on IPv6-only hosting. OpenSim viewers, UDP region handshakes, NAT/proxy setups, and older grid assumptions all work more reliably over IPv4.

- Use a VPS with a public IPv4 address
- Create a DNS A record only
- Forward both TCP and UDP region ports
- Set ExternalHostName to the IPv4-backed hostname
- Do not publish an AAAA record until you have confirmed IPv6 works end-to-end

---

## WireGuard Setup

The VPS acts as the WireGuard server. The home machine connects outbound to it. This usually works even if your home internet is behind NAT.

| Host     | WireGuard IP |
|----------|--------------|
| VPS      | 10.99.0.1    |
| Home     | 10.99.0.2    |

### VPS WireGuard Configuration

Create `/etc/wireguard/wg0.conf` on the VPS:

```ini
[Interface]
Address = 10.99.0.1/24
ListenPort = 51820
PrivateKey = VPS_PRIVATE_KEY

[Peer]
PublicKey = HOME_PUBLIC_KEY
AllowedIPs = 10.99.0.2/32
```

Enable and start the tunnel:

```bash
sudo systemctl enable --now wg-quick@wg0
```

Open the WireGuard port on the VPS firewall:

```bash
sudo ufw allow 51820/udp
```

### Home Machine WireGuard Configuration

Create `/etc/wireguard/wg0.conf` on the home machine:

```ini
[Interface]
Address = 10.99.0.2/24
PrivateKey = HOME_PRIVATE_KEY

[Peer]
PublicKey = VPS_PUBLIC_KEY
Endpoint = VPS_PUBLIC_IP:51820
AllowedIPs = 10.99.0.1/32
PersistentKeepalive = 25
```

Enable and start the tunnel:

```bash
sudo systemctl enable --now wg-quick@wg0
```

### Test the Tunnel

From the VPS, ping the home machine:

```bash
ping 10.99.0.2
```

If that succeeds, the tunnel is working.

---

## Forwarding an OpenSim Region Port

For a region running on port 8050, you need to forward both TCP and UDP on that port from the VPS to `10.99.0.2:8050`.

The easiest method is nginx with stream proxy support.

### Install nginx Stream Support

```bash
sudo apt update
sudo apt install nginx libnginx-mod-stream
```

### Configure the Stream Block

Add the following to `/etc/nginx/nginx.conf`, outside the existing `http {}` block:

```nginx
stream {
    upstream opensim_region_8050_tcp {
        server 10.99.0.2:8050;
    }

    upstream opensim_region_8050_udp {
        server 10.99.0.2:8050;
    }

    server {
        listen 8050;
        proxy_pass opensim_region_8050_tcp;
        proxy_timeout 1h;
        proxy_connect_timeout 10s;
    }

    server {
        listen 8050 udp;
        proxy_pass opensim_region_8050_udp;
        proxy_timeout 1h;
        proxy_responses 0;
    }
}
```

### Reload nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Open the Port on the VPS Firewall

```bash
sudo ufw allow 8050/tcp
sudo ufw allow 8050/udp
```

Public traffic to `region.example.com:8050` will now be forwarded through the tunnel to your home OpenSim server at `10.99.0.2:8050`.

---

## OpenSim Region Configuration

On the home machine, configure OpenSim to listen on port 8050 but advertise the public VPS hostname.

Edit your `Regions.ini`:

```ini
[My Home Region]
RegionUUID = 11111111-2222-3333-4444-555555555555
Location = 1000,1000
SizeX = 256
SizeY = 256

InternalAddress = 0.0.0.0
InternalPort = 8050
AllowAlternatePorts = False
ExternalHostName = region.example.com
```

### What to Set and What to Avoid

Set `ExternalHostName` to the VPS hostname:

```
ExternalHostName = region.example.com
```

Do not use the WireGuard tunnel IP:

```
ExternalHostName = 10.99.0.2   <- wrong, not reachable from the internet
```

Do not use a private home LAN address:

```
ExternalHostName = 192.168.1.50  <- wrong, not reachable from the internet
```

---

## Running Multiple Regions

Each region needs its own port. For example:

| Region   | Port |
|----------|------|
| Region 1 | 8050 |
| Region 2 | 8051 |
| Region 3 | 8052 |

The VPS must forward both TCP and UDP for each port.

### nginx Stream Configuration for Multiple Regions

```nginx
stream {
    upstream opensim_region_8050 {
        server 10.99.0.2:8050;
    }

    upstream opensim_region_8051 {
        server 10.99.0.2:8051;
    }

    server {
        listen 8050;
        proxy_pass opensim_region_8050;
    }

    server {
        listen 8050 udp;
        proxy_pass opensim_region_8050;
        proxy_responses 0;
    }

    server {
        listen 8051;
        proxy_pass opensim_region_8051;
    }

    server {
        listen 8051 udp;
        proxy_pass opensim_region_8051;
        proxy_responses 0;
    }
}
```

### Open All Region Ports on the Firewall

```bash
sudo ufw allow 8050/tcp
sudo ufw allow 8050/udp
sudo ufw allow 8051/tcp
sudo ufw allow 8051/udp
```

---

## Standalone Mode vs. Grid Mode

For most people, standalone mode is the right place to start.

Standalone mode gives you:

- Your own users
- Your own inventory and assets
- Your own region and database
- No dependency on another grid's services

This is the simplest and safest self-hosting configuration. Once you are comfortable with it, you can explore Hypergrid, which allows travel between compatible OpenSim grids without giving any external grid direct access to your server infrastructure.

---

## Why This Is Better Than Hosting on Someone Else's Grid Server

Running your own OpenSim server gives you full control over:

- Region files and OAR exports
- Backups and OpenSim version
- Estate and script settings
- Database and asset storage
- Server restarts

It also avoids needing privileged access to another grid operator's infrastructure. A grid operator generally cannot safely grant unknown users direct access to their production servers, Robust services, databases, or asset services. This VPS and WireGuard setup gives you your own public-facing entry point without depending on anyone else's systems.

---

## Limitations and Important Notes

### Home Upload Speed Matters

Region performance depends on your home internet upload speed. A slow upload connection may cause:

- Slow texture loading
- Movement rubber-banding
- Slow object rez
- Teleport failures

### VPS Location Matters

Pick a VPS geographically close to your home. Traffic travels:

```
viewer -> VPS -> home -> VPS -> viewer
```

A nearby VPS minimizes added latency.

### UDP Is Required

Forwarding TCP alone is not sufficient. OpenSim regions require UDP for viewer traffic. For every region port, forward both TCP and UDP.

### Use One Port Per Region

Do not attempt to run multiple regions on the same public port. Assign each region a unique port number.

### Keep the VPS Locked Down

Expose only the ports you actually need:

| Port         | Purpose                          |
|--------------|----------------------------------|
| 22/tcp       | SSH (restrict access if possible)|
| 51820/udp    | WireGuard                        |
| 8050/tcp+udp | OpenSim region                   |

Do not expose databases, admin panels, or OpenSim remote admin ports to the public internet.

---

## Summary

The complete setup in brief:

1. DNS A record points your subdomain to the VPS public IP
2. WireGuard tunnel connects the home machine to the VPS
3. nginx on the VPS forwards TCP and UDP region ports through the tunnel to the home machine
4. OpenSim on the home machine uses the VPS hostname as ExternalHostName

```
region.example.com:8050
        |
        v
VPS public port 8050
        |
        v
WireGuard tunnel
        |
        v
Home OpenSim server at 10.99.0.2:8050
```

This gives you full self-hosting control without needing another grid operator to host your simulator or grant access to their infrastructure.
