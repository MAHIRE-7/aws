# 🚀 Tic-Tac-Toe CI/CD on AWS (Production Architecture)

This repository demonstrates a **production-grade deployment** of a Node.js application using **Docker, AWS EC2, Application Load Balancer (ALB), and AWS WAF**, following real-world DevOps and security best practices.

---

## 📌 What This Project Shows

- Containerized application running on **EC2**
- **Application Load Balancer (ALB)** as the only public entry point
- **AWS WAF** protecting the application
- **Direct EC2 access completely blocked**
- Clean separation between **public layer** and **backend**

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
               │        (Internet Facing)  │
               └───────────┬──────────────┘
                           │
                           │  WAF Rules Applied
                           ▼
                ┌─────────────────────────┐
                │        AWS WAF           │
                │  (SQLi, XSS, Rate Limit)│
                └───────────┬─────────────┘
                            │
                            │  HTTP : 80
                            ▼
                ┌─────────────────────────┐
                │     EC2 (Ubuntu)         │
                │  Security Group allows   │
                │  traffic ONLY from ALB   │
                └───────────┬─────────────┘
                            │
                            │  Docker Port Mapping
                            ▼
                ┌─────────────────────────┐
                │   Docker Container       │
                │   Node.js App (3000)     │
                └─────────────────────────┘
```

---

## 🛠️ Tech Stack

- **AWS EC2 (Ubuntu)**
- **Docker**
- **Application Load Balancer (ALB)**
- **AWS WAF**
- **Node.js**
- **GitHub**

---

## 🔧 Implementation Summary

### 1️⃣ EC2 & Docker
- Launched Ubuntu EC2
- Installed Docker
- Added user to Docker group
- Ran Node.js app inside Docker container

---

### 2️⃣ Application Deployment
- App runs internally on port **3000**
- Docker mapped container port to **EC2 port 80**

---

### 3️⃣ Application Load Balancer
- Created Target Group (HTTP:80)
- Registered EC2 instance
- Created Internet-facing ALB
- Forwarded traffic from ALB → EC2

---

### 4️⃣ Security Hardening (Critical)
- ❌ Removed public access to EC2
- ✅ Allowed inbound traffic to EC2 **ONLY from ALB Security Group**

---

### 5️⃣ AWS WAF Protection
- Attached Web ACL to ALB
- Enabled AWS Managed Rules:
  - SQL Injection protection
  - XSS protection
  - Common web exploits
- Added rate-based rule for abuse prevention

---

## 🔐 Security Best Practices Applied

- Single entry point via ALB
- Backend EC2 isolated from internet
- WAF protection at edge
- Security Group chaining (ALB → EC2)
- No public exposure of compute layer

---

## 🚀 Future Enhancements

- Enable HTTPS using **AWS ACM**
- Map custom domain to ALB
- Add Auto Scaling Group
- Implement Jenkins CI/CD pipeline
- Push images to Amazon ECR

---

## 🧠 Interview-Ready Summary

> Deployed a Dockerized Node.js application on EC2, fronted it with an Application Load Balancer, secured it with AWS WAF, and restricted backend access so traffic flows only through the load balancer following production-grade AWS architecture.

---

## 📂 Repository

https://github.com/MAHIRE-7/Tic-Tac-Toe_CICD