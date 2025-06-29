---
date: 2025-02-18
draft: false
title: 'Expose Kubernetes Applications Securely to the Internet with Cloudflare Tunnel and Nginx Ingress'
slug: 'kubernetes-expose-app-cloudflare-tunnel-nginx-ingress'
tags: [☸️ kubernetes, 🌐 cloudflare]
---

# Introduction

In this guide, we will learn how to expose a Kubernetes application securely to the internet using Cloudflare Tunnel and Nginx Ingress. This setup allows you to leverage Cloudflare's security features while managing your application traffic efficiently.

![intro-expose-k8s-apps-to-internet.png](/images/intro-expose-k8s-apps-to-internet.png)

We are going to use:

- Cloudflare Tunnel to expose our application securely to the internet.
- Kubernetes Nginx Ingress to route traffic to our application.

# Prerequisites

- A Cloudflare account with the domain added.
- A Kubernetes cluster set up with Nginx Ingress Controller installed.
- Root or sudo access to the Kubernetes cluster.

# Deployment Guide

### Step 1: Install Cloudflare Tunnel

```bash
sudo apt-get update && sudo apt-get install cloudflared
```

---

### Step 2: Authenticate Cloudflare Tunnel

```bash
sudo cloudflared tunnel login
```

Don't worry, if you see login url in the server terminal, just copy it and paste it in your personal browser. After logging in, you will see a success message in the server terminal.

---

### Step 3: Create a Tunnel

```bash
sudo cloudflared tunnel create <tunnel-name>
```

Example:

```bash
sudo cloudflared tunnel create nginx-tunnel
```

Once the tunnel is created, you will see a message with the tunnel ID and a certificate file path.

Example output:

```
vijay@controlplane:~$ cloudflared tunnel create nginx-tunnel
Tunnel credentials written to /home/vijay/.cloudflared/lmnop-0ce8-efgh-8c67-abcd.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel.

Created tunnel nginx-tunnel with id lmnop-0ce8-efgh-8c67-abcd
```

You can check the list of tunnels created by running:

```bash
sudo cloudflared tunnel list
sudo cloudflared tunnel info <tunnel-name>
```

Copy the tunnel ID and certificate file path for later use. I would recommend to rename the certificate file to something more meaningful, like `nginx-tunnel-credential.json`.

```bash
sudo cp /home/vijay/.cloudflared/lmnop-0ce8-efgh-8c67-abcd.json /home/vijay/.cloudflared/nginx-tunnel-credential.json
```

---

### Step 4: Create a Kubernetes Secret for Cloudflare Tunnel Credentials

I am creating a secret in the `nginx` namespace, but you can create it in any namespace you prefer. Just make sure to update the ConfigMap and Deployment accordingly.
Feel free change the path to the credential file if you have it in a different location.

```bash
kubectl -n nginx create secret generic cloudflared-creds \
 --from-file=nginx-tunnel-credential.json=/home/vijay/.cloudflared/nginx-tunnel-credential.json
```

The next step is to create a ConfigMap with the Cloudflare Tunnel configuration.

---

### Step 5: Create a Kubernetes ConfigMap for Cloudflare Tunnel

```yaml
# cloudflared-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: cloudflare
data:
  config.yaml: |
    tunnel: nginx-tunnel
    credentials-file: /etc/cloudflared/creds/nginx-tunnel-credential.json
    ingress:
      - hostname: '*.yourdomain.com'
        originRequest:
          noTLSVerify: true
        service: https://ingress-nginx-controller.ingress-nginx.svc.cluster.local:443
      - service: http_status:404
```

Explanation:

- `tunnel`: The name of the tunnel you created.
- `credentials-file`: Don't be confused with the path to the credential file, this is the path inside the container where the credential file will be mounted.
- `ingress`: This section defines the routing rules for the tunnel.
  - `hostname`: The domain or subdomain you want to expose. Replace `yourdomain.com` with your actual domain.
  - `originRequest.noTLSVerify`: Set to true to skip TLS verification for the service.
  - `service`: The service URL that the tunnel will route traffic to. In this case, it routes to the Nginx Ingress Controller service which is typically named `ingress-nginx-controller` in the `ingress-nginx` namespace. Adjust the service name and namespace as per your setup.
  - `http_status:404`: This is a catch-all rule that returns a 404 status for any unmatched requests.

---

### Step 6: Create Deployment for Cloudflare Tunnel

```yaml
# cloudflared-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:2025.6.1
          args: ["tunnel", "--config", "/etc/cloudflared/config.yaml", "run"]
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config.yaml
              subPath: config.yaml
            - name: creds
              mountPath: /etc/cloudflared/creds
      volumes:
        - name: config
          configMap:
            name: cloudflared-config
        - name: creds
          secret:
            secretName: cloudflared-creds
```

Explanation:
 - `args`: The command to run the Cloudflare Tunnel with the specified configuration file.
 - `volumeMounts`: Mounts the ConfigMap and secret containing the Cloudflare Tunnel configuration and credentials. This is the path above ConfigMap uses `/etc/cloudflared/creds/nginx-tunnel-credential.json`.

---

### Step 7: Apply the ConfigMap and Deployment

```bash
kubectl apply -f cloudflared-config.yaml
kubectl apply -f cloudflared-deployment.yaml
```

Make sure your Secret, ConfigMap, and Deployment are in the same namespace or adjust the commands accordingly. Otherwise, your Deployment won't be able to access the ConfigMap and Secret.

---

### Step 8: Expose Nginx Deployment using Ingress

```bash
kubectl create namespace nginx
kubectl -n nginx create deployment nginx --image=nginx:alpine --expose=nginx --port=443
```

---

### Step 9: Create an Ingress Resource

```bash
kubectl -n nginx create ingress nginx-ingress \
  --rule="nginx.yourdomain.com/*=nginx:80"
```

---

### Step 10: Verify the Ingress Resource

```bash
kubectl -n nginx get ingress nginx-ingress
```

You should see the Ingress resource with the hostname and backend service.

Whola! Your application is now exposed securely to the internet.

![expose-k8s-apps-to-internet.png](/images/expose-k8s-apps-to-internet.png)

---

# Conclusion

In this guide, we have successfully exposed a Kubernetes application securely to the internet using Cloudflare Tunnel and Nginx Ingress. This setup allows you to leverage Cloudflare's security features while managing your application traffic efficiently.

---
