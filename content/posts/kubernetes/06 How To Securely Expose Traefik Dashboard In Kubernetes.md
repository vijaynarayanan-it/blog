---
date: 2025-07-05
draft: false
title: 'How To Securely Expose Traefik Dashboard In Kubernetes'
slug: 'how-to-securely-expose-traefik-dashboard-in-kubernetes'
tags: [☸️ kubernetes, 🌐 cloudflare, 🔐 Authentik, 🛠️ traefik]
---

# Introduction

In this guide, I will explain how to securely expose the Traefik dashboard in a Kubernetes cluster using Cloudflare.
The Traefik dashboard provides insights into the traffic and routing within your cluster, but it should be secured to prevent unauthorized access.

# Prerequisites
- A Kubernetes cluster with Traefik installed.
- Helm package manager installed and has required permissions to install and manage resources in the cluster.
- Cloudflare account with your domain configured.
- [Already configured Cloudflare tunnel to use Traefik Ingress Controller](https://www.vijay-narayanan.com/posts/kubernetes/how-to-set-up-traefik-ingress-controller-in-kubernetes).





