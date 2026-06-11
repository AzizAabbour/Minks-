# Kubernetes Fundamentals Lab – Minikube on Ubuntu 22.04

## 📋 Table of Contents
1. Prerequisites
2. Verify Installations
3. Start Docker Service (if needed)
4. Part 1 – Start Minikube
5. Part 2 – Verify Cluster Status
6. Part 3 – Create an Nginx Deployment
7. Part 4 – Check Pods Status
8. Part 5 – Expose the Deployment as a Service (NodePort)
9. Part 6 – Test Self‑Healing
10. Part 7 – Horizontal Scaling
11. Part 8 – Cleanup (Optional)
12. Part 9 – Troubleshooting (Bonus)
13. Quick‑Reference Command Summary

---

## 🛠️ Prerequisites
| Tool | Minimum Version | Install Command |
|------|----------------|-----------------|
| Ubuntu | 22.04 LTS | – |
| Docker | 24.0+ | `sudo apt-get update && sudo apt-get install -y docker.io` |
| kubectl | 1.28+ (client) | `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && chmod +x kubectl && sudo mv kubectl /usr/local/bin/` |
| Minikube | 1.32+ | `curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube-linux-amd64 && sudo mv minikube-linux-amd64 /usr/local/bin/minikube` |

> **⚠️ Common errors**: Docker not starting → see Troubleshooting section.

---

## ✅ Verify Installations
```bash
user@ubuntu:~$ minikube version
```
```
minikube version: v1.32.0
commit: 1234abcd (2024‑03‑01)
```
```bash
user@ubuntu:~$ kubectl version --client
```
```
Client Version: version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", ...}
```
```bash
user@ubuntu:~$ docker --version
```
```
Docker version 24.0.5, build 5f2c1e6
```
📘 These commands confirm that the required tools are installed and meet version requirements.

---

## 🧰 Start Docker Service (if not running)
```bash
user@ubuntu:~$ sudo systemctl status docker
```
```
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled)
   Active: active (running) since Mon 2026‑06‑11 20:45:12 CEST; 1h 24min ago
```
If inactive:
```bash
user@ubuntu:~$ sudo systemctl start docker
```
```bash
user@ubuntu:~$ sudo systemctl enable docker
```
📘 Minikube’s Docker driver requires the Docker daemon to be running.

---

## 🚀 Part 1 – Start Minikube
```bash
user@ubuntu:~$ minikube start --driver=docker --cpus=2 --memory=4000
```
```
😄  minikube v1.32.0 on Ubuntu 22.04 (docker driver)
✨  Using the docker driver based on existing profile.
👍  Starting control plane node minikube in cluster minikube
🔄  Pulling image quay.io/kubernetes-sigs/kind-node:v1.28.2
💾  Downloading the Docker image for the node ...
🛠  Creating Docker container (CPUs=2, Memory=4000MB) ...
🔧  Configuring Kubernetes components ...
🔎  Verifying Kubernetes components...
🏃  Starting control plane...
🔸  Enabled addons: storage-provisioner, default-storageclass
💡  kubectl is now configured for minikube.
```
📘 **Explanation**: The Docker driver runs a lightweight `kind-node` container that hosts a single‑node Kubernetes cluster—no VM is required. The driver pulls the base image, creates a container with the requested resources, starts control‑plane components, and enables default addons.

---

## 🔍 Part 2 – Verify Cluster Status
```bash
user@ubuntu:~$ kubectl get nodes
```
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.28.2
```
```bash
user@ubuntu:~$ kubectl cluster-info
```
```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
```bash
user@ubuntu:~$ kubectl get namespaces
```
```
NAME              STATUS   AGE
default           Active   2m
kube-system       Active   2m
kube-public       Active   2m
kube-node-lease   Active   2m
```
📘 Minikube provides a **single‑node** cluster where the node acts as both control‑plane and worker.

---

## 🧩 Part 3 – Create an Nginx Deployment
### Quick one‑liner (default replica = 1)
```bash
user@ubuntu:~$ kubectl create deployment nginx --image=nginx:latest
```
```
deployment.apps/nginx created
```
*To create 3 replicas in one command:*
```bash
user@ubuntu:~$ kubectl create deployment nginx --image=nginx:latest --replicas=3
```
```
deployment.apps/nginx created
```
### Declarative YAML (recommended)
Create **nginx-deployment.yaml** with the following content:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
Apply it:
```bash
user@ubuntu:~$ kubectl apply -f nginx-deployment.yaml
```
```
deployment.apps/nginx created
```
Inspect:
```bash
user@ubuntu:~$ kubectl get deployments
```
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           1m
```
```bash
user@ubuntu:~$ kubectl get pods -o wide
```
```
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-6b9c9c6c9f-1k2hm   1/1     Running   0          1m    172.17.0.5   minikube   <none>           <none>
nginx-6b9c9c6c9f-jx9dp   1/1     Running   0          1m    172.17.0.6   minikube   <none>           <none>
nginx-6b9c9c6c9f-w2vzl   1/1     Running   0          1m    172.17.0.7   minikube   <none>           <none>
```
📘 **Key concepts**: Deployment (desired state), ReplicaSet (auto‑generated), Pod (runtime unit), Container (Nginx process).

---

## 🧪 Part 4 – Check Pods Status
```bash
user@ubuntu:~$ kubectl get pods --watch
```
*(Press `Ctrl+C` to stop.)*
```bash
user@ubuntu:~$ kubectl logs nginx-6b9c9c6c9f-1k2hm
```
```
172.17.0.5 - - [11/Jun/2026:21:00:12 +0100] "GET / HTTP/1.1" 200 612 "-" "curl/7.81.0"
```
```bash
user@ubuntu:~$ kubectl exec -it nginx-6b9c9c6c9f-1k2hm -- /bin/bash
```
```
root@nginx-6b9c9c6c9f-1k2hm:/# curl -s http://localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
root@nginx-6b9c9c6c9f-1k2hm:/# exit
```
📘 Pods go through **Pending → ContainerCreating → Running**.

---

## 🌐 Part 5 – Expose the Deployment as a Service (NodePort)
```bash
user@ubuntu:~$ kubectl expose deployment nginx \
    --type=NodePort \
    --port=80 \
    --target-port=80 \
    --name=nginx-service
```
```
service/nginx-service exposed
```
```bash
user@ubuntu:~$ kubectl get services
```
```
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP  10.96.0.1        <none>        443/TCP        5m
nginx-service   NodePort   10.108.221.250   <none>        80:31123/TCP   30s
```
Obtain the NodePort value:
```bash
user@ubuntu:~$ NODEPORT=$(kubectl get svc nginx-service -o jsonpath='{.spec.ports[0].nodePort}')
user@ubuntu:~$ echo $NODEPORT
31123
```
Get Minikube IP:
```bash
user@ubuntu:~$ MINIP=$(minikube ip)
user@ubuntu:~$ echo $MINIP
192.168.64.2
```
Test with curl:
```bash
user@ubuntu:~$ curl http://$MINIP:$NODEPORT
```
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```
Shortcut URLs:
```bash
user@ubuntu:~$ minikube service nginx-service --url
```
```
http://192.168.64.2:31123
```
```bash
user@ubuntu:~$ minikube service nginx-service
```
*(Browser opens the Nginx welcome page.)*
📘 **Service types**: ClusterIP (internal), NodePort (external on high port range 30000‑32767), LoadBalancer (cloud). The NodePort method is simplest for local labs.

---

## ♻️ Part 6 – Test Self‑Healing
```bash
user@ubuntu:~$ kubectl get pods
```
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6b9c9c6c9f-1k2hm   1/1     Running   0          6m
nginx-6b9c9c6c9f-jx9dp   1/1     Running   0          6m
nginx-6b9c9c6c9f-w2vzl   1/1     Running   0          6m
```
Delete one pod:
```bash
user@ubuntu:~$ kubectl delete pod nginx-6b9c9c6c9f-jx9dp
```
```
pod "nginx-6b9c9c6c9f-jx9dp" deleted
```
Watch replacement:
```bash
user@ubuntu:~$ kubectl get pods --watch
```
```
nginx-6b9c9c6c9f-1k2hm   1/1   Running   0   6m
nginx-6b9c9c6c9f-w2vzl   1/1   Running   0   6m
nginx-6b9c9c6c9f-abc12   0/1   Pending   0   0s
nginx-6b9c9c6c9f-abc12   1/1   Running   0   5s
```
The Deployment’s desired state (3 replicas) triggered the ReplicaSet to create a replacement pod automatically.
```bash
user@ubuntu:~$ kubectl describe replicaset $(kubectl get rs -o name | head -n1)
```
Shows a `SuccessfulCreate` event for the new pod.
✅ After deletion, the replica count returns to **3/3**.

---

## 📈 Part 7 – Horizontal Scaling
Scale up:
```bash
user@ubuntu:~$ kubectl scale deployment nginx --replicas=5
```
```
deployment.apps/nginx scaled
```
Watch new pods:
```bash
user@ubuntu:~$ kubectl get pods --watch
```
Two new pods appear and reach **Running**.
```bash
user@ubuntu:~$ kubectl get deployments
```
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   5/5     5            5           10m
```
Scale down to 2:
```bash
user@ubuntu:~$ kubectl scale deployment nginx --replicas=2
```
```
deployment.apps/nginx scaled
```
```bash
user@ubuntu:~$ kubectl get pods --watch
```
Three pods terminate, leaving two.
✅ The `READY` column always matches the replica count you set.
> *Alternative*: `kubectl edit deployment nginx` → change `spec.replicas` interactively.

---

## 🗑️ Part 8 – Cleanup (Optional)
```bash
user@ubuntu:~$ kubectl delete service nginx-service
```
```
service "nginx-service" deleted
```
```bash
user@ubuntu:~$ kubectl delete deployment nginx
```
```
deployment.apps/nginx deleted
```
```bash
user@ubuntu:~$ minikube stop
```
```
💤  Stopping "minikube" ...
```
```bash
user@ubuntu:~$ minikube delete --all
```
```
🔥  Removing all traces of "minikube" ...
```
Cleaning up frees CPU, memory, and disk space.

---

## 🛠️ Part 9 – Troubleshooting (Bonus)
| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Pod stays `Pending` | Insufficient resources or Docker driver mis‑config | `kubectl describe pod <pod>` → examine events. Reduce replicas or allocate more memory. |
| NodePort not reachable | Host firewall blocking 30000‑32767 | `sudo ufw allow 30000:32767/tcp` or disable firewall. |
| `ErrImagePull` | No internet or Docker Hub rate‑limit | Verify network, add `--image-pull-policy=IfNotPresent`, or manually `docker pull nginx:latest`. |
| Minikube fails to start | Corrupt cache | `minikube delete --purge && rm -rf ~/.minikube` then restart. |
| `kubectl` command not found after start | `$PATH` missing `/usr/local/bin` | `export PATH=$PATH:/usr/local/bin` (add to `~/.bashrc`). |
| Browser cannot open service URL (remote VM) | No port forwarding | SSH tunnel: `ssh -L 31123:localhost:31123 user@remote-host` then browse `http://localhost:31123`. |

---

## 🔖 Quick‑Reference Command Summary
| # | Command | Purpose |
|---|---------|---------|
| 1 | `minikube version` | Verify Minikube installed |
| 2 | `kubectl version --client` | Verify kubectl client |
| 3 | `docker --version` | Verify Docker |
| 4 | `sudo systemctl status docker` | Check Docker daemon |
| 5 | `minikube start --driver=docker --cpus=2 --memory=4000` | Start cluster |
| 6 | `kubectl get nodes` | Show node status |
| 7 | `kubectl cluster-info` | Show control‑plane endpoints |
| 8 | `kubectl get namespaces` | List namespaces |
| 9 | `kubectl create deployment nginx --image=nginx:latest --replicas=3` | One‑liner Deployment |
| 10 | *(or)* `kubectl apply -f nginx-deployment.yaml` | Declarative YAML |
| 11 | `kubectl get deployments` | Verify deployment status |
| 12 | `kubectl get pods -o wide` | List pods with IPs |
| 13 | `kubectl logs <pod>` | View container logs |
| 14 | `kubectl exec -it <pod> -- /bin/bash` | Enter container shell |
| 15 | `kubectl expose deployment nginx --type=NodePort --port=80 --target-port=80 --name=nginx-service` | Create Service |
| 16 | `kubectl get svc` | Show Service & NodePort |
| 17 | `minikube ip` | Get node IP |
| 18 | `curl http://$(minikube ip):<nodeport>` | Test external access |
| 19 | `minikube service nginx-service --url` | Shortcut URL |
| 20 | `kubectl delete pod <pod>` | Test self‑healing |
| 21 | `kubectl scale deployment nginx --replicas=5` | Scale up |
| 22 | `kubectl scale deployment nginx --replicas=2` | Scale down |
| 23 | `kubectl delete service nginx-service` | Remove Service |
| 24 | `kubectl delete deployment nginx` | Remove Deployment |
| 25 | `minikube stop` | Stop cluster |
| 26 | `minikube delete --all` | Delete all Minikube data |

---

**Congratulations!** You now have a complete, copy‑paste ready lab that walks a beginner through installing Minikube, deploying an app, exposing it, and exploring core Kubernetes concepts. Feel free to experiment further—try other images, add ConfigMaps, or explore Helm charts!
#   M i n k s -  
 