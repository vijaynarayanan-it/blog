---
date: '2025-06-15T09:30:42+02:00'
draft: false
title: 'Disable Proxmox Enterprise Subscription Alert'
tags: [proxmox, homelab]
---
# How to Remove the Proxmox VE Subscription Warning

![Proxmox Subscription Warning Image](/images/proxmox-ve-enterprise-subscription-alert-box-screenshot.jpeg)

## Introduction

If you don’t have a paid Proxmox subscription, you’ll see a warning about the Enterprise repository. This is normal for home labs, but you can easily switch to the free no-subscription repository and get rid of the alert.

---

## Steps to Fix the Warning

1. **Open the Proxmox APT sources file:**

```
nano /etc/apt/sources.list.d/pve-enterprise.list
```

2. **Disable the enterprise repository:**

Just add a `#` at the start of the line so it looks like this:

```
# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
```

3. **Add the no-subscription repository:**

Open your main sources list:

```
nano /etc/apt/sources.list
```

Add this line if it’s not already there:

```
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

4. **(Optional) If you use Ceph, disable the enterprise Ceph repo:**

Edit the Ceph sources file:

```
nano /etc/apt/sources.list.d/ceph.list
```

Comment out the enterprise line and add the no-subscription one:

```
# deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise
deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription
```

5. **Update your package lists:**

```
apt-get update
```

That’s it! The warning should be gone, and you’ll still get updates from the free repository.
```