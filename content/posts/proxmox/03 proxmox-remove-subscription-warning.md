---
date: 2025-02-10
draft: false
title: 'How to Remove the Proxmox Subscription Warning'
slug: 'remove-proxmox-subscription-warning'
tags: [proxmox, 🏠 homelab]
---

## Introduction

![Proxmox Subscription Warning Image](/images/proxmox-ve-enterprise-subscription-alert-box-screenshot.jpeg)

If you don’t have a paid Proxmox subscription, you’ll see a warning about the Enterprise repository. This is normal for home labs, but you can easily switch to the free no-subscription repository and get rid of the alert.

---

## Steps to Fix the Warning

### Step 1: Open the Proxmox APT sources file

```
nano /etc/apt/sources.list.d/pve-enterprise.list
```

---

### Step 2: Disable the enterprise repository

Add a `#` at the start of the line so it looks like this:

```
# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
```

---

### Step 3: Add the no-subscription repository

Open your main sources list:

```
nano /etc/apt/sources.list
```

Add this line if it’s not already there:

```
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

---

### Step 4: If you use Ceph, disable the enterprise Ceph repo (Optional)

Edit the Ceph sources file:

```
nano /etc/apt/sources.list.d/ceph.list
```

Comment out the enterprise line and add the no-subscription one:

```
# deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise
deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription
```

---

### Step 5: Update your package lists

```
apt-get update
```

That’s it! The warning should be gone, and you’ll still get updates from the free repository.

---

## Conclusion

Now you can use Proxmox without the subscription warning. This is perfect for home labs or testing environments where you don’t need a paid subscription. Enjoy your Proxmox experience without the distraction of the subscription alert!