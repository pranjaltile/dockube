# Project 2 — Deploy Flask App on Kubernetes with APISIX Gateway

> **Goal:** Take the containerized Flask app from Project 1 → Deploy it on Kubernetes with 3 replicas → Expose it through APISIX API Gateway → Hit it via `curl` from outside the cluster.
>
> **What you learn:** Kubernetes basics (pod, deployment, service), APISIX routing, Helm, Docker contexts, and how real end-to-end traffic flow works.

---

## 🗂️ Project Structure

```
k8s-project/
├── deployment.yaml       ← tells K8s to run 3 copies of your app
└── service.yaml          ← exposes your app internally in the cluster
```

---

## 🧰 Prerequisites

| Tool | Purpose | Install |
|---|---|---|
| Docker Desktop + WSL | Container engine | From Project 1 |
| minikube | Local Kubernetes cluster | See Step 1 |
| kubectl | K8s command line tool | Pre-installed |
| helm | K8s package manager | See Step 1 |
| Flask app image | From Project 1 | `localhost/firstproject/first_app:v1` |

---

## 📌 Concepts Learned

### What is Kubernetes?

Docker runs containers on ONE machine. Kubernetes manages containers across MANY machines automatically.

```
Your laptop
└── Docker = runs 1 container manually

10 Servers (cluster)
└── Kubernetes = runs 1000 containers automatically
    ├── Auto-restarts crashed containers
    ├── Scales up/down based on traffic
    └── Balances load across all pods
```

### Key Kubernetes Objects

| Object | What it is | Analogy |
|---|---|---|
| **Pod** | Smallest unit — runs one container | One worker |
| **Deployment** | Manages multiple pods, handles restarts | HR manager |
| **Service** | Stable network endpoint for pods | Reception desk |
| **Namespace** | Isolated group of resources | Department |

### ClusterIP vs NodePort

| Type | Accessible from | Used for |
|---|---|---|
| `ClusterIP` | Inside cluster only | Internal service-to-service |
| `NodePort` | Outside cluster via port | Exposing to external traffic |

### What is APISIX?

APISIX is an API Gateway — it sits in front of all your services and controls incoming traffic.

```
Internet
    ↓
APISIX (API Gateway)
├── Authentication
├── Rate limiting
├── SSL termination
├── Load balancing
└── Routing → sends to correct K8s service
```

### What is Helm?

Helm is a package manager for Kubernetes — like `apt` for Ubuntu or `npm` for Node.js. Instead of applying 20 YAML files manually, Helm installs everything with one command.

```
Without Helm: kubectl apply -f file1.yaml, file2.yaml, file3.yaml...
With Helm:    helm install apisix apisix/apisix
```

### Docker Contexts — Important!

Docker can connect to different engines. Minikube has its own internal Docker, separate from your local Docker Desktop.

```
default context     → your local Docker (Harbor images live here)
minikube context    → Minikube's internal Docker (K8s uses this)
```

This is why we use `eval $(minikube docker-env)` — it switches Docker to talk to Minikube's engine so we can build images directly inside it.

### imagePullPolicy: Never

Tells Kubernetes "don't try to pull this image from a registry — use what's already loaded locally." Essential for local development where your registry isn't accessible from inside Minikube.

---

## 🔵 Step 1 — Install Minikube and Helm

Open Ubuntu (WSL) terminal:

**Install Minikube:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

**Install Helm:**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Verify both:**
```bash
minikube version
helm version
kubectl version --client
```

---

## 🔵 Step 2 — Start Minikube

```bash
minikube start --driver=docker
```

Wait 3-5 minutes for cluster to start.

**Verify cluster is running:**
```bash
kubectl get nodes
```

Expected:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.x.x
```

**Verify kubectl can talk to cluster:**
```bash
kubectl cluster-info
```

Expected:
```
Kubernetes control plane is running at https://127.0.0.1:xxxxx
```

---

## 🔵 Step 3 — Build Image Inside Minikube

Since Minikube has its own internal Docker, we need to build the image there directly.

**Switch Docker context to Minikube's engine:**
```bash
eval $(minikube docker-env)
```

**Navigate to your Flask app:**
```bash
cd /mnt/c/Users/<YourUsername>/Docker_projects/first_app
```

**Build image inside Minikube:**
```bash
docker build -t first_app:v1 .
```

**Verify image exists inside Minikube:**
```bash
docker images | grep first_app
```

Expected:
```
first_app   v1   abc123   1 min ago   xxx MB
```

> ⚠️ **Important:** Every time you restart Minikube, you need to rebuild the image inside it. The image does not persist across Minikube restarts.

---

## 🔵 Step 4 — Create Kubernetes YAML Files

```bash
mkdir ~/k8s-project && cd ~/k8s-project
```

**Create deployment.yaml:**
```bash
nano deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: first_app:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 5000
```

**What each field means:**

| Field | Meaning |
|---|---|
| `replicas: 3` | Run 3 copies of the app |
| `selector.matchLabels` | Find pods with label `app: flask-app` |
| `image: first_app:v1` | Use this Docker image |
| `imagePullPolicy: Never` | Use local image, don't pull from registry |
| `containerPort: 5000` | Flask app listens on port 5000 |

---

**Create service.yaml:**
```bash
nano service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  selector:
    app: flask-app
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP
```

**What each field means:**

| Field | Meaning |
|---|---|
| `selector: flask-app` | Route traffic to pods with this label |
| `port: 80` | Service listens on port 80 |
| `targetPort: 5000` | Forwards to Flask app on port 5000 |
| `type: ClusterIP` | Internal only — APISIX will expose it outside |

---

## 🔵 Step 5 — Deploy to Kubernetes

**Apply both files:**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

**Verify pods are running:**
```bash
kubectl get pods
```

Expected:
```
NAME                         READY   STATUS    RESTARTS   AGE
flask-app-745b45857c-6z4bv   1/1     Running   0          5s
flask-app-745b45857c-8nrjv   1/1     Running   0          10s
flask-app-745b45857c-jnd2b   1/1     Running   0          8s
```

**Verify service is created:**
```bash
kubectl get svc
```

Expected:
```
NAME                TYPE        CLUSTER-IP       PORT(S)   AGE
flask-app-service   ClusterIP   10.103.241.228   80/TCP    30s
```

> ⚠️ If pods show `ImagePullBackOff` — image not found in Minikube. Run `eval $(minikube docker-env)` and rebuild the image.
> ⚠️ If pods show `ContainerCreating` — wait 1-2 minutes, K8s is starting up.

---

## 🔵 Step 6 — Install APISIX using Helm

**Add APISIX Helm repository:**
```bash
helm repo add apisix https://charts.apiseven.com
helm repo update
```

**Install APISIX on the cluster:**
```bash
helm install apisix apisix/apisix \
  --set gateway.type=NodePort \
  --set ingress-controller.enabled=true \
  --namespace ingress-apisix \
  --create-namespace
```

Wait 2-3 minutes for all pods to start.

**Verify APISIX pods:**
```bash
kubectl get pods -n ingress-apisix
```

Expected:
```
NAME                                        READY   STATUS
apisix-xxxx                                 1/1     Running
apisix-etcd-0                               1/1     Running
apisix-etcd-1                               1/1     Running
apisix-etcd-2                               1/1     Running
apisix-ingress-controller-xxxx              2/2     Running
```

**Verify APISIX services:**
```bash
kubectl get svc -n ingress-apisix
```

Expected:
```
NAME                TYPE        PORT(S)
apisix-gateway      NodePort    80:31994/TCP
apisix-admin        ClusterIP   9180/TCP
```

---

## 🔵 Step 7 — Create APISIX Route

A route tells APISIX where to forward incoming requests.

**Terminal 1 — Port-forward APISIX admin API:**
```bash
kubectl port-forward svc/apisix-admin -n ingress-apisix 9180:9180
```

Keep this running.

**Terminal 2 — Create the route:**
```bash
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" \
  -X PUT \
  -d '{
    "uri": "/*",
    "name": "flask-route",
    "upstream": {
      "type": "roundrobin",
      "nodes": {
        "flask-app-service.default.svc.cluster.local:80": 1
      }
    }
  }'
```

Expected response:
```json
{"key":"/apisix/routes/1","value":{"id":"1","uri":"/*","name":"flask-route"...}}
```

**What the route means:**

| Field | Meaning |
|---|---|
| `uri: "/*"` | Match all incoming requests |
| `type: roundrobin` | Spread traffic equally across all pods |
| `flask-app-service.default.svc.cluster.local:80` | K8s DNS name for your service |

> 💡 **K8s DNS format:** `<service-name>.<namespace>.svc.cluster.local`
> Your service `flask-app-service` is in `default` namespace → `flask-app-service.default.svc.cluster.local`

---

## 🔵 Step 8 — Test End-to-End

**Terminal 2 — Port-forward APISIX gateway:**
```bash
kubectl port-forward svc/apisix-gateway -n ingress-apisix 9080:80
```

Keep this running.

**Terminal 3 — Hit your app:**
```bash
curl http://localhost:9080/
```

Expected:
```
Hello from my first DevOps app! 🚀
```

---

## ✅ Success Checklist

```
□ minikube start shows cluster Ready
□ kubectl get nodes shows minikube Ready
□ Image built inside Minikube with eval $(minikube docker-env)
□ kubectl get pods shows 3 pods Running
□ kubectl get svc shows flask-app-service
□ APISIX pods all Running in ingress-apisix namespace
□ APISIX route created successfully
□ curl http://localhost:9080/ returns Hello from my first DevOps app!
```

---

## 🐛 Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | K8s can't find image | Run `eval $(minikube docker-env)` and rebuild image inside Minikube |
| `502 Bad Gateway` | APISIX can't reach Flask service | Check pods are Running with `kubectl get pods` |
| `Could not connect to server` | Port-forward not running | Run `kubectl port-forward svc/apisix-gateway -n ingress-apisix 9080:80` |
| `ContainerCreating` for long time | Image still loading | Wait 2 min, if stuck check `kubectl describe pod <pod-name>` |
| Pods running but curl fails | Route not created | Re-run the APISIX route curl command |
| Port-forward stops working | Terminal closed | Re-run port-forward command in a new terminal |

---

## 🗺️ Full Architecture — What You Built

```
Your Machine (Windows + WSL)
│
├── curl http://localhost:9080/
│           ↓
│   ┌───────────────────────────────────────┐
│   │         Minikube K8s Cluster          │
│   │                                       │
│   │   APISIX Gateway (port-forward 9080)  │
│   │           ↓ route: "/*"               │
│   │   flask-app-service (ClusterIP :80)   │
│   │           ↓ roundrobin                │
│   │   ┌──────┬──────┬──────┐             │
│   │   │ Pod1 │ Pod2 │ Pod3 │             │
│   │   │Flask │Flask │Flask │             │
│   │   └──────┴──────┴──────┘             │
│   └───────────────────────────────────────┘
│
└── Harbor Registry (separate — stores images)
```

---

## 📚 Key Commands Reference

```bash
# Minikube
minikube start --driver=docker     # start cluster
minikube stop                      # stop cluster
minikube status                    # check status
minikube delete                    # delete cluster
eval $(minikube docker-env)        # switch to Minikube's Docker

# Kubectl - Pods
kubectl get pods                   # list pods
kubectl get pods -n ingress-apisix # list pods in namespace
kubectl describe pod <name>        # detailed pod info
kubectl logs <pod-name>            # view pod logs

# Kubectl - Services
kubectl get svc                    # list services
kubectl get svc -n ingress-apisix  # list services in namespace

# Kubectl - Deployments
kubectl apply -f deployment.yaml   # apply/update deployment
kubectl delete -f deployment.yaml  # delete deployment
kubectl rollout restart deployment/flask-app  # restart all pods

# Port Forwarding
kubectl port-forward svc/apisix-admin -n ingress-apisix 9180:9180
kubectl port-forward svc/apisix-gateway -n ingress-apisix 9080:80
kubectl port-forward svc/flask-app-service 5000:80

# Helm
helm repo add apisix https://charts.apiseven.com
helm repo update
helm install apisix apisix/apisix --namespace ingress-apisix --create-namespace
helm list -n ingress-apisix        # list installed helm releases
helm uninstall apisix -n ingress-apisix  # uninstall APISIX
```

---

## 🔁 Restart Checklist (after PC reboot)

Every time you restart your machine, run these in order:

```bash
# 1. Start Harbor
cd ~/harbor-install/harbor && sudo docker compose up -d

# 2. Start Minikube
minikube start --driver=docker

# 3. Switch to Minikube Docker and rebuild image
eval $(minikube docker-env)
cd /mnt/c/Users/<YourUsername>/Docker_projects/first_app
docker build -t first_app:v1 .

# 4. Reapply K8s manifests
cd ~/k8s-project
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 5. Start port-forwards (each in separate terminal)
kubectl port-forward svc/apisix-admin -n ingress-apisix 9180:9180
kubectl port-forward svc/apisix-gateway -n ingress-apisix 9080:80

# 6. Test
curl http://localhost:9080/
```

---


