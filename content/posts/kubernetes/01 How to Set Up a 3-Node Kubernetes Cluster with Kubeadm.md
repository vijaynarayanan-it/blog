---
date: 2025-02-15
draft: false
title: 'How to Set Up a 3-Node Kubernetes Cluster with Kubeadm'
slug: 'kubeadm-3-node-kubernetes-cluster'
tags: [☸️ kubernetes, kubeadm]
---

![kubeadm-3node-cluster-setup.png](/images/kubeadm-3node-cluster-setup.png)

## Introduction

This guide will walk you through the following steps to set up a 3-node Kubernetes cluster using kubeadm:

- Configure unique hostnames for each node.
- Set up networking and update the `/etc/hosts` file.
- Install required system packages and disable swap.
- Install and configure the container runtime (`containerd`) and enable IP forwarding.
- Install Kubernetes components: `kubeadm`, `kubelet`, and `kubectl`.
- Initialize the control plane node with `kubeadm`.
- Set up pod networking using Calico CNI.
- Join worker nodes to the cluster.
- Verify the cluster status and apply additional configurations.

---

## Prerequisites

- Create three VMs or physical servers with Ubuntu 22.04 LTS or later.
- Make sure all nodes can communicate with each other over the network and has internet access.

---

## Setup

### Step 1: Setup Hostnames on all nodes

On each node, set a unique hostname using the following command:

```
sudo hostnamectl set-hostname <your-node-hostname>
```

For example, you can use the following hostnames:

```bash
# on control plane node
sudo hostnamectl set-hostname controlplane

# on worker1 node
sudo hostnamectl set-hostname worker-node01

# on worker2 node
sudo hostnamectl set-hostname worker-node02
```

---

### Step 2: Configure Node IP address details in all nodes

Edit the `/etc/hosts` file on all nodes to include the IP addresses and hostnames of all nodes in the cluster.

```bash
sudo nano /etc/hosts
```

Add the following lines to the file:

Make sure to replace the IP addresses with your actual node IPs. In my case, IPs of the nodes are:

```
# /etc/hosts

10.10.10.50 controlplane
10.10.10.51 worker-node01
10.10.10.52 worker-node02
```

---

### Step 3: Update and Install Required Packages on all nodes

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg lsb-release gnupg software-properties-common
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab
```

| Package                     | Purpose                                                                 |
|-----------------------------|-------------------------------------------------------------------------|
| apt-transport-https         | Enables apt to use HTTPS for downloading packages.                      |
| ca-certificates             | Provides trusted SSL certificates for secure connections.               |
| curl                        | Tool for transferring data from URLs, often used to download files.     |
| gpg and gnupg               | Tools for managing GPG keys, used to verify package authenticity.       |
| lsb-release                 | Provides Linux Standard Base information about the system.              |
| software-properties-common  | Adds scripts for managing apt repositories.                             |

| Command                                             | Purpose                                                                                                    |
|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `sudo swapoff -a`                                   | Ensures that swap is disabled, which is a requirement for Kubernetes to function properly.                 |
| `sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab` | Comments out any swap entries in the `/etc/fstab` file to prevent swap from being enabled on reboot.       |

---

### Step 4: Install Containerd and Enable IP Forwarding

Before installing Kubernetes components, we need to set up the container runtime and enable IP forwarding.
IP forwarding is required for Kubernetes to allow communication between pods across nodes.

Make sure to run the following steps on **all nodes** (control plane and worker nodes).

#### Step 4.1: Enable IP Forwarding on all nodes

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Then make it persistent across reboots by editing the sysctl configuration:

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-kubernetes.conf
sudo sysctl --system
```

#### Step 4.2: Install Containerd on all nodes

```bash
sudo apt install -y containerd

# Create default config file
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Change cgroupDriver to systemd
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

### Step 5: Install kubeadm, kubelet, kubectl on all nodes

Note: I am using Kubernetes version `v1.32.0` in this guide. You can change it to the latest stable version if needed.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### Step 6: Initialize the ControlPlane Node (Only on controlplane node)

#### Step 6.1: Initialize the kubeadm on control plane

Run below command only on the controlplane node. **Not on worker nodes**.

If you want to use different CIDR for pod or service, you can change the `--pod-network-cidr` and `--service-cidr` options accordingly.

Kept `192.168.0.0/16` for pod network CIDR as it is the default config for Calico CNI. If you plan to change it, make sure to update the Calico manifest later.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.10.10.50 \
  --pod-network-cidr=192.168.0.0/16 \
  --service-cidr=10.100.0.0/16 \
  --kubernetes-version=v1.32.0
```

##### Step 6.2: Set up kubeconfig for the controlplane node

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### Step 7: Install Calico CNI for Pod Networking (Only on controlplane node)

Since our CIDR for pod and service are below:

- `--pod-network-cidr=192.168.0.0/16`: for Calico or your CNI

- `--service-cidr=10.100.0.0/16`: for Kubernetes services (ClusterIP, etc.)

- Make sure these do not overlap with your existing network ranges.

When using a non-default pod CIDR, you must modify the Calico manifest accordingly.

#### Step 7.1: Download and Apply Calico Manifest

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

#### Step 7.2: Edit Calico Manifest

```bash
sudo nano calico.yaml
```

Search for the following block:

```yaml
- name: CALICO_IPV4POOL_CIDR     
  value: "192.168.0.0/16"
```

Make sure it matches your `--pod-network-cidr` you used during `kubeadm init`.

#### Step 7.3: Apply the Calico Manifest

```bash
kubectl apply -f calico.yaml -n kube-system
```

Wait for the Calico pods to be up and running.

```bash
kubectl get pods -n kube-system
```

Example output:
```
vijay@controlplane:~$ kubectl get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS        AGE
calico-kube-controllers-7967497bcf-j2tj2    1/1     Running   0               60s
calico-node-8sqtx                           1/1     Running   0               40s
calico-node-mgjbt                           1/1     Running   0               41s
calico-node-qkdsz                           1/1     Running   0               43s
```

Now your control plane node is ready with Calico CNI installed and ready for worker nodes to join.

---

### Step 8: Join Worker Nodes to the Cluster

From `kubeadm init`, you’ll get a `kubeadm join` command. Run it on both `worker-node01` and `worker-node02`.

Example:

```bash
sudo kubeadm join 10.10.10.50:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If you lost the token, re-generate it using the following command on the controlplane node:

```bash
sudo kubeadm token create --print-join-command
```

---

### Step 9: Verify Cluster Status and add additional configurations

After joining the worker nodes, you can verify the cluster status from the controlplane node.

```
kubectl get nodes
```

You should see:

```
controlplane   Ready    control-plane
worker-node01  Ready    <none>
worker-node02  Ready    <none>
```

#### Step 9.1: Set up kubeconfig for root user

On the control plane node, only if you want to use kubectl from your controlplane node as root user.

```bash
sudo cp /etc/kubernetes/admin.conf /root/.kube/config
sudo chown root:root /root/.kube/config
```

#### Step 9.2: Enable Bash Completion for kubectl

```bash
sudo apt install -y bash-completion
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

---

## Conclusion

You have successfully set up a 3-node Kubernetes cluster using `kubeadm`. The control plane node is running with Calico CNI for pod networking, and the worker nodes are joined to the cluster. You can now deploy applications and manage your Kubernetes cluster effectively.