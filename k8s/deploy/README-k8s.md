# 📦 Review-Filmes — Deploy no Kubernetes (Kustomize)

Este guia explica a estrutura da pasta `k8s/deploy` e como fazer o deploy da aplicação **Review-Filmes** no **Amazon EKS** usando **Kustomize**.

---

## 🌳 Estrutura

```text
k8s/deploy/
├─ base/
│  ├─ deployment.yaml      # Deployment genérico (imagem placeholder, probes, envs básicos)
│  └─ kustomization.yaml   # Define resources e imagem base
├─ hml/
│  ├─ ingress.yaml         # Ingress (host: homolog.app.wsnobrega.life)
│  ├─ kustomization.yaml   # Importa base/ + patches + imagem HML do ECR
│  ├─ patch-db.yaml        # Ajustes do banco (env vars, secrets)
│  ├─ postgre.yaml         # PostgreSQL (opcional, caso não use RDS)
│  ├─ replicas.yaml        # Define réplicas (HML: geralmente 1)
│  └─ service.yaml         # Service do app
└─ prd/
   ├─ ingress.yaml         # Ingress (host: prod.app.wsnobrega.life)
   ├─ kustomization.yaml   # Importa base/ + patches + imagem PRD do ECR
   ├─ patch-db.yaml        # Ajustes do banco (env vars, secrets)
   ├─ postgre.yaml         # PostgreSQL (opcional, caso não use RDS)
   ├─ replicas.yaml        # Define réplicas (PRD: >1 para HA)
   └─ service.yaml         # Service do app
```

---

## 📌 Arquivos e Funções

### 🔹 `base/`
- **deployment.yaml**  
  - Define o `Deployment` principal da aplicação.  
  - Usa imagem placeholder (`placeholder`) que será sobrescrita via `kustomization.yaml` dos overlays (`hml`/`prd`).  
  - Inclui probes (`livenessProbe` e `readinessProbe`).  

- **kustomization.yaml**  
  - Lista `resources` que compõem a base.  
  - Define o bloco `images:` com `newName: placeholder`.

---

### 🔹 `hml/`
- **ingress.yaml**  
  - Configura um ALB ingress público (`alb.ingress.kubernetes.io/scheme: internet-facing`).  
  - Host: `homolog.app.wsnobrega.life`.  
  - Annotations podem incluir SSL redirect, certificado ACM etc.  

- **kustomization.yaml**  
  - Inclui `../base`.  
  - Aplica patches (`replicas.yaml`, `patch-db.yaml`, `postgre.yaml`).  
  - Substitui `placeholder` pela imagem do ECR:  
    ```
    216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes:<tag>
    ```

- **patch-db.yaml**  
  - Injeta variáveis de conexão para o banco (`ConnectionStrings__Default`).  
  - Pode referenciar um `Secret` no namespace `hml-reviewfilmes`.

- **postgre.yaml**  
  - Caso queira rodar PostgreSQL dentro do cluster.  
  - Recomenda-se RDS para produção, mas pode ser útil em HML.

- **replicas.yaml**  
  - Define `replicas: 1`.  

- **service.yaml**  
  - Expõe o app no cluster (geralmente `ClusterIP`, usado pelo Ingress).  

---

### 🔹 `prd/`
Mesmo padrão do `hml/`, com ajustes:

- **ingress.yaml** → Host: `prod.app.wsnobrega.life`
- **kustomization.yaml** → Imagem:  
  ```
  216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes:<tag>
  ```
- **replicas.yaml** → `replicas: 2+` (alta disponibilidade)
- **patch-db.yaml** → Secrets do namespace `prd-reviewfilmes`
- **service.yaml** → expõe app internamente para o ALB
- **postgre.yaml** → opcional (substituível por RDS)

---

## ☸️ Namespaces

Namespaces esperados:
- `hml-reviewfilmes`
- `prd-reviewfilmes`

Criação:
```bash
kubectl create ns hml-reviewfilmes
kubectl create ns prd-reviewfilmes
```

---

## 🚀 Deploy Manual

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

## 🔐 Secrets no cluster

Antes do deploy, crie os `Secrets` com credenciais do banco:

```bash
kubectl create secret generic db-credentials   --from-literal=ConnectionStrings__Default="Host=<host>;Database=review;Username=review;Password=xxxx"   -n hml-reviewfilmes

kubectl create secret generic db-credentials   --from-literal=ConnectionStrings__Default="Host=<host>;Database=review;Username=review;Password=xxxx"   -n prd-reviewfilmes
```

Os overlays (`patch-db.yaml`) devem referenciar esse Secret.

---

## 📈 Boas práticas

- **Não rodar PostgreSQL no cluster de produção** → use **Amazon RDS**.  
- **Ingress** com **AWS Load Balancer Controller** + **ACM** para HTTPS.  
- **Replicas** maiores em PRD (`min 2`).  
- **Autoscaling** opcional via `HorizontalPodAutoscaler`.  
- **Secrets** via AWS Secrets Manager (ou pelo menos Kubernetes Secrets).  

---
