---
date: 2025-07-02
draft: false
title: 'How To Self-Host JetBrains YouTrack Project Management Tool In Kubernetes'
slug: 'how-to-self-host-jetbrains-youtrack-project-management-tool-in-kubernetes'
tags: [self-hosting, kubernetes, jetbrains, youtrack, project-management]
---

# Introduction

In this guide, I will walk through the process of self-hosting JetBrains YouTrack, a powerful project management tool, in a Kubernetes environment. YouTrack is designed to help teams manage their projects efficiently with features like issue tracking, agile boards, and customizable workflows.
It is also free for up to 10 users, making it an excellent choice for small teams or personal projects.

![youtrack-homepage.png](/images/youtrack-homepage.png)

---

# Prerequisites

Before we begin, ensure you have the following prerequisites:

- A Kubernetes cluster up and running (not applicable for cloud providers like GKE, EKS, or AKS).
- Storage Class configured in your cluster. In this guide, I will use the `longhorn` storage class.
- To securely access the YouTrack instance, I will use Nginx Ingress Controller with Cloudflare Tunnel (Argo Tunnel) for secure access.
- A domain name that registers with Cloudflare.

---

# Deployment Steps

## Step 1: Create a Namespace

Create a dedicated namespace for YouTrack to keep resources organized:

```bash
kubectl create namespace youtrack
```

---

## Step 2: Create a Persistent Volume Claim

Create a Persistent Volume Claim (PVC) to store YouTrack data. This ensures that your data persists even if the pod is deleted or recreated.

For more information on how to create a PVC, refer to the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims).
To know more about the Longhorn storage class, refer to the [Longhorn documentation](https://longhorn.io/docs/1.9.0/nodes-and-volumes/volumes/create-volumes/).

We are going to create four PVCs for different purposes:

- `youtrack-data-pv-claim`: For YouTrack application data.
- `youtrack-conf-pv-claim`: For YouTrack configuration files.
- `youtrack-logs-pv-claim`: For YouTrack logs.
- `youtrack-backups-pv-claim`: For YouTrack backups.

```yaml
# youtrack-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: youtrack-data-pv-claim
  namespace: youtrack
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: youtrack-conf-pv-claim
  namespace: youtrack
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: youtrack-logs-pv-claim
  namespace: youtrack
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: youtrack-backups-pv-claim
  namespace: youtrack
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

Apply the PVCs:

```bash
kubectl apply -f youtrack-pvc.yaml
```

---

## Step 3: Create a Deployment and Mount the PVCs

Create a Deployment for YouTrack. This deployment will use the official YouTrack Docker image and mount the PVCs created in the previous step.

```yaml
# youtrack-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: youtrack-deployment
  namespace: youtrack
  labels:
    app: youtrack
spec:
  replicas: 1
  selector:
    matchLabels:
      app: youtrack
  template:
    metadata:
      labels:
        app: youtrack
    spec:
      securityContext:
        runAsUser: 0
        runAsGroup: 0
      containers:
        - name: youtrack
          image: jetbrains/youtrack:2025.1.82518
          volumeMounts:
            - name: opt-youtrack-data
              mountPath: /opt/youtrack/data
            - name: opt-youtrack-conf
              mountPath: /opt/youtrack/conf
            - name: opt-youtrack-logs
              mountPath: /opt/youtrack/logs
            - name: opt-youtrack-backups
              mountPath: /opt/youtrack/backups
      volumes:
        - name: opt-youtrack-data
          persistentVolumeClaim:
            claimName: youtrack-data-pv-claim
        - name: opt-youtrack-conf
          persistentVolumeClaim:
            claimName: youtrack-conf-pv-claim
        - name: opt-youtrack-logs
          persistentVolumeClaim:
            claimName: youtrack-logs-pv-claim
        - name: opt-youtrack-backups
          persistentVolumeClaim:
            claimName: youtrack-backups-pv-claim
```

Apply the Deployment:

```bash
kubectl apply -f youtrack-deployment.yaml
```

Make sure the PVCs are bound correctly by checking their status:

```bash
kubectl get pvc -n youtrack
```

You should see the status as `Bound` for all PVCs.

```bash
vijay@controlplane:~$ k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-35efef16-c771-4d39-88f4-6e5b15cc2ea2   1Gi        RWO            Delete           Bound    youtrack/youtrack-conf-pv-claim          longhorn       <unset>                          60s
pvc-64f43503-41f3-42cc-98d7-a5e0632b8753   5Gi        RWO            Delete           Bound    youtrack/youtrack-data-pv-claim          longhorn       <unset>                          60s
pvc-a1a5ae9d-402c-4913-a6d8-5f5282c6de73   2Gi        RWO            Delete           Bound    youtrack/youtrack-backups-pv-claim       longhorn       <unset>                          60s
pvc-ff4dd470-1c35-4f00-b6bc-11c1349b5b18   1Gi        RWO            Delete           Bound    youtrack/youtrack-logs-pv-claim          longhorn       <unset>                          60s
```

```bash
vijay@controlplane:~$ k get pvc -n youtrack
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
youtrack-backups-pv-claim   Bound    pvc-a1a5ae9d-402c-4913-a6d8-5f5282c6de73   2Gi        RWO            longhorn       <unset>                 2m
youtrack-conf-pv-claim      Bound    pvc-35efef16-c771-4d39-88f4-6e5b15cc2ea2   1Gi        RWO            longhorn       <unset>                 2m
youtrack-data-pv-claim      Bound    pvc-64f43503-41f3-42cc-98d7-a5e0632b8753   5Gi        RWO            longhorn       <unset>                 2m
youtrack-logs-pv-claim      Bound    pvc-ff4dd470-1c35-4f00-b6bc-11c1349b5b18   1Gi        RWO            longhorn       <unset>                 2m
```

---

## Step 4: Create a Service

Create a Service to expose the YouTrack application within the cluster. Target the port 8080, which is the default port for YouTrack.

```yaml
# youtrack-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: youtrack
  namespace: youtrack
spec:
  ports:
    - name: "http"
      port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app: youtrack
  type: ClusterIP
status:
  loadBalancer: {}
```

Apply the Service:

```bash
kubectl apply -f youtrack-service.yaml
```

---

## Step 5: Create an Ingress Resource

Create an Ingress resource to expose YouTrack externally. This will allow you to access YouTrack via a domain name.

```yaml
# youtrack-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: youtrack-ingress
  namespace: youtrack
spec:
  ingressClassName: nginx
  rules:
    - host: track.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: youtrack
                port:
                  number: 80
```

Apply the Ingress resource:

```bash
kubectl apply -f youtrack-ingress.yaml
```

---

## Step 6: Configure Cloudflare Tunnel

To securely access your YouTrack instance, you can use Cloudflare Tunnel (Argo Tunnel). This allows you to expose your service without opening ports on your firewall.
Also, you can skip Built-in TLS of YouTrack and use Cloudflare's TLS for secure access.

Instead of repeating the steps to set up Cloudflare Tunnel, I recommend you to follow my previous guide on [Expose Kubernetes Applications Securely to the Internet with Cloudflare Tunnel and Nginx Ingress](https://www.vijay-narayanan.com/posts/kubernetes/expose-k8s-apps-to-internet-using-cloudflare-tunnel/).
Once you have set up the Cloudflare Tunnel, you can access your YouTrack instance at `https://track.yourdomain.com`

---

## Step 7: Access YouTrack and Use the Wizard Token

Once everything is set up, open your web browser and navigate to `https://track.yourdomain.com`.
You should see the loading screen of YouTrack, and then you will be prompted to enter the wizard token.

### Step 7.1: Obtain the Wizard Token

Check the logs of the YouTrack pod to find the wizard token:

Example:
At the end of the logs, you will see a line similar to this:

```bash
vijay@controlplane:~$ k logs youtrack-deployment-69fd966796-xb67v -f
Starting YouTrack...
* Configuring JetBrains YouTrack 2025.1
* Made default base-url 'http://localhost:8080/' from hostname 'localhost' and listen port '8080'
* JetBrains YouTrack 2025.1 runtime environment is successfully configured
* Loading logging configuration from /opt/youtrack/lib/ext/log4j.xml
* Redirecting JetBrains YouTrack 2025.1 logging to /opt/youtrack/logs/internal/services/bundleProcess
* Configuring Service-Container[bundleProcess]
* Configuring Bundle Backend Service
* Configuring Configuration Wizard
* Starting Service-Container[bundleProcess]
* Starting Bundle Backend Service
* Starting Configuration Wizard
* JetBrains YouTrack 2025.1 Configuration Wizard will listen inside container on {0.0.0.0:8080}/ 
after start and can be accessed by URL 
[http://<put-your-docker-HOST-name-here>:<put-host-port-mapped-to-container-port-8080-here>//?wizard_token=ABCDEFGHIJKLMNOPQRSTUVWX]
```

### Step 7.2: Use the Wizard Token

Copy the wizard token from the logs and paste it into the YouTrack setup wizard page. This token is used to configure the initial settings of YouTrack.

---

## Step 8: Complete the Setup Wizard

Once you have entered the wizard token, you click on `Setup` button to proceed with the setup wizard.

Since you already setup the Cloudflare Tunnel, you can skip the Https configuration in the setup wizard and proceed with the HTTP configuration.

![youtrack-setup-wizard.png](/images/youtrack-setup-wizard.png)

As you see in the image above, you can skip the HTTPS configuration and proceed with HTTP configuration.

Make sure to change the `Base URL` to your domain name, e.g., `https://track.yourdomain.com`.

Keep the Application Port as `8080` and Application Listen Address as `0.0.0.0` in advanced settings.

In my case, I got an error while setting up the YouTrack instance because I had to restart my deployment after creating the PVCs. So youTrack complained that the Data directory is not empty. If you encounter this error, you can do the following:

Exec into the YouTrack pod and then, remove the contents of the `/opt/youtrack/data` directory:

```bash
kubectl exec -it -n youtrack <your-pod-name> -- bash
rm -rf /opt/youtrack/data/*
```

After clearing the data directory, you can restart the YouTrack deployment:

```bash
kubectl rollout restart deployment youtrack-deployment -n youtrack
```

Repeat the setup wizard steps again, and this time it should work without any issues.

---

## Step 9: Use Default License

In the next screen, you will be prompted to enter a license key. Since you are self-hosting YouTrack for personal use or for a small team, you can use the default license key provided by JetBrains.

Next to that, add your administrator account by entering your username and password. This account will have full administrative privileges in YouTrack.

---

## Step 10: Access YouTrack

Once the setup is complete, you can access your YouTrack instance at `https://track.yourdomain.com`.

![youtrack-login-page.png](/images/youtrack-login-page.png)

---

# Conclusion

Congratulations! You have successfully self-hosted JetBrains YouTrack in a Kubernetes environment.

---

# Reference Resources

- [JetBrains YouTrack Official Site](https://www.jetbrains.com/youtrack/)
- [YouTrack Hangs on Start - Support Article](https://youtrack-support.jetbrains.com/hc/en-us/articles/207242645-YouTrack-hangs-on-start)
- [Deploy YouTrack in Kubernetes - Documentation](https://www.jetbrains.com/help/youtrack/server/deploy-youtrack-kubernetes.html)
- [YouTrack Pricing and Buy Page](https://www.jetbrains.com/youtrack/buy/)

---
