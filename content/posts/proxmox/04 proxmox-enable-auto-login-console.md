---
date: 2025-02-11
draft: false
title: 'How to Enable Auto Login on Proxmox Console'
slug: 'proxmox-enable-auto-login-console'
tags: [proxmox, homelab]
---
# How to Enable Auto Login on Proxmox Console (TTY)

#### What is tty?

`tty` stands for "teletypewriter" and refers to a terminal interface in Unix-like operating systems. It allows users to interact with the system through a command-line interface.

#### What is `getty`?

`getty` is a program that manages physical or virtual terminals on Unix-like systems. It is responsible for prompting for a login name and starting the login process.

If you want to enable auto-login on the Proxmox console (TTY), you can do this by modifying the `getty` service configuration. This allows you to log in automatically without entering a username and password each time you access the console.
This is helpful when your homelab server restarts for some reason, and you want to avoid manual login.

### Step 1: Edit the `getty` service configuration

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d/
```

Create an override config:

```bash
sudo nano /etc/systemd/system/getty@tty1.service.d/override.conf
```

Add this content:

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin root --noclear %I $TERM
```

The line `ExecStart=` clears the default start command.

The next line sets a new start command:

`ExecStart=-/sbin/agetty --autologin root --noclear %I $TERM`
This tells agetty to automatically log in as root on TTY1 without prompting for credentials.

> If you want to auto-login as a different user, replace `root` with your desired username.

---

### Step 2: Reload systemd and restart getty

```bash
sudo systemctl daemon-reload
sudo systemctl restart getty@tty1
```

---

### Step 3: Test it

- Switch to tty1 by pressing `Ctrl + Alt + F1` or reboot to test fresh login
- It should log in automatically as `root` or your user

---

# Conclusion

Now you have successfully enabled auto-login on the Proxmox console (TTY). This setup is particularly useful for homelab environments where you want quick access without manual login steps after reboots or power failures.

But be cautious with security, as auto-login can expose your system to unauthorized access if someone has physical access to the server.

Also note when you access the Proxmox web interface, you will still need to log in with your credentials.