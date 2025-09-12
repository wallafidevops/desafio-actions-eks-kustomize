# Review-Filmes — CI/CD com GitHub Actions, Kustomize e EKS

> Aplicação .NET para reviews de filmes com PostgreSQL, testes (unit, integration e e2e) e pipeline completa de CI/CD para Kubernetes (EKS), organizada por **overlays** (`hml` e `prd`) via **Kustomize**.

![CI](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/main.yml/badge.svg)
![Tests](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/testes.yml/badge.svg)
![Release](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/releases.yml/badge.svg)
![Deploy](https://github.com/wallafidevops/desafio-actions-eks-kustomize/actions/workflows/deploy.yml/badge.svg)

## 🌳 Estrutura do repositório

```
.github/workflows/
├─ main.yml        # CI: build, testes, análise
├─ testes.yml      # (opcional) job dedicado de testes
├─ releases.yml    # Release/tag + build de imagem
└─ deploy.yml      # CD: deploy no EKS por overlay

k8s/deploy/
├─ base/
│  ├─ deployment.yaml
│  └─ kustomization.yaml
├─ hml/
│  ├─ ingress.yaml
│  ├─ kustomization.yaml
│  ├─ patch-db.yaml
│  ├─ postgre.yaml
│  ├─ replicas.yaml
│  └─ service.yaml
└─ prd/
   ├─ ingress.yaml
   ├─ kustomization.yaml
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

## ⚙️ Requisitos

- .NET SDK 8+
- Docker & Docker Compose
- Kubectl, Kustomize (ou `kubectl kustomize`)
- (CD) Acesso a um cluster **Amazon EKS** configurado
- (CD) **AWS CLI** autenticado com permissões para ECR/EKS/KMS/Secrets Manager

## 🧪 Como rodar testes localmente

```bash
dotnet restore
dotnet build -c Release
dotnet test -c Release --collect:"XPlat Code Coverage"
```

## 🐳 Executar localmente com Docker Compose

Crie um `.env` com variáveis:

```env
POSTGRES_DB=review
POSTGRES_USER=review
POSTGRES_PASSWORD=postgrespwd
ASPNETCORE_ENVIRONMENT=Development
ConnectionStrings__Default=Host=postgres;Database=review;Username=review;Password=postgrespwd
```

Suba os serviços:

```bash
docker compose up -d --build
```

Acesse: `http://localhost:8080`

## 🛠️ Build da imagem Docker (manual)

```bash
docker build -t review-filmes:local -f src/Review-Filmes.Web/Dockerfile .
docker run --rm -p 8080:8080 --env-file .env review-filmes:local
```

### Push para o ECR

```bash
aws ecr get-login-password --region us-east-1  | docker login --username AWS --password-stdin 216989136189.dkr.ecr.us-east-1.amazonaws.com

docker tag review-filmes:local 216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes:<tag>
docker push 216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes:<tag>
```

## ☸️ Kubernetes com Kustomize

Namespaces:
- `hml-reviewfilmes`
- `prd-reviewfilmes`

Ingress:
- Homologação → `homolog.app.wsnobrega.life`
- Produção → `prod.app.wsnobrega.life`

### Aplicar HML

```bash
kubectl apply -k k8s/deploy/hml -n hml-reviewfilmes
```

### Aplicar PRD

```bash
kubectl apply -k k8s/deploy/prd -n prd-reviewfilmes
```

## 🔐 Segredos e variáveis

- GitHub Actions:  
  `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `ECR_REPOSITORY`, `KUBE_CONFIG`
- Kubernetes Secrets: credenciais do PostgreSQL, connection string etc.

## 🚀 Pipelines (GitHub Actions)

### `main.yml` — CI
- Build, testes, análise

### `testes.yml` — Testes dedicados
- Unit, integration, e2e

### `releases.yml` — Release & Imagem
- Builda imagem Docker, publica no **ECR**

### `deploy.yml` — CD para EKS
- Autentica na AWS/EKS
- Aplica overlay (`hml` ou `prd`)

## ✅ Checklist

1. Criar ECR (já configurado):
   - `216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes`
   - `216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes`
2. Configurar Secrets no GitHub
3. Criar Namespaces no EKS (`hml-reviewfilmes`, `prd-reviewfilmes`)
4. Criar Secrets no K8s para DB
5. Rodar pipelines `releases.yml` e `deploy.yml`
6. Validar rollout no cluster

## 👥 Contribuição

1. `git checkout -b feature/minha-feature`
2. Commits claros
3. PR para `develop` (HML) ou `main` (PRD)

## 📄 Licença

MIT (ou a de sua preferência).

