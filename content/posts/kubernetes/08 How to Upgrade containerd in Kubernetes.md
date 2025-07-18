---
date: 2025-07-05
draft: false
title: 'How to Upgrade containerd in Kubernetes'
slug: 'upgrade-containerd-kubernetes'
tags: [☸️ kubernetes, 📦 containerd]
---

![upgrade-containerd-version.png](/images/upgrade-containerd-kubernetes.png)

# Introduction

Containerd is a core component of Kubernetes that manages container lifecycle.
Upgrading containerd can bring performance improvements, bug fixes, and new features.
This guide will walk you through the steps to upgrade Containerd on a Kubernetes node.

# Deployment Script

```bash
CURRENT_VERSION="v2.1.3"
ARCH="linux-amd64"
DOWNLOAD_URL="https://github.com/containerd/containerd/releases/download/${CURRENT_VERSION}/containerd-${CURRENT_VERSION#v}-${ARCH}.tar.gz"

echo "Draining node..."
kubectl drain $(hostname) --ignore-daemonsets --delete-emptydir-data

echo "Stopping containerd..."
sudo systemctl stop containerd

echo "Removing old containerd..."
sudo apt remove -y containerd

echo "Downloading containerd $CURRENT_VERSION..."
wget -q $DOWNLOAD_URL -O containerd.tar.gz

echo "Extracting and installing containerd..."
tar -xvf containerd.tar.gz
sudo cp bin/* /usr/local/bin/

echo "Setting up systemd service..."
sudo systemctl unmask containerd
sudo wget -q -O /etc/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

echo "Generating config and setting SystemdCgroup..."
sudo mkdir -p /etc/containerd
sudo bash -c "containerd config default > /etc/containerd/config.toml"
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

echo "Restarting containerd and kubelet..."
sudo systemctl restart containerd
sudo systemctl restart kubelet

echo "Updated. Current containerd version:"
containerd --version
```