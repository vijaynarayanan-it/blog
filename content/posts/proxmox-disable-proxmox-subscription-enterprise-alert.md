# How to Remove Enterprise Proxmox VE Subscription

## **Introduction**

Proxmox tries to access the Enterprise repositories, which require a paid subscription. Without a subscription, access is denied with a `401 Unauthorized` error.

### How to Fix (Use the No-Subscription Repository):

You can switch from the Enterprise repository to the No-Subscription repository, which is free to use and intended for home labs and non-commercial setups.

---

### Step-by-Step Fix:

1. **Edit the Proxmox APT sources:**

```
nano /etc/apt/sources.list.d/pve-enterprise.list
```

2. **Comment out or delete the enterprise repo:**

Change:

```
deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
```

To:

```
# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
```

3. **Add the no-subscription repository:**

```
nano /etc/apt/sources.list
```

Add this line if it's not already present:

```
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

4. **(Optional) If using Ceph, also disable the enterprise Ceph repo:**

Edit:

```
nano /etc/apt/sources.list.d/ceph.list
```

Comment out:

```
deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise
```

Replace it with (or just add this in your `/etc/apt/sources.list`):

```
deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription
```

5. **Update APT again:**

```
apt-get update
```