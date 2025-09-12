# Review-Filmes — CI/CD com GitHub Actions, Kustomize e EKS

> Aplicação **.NET 8** com **PostgreSQL**, testes (unit, integration e e2e) e pipeline **CI/CD** completa para **Amazon EKS** usando **GitHub Actions** + **Kustomize** (overlays `hml` e `prd`).

![CI](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/main.yml/badge.svg)
![Tests](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/testes.yml/badge.svg)
![Release](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/releases.yml/badge.svg)
![Deploy](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/deploy.yml/badge.svg)

---

## 🚦 Fluxo CI/CD (imagem)

![Fluxo CI/CD](fluxo_cicd.png)

---

## 🌳 Estrutura do repositório

```text
.github/workflows/
├─ main.yml        # Orquestra CI/CD em PR mergeado (develop/main)
├─ build.yml       # build dotnet
├─ testes.yml      # unit + integration + SonarQube
├─ releases.yml    # build/push da imagem no ECR + Trivy (SARIF)
└─ deploy.yml      # update de imagem no overlay + sync ArgoCD

k8s/deploy/
├─ base/
│  ├─ deployment.yaml
│  └─ kustomization.yaml
├─ hml/
│  ├─ ingress.yaml            # host: homolog.app.wsnobrega.life
│  ├─ kustomization.yaml      # image: 216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes:<tag>
│  ├─ patch-db.yaml
│  ├─ postgre.yaml
│  ├─ replicas.yaml
│  └─ service.yaml
└─ prd/
   ├─ ingress.yaml            # host: prod.app.wsnobrega.life
   ├─ kustomization.yaml      # image: 216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes:<tag>
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

## 🔧 Pré‑requisitos

- .NET SDK 8+
- Docker & Docker Compose
- kubectl + kustomize (ou `kubectl kustomize`)
- AWS CLI autenticado
- Cluster **EKS** com **AWS Load Balancer Controller**
- **ArgoCD** acessível em `argocd.app.wsnobrega.life`

---

## 🔐 Secrets & Vars necessários (GitHub)

### Variables (vars)
| Nome | Exemplo | Uso |
|---|---|---|
| `AWS_REGION` | `us-east-1` | Região AWS |
| `ID_ACCOUNT` | `216989136189` | Conta AWS |
| `STAGE` | `dev` ou `prod` | Usado no run-name/Sonar e naming das imagens |

### Secrets
| Nome | Uso |
|---|---|
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | `releases.yml` (se não usar OIDC) |
| `SONAR_HOST_URL`, `SONAR_TOKEN` | `testes.yml` |
| `GIT_USERNAME`, `GIT_PASSWORD` | `deploy.yml` faz push no repo |
| `ARGOCD_TOKEN` | `deploy.yml` (argocd login) |

**ECR** por ambiente:
- HML → `216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes`
- PRD → `216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes`

---

## 🧪 Testes locais

```bash
dotnet restore
dotnet build -c Release
dotnet test -c Release --collect:"XPlat Code Coverage"
```

## 🐳 Subir local com Docker Compose

Crie um `.env` (opcional):

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

## ☸️ Namespaces e Ingress

Namespaces:
- `hml-reviewfilmes`
- `prd-reviewfilmes`

Ingress hosts:
- HML → `homolog.app.wsnobrega.life`
- PRD → `prod.app.wsnobrega.life`

Deploy manual (debug):
```bash
kubectl apply -k k8s/deploy/hml -n hml-reviewfilmes
kubectl apply -k k8s/deploy/prd -n prd-reviewfilmes
kubectl rollout status deploy/<nome-do-deployment> -n <ns>
```

---

## 🚀 CI/CD (quando PR é mergeado)

1. `build.yml` → build .NET
2. `testes.yml` → unit + integration + SonarQube
3. `releases.yml` → build/push de imagem no **ECR** + **Trivy** (SARIF para S3)
4. `deploy.yml` → atualiza `kustomization.yaml` do overlay (`hml`/`prd`) e executa **ArgoCD sync**

---

## 👥 Contribuição

1. `git checkout -b feature/minha-feature`
2. Commits semânticos
3. PR para `develop` (hml) ou `main` (prd)



