# 🐳 Docker, CI/CD e Kubernetes

Como a DampCode vira imagem Docker, como o GitHub Actions testa e publica essas imagens e como elas chegam num cluster Kubernetes.

> Para desenvolver no dia a dia continue usando [Instalação em uma máquina nova](./instalacao.md) (`dotnet run` + `npm run dev`).
> Este guia é para rodar **como em produção**.

---

## 📖 Sumário
1. [Visão do fluxo](#1-visão-do-fluxo)
2. [As imagens Docker](#2-as-imagens-docker)
3. [Rodar tudo com Docker Compose](#3-rodar-tudo-com-docker-compose)
4. [CI/CD no GitHub Actions](#4-cicd-no-github-actions)
5. [Preparar o cluster Kubernetes (uma vez)](#5-preparar-o-cluster-kubernetes-uma-vez)
6. [Ligar o deploy automático](#6-ligar-o-deploy-automático)
7. [Operação: logs, rollback, migrations, backup](#7-operação-logs-rollback-migrations-backup)
8. [Problemas comuns](#8-problemas-comuns)

---

## 1. Visão do fluxo

```mermaid
flowchart LR
    PR[Pull Request] --> CI["CI<br/>testes / build / lint<br/>build da imagem"]
    M["merge na master / main"] --> CI2["CI<br/>mesmos passos"]
    CI2 --> GHCR[("GHCR<br/>ghcr.io/damp-code/...<br/>:sha e :latest")]
    GHCR --> D{"K8S_DEPLOY<br/>= true?"}
    D -- não --> FIM[para aqui]
    D -- sim --> J["Job de migrations<br/>(só API)"]
    J --> R["kubectl apply<br/>+ rollout status"]
```

Dentro do cluster:

```mermaid
flowchart LR
    U[Navegador] --> I["Ingress<br/>(ingress-nginx)"]
    I --> F["dampcode-front<br/>nginx :8080"]
    F -- "/api/*" --> A["dampcode-api<br/>:8080"]
    A --> P[("postgres<br/>StatefulSet")]
    A --> M[("mongo<br/>StatefulSet")]
    J["Job dampcode-migrations<br/>./efbundle"] --> P
```

O front e a API ficam no **mesmo domínio**: o nginx do front repassa `/api/` para a API. Por isso não há CORS em produção e o front não precisa saber o endereço da API.

| Onde está | O quê |
|-----------|-------|
| `DampCode_API/Dockerfile` | imagem da API (inclui o `efbundle` das migrations) |
| `DampCode_API/.github/workflows/ci-cd.yml` | testes → imagem → migrations + deploy |
| `Damp-Code-Front/Dockerfile` + `nginx/default.conf.template` | imagem do front (Vite build + nginx) |
| `Damp-Code-Front/.github/workflows/ci-cd.yml` | build/lint → imagem → deploy |
| `DAMPCode-Base/docker-compose.yml` + `.env.example` | stack completa local em containers |
| `DAMPCode-Base/k8s/` | manifests do cluster (Kustomize) |

---

## 2. As imagens Docker

### API — `ghcr.io/damp-code/dampcode-api`

- Build com o SDK do .NET 8, runtime `mcr.microsoft.com/dotnet/aspnet:8.0`, usuário sem root, porta **8080**, `ASPNETCORE_ENVIRONMENT=Production` (sem Swagger, sem User Secrets).
- Traz junto o **`efbundle`**: um executável que aplica as migrations. A mesma imagem roda a API (padrão) ou as migrations (`./efbundle`).
- Rota **`/health`** para as sondas do Kubernetes.
- Configuração só por variável de ambiente:

| Variável | Exemplo |
|----------|---------|
| `ConnectionStrings__PostgreSql` | `Host=postgres;Port=5432;Database=dampcode;Username=dampcode;Password=...` |
| `ConnectionStrings__DbConnection` | `mongodb://dampcode:...@mongo:27017/damp_code?authSource=admin` |
| `Jwt__Secret` | 32+ caracteres |
| `Cors__Origins__0` (opcional) | `https://outro-dominio.com` — só se o front ficar em **outro** domínio |

Build manual, na raiz do repositório da API:

```bash
docker build -t dampcode-api .
```

### Front — `ghcr.io/damp-code/damp-code-front`

- Build com `node:22-alpine`, servido por `nginxinc/nginx-unprivileged` (sem root) na porta **8080**.
- O nginx devolve o `index.html` para qualquer rota (SPA), guarda em cache os arquivos de `/assets/` e repassa `/api/` para `API_UPSTREAM`.

| Variável | Quando | Padrão |
|----------|--------|--------|
| `API_UPSTREAM` | ao **rodar** o container | `http://api:8080` (compose). No Kubernetes: `http://dampcode-api:8080` |
| `VITE_API_URL` | no **build** (`--build-arg`) | vazio = chamadas relativas `/api/...`. Só use se a API estiver em outro domínio. |

```bash
docker build -t damp-code-front .
```

> O modo mock nunca entra na imagem: ele só existe no `npm run dev`.

---

## 3. Rodar tudo com Docker Compose

Útil para testar a versão "de produção" na sua máquina ou mostrar o projeto sem instalar .NET, PostgreSQL e MongoDB. Só precisa do [Docker Desktop](https://www.docker.com/products/docker-desktop/).

Os três repositórios clonados lado a lado:

```
DAMPCODE/
├── DAMPCode-Base/      ← rode os comandos aqui
├── DampCode_API/
└── Damp-Code-Front/
```

```bash
cd DAMPCode-Base
cp .env.example .env
# edite o .env e preencha JWT_SECRET (gere com o comando que está no próprio arquivo)
docker compose up --build
```

Abra **http://localhost:8080**. O que acontece na ordem:

1. `postgres` e `mongo` sobem e ficam saudáveis.
2. `migrations` roda o `efbundle`, cria as tabelas e termina.
3. `api` sobe (só depois das migrations terminarem sem erro).
4. `front` sobe e publica a porta 8080.

| Comando | Para quê |
|---------|----------|
| `docker compose up -d --build` | sobe em segundo plano |
| `docker compose logs -f api` | acompanha os logs da API |
| `docker compose ps -a` | status de cada serviço |
| `docker compose down` | para tudo (os dados ficam nos volumes) |
| `docker compose down -v` | para tudo e **apaga** os bancos |
| `docker compose pull && docker compose up -d` | usa as imagens publicadas pelo CI em vez de compilar |

Se os repositórios estiverem em outro lugar, ajuste `API_CONTEXT` e `FRONT_CONTEXT` no `.env`. Para promover um Admin no compose:

```bash
docker compose exec postgres psql -U dampcode -d dampcode -c "UPDATE accounts SET account_type = 'Admin' WHERE normalized_email = 'ADMIN@EXEMPLO.COM';"
```

---

## 4. CI/CD no GitHub Actions

Cada repositório de código tem um `.github/workflows/ci-cd.yml`:

| Evento | API (`DampCode_API`) | Front (`Damp-Code-Front`) |
|--------|----------------------|---------------------------|
| Pull Request / push em qualquer branch | `dotnet test` (SDK 8 + 10) e build da imagem | `npm ci`, `npm run build`, `npm run lint` (não bloqueia) e build da imagem |
| push na `master` / `main` | + publica `ghcr.io/damp-code/dampcode-api:<sha>` e `:latest` | + publica `ghcr.io/damp-code/damp-code-front:<sha>` e `:latest` |
| push na `master` / `main` **com `K8S_DEPLOY=true`** | + Job de migrations e rollout da API | + rollout do front |
| `workflow_dispatch` | permite rodar à mão na aba **Actions** | idem |

- A imagem é publicada com o `GITHUB_TOKEN` do próprio Actions: não precisa criar token para o GHCR.
- O deploy busca os manifests do repositório **DAMPCode-Base** (`k8s/`) e troca `:latest` pela tag do commit, então cada deploy aponta para uma imagem exata.
- Enquanto `K8S_DEPLOY` não existir, o CI/CD **só publica as imagens**. Assim dá para fazer o merge antes de ter um cluster.
- O job `deploy` usa o *environment* `producao`. Em **Settings → Environments → producao** dá para exigir aprovação manual antes de cada deploy.

---

## 5. Preparar o cluster Kubernetes (uma vez)

Qualquer cluster serve (DigitalOcean, GKE, AKS, EKS, k3s numa VPS, ou minikube/kind para teste). É preciso:

- `kubectl` configurado apontando para o cluster;
- uma **StorageClass padrão** (os bancos pedem 5 GiB de disco cada);
- um **Ingress Controller** `nginx`. Se o cluster não tiver, siga a [instalação do ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/) para o seu provedor.

### 5.1 Namespace e segredos

Os segredos **não** ficam no Git. Crie direto no cluster (gere valores novos, diferentes dos de desenvolvimento):

```bash
kubectl create namespace dampcode

kubectl -n dampcode create secret generic dampcode-secrets \
  --from-literal=postgres-password="$(node -e "console.log(require('crypto').randomBytes(24).toString('hex'))")" \
  --from-literal=mongo-password="$(node -e "console.log(require('crypto').randomBytes(24).toString('hex'))")" \
  --from-literal=jwt-secret="$(node -e "console.log(require('crypto').randomBytes(48).toString('base64'))")"
```

| Chave do Secret | Usada por |
|-----------------|-----------|
| `postgres-password` | `postgres`, `dampcode-api`, Job de migrations |
| `mongo-password` | `mongo`, `dampcode-api` |
| `jwt-secret` | `dampcode-api` |

> A senha do PostgreSQL e a do MongoDB só são gravadas no **primeiro** start dos bancos (volume vazio). Trocar o Secret depois não muda a senha dentro do banco.

### 5.2 Acesso às imagens do GHCR

Os pacotes publicados pelo Actions podem nascer **privados**. Escolha uma opção:

- **Deixar públicos** (repositórios já são públicos): em github.com/orgs/DAMP-Code/packages → pacote → *Package settings* → *Change visibility* → Public. Faça para `dampcode-api` e `damp-code-front`.
- **Ou** criar o Secret de pull com um token pessoal (*classic*, escopo `read:packages`):

  ```bash
  kubectl -n dampcode create secret docker-registry ghcr-pull \
    --docker-server=ghcr.io --docker-username=<usuario-github> --docker-password=<token>
  ```

Os manifests já referenciam `ghcr-pull`. Se o Secret não existir e as imagens forem públicas, o Kubernetes só ignora.

### 5.3 Primeira instalação

Os pushes na `master`/`main` precisam ter rodado ao menos uma vez para existirem as imagens `:latest`. **Antes de aplicar, troque `dampcode.exemplo.com` pelo domínio real em `k8s/ingress.yaml`** (seção 5.4). Na pasta do DAMPCode-Base:

```bash
kubectl apply -k k8s/
kubectl -n dampcode rollout status statefulset/postgres
kubectl -n dampcode apply -f k8s/migrations-job.yaml
kubectl -n dampcode wait --for=condition=complete job/dampcode-migrations --timeout=300s
kubectl -n dampcode get pods
```

A API sobe antes das tabelas existirem (o `/health` não consulta o banco), mas as chamadas só funcionam depois que o Job de migrations termina.

Para descobrir o endereço:

```bash
kubectl -n dampcode get ingress dampcode
```

Sem domínio ainda, teste com port-forward (o tráfego fica só entre a sua máquina e o cluster): `kubectl -n dampcode port-forward svc/dampcode-front 8080:80` e abra http://localhost:8080.

### 5.4 Domínio e HTTPS

O Ingress já vem com **HTTPS obrigatório**: só responde no domínio configurado e redireciona HTTP para HTTPS. Isso é proposital, porque login, senha e token passam por ele. Para funcionar:

1. Troque `dampcode.exemplo.com` pelo domínio nos dois lugares de `k8s/ingress.yaml` (`tls.hosts` e `rules.host`).
2. Aponte o DNS do domínio para o IP externo do Ingress Controller.
3. Instale o [cert-manager](https://cert-manager.io/docs/installation/) e crie o emissor `letsencrypt` (troque o e-mail):

   ```yaml
   # cluster-issuer.yaml — kubectl apply -f cluster-issuer.yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: <email-do-time>
       privateKeySecretRef:
         name: letsencrypt
       solvers:
         - http01:
             ingress:
               ingressClassName: nginx
   ```

4. `kubectl apply -k k8s/`. Em alguns minutos o certificado aparece em `kubectl -n dampcode get certificate` com `READY True`.

Enquanto o certificado não sai, o ingress-nginx responde com um certificado provisório (o navegador avisa). Não use a aplicação assim: espere o `READY True`.

### 5.5 Primeiro Admin

```bash
kubectl -n dampcode exec -it statefulset/postgres -- psql -U dampcode -d dampcode \
  -c "UPDATE accounts SET account_type = 'Admin' WHERE normalized_email = 'ADMIN@EXEMPLO.COM';"
```

---

## 6. Ligar o deploy automático

Faça nos **dois** repositórios (`DampCode_API` e `Damp-Code-Front`), em **Settings → Secrets and variables → Actions**:

1. **Secret** `KUBE_CONFIG`: o kubeconfig do cluster em base64.

   ```bash
   # Linux/macOS
   base64 -w0 ~/.kube/config        # macOS: base64 -i ~/.kube/config
   ```

   ```powershell
   # Windows
   [Convert]::ToBase64String([IO.File]::ReadAllBytes("$HOME\.kube\config"))
   ```

   O ideal é um kubeconfig de uma *ServiceAccount* com permissão só no namespace `dampcode`, não o de administrador do cluster.

2. **Variable** `K8S_DEPLOY` = `true`.

No próximo push na `master`/`main`, o job **Deploy no Kubernetes** aparece na aba Actions.

---

## 7. Operação: logs, rollback, migrations, backup

```bash
# Situação geral
kubectl -n dampcode get pods,svc,ingress,jobs

# Logs
kubectl -n dampcode logs deploy/dampcode-api -f
kubectl -n dampcode logs deploy/dampcode-front -f
kubectl -n dampcode logs job/dampcode-migrations

# Qual imagem está rodando
kubectl -n dampcode get deploy -o custom-columns=NOME:.metadata.name,IMAGEM:.spec.template.spec.containers[0].image

# Voltar para a versão anterior
kubectl -n dampcode rollout undo deployment/dampcode-api
kubectl -n dampcode rollout undo deployment/dampcode-front

# Reiniciar sem trocar a versão
kubectl -n dampcode rollout restart deployment/dampcode-api
```

- **Rollback da API com migration nova**: o `rollout undo` volta o código, mas **não** desfaz a migration. Como as migrations devem ser compatíveis com a versão anterior (só adicionar coluna/tabela), normalmente basta. Se não for, rode o `efbundle` da versão certa apontando para a migration anterior: `./efbundle <NomeDaMigrationAnterior>`.
- **Backup do PostgreSQL**:

  ```bash
  kubectl -n dampcode exec statefulset/postgres -- pg_dump -U dampcode -d dampcode > backup-$(date +%F).sql
  ```

- **Backup do MongoDB**:

  ```bash
  kubectl -n dampcode exec statefulset/mongo -- sh -c 'mongodump --username dampcode --password "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin --db damp_code --archive' > mongo-$(date +%F).archive
  ```

---

## 8. Problemas comuns

| Sintoma | Causa provável | Como resolver |
|---------|----------------|---------------|
| Pod em `ImagePullBackOff` | Pacote do GHCR privado sem `ghcr-pull`, ou imagem `:latest` ainda não publicada | Seção 5.2; confira na aba Actions se o job **Imagem Docker** rodou na master/main. |
| Pod em `CreateContainerConfigError` | Secret `dampcode-secrets` não existe ou falta uma chave | Seção 5.1 (`kubectl -n dampcode describe pod <pod>` mostra qual). |
| Bancos em `Pending` | Cluster sem StorageClass padrão | `kubectl get storageclass`; defina uma como padrão. |
| Job de migrations falha com `28P01` | Senha do Secret diferente da gravada no volume do PostgreSQL | Use a senha original no Secret, ou apague o PVC `data-postgres-0` (**perde os dados**). |
| API em `CrashLoopBackOff` com `Configure Jwt:Secret...` | `jwt-secret` vazio ou curto | Recrie o Secret com 32+ caracteres e `kubectl rollout restart`. |
| Front abre, mas toda chamada dá 502 | nginx não alcança a API | `kubectl -n dampcode get endpoints dampcode-api`; veja os logs da API. |
| Upload de logo/foto dá 413 | Limite de tamanho do corpo | Já está em 10 MB no nginx do front e no Ingress (`proxy-body-size`). Se o Ingress Controller não for o nginx, ajuste o equivalente. |
| Navegador avisa de certificado inválido no domínio | Certificado do Let's Encrypt ainda não emitido | `kubectl -n dampcode describe certificate dampcode-tls`; confira o DNS e o `ClusterIssuer` (seção 5.4). |
| Ingress não responde pelo IP | O Ingress só atende o `host` configurado | Acesse pelo domínio, ou teste com `port-forward` (seção 5.3). |
| Compose: `required variable JWT_SECRET is missing a value` | `.env` ausente ou sem `JWT_SECRET` | `cp .env.example .env` e preencha. |
| Compose: `migrations` sai com erro e a API não sobe | Banco com senha antiga no volume | `docker compose down -v` e suba de novo (apaga os dados locais do compose). |
| Deploy no Actions falha em "Acesso ao cluster" | `KUBE_CONFIG` ausente ou não está em base64 | Seção 6. |
