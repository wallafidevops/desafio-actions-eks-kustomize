# 📦 Review-Filmes — Kubernetes Deployment (Kustomize)

This guide explains the structure of the `k8s/deploy` folder and how to deploy the **Review-Filmes** application to **Amazon EKS** using **Kustomize**.

---

## 🌳 Structure

```text
k8s/deploy/
├─ base/
│  ├─ deployment.yaml      # Generic Deployment (placeholder image, probes, basic envs)
│  └─ kustomization.yaml   # Defines resources and base image
├─ hml/
│  ├─ ingress.yaml         # Ingress (host: homolog.app.wsnobrega.life)
│  ├─ kustomization.yaml   # Imports base/ + patches + HML ECR image
│  ├─ patch-db.yaml        # DB adjustments (env vars, secrets)
│  ├─ postgre.yaml         # Optional PostgreSQL (not recommended for prod)
│  ├─ replicas.yaml        # Sets replicas (HML: usually 1)
│  └─ service.yaml         # App Service
└─ prd/
   ├─ ingress.yaml         # Ingress (host: prod.app.wsnobrega.life)
   ├─ kustomization.yaml   # Imports base/ + patches + PRD ECR image
   ├─ patch-db.yaml        # DB adjustments (env vars, secrets)
   ├─ postgre.yaml         # Optional PostgreSQL (replace with RDS in prod)
   ├─ replicas.yaml        # Sets replicas (PRD: >1 for HA)
   └─ service.yaml         # App Service
```

---

## 📌 Files and Purpose

### 🔹 `base/`
- **deployment.yaml**  
  - Defines the main `Deployment`.  
  - Uses `placeholder` image replaced by overlays.  
  - Includes probes (liveness, readiness).  

- **kustomization.yaml**  
  - Lists base resources.  
  - Defines `images:` block with `newName: placeholder`.

---

### 🔹 `hml/`
- **ingress.yaml** → ALB ingress, host `homolog.app.wsnobrega.life`.  
- **kustomization.yaml** → Replaces `placeholder` with HML ECR image.  
- **patch-db.yaml** → Injects DB connection string.  
- **postgre.yaml** → Optional PostgreSQL inside cluster.  
- **replicas.yaml** → `replicas: 1`.  
- **service.yaml** → ClusterIP service for ingress.  

### 🔹 `prd/`
- **ingress.yaml** → Host `prod.app.wsnobrega.life`.  
- **kustomization.yaml** → Uses PRD ECR image.  
- **patch-db.yaml** → Injects secrets for PRD.  
- **postgre.yaml** → Optional, replace with RDS.  
- **replicas.yaml** → `replicas: 2+`.  
- **service.yaml** → Internal service for ALB.  

---

## ☸️ Namespaces

Namespaces required:
- `hml-reviewfilmes`
- `prd-reviewfilmes`

```bash
kubectl create ns hml-reviewfilmes
kubectl create ns prd-reviewfilmes
```

---

## 🚀 Manual Deploy

### HML
```bash
kubectl apply -k k8s/deploy/hml -n hml-reviewfilmes
kubectl rollout status deploy/review-filmes -n hml-reviewfilmes
```

### PRD
```bash
kubectl apply -k k8s/deploy/prd -n prd-reviewfilmes
kubectl rollout status deploy/review-filmes -n prd-reviewfilmes
```

---

## 🔐 Secrets

```bash
kubectl create secret generic db-credentials   --from-literal=ConnectionStrings__Default="Host=<host>;Database=review;Username=review;Password=xxxx"   -n hml-reviewfilmes

kubectl create secret generic db-credentials   --from-literal=ConnectionStrings__Default="Host=<host>;Database=review;Username=review;Password=xxxx"   -n prd-reviewfilmes
```

---

## 📈 Best Practices

- Do not run PostgreSQL in prod cluster → use **Amazon RDS**.  
- Ingress with **AWS Load Balancer Controller** + **ACM** for HTTPS.  
- Replicas ≥2 in PRD.  
- Optionally add HPA for autoscaling.  
- Store secrets in AWS Secrets Manager.  

---