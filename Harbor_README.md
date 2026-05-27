# Project 1 — Containerize a Flask App & Push to Harbor Registry

> **Goal:** Build a simple Python Flask app → Dockerize it → Run a private Harbor registry → Push your image to Harbor → Pull it back and run it.
>
> **What you learn:** Docker end-to-end, Harbor as a private container registry, image tagging & versioning, WSL + Docker Desktop integration on Windows.

---

## 🗂️ Project Structure

```
my-app/
├── app.py               ← Flask web app
├── requirements.txt     ← Python dependencies
└── Dockerfile           ← Instructions to containerize the app
```

---

## 🧰 Prerequisites

| Tool | Purpose | Install |
|---|---|---|
| Docker Desktop | Runs containers on Windows | [docker.com](https://www.docker.com/products/docker-desktop/) |
| WSL 2 (Ubuntu) | Runs Harbor setup scripts (Linux only) | `wsl --install -d Ubuntu` in PowerShell as Admin |
| Windows Browser | Access Harbor UI | Already installed |

> **Important:** Enable WSL Integration in Docker Desktop → ⚙️ Settings → Resources → WSL Integration → turn on Ubuntu ✅

---

## 📌 Concepts Learned

### Docker vs Harbor vs Kubernetes

| Tool | Role | Analogy |
|---|---|---|
| **Docker** | Builds and runs containers | Camera that takes photos |
| **Harbor** | Stores and manages container images | Google Photos album |
| **Kubernetes** | Orchestrates containers at scale | Cargo ship manager |

### docker run vs docker compose up

| Command | Use case |
|---|---|
| `docker run` | Start a single container |
| `docker compose up` | Start multiple containers together from a `docker-compose.yml` file |

### Why HTTP not HTTPS (local setup)

HTTPS requires an SSL certificate issued for a real domain name. Since we run Harbor on `localhost` (not a real domain), we use HTTP on port 80 instead. In production, Harbor runs on a real domain with a valid SSL certificate.

### Why WSL for Harbor?

Harbor's installer uses Linux shell scripts (`.sh` files) that cannot run on Windows Command Prompt. WSL provides a Linux environment inside Windows to run these scripts. After setup, Harbor runs inside Docker Desktop which is shared between Windows and WSL.

---

## 🔵 Step 1 — Install Docker Desktop

1. Download from [docker.com](https://www.docker.com/products/docker-desktop/)
2. Run installer — keep **WSL 2** option checked ✅
3. Restart PC after install
4. Open Docker Desktop and wait for engine to start

**Verify:**
```bash
docker --version
docker run hello-world
```

Expected output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## 🔵 Step 2 — Create the Flask App

Create a folder `my-app` and add these files:

**`app.py`**
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from my first DevOps app! 🚀"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`**
```
flask
```

---

## 🔵 Step 3 — Write the Dockerfile

Create a file named `Dockerfile` (no extension) in the same folder:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

**What each line does:**

| Line | Meaning |
|---|---|
| `FROM python:3.11-slim` | Start from an official lightweight Python base image |
| `WORKDIR /app` | Set working directory inside the container |
| `COPY requirements.txt .` | Copy dependencies file into container |
| `RUN pip install` | Install Flask inside the container |
| `COPY app.py .` | Copy your app code into container |
| `EXPOSE 5000` | Tell Docker the app runs on port 5000 |
| `CMD` | Command to run when container starts |

---

## 🔵 Step 4 — Build and Run Locally

Navigate to your project folder and build:

```bash
docker build -t myapp:v1 .
```

Run it:
```bash
docker run -p 5000:5000 myapp:v1
```

Open browser → `http://localhost:5000`

✅ You should see: `Hello from my first DevOps app! 🚀`

Press `Ctrl+C` to stop.

---

## 🔵 Step 5 — Install Harbor (via WSL Ubuntu)

Harbor is installed in a separate location — it is a global tool, not part of your project.

**Open Ubuntu (WSL) terminal and run:**

```bash
cd ~
mkdir harbor-install && cd harbor-install

# Download offline installer (~600MB)
curl -L https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz -o harbor-offline.tgz

# Extract
tar xzvf harbor-offline.tgz
cd harbor
```

**Copy and edit config:**
```bash
cp harbor.yml.tmpl harbor.yml
nano harbor.yml
```

Make these two changes in `harbor.yml`:
```yaml
# Change hostname from:
hostname: reg.mydomain.com
# To:
hostname: localhost

# Comment out the entire https section:
# https:
#   port: 443
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path
```

Save with `Ctrl+X` → `Y` → `Enter`

**Generate docker-compose.yml:**
```bash
sudo bash prepare
```

**Start Harbor:**
```bash
sudo docker compose up -d
```

**Verify all containers are healthy:**
```bash
sudo docker compose ps
```

All 9 services should show `(healthy)` status:
```
harbor-core        healthy
harbor-db          healthy
harbor-jobservice  healthy
harbor-log         healthy
harbor-portal      healthy
nginx              healthy   ← port 80
redis              healthy
registry           healthy
registryctl        healthy
```

> **Note:** Harbor auto-starts every time by adding this to `~/.bashrc`:
> ```bash
> cd ~/harbor-install/harbor && sudo docker compose up -d 2>/dev/null
> ```

---

## 🔵 Step 6 — Configure Docker to Trust Harbor (HTTP)

Since Harbor runs on HTTP (not HTTPS), Docker needs to be told to trust it.

Open **Docker Desktop** → ⚙️ Settings → **Docker Engine**

Add `insecure-registries` to the JSON:
```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "insecure-registries": ["localhost:80"]
}
```

Click **Apply & Restart** and wait for Docker Desktop to restart.

**Verify it applied:**
```bash
docker info | grep -i insecure
```

Expected:
```
Insecure Registries:
  localhost:80
```

---

## 🔵 Step 7 — Create a Project in Harbor UI

1. Open browser → `http://localhost`
2. Login:
   - Username: `admin`
   - Password: `Harbor12345`
3. Click **"New Project"**
4. Name: `myproject` → Access Level: Public → Click **OK**

---

## 🔵 Step 8 — Tag and Push Image to Harbor

**Login to Harbor:**
```bash
docker login localhost:80 -u admin -p Harbor12345
```

Expected:
```
Login Succeeded
```

**Tag your image with Harbor address:**
```bash
docker tag myapp:v1 localhost/myproject/myapp:v1
```

**Push to Harbor:**
```bash
docker push localhost/myproject/myapp:v1
```

Go to Harbor UI → **myproject** → **Repositories** → you should see `myproject/myapp` with tag `v1` ✅

---

## 🔵 Step 9 — Pull Back and Prove it Works

**Delete local images:**
```bash
# First stop and remove any running containers using the image
docker ps -a                          # find container ID
docker stop <container_id>
docker rm <container_id>

# Then delete the images
docker rmi myapp:v1
docker rmi localhost/myproject/myapp:v1
```

**Verify images are gone:**
```bash
docker images
```

**Pull fresh from Harbor:**
```bash
docker pull localhost/myproject/myapp:v1
```

**Run from Harbor-pulled image:**
```bash
docker run -p 5000:5000 localhost/myproject/myapp:v1
```

Open browser → `http://localhost:5000`

✅ **Same result — but this time the image came from Harbor, not your local build!**

---

## ✅ Success Checklist

```
□ docker run hello-world works
□ Flask app runs locally on http://localhost:5000
□ Harbor UI accessible at http://localhost
□ All 9 Harbor containers show healthy
□ docker login localhost:80 shows "Login Succeeded"
□ Image visible in Harbor UI under myproject/myapp
□ Local image deleted successfully
□ docker pull from Harbor works
□ App runs from Harbor-pulled image
```

---

## 🐛 Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Docker Desktop is unable to start` | Docker Desktop not opened | Open Docker Desktop and wait for engine to start |
| `no configuration file provided` | `docker-compose.yml` missing | Run `sudo bash prepare` first |
| `Get https://localhost/v2/`: timeout | Insecure registry not configured | Add `localhost:80` to insecure-registries in Docker Desktop |
| `must be forced` on docker rmi | Container still using the image | Stop and remove container first with `docker stop` + `docker rm` |
| `docker` not found in WSL | WSL integration not enabled | Enable in Docker Desktop → Settings → Resources → WSL Integration |
| Harbor containers offline | WSL session closed or PC restarted | Run `sudo docker compose up -d` in harbor folder |

---

## 🗺️ What's Next — Project 2

Deploy this same app on **Kubernetes** with **APISIX** as the API gateway:

```
Internet
    ↓
APISIX (API Gateway)     ← routes and protects traffic
    ↓
Kubernetes Pods          ← runs 3 copies of your app
    ↓
Harbor                   ← K8s pulls image from here
```

Tools needed: `minikube` or `kind`, `kubectl`, `helm`

---

## 📚 Key Commands Reference

```bash
# Docker
docker build -t myapp:v1 .                        # build image
docker run -p 5000:5000 myapp:v1                  # run container
docker images                                      # list images
docker ps -a                                       # list all containers
docker stop <id>                                   # stop container
docker rm <id>                                     # remove container
docker rmi <image>                                 # remove image

# Harbor
sudo docker compose up -d                          # start Harbor
sudo docker compose ps                             # check status
sudo docker compose down                           # stop Harbor

# Registry
docker login localhost:80 -u admin -p Harbor12345  # login
docker tag myapp:v1 localhost/myproject/myapp:v1   # tag image
docker push localhost/myproject/myapp:v1           # push to Harbor
docker pull localhost/myproject/myapp:v1           # pull from Harbor
```

---

*Built as part of DevOps learning journey — Day 1 hands-on project.*
