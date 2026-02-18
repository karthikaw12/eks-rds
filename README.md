# EKS + RDS Database Architecture (OTP Application)

This repository documents the **production-style architecture** used to deploy an OTP application where:

* Frontend and Backend run on **Amazon EKS**
* Database runs on **Amazon RDS** (managed, private)
* Secure access is provided via a **Bastion Host**

---

## 🏗️ Architecture Overview

* **Frontend**: Kubernetes Deployment + Service (EKS)
* **Backend (OTP App)**: Kubernetes Deployment (EKS)
* **Database**: Amazon RDS (PostgreSQL / MySQL)
* **Networking**: Single VPC with public & private subnets
* **Security**: Security Groups, IAM, Kubernetes Secrets / AWS Secrets Manager

---

## 📐 Architecture Diagrams

### Basic Architecture (Current Implementation)

![image4](image4)

*Figure 1: Simple VPC architecture showing EKS cluster connecting to RDS through private networking with Bastion Host for admin access*

### Advanced Multi-VPC Architecture

![image3](image3)

*Figure 2: Enterprise-grade multi-VPC setup with Transit Gateway for complex networking scenarios*

### EKS with Integrated Services

![image2](image2)

*Figure 3: EKS cluster integrated with various AWS services including S3, EFS, and RDS*

### Multi-Account Organization Setup

![image1](image1)

*Figure 4: AWS Organizations setup with Resource Access Manager for cross-account resource sharing*

---

## 🌐 VPC Design

**VPC CIDR:** `10.0.0.0/16`

### Subnets (Multi-AZ)

| Subnet Type                 | Purpose                   |
| --------------------------- | ------------------------- |
| Public Subnet (AZ-A, AZ-B)  | Bastion Host, NAT Gateway |
| Private Subnet (AZ-A, AZ-B) | EKS Worker Nodes, RDS     |

---

## 🔐 Security Groups

### Bastion Host SG

* Inbound: `SSH (22)` from **your IP only**
* Outbound: All traffic

### EKS Node Group SG

* Outbound: `5432 / 3306` to RDS SG

### RDS SG

* Inbound:

  * `5432 (Postgres)` or `3306 (MySQL)` from:

    * EKS Node Group SG
    * Bastion Host SG
* ❌ No public access

---

## 🗄️ RDS Configuration

* Engine: PostgreSQL / MySQL
* Public access: ❌ Disabled
* Subnet group: **Private subnets only**
* Multi-AZ: Optional (recommended for prod)

---

## ☸️ EKS Configuration

* Cluster and Node Groups deployed into **private subnets**
* Pods communicate with RDS using **private DNS endpoint**
* Images pulled via NAT Gateway

---

## 🔑 Database Credential Management

### Option 1: Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_HOST: mydb.xxxxxx.region.rds.amazonaws.com
  DB_PORT: "5432"
  DB_NAME: otpdb
  DB_USER: otpuser
  DB_PASSWORD: strongpassword
```

Backend Deployment references the secret as environment variables.

---

### Option 2 (Best Practice): AWS Secrets Manager + IRSA

* Store DB credentials in **AWS Secrets Manager**
* Create IAM role with secret access
* Attach IAM role to Kubernetes ServiceAccount (IRSA)
* Backend fetches secrets dynamically at runtime

**Benefits:**

* No secrets in YAML
* Easy rotation
* Enterprise-grade security

---

## 🔗 Backend → RDS Connection Flow

1. Backend Pod starts in EKS
2. Reads DB credentials (Secret / Secrets Manager)
3. Resolves RDS private DNS
4. Connects via VPC internal network
5. Security Groups allow traffic

---

## 🛡️ Bastion Host Access (Admin Only)

```bash
ssh ec2-user@<bastion-public-ip>
psql -h mydb.rds.amazonaws.com -U otpuser -d otpdb
```

Used only for debugging, migrations, or admin access.

---

## ✅ Best Practices Followed

* Database not inside Kubernetes
* No public database exposure
* Principle of least privilege (SG + IAM)
* Secrets not hardcoded
* Highly available networking

---

## 🎯 Architecture Summary

> "Frontend and backend run in EKS private subnets, while the database is hosted in Amazon RDS within the same VPC. Backend connects to RDS using private networking secured by security groups, with credentials managed via Kubernetes Secrets or AWS Secrets Manager."

---

## 📌 Future Enhancements

* RDS Proxy for connection pooling
* Automated secret rotation
* Terraform IaC
* Network policies in EKS

---

**Author:** Karthik Sivakumar

