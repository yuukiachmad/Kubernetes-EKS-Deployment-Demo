# 🚀 Vprofile — Multi-Tier Java Application on AWS EKS

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![AWS EKS](https://img.shields.io/badge/AWS%20EKS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/eks/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com)
[![Java](https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://openjdk.org)
[![Spring Boot](https://img.shields.io/badge/Spring-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](https://spring.io)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org)

---

## 📖 Overview

**Vprofile** is a production-grade, containerised **multi-tier Java web application** deployed on **AWS Elastic Kubernetes Service (EKS)**. The project demonstrates end-to-end cloud-native engineering — from local development using Docker Compose to a fully managed Kubernetes deployment on AWS.

**Skills demonstrated:**
- Container orchestration with Kubernetes
- Infrastructure management on AWS EKS
- Multi-service architecture with MySQL, Memcache, and RabbitMQ
- Ingress routing via NGINX Ingress Controller
- Persistent storage with AWS EBS (PersistentVolumeClaims)

---

## 🏗️ Architecture
<img width="1440" height="900" alt="Architecture" src="https://github.com/user-attachments/assets/7a2a2812-16d6-411d-a9ad-9a292c2e3e7f" />


## 🧩 Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Application** | Java 11, Spring MVC, Tomcat 9 | Core web application |
| **Database** | MySQL 8 | User accounts & data persistence |
| **Caching** | Memcached | Session & query caching |
| **Messaging** | RabbitMQ | Async message queue |
| **Web Server** | Nginx | Reverse proxy |
| **Containerisation** | Docker, Docker Compose | Image builds & local development |
| **Orchestration** | Kubernetes (AWS EKS) | Production container management |
| **Ingress** | NGINX Ingress Controller | HTTP routing & load balancing |
| **Cloud Provider** | AWS (EKS, ELB, EBS) | Managed Kubernetes & storage |

---

## 📁 Repository Structure

```
vprofile-project/
│
├── Docker-files/             # Dockerfiles for each service
│   ├── app/                  # Tomcat application image
│   ├── db/                   # MySQL image with seed data
│   └── web/                  # Nginx reverse proxy image
│
├── kubedefs/                 # Kubernetes manifests (deploy to EKS)
│   ├── secret.yaml           # Base64-encoded credentials
│   ├── dbpvc.yaml            # MySQL PersistentVolumeClaim (3Gi)
│   ├── dbdeploy.yaml         # MySQL Deployment
│   ├── dbservice.yaml        # MySQL ClusterIP Service
│   ├── mcdep.yaml            # Memcached Deployment
│   ├── mcservice.yaml        # Memcached ClusterIP Service
│   ├── rmqdeploy.yaml        # RabbitMQ Deployment
│   ├── rmqservice.yaml       # RabbitMQ ClusterIP Service
│   ├── appdeploy.yaml        # Tomcat App Deployment
│   ├── appservice.yaml       # App ClusterIP Service
│   └── appingress.yaml       # NGINX Ingress → vproapp-service
│
└── docker-compose.yml        # Local multi-service dev environment
```

---

## ⚙️ Service Connectivity

The application resolves its backing services using **Kubernetes DNS** — each service name maps directly to a K8s ClusterIP Service:

| Service | K8s Service Name | Port |
|---|---|---|
| MySQL Database | `vprodb` | `3306` |
| Memcached | `vprocache01` | `11211` |
| RabbitMQ | `vpromq01` | `5672` |

Credentials are stored in a Kubernetes `Secret` (`app-secret`) and injected as environment variables into the relevant pods.

---

## 🚀 Deploying to AWS EKS

### Prerequisites

Ensure the following tools are installed on your local machine:

| Tool | Minimum Version | Install |
|---|---|---|
| AWS CLI | v2 | [docs.aws.amazon.com](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) |
| `eksctl` | v0.150+ | [eksctl.io](https://eksctl.io/installation/) |
| `kubectl` | v1.28+ | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| Helm | v3+ | [helm.sh](https://helm.sh/docs/intro/install/) |

Configure your AWS credentials:
```bash
aws configure
# Enter your AWS Access Key ID, Secret, region, and output format
```

---

### Step 1 — Create the EKS Cluster

```bash
eksctl create cluster \
  --name vprofile-eks \
  --region ap-southeast-1 \
  --nodegroup-name vprofile-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```

> ⏱️ Cluster creation takes approximately **15 minutes**.

Verify nodes are ready:
```bash
kubectl get nodes
# All nodes should show STATUS = Ready
```

---

### Step 2 — Install the NGINX Ingress Controller

The `appingress.yaml` manifest uses `ingressClassName: nginx`, so the NGINX Ingress Controller must be installed via Helm. This also provisions an AWS Elastic Load Balancer automatically.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

Retrieve the AWS Load Balancer DNS name:
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
# Copy the value under EXTERNAL-IP — this is your AWS ELB endpoint
```

---

### Step 3 — Deploy the Application

Apply the Kubernetes manifests in dependency order:

```bash
cd kubedefs/

# 1. Secrets (must be applied first — referenced by other resources)
kubectl apply -f secret.yaml

# 2. Storage (PVC for MySQL data persistence)
kubectl apply -f dbpvc.yaml

# 3. Backing services
kubectl apply -f dbdeploy.yaml
kubectl apply -f dbservice.yaml
kubectl apply -f mcdep.yaml
kubectl apply -f mcservice.yaml
kubectl apply -f rmqdeploy.yaml
kubectl apply -f rmqservice.yaml

# 4. Application layer (init containers wait for DB & cache to be ready)
kubectl apply -f appdeploy.yaml
kubectl apply -f appservice.yaml

# 5. Ingress routing
kubectl apply -f appingress.yaml
```

Verify all pods are running:
```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

Expected output — all pods should show `Running`:
```
NAME                        READY   STATUS    RESTARTS   AGE
vproapp-xxxx                1/1     Running   0          2m
vprodb-xxxx                 1/1     Running   0          3m
vpromc-xxxx                 1/1     Running   0          3m
vpromq01-xxxx               1/1     Running   0          3m
```

---

### Step 4 — Configure DNS

Point your domain to the AWS Load Balancer provisioned in Step 2.

| Record Type | Name | Value |
|---|---|---|
| CNAME | `vprofile` | `<your-elb>.elb.amazonaws.com` |

> **Route 53 users:** Use an **Alias A record** pointing to the ELB instead of a CNAME.

Once DNS propagates, the app will be accessible at your configured domain.

---

## 🖥️ Local Development (Docker Compose)

To run the full stack locally without Kubernetes:

```bash
# Build and start all services
docker compose up --build -d

# Tear down
docker compose down -v
```

Services available locally:

| Service | Local URL |
|---|---|
| Nginx (Web) | http://localhost:80 |
| Tomcat (App) | http://localhost:8080 |
| MySQL | localhost:3306 |
| Memcached | localhost:11211 |
| RabbitMQ | localhost:5672 |

---

## 📊 Kubernetes Resources Summary

| Resource Name | Kind | Port |
|---|---|---|
| `vproapp` | Deployment | 8080 |
| `vprodb` | Deployment | 3306 |
| `vpromc` | Deployment | 11211 |
| `vpromq01` | Deployment | 5672 |
| `vproapp-service` | ClusterIP Service | 8080 |
| `vprodb` | ClusterIP Service | 3306 |
| `vprocache01` | ClusterIP Service | 11211 |
| `vpromq01` | ClusterIP Service | 5672 |
| `db-pv-claim` | PersistentVolumeClaim | 3Gi |
| `vpro-ingress` | Ingress | 80 |

---

## 🩺 Troubleshooting

```bash
# Watch pod startup in real time
kubectl get pods -w

# View application logs
kubectl logs deployment/vproapp --tail=100 -f

# Describe a pod for events and errors
kubectl describe pod <pod-name>

# Check ingress routing
kubectl describe ingress vpro-ingress

# Exec into a running pod
kubectl exec -it deployment/vproapp -- /bin/bash
```

| Symptom | Likely Cause | Fix |
|---|---|---|
| `vproapp` stuck at `Init:0/2` | DB or Memcache not ready yet | Wait, or check `vprodb` / `vpromc` pod status |
| Ingress shows no ADDRESS | NGINX controller not running | Check pods in `ingress-nginx` namespace |
| DB connection refused | Secret misconfigured | Run `kubectl get secret app-secret -o yaml` |
| 504 Gateway Timeout | App pod unhealthy | Check logs with `kubectl logs deployment/vproapp` |

---

## 🧹 Cleanup

Remove all resources to avoid AWS charges:

```bash
# Delete all application resources
kubectl delete -f kubedefs/

# Uninstall NGINX Ingress Controller
helm uninstall ingress-nginx -n ingress-nginx

# Delete the EKS cluster and all associated AWS resources
eksctl delete cluster --name vprofile-eks --region ap-southeast-1
```

---

*Built to demonstrate production-ready cloud-native engineering on AWS EKS.*
