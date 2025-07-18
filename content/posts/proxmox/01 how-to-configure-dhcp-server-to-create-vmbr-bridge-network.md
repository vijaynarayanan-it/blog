---
date: 2025-02-09
draft: false
title: 'How to Configure a DHCP Server for vmbr Bridge Networking'
slug: 'configure-dhcp-vmbr-bridge'
tags: [proxmox, 🏠 homelab]
---

## Introduction

This guide will help you set up a DHCP server on your Proxmox host to create a bridge network (`vmbr1`) for your virtual machines (VMs). This setup allows VMs to automatically receive IP addresses and network configuration from the DHCP server.

Also, helps in isolating the VM network while still providing internet access.

## Install DHCP Server and Configure Bridge Network

Install DHCP server on Proxmox host:

```bash
apt install isc-dhcp-server
```

Set interface in `/etc/default/isc-dhcp-server`:

```ini
INTERFACESv4="vmbr1"
```

Configure DHCP in `/etc/dhcp/dhcpd.conf`:

```conf
# dhcpd.conf

default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 10.10.10.0 netmask 255.255.255.0 {
  range 10.10.10.50 10.10.10.100;
  option routers 10.10.10.1;
  option domain-name-servers 1.1.1.1, 8.8.8.8;
}
```

Configuration Breakdown:

| Configuration Option                                      | Description                                                                                      |
|-----------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| `subnet 10.10.10.0 netmask 255.255.255.0 { ... }`         | Defines a DHCP scope for the 10.10.10.0/24 network.                                              |
| `range 10.10.10.50 10.10.10.100;`                         | Specifies the pool of IP addresses to assign to clients.                                         |
| `option routers 10.10.10.1;`                              | Sets the default gateway for DHCP clients.                                                       |
| `option domain-name-servers 1.1.1.1, 8.8.8.8;`            | Specifies DNS servers for DHCP clients. `1.1.1.1` is Cloudflare's DNS, and `8.8.8.8` is Google's DNS. |

Start the service:

```bash
systemctl restart isc-dhcp-server

systemctl status isc-dhcp-server
```

Then reboot your VM or run:

```bash
sudo dhclient eth0
```

## Conclusion

This setup allows your VMs to automatically receive IP addresses from the DHCP server configured on `vmbr1`. The DHCP server will assign IPs within the specified range, set the default gateway, and provide DNS servers for name resolution.
