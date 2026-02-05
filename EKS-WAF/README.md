# 🚀 Tic-Tac-Toe on AWS EKS (Production-Grade Architecture)

This project demonstrates a **production-ready deployment** of a containerized Node.js application on **Amazon EKS**, using **Application Load Balancer (ALB)** and **AWS WAF**, following the same security and traffic-flow principles used in a hardened EC2 setup.

---

## 📌 Key Idea

> **EKS does the same job as EC2 — just at a higher abstraction level.**  
The entry point, security model, and traffic flow remain the same.

---

## 🏗️ Architecture Diagram (Single View)
```
                     ┌──────────────┐
                     │    User      │
                     └──────┬───────┘
                            │
                            │  HTTP / HTTPS
                            ▼
               ┌──────────────────────────┐
               │  Application Load Balancer│
               │        (Public Entry)     │
               └───────────┬──────────────┘
                           │
                           │  AWS WAF (Attached)
                           ▼
                ┌─────────────────────────┐
                │        AWS WAF           │
                │  SQLi | XSS | Rate Limit│
                └───────────┬─────────────┘
                            │
                            │  ALB Ingress
                            ▼
          ┌────────────────────────────────────┐
          │          Amazon EKS Cluster          │
          │                                      │
          │   ┌────────────┐   ┌────────────┐   │
          │   │   Pod      │   │   Pod      │   │
          │   │ Node.js    │   │ Node.js    │   │
          │   │ App        │   │ App        │   │
          │   └────────────┘   └────────────┘   │
          │                                      │
          │   Pods are NOT publicly accessible   │
          └────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

- Amazon EKS
- Kubernetes
- Docker
- Application Load Balancer (ALB)
- AWS WAF
- AWS Load Balancer Controller
- Node.js
- GitHub

---

## 🔁 Request Flow (Important)

1. **User** → **ALB** (Public Entry)
2. **ALB** → **AWS WAF** (Security Filtering)
3. **WAF** → **EKS Ingress** (Clean Traffic)
4. **Ingress** → **Service** (Internal Routing)
5. **Service** → **Pods** (Application)

✔️ User never accesses Pods or Nodes directly  
✔️ Single controlled entry point

---

## 🔧 Implementation Flow (After EKS Cluster Creation)

### 1️⃣ EKS Cluster
- Cluster created with managed node group
- Worker nodes running in private subnets
- `kubectl get nodes` shows nodes in **Ready** state

---

### 2️⃣ Application Deployment
- Docker image built and pushed to ECR
- Kubernetes **Deployment** created
- Pods running Node.js app

---

### 3️⃣ Kubernetes Service
- Service type: **ClusterIP**
- Internal communication only
- No public exposure

---

### 4️⃣ ALB via Ingress
- AWS Load Balancer Controller installed
- Kubernetes **Ingress** created
- ALB automatically provisioned by AWS
- Target Group points to **Pods**, not Nodes

---

### 5️⃣ AWS WAF Protection
- Web ACL created
- WAF attached to ALB
- Enabled protections:
  - SQL Injection
  - XSS
  - Common web exploits
  - Rate-based limiting

---

## 🔐 Security Model (Same as EC2 Version)

❌ No public access to:
- Pods
- Worker nodes
- NodePorts

✅ Allowed:
- Traffic only through ALB
- WAF inspection before reaching cluster
- Private backend workloads

---

## 🔄 EC2 vs EKS (Easy Mapping)

| EC2 Setup | EKS Setup |
|---------|-----------|
| EC2 Instance | Worker Node |
| Docker Container | Pod |
| Target Group → EC2 | Target Group → Pods |
| ALB | ALB (via Ingress) |
| Manual scaling | HPA |
| SSH | kubectl |

---

## 🔐 Best Practices Applied

- Single public entry point (ALB)
- WAF protection at edge
- Backend fully private
- Kubernetes-native scaling
- Cloud-native security boundaries

---

## 🚀 Future Enhancements

- HTTPS using AWS ACM
- Domain mapping to ALB
- Horizontal Pod Autoscaler (HPA)
- Jenkins CI/CD → EKS
- Blue/Green or Canary deployments

---

## 🎤 Interview-Ready Summary

> Deployed a Dockerized Node.js application on Amazon EKS using ALB Ingress, secured it with AWS WAF, and ensured all traffic flows through a single load balancer while keeping pods and nodes private, following production-grade AWS architecture principles.

---

## 📂 Repository

https://github.com/MAHIRE-7/Tic-Tac-Toe_CICD