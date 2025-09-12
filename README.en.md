# Review-Filmes — CI/CD with GitHub Actions, Kustomize and EKS

> **.NET 8** application with **PostgreSQL**, tests (unit, integration, and e2e), and a full **CI/CD pipeline** for **Amazon EKS** using **GitHub Actions** + **Kustomize** (`hml` and `prd` overlays).

![CI](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/main.yml/badge.svg)
![Tests](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/testes.yml/badge.svg)
![Release](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/releases.yml/badge.svg)
![Deploy](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/deploy.yml/badge.svg)

---

## 🚦 CI/CD Flow (image)

> Mermaid diagrams sometimes fail on GitHub, so a **PNG fallback** is included.

![CI/CD Flow](fluxo_cicd.png)

---

## 🌳 Repository Structure

```text
.github/workflows/
├─ main.yml        # Orchestrates CI/CD on PR merged (develop/main)
├─ build.yml       # .NET build
├─ testes.yml      # unit + integration + SonarQube
├─ releases.yml    # build/push Docker image to ECR + Trivy (SARIF)
└─ deploy.yml      # updates overlay image + ArgoCD sync

k8s/deploy/
├─ base/
│  ├─ deployment.yaml
│  └─ kustomization.yaml
├─ hml/
│  ├─ ingress.yaml           # host: homolog.app.wsnobrega.life
│  ├─ kustomization.yaml     # image: 216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes:<tag>
│  ├─ patch-db.yaml
│  ├─ postgre.yaml
│  ├─ replicas.yaml
│  └─ service.yaml
└─ prd/
   ├─ ingress.yaml           # host: prod.app.wsnobrega.life
   ├─ kustomization.yaml     # image: 216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes:<tag>
   ├─ patch-db.yaml
   ├─ postgre.yaml
   ├─ replicas.yaml
   └─ service.yaml

src/
├─ Review-Filmes.Domain/
├─ Review-Filmes.Web/
├─ Review-Filmes.Test.Unit/
├─ Review-Filmes.Test.Integration/
└─ Review-Filmes.Test.EndToEnd/
```

---

## 🔧 Requirements

- .NET SDK 8+
- Docker & Docker Compose
- kubectl + kustomize (or `kubectl kustomize`)
- AWS CLI authenticated
- **EKS** cluster with **AWS Load Balancer Controller**
- **ArgoCD** accessible at `argocd.app.wsnobrega.life`

---

## 🔐 Required GitHub Secrets & Vars

### Variables (vars)
| Name | Example | Purpose |
|---|---|---|
| `AWS_REGION` | `us-east-1` | AWS Region |
| `ID_ACCOUNT` | `216989136189` | AWS Account ID |
| `STAGE` | `dev` or `prod` | Used in run-name/Sonar and image naming |

### Secrets
| Name | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Used by `releases.yml` (if OIDC not enabled) |
| `SONAR_HOST_URL`, `SONAR_TOKEN` | Used by `testes.yml` |
| `GIT_USERNAME`, `GIT_PASSWORD` | `deploy.yml` pushes back to repo |
| `ARGOCD_TOKEN` | Used in `deploy.yml` to login into ArgoCD |

**ECR repos per environment:**
- HML → `216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes`
- PRD → `216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes`

---

## 🧪 Local Tests

```bash
dotnet restore
dotnet build -c Release
dotnet test -c Release --collect:"XPlat Code Coverage"
```

## 🐳 Run Locally with Docker Compose

`.env` example:
```env
POSTGRES_DB=review
POSTGRES_USER=review
POSTGRES_PASSWORD=postgrespwd
ASPNETCORE_ENVIRONMENT=Development
ConnectionStrings__Default=Host=postgres;Database=review;Username=review;Password=postgrespwd
```

```bash
docker compose up -d --build
# http://localhost:8080
```

---

## ☸️ Namespaces and Ingress

Namespaces:
- `hml-reviewfilmes`
- `prd-reviewfilmes`

Ingress hosts:
- HML → `homolog.app.wsnobrega.life`
- PRD → `prod.app.wsnobrega.life`

Manual deploy (debug):
```bash
kubectl apply -k k8s/deploy/hml -n hml-reviewfilmes
kubectl apply -k k8s/deploy/prd -n prd-reviewfilmes
kubectl rollout status deploy/<deployment-name> -n <ns>
```

---

## 🚀 CI/CD (when PR merged)

1. `build.yml` → .NET build
2. `testes.yml` → unit + integration + SonarQube
3. `releases.yml` → build/push image to **ECR** + **Trivy** (SARIF to S3)
4. `deploy.yml` → updates `kustomization.yaml` in the overlay (`hml`/`prd`) and performs **ArgoCD sync**

---

## 👥 Contribution

1. `git checkout -b feature/my-feature`
2. Semantic commits
3. PR to `develop` (hml) or `main` (prd)

## 📄 License

MIT (or your choice).
