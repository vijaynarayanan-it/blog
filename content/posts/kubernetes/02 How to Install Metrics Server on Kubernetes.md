---
date: 2025-02-15
draft: false
title: 'How to install Metrics Server on Kubernetes'
slug: 'install-metrics-server-kubernetes'
tags: [kubernetes]
---

# Introduction

Metrics Server is a cluster-wide aggregator of resource usage data in Kubernetes.
It collects metrics from the kubelet on each node and provides them to the Kubernetes API server, which can be used for horizontal pod autoscaling and other purposes.

![how-metrics-server-works.png](/images/how-metrics-server-works.png)

# Prerequisites
- A running Kubernetes cluster (version 1.8 or later).
- `kubectl` command-line tool installed and configured to communicate with your cluster.

# Installation Steps

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

# Verification

To verify that Metrics Server is running correctly, you can check the status of the Metrics Server pod:

```bash
kubectl get pods -n kube-system
```

You should see a pod named `metrics-server-<hash>` in the `Running` state.

You can also check if Metrics Server is collecting metrics by running:

```bash
kubectl top nodes
```

Example output:

```
vijay@controlplane:~$ kubectl top nodes
NAME           CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
controlplane   89m          1%       2672Mi          22%
node01         41m          2%       1718Mi          44%
node02         59m          2%       2460Mi          31%
```

To check metrics for pods in all namespaces, you can run:

```bash
kubectl top pods --all-namespaces
```

To check metrics for a specific namespace, you can run:

```bash
kubectl top pods -n <namespace>
```

---