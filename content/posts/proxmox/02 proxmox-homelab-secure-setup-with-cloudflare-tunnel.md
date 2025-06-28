---
date: 2025-02-10
draft: false
title: 'Secure and Isolated Proxmox with Cloudflare Tunnel'
slug: 'secure-proxmox-with-cloudflare-tunnel'
tags: [proxmox, 🏠 homelab]
---

## Introduction

This guide shows you how to set up a secure and flexible Proxmox VE homelab. You will:

- Isolate your VM network but keep internet access.
- Securely access the Proxmox web UI using Cloudflare Tunnel and custom DNS.
- Block direct IP access to the Proxmox UI.
- Prepare for adding more services in the future.

![proxmox-ve-setup-title-image.png](/images/proxmox-ve-setup-title-image.png)

## Prerequisite

Before you start, ensure you have:

- Already installed Proxmox VE on your machine.
- A basic understanding of Linux command line.
- A Cloudflare account with a domain set up (e.g., `yourdomain.com`).
- Already created a linux bridge network in Proxmox for your VMs. Check out my blog post on [How to Configure DHCP Server to Create vmbr Bridge Network](https://www.vijay-narayanan.com/posts/how-to-configure-dhcp-server-to-create-vmbr-bridge-network) for guidance.

## Example values we are going to use

Assuming you have a Proxmox VE installation with the following network configuration:
Note that these values are examples. You should replace them with your actual network settings.

| Setting         | Example Value                | Description                |
|-----------------|-----------------------------|----------------------------|
| **LAN IP**      | `10.20.30.40/24`            | Proxmox server IP address  |
| **Gateway/DNS** | `10.20.30.1`                | Default gateway and DNS    |
| **Hostname**    | `homelab.yourdomain.com`    | Proxmox hostname           |
| **Domain**      | `yourdomain.com`            | Cloudflare-managed domain  |

---

## Steps to Secure and Isolate Proxmox VE Homelab

### Step 1: Initial Proxmox VE Setup

```
apt-get update
apt-get upgrade
```

#### Step 1.1: Open `/etc/hostname` and set the following entry:

You can use any hostname you prefer, but for this guide, we will use: homelab

```
homelab
```

#### Step 1.2: Open and check values for `/etc/hosts`:

```
127.0.0.1 localhost.localdomain localhost
10.20.30.40 homelab.yourdomain.com homelab
```

---

### Step 2: Isolate your VM network but keep internet access

To create an isolated linux bridge network, check out my blog [How to Configure DHCP Server to Create vmbr Bridge Network](https://www.vijay-narayanan.com/posts/how-to-configure-dhcp-server-to-create-vmbr-bridge-network).

If you want your Proxmox and its VMs on an isolated internal virtual network, but still able to access the internet. Follow these steps:

#### Step 2.1: Network Plan

Use Proxmox’s Linux bridges for VM/Container networks.

- `vmbr0`: Connected to physical NIC (`eno1`) — your main interface (`10.20.30.40`)
- `vmbr1`: Internal-only isolated virtual bridge (no physical NIC attached)
- Configure NAT for `vmbr1` to allow internet for VMs while keeping them isolated

#### Step 2.2: Create NAT Bridge (Isolated VM Network)

Edit `/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

iface enp3s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 10.20.30.40/24
        gateway 10.20.30.1
        bridge-ports enp3s0
        bridge-stp off
        bridge-fd 0
        dns-nameservers 10.20.30.1

auto vmbr1
iface vmbr1 inet static
        address 10.10.10.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE

iface enp2s0 inet manual

iface wlp4s0 inet manual

source /etc/network/interfaces.d/*
```

Then run to apply the changes

```
ifreload -a
```

Now, your VMs on `vmbr1` can access the internet but are not exposed to your LAN.

---

### Step 3: Secure Remote Access via Cloudflare Tunnel

To expose the Proxmox UI (`8006`) securely via your domain, use Cloudflare Tunnel (`cloudflared`). No port forwarding is needed.

#### Step 3.1: Install `cloudflared` on Proxmox

```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
```

#### Step 3.2: Authenticate with Cloudflare

```
cloudflared tunnel login
```

Follow the browser instructions. Select your domain: `yourdomain.com`.

#### Step 3.3: Create the Tunnel

```
cloudflared tunnel create proxmox-tunnel
```

This gives a tunnel UUID like `a1b2c3d4...`

#### Step 3.4: Configure the Tunnel

Create `/etc/cloudflared/config.yml`:

```yaml
tunnel: a1b2c3d4...         # from previous step
credentials-file: /root/.cloudflared/875fc262-b752-46cf-958d-86f35815deed.json  # from previous step

ingress:
  - hostname: homelab.yourdomain.com
    service: https://localhost:8006
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

#### Step 3.5: Route DNS in Cloudflare

Run below command. This will create a DNS record in Cloudflare for your tunnel.

```
cloudflared tunnel route dns proxmox-tunnel homelab.yourdomain.com
```

#### Step 3.6: Run the Tunnel as a Service

```
cloudflared service install
systemctl enable --now cloudflared
```

Now, only users with access to `homelab.yourdomain.com` via Cloudflare can access your Proxmox UI.

---

### Step 4: Block IP Access to `https://<your-proxmox-ip>:8006`

We’ll achieve this using Proxmox firewall rules using `iptables`.

Use `iptables` (if you don’t want to use Proxmox firewall)

#### Step 4.1: Block direct access to Proxmox UI

```
iptables -A INPUT -p tcp --dport 8006 ! -s 127.0.0.1 -j DROP
```

#### Step 4.2: Save the iptables rules by running below command

```
apt install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

---

### Step 5: Verify It Works

Now try:

- `https://<your-public-LAN-IP>:8006` → Should fail (connection refused or timeout)
- `https://homelab.yourdomain.com` → Should work perfectly

---

## Conclusion

| Objective | Implementation Details                                        |
|---|---------------------------------------------------------------|
| Isolated network with internet | Created `vmbr1`, apply NAT masquerading via `vmbr0`           |
| Secure remote Proxmox access | Used Cloudflare Tunnel and custom DNS (`homelab.yourdomain.com`) |
| Block access via IP/Port | web UI to `127.0.0.1:8006` blocked                    |
