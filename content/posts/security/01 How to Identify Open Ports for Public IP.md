---
date: 2025-06-28
draft: false
title: 'Check Your Public IP for Exposed Open Ports'
slug: 'check-public-ip-open-ports'
tags: [🛡️ security]
---

# Introduction

This guide will help you identify if your public IP address has any open ports or services that could be unintentionally exposed to the internet.
This is crucial for maintaining the security of your network and devices.

Based on RFC1918, private IP addresses are not routable on the public internet.

What is RFC1918?

- RFC1918 is a standard that defines private IP address ranges that are reserved for use within private networks.
- RFC1918 designates the following three ranges for private networks:
  - 10.0.0.0 - 10.255.255.255: (10.0.0.0/8)
  - 172.16.0.0 - 172.31.255.255: (172.16.0.0/12)
  - 192.168.0.0 - 192.168.255.255: (192.168.0.0/16)

# Steps

You can use online tools to scan your public IP address for open ports and services.
Use a port scan tool like **Censys** to search for a public IP address.

## Step 1: Find Your Public IP Address

To check if, any devices or services (e.g. Proxmox, SSH, or web servers) are unintentionally exposed to the internet.

Find your public IP address:

    - macOS/Linux:
      ```bash
      curl ifconfig.me
      ```
    - Windows (PowerShell):
      ```powershell
      curl ifconfig.me
      ```

Lets say your public IP is `88.90.123.456`. Copy this IP address and proceed to the next step.

---

## Step 2: Use Censys to Search for Open Ports

Go to [https://search.censys.io](https://search.censys.io)

Paste the public IP into the search bar and view any open ports or services listed.

For example:

![censys-public-ip-result.png](/images/censys-public-ip-result.png)

---

# Conclusion

✅ If no results show up, your network is likely not exposing anything publicly.

❌ If results show up, review the services and ports listed to ensure they are intended to be public.

---