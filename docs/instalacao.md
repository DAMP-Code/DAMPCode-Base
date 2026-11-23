# 🛠️ Instalação em uma máquina nova

Passo a passo para sair de um computador zerado e ter a **DampCode** rodando: API, front, bancos de dados e segredos.

> Para entender cada variável e segredo em detalhe, veja [Configuração e segredos](./configuracao-e-segredos.md).
> Para o dia a dia depois de instalado (branches, commits, migrations, PRs), veja [Processos do time](./processos.md).

---

## 📖 Sumário
1. [Como as peças se encaixam](#1-como-as-peças-se-encaixam)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Clonar os repositórios](#3-clonar-os-repositórios)
4. [Bancos de dados](#4-bancos-de-dados)
5. [Segredos da API (User Secrets)](#5-segredos-da-api-user-secrets)
6. [Criar as tabelas (migrations)](#6-criar-as-tabelas-migrations)
7. [Rodar a API](#7-rodar-a-api)
8. [Rodar o front](#8-rodar-o-front)
9. [Primeiro uso com a API de verdade](#9-primeiro-uso-com-a-api-de-verdade)
10. [Checklist final](#10-checklist-final)
11. [Problemas comuns](#11-problemas-comuns)

---

## 1. Como as peças se encaixam

```
Navegador
   │
   ▼
Front (React + Vite) ── http://localhost:5173
   │  fetch para https://localhost:7092/api/...
   ▼
API (ASP.NET Core 8) ── https://localhost:7092  (e http://localhost:5062)
   ├──► PostgreSQL :5432   → contas, empresas, Hackathons, equipes, avaliações...
   └──► MongoDB    :27017  → código de cada versão das entregas
```

Existem **três jeitos** de rodar:

| Modo | O que precisa | Quando usar |
|------|---------------|-------------|
| **A. Só o front, com mock** | Git + Node.js | Mexer em tela/visual. O front simula a API no navegador (`VITE_USE_MOCK=true`, padrão). Leva uns 5 minutos. |
| **B. Completo** | Tudo da seção 2 | Mexer na API, testar integração de verdade, migrations. É o modo deste guia. |
| **C. Tudo em containers** | Git + Docker Desktop | Ver o sistema inteiro rodando "como em produção" sem instalar .NET nem bancos: `docker compose up --build` no DAMPCode-Base. Ver [Docker, CI/CD e Kubernetes](./deploy-kubernetes.md#3-rodar-tudo-com-docker-compose). Não serve para desenvolver (precisa reconstruir a cada mudança). |

Se for só o modo A, faça as seções [2](#2-pré-requisitos) (Git e Node), [3](#3-clonar-os-repositórios) e [8](#8-rodar-o-front).

---

## 2. Pré-requisitos

| Ferramenta | Versão | Para quê | Conferir com |
|------------|--------|----------|--------------|
| [Git](https://git-scm.com/downloads) | recente | clonar e versionar | `git --version` |
| [Node.js](https://nodejs.org/) | **20.19+ ou 22.12+** (exigência do Vite 8). Testado com 24. | front | `node -v` |
| [.NET SDK 10](https://dotnet.microsoft.com/download) | 10.0.x | compilar/rodar a API e rodar os testes | `dotnet --list-sdks` |
| [.NET SDK 8](https://dotnet.microsoft.com/download/dotnet/8.0) | 8.0.x (opcional) | rodar a API no runtime 8 original | `dotnet --list-sdks` |
| [PostgreSQL](https://www.postgresql.org/download/) | **13 ou mais novo** (uma migration usa `gen_random_uuid()`). Testado com 18. | banco principal | `psql --version` |
| [MongoDB Community](https://www.mongodb.com/try/download/community) | atual. Testado com 8. | código das entregas | `mongod --version` |

Por que só o SDK 10 já basta: a API é `net8.0`, mas o `DampCode_API.csproj` tem `<RollForward>Major</RollForward>`, então ela (e o `dotnet ef`) rodam no runtime 10. O projeto de testes é `net10.0` e **exige** o SDK 10.

Opcionais que ajudam: **pgAdmin** (vem com o instalador do PostgreSQL), **MongoDB Compass**, **VS Code** ou **Visual Studio 2022+/Rider**.

### Instalando no Windows

- **PostgreSQL**: no instalador, anote a senha do usuário `postgres` (ela vai na connection string) e mantenha a porta `5432`. Ele fica como serviço do Windows e sobe sozinho.
- **MongoDB**: marque "Install MongoD as a Service". Porta padrão `27017`, sem usuário/senha. Também sobe sozinho.
- Depois de instalar Node e .NET, **feche e abra o terminal** para o `PATH` atualizar.

### Alternativa: bancos no Docker

Se preferir não instalar os bancos, com [Docker](https://www.docker.com/) rodando:

```bash
docker run -d --name dampcode-postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 -v dampcode-pg:/var/lib/postgresql/data postgres:17
docker run -d --name dampcode-mongo -p 27017:27017 -v dampcode-mongo:/data/db mongo:7
```

Nesse caso a senha do `postgres` é `postgres`. Para subir de novo depois de reiniciar: `docker start dampcode-postgres dampcode-mongo`.
Se já houver um PostgreSQL ou MongoDB instalado na máquina, as portas vão conflitar: use um ou outro.

---

## 3. Clonar os repositórios

Todos ficam na organização [DAMP-Code](https://github.com/DAMP-Code). Sugestão de pasta:

```bash
mkdir DAMPCODE
cd DAMPCODE
git clone https://github.com/DAMP-Code/DampCode_API.git
git clone https://github.com/DAMP-Code/Damp-Code-Front.git
git clone https://github.com/DAMP-Code/DAMPCode-Base.git
```

| Repositório | Branch principal | Conteúdo |
|-------------|------------------|----------|
| `DampCode_API` | `master` | API ASP.NET Core + testes |
| `Damp-Code-Front` | `main` | Front React |
| `DAMPCode-Base` | `main` | Esta documentação |

---

## 4. Bancos de dados

### PostgreSQL

Não precisa criar nada à mão: o comando de migrations da [seção 6](#6-criar-as-tabelas-migrations) **cria o banco** `dampcode_dev` se ele não existir (o usuário `postgres` tem permissão para isso).

Se preferir criar antes (ou usar outro usuário), no pgAdmin ou com:

```bash
createdb -U postgres dampcode_dev
```

> No Windows o `createdb`/`psql` ficam em `C:\Program Files\PostgreSQL\<versão>\bin` e normalmente não estão no `PATH`.
> Use o caminho completo, por exemplo `& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres` no PowerShell.

### MongoDB

Também não precisa criar nada. A API usa `mongodb://localhost/damp_code` (definido em `appsettings.json`): o banco `damp_code` e a coleção `SubmissionVersions` nascem na primeira entrega salva.

Se o seu MongoDB tiver usuário e senha ou estiver em outro endereço, sobrescreva a connection string nos User Secrets (próxima seção) em vez de editar o `appsettings.json`.

> A API **inicia sem o MongoDB**, mas salvar ou abrir uma entrega vai falhar. Se as entregas derem erro 500, confira se o serviço do MongoDB está rodando.

---

## 5. Segredos da API (User Secrets)

A API tem **dois segredos obrigatórios** e eles **nunca** vão para o repositório. Em desenvolvimento usamos o [User Secrets](https://learn.microsoft.com/aspnet/core/security/app-secrets) do .NET, que grava um arquivo fora da pasta do projeto, só na sua máquina.

Entre na pasta do projeto da API (a que tem o `.csproj`):

```bash
cd DampCode_API/DampCode_API
```

**1) Connection string do PostgreSQL** (troque `<sua-senha>` pela senha do usuário `postgres`):

```bash
dotnet user-secrets set "ConnectionStrings:PostgreSql" "Host=localhost;Port=5432;Database=dampcode_dev;Username=postgres;Password=<sua-senha>"
```

**2) Chave de assinatura do JWT** — um texto aleatório com **pelo menos 32 caracteres**. Para gerar um (funciona em qualquer sistema, já que o Node está instalado):

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('base64'))"
```

Copie o resultado e grave:

```bash
dotnet user-secrets set "Jwt:Secret" "<cole-aqui-o-texto-gerado>"
```

**3) (Opcional) MongoDB com senha ou em outro endereço:**

```bash
dotnet user-secrets set "ConnectionStrings:DbConnection" "mongodb://usuario:senha@localhost:27017/damp_code?authSource=admin"
```

Confira o que ficou gravado:

```bash
dotnet user-secrets list
```

Onde o arquivo fica (caso precise apagar ou editar à mão):

| Sistema | Caminho |
|---------|---------|
| Windows | `%APPDATA%\Microsoft\UserSecrets\DampCode_API-59e4d42a-d7ba-4372-bdf2-68b6902e4314\secrets.json` |
| Linux/macOS | `~/.microsoft/usersecrets/DampCode_API-59e4d42a-d7ba-4372-bdf2-68b6902e4314/secrets.json` |

Cada pessoa do time tem **seus próprios** segredos: não copie o `secrets.json` de outra máquina nem mande senha pelo grupo. A chave do JWT pode ser diferente em cada computador; o único efeito é que um token gerado numa máquina não vale na outra.

---

## 6. Criar as tabelas (migrations)

Na **raiz** do repositório da API (a pasta que tem o `DampCode_API.slnx`):

```bash
cd DampCode_API
dotnet tool restore
dotnet ef database update --project DampCode_API --startup-project DampCode_API
```

- `dotnet tool restore` instala o `dotnet-ef` na versão fixada em `dotnet-tools.json` (8.0.11). Só precisa uma vez por clone.
- O `dotnet ef` sobe a API por baixo para achar o `DbContext`, por isso **os segredos da seção 5 precisam estar gravados antes**.
- No fim deve aparecer `Done.`. Hoje são 8 migrations, de `InitialCore` a `AccountAvatar`. Para ver a lista: `dotnet ef migrations list --project DampCode_API --startup-project DampCode_API`.

> Se aparecer erro dizendo que o framework `Microsoft.NETCore.App 8.0` não foi encontrado, rode antes
> `$env:DOTNET_ROLL_FORWARD="Major"` (PowerShell) ou `export DOTNET_ROLL_FORWARD=Major` (bash), ou instale o runtime do .NET 8.

---

## 7. Rodar a API

### Certificado HTTPS de desenvolvimento (uma vez por máquina)

O front chama a API em **https**://localhost:7092, então o certificado local precisa ser confiável:

```bash
dotnet dev-certs https --trust
```

Aceite a janela que aparecer. Depois disso, feche e abra o navegador.

### Subindo

Na raiz do repositório da API:

```bash
dotnet run --project DampCode_API --launch-profile https
```

Deve aparecer:

```
Now listening on: https://localhost:7092
Now listening on: http://localhost:5062
```

Abra **https://localhost:7092/swagger** para ver e testar todas as rotas.

> ⚠️ **Use sempre `--launch-profile https`.** Sem ele, o `dotnet run` pega o primeiro perfil do `launchSettings.json` (`http`),
> que sobe só em `http://localhost:5062`, e o front não encontra a API.
> No Visual Studio/Rider, escolha o perfil **https** antes de dar play.

### Testando o login no Swagger

1. `POST /api/Auth/register/participante` (ou faça login com uma conta existente em `POST /api/Auth/login`) e copie o campo `token`.
2. Clique em **Authorize** e cole **só o token**, sem `Bearer `.
3. Chame `GET /api/Auth/me`.

### Testes automatizados

```bash
dotnet test DampCode_API.Tests
```

Os testes **não precisam** de PostgreSQL nem MongoDB (usam banco em memória). Todos devem passar.

---

## 8. Rodar o front

Na pasta do front:

```bash
cd Damp-Code-Front
npm ci
npm run dev
```

- `npm ci` instala exatamente as versões do `package-lock.json`. Use `npm install` só quando for **adicionar** uma dependência.
- Abra **http://localhost:5173**.

### Mock ou API de verdade?

O arquivo `.env.development` (versionado) vem com:

```env
VITE_USE_MOCK=true
```

| `VITE_USE_MOCK` | Comportamento |
|-----------------|---------------|
| `true` (padrão) | O front intercepta as chamadas e simula a API no navegador, com dados guardados no `localStorage`. Não precisa de API nem banco. A tela de login mostra a caixa **"Contas de teste"**. |
| `false` | O front chama a API real em `https://localhost:7092`. |

**Para usar a API real sem alterar o arquivo versionado**, crie na raiz do front o arquivo `.env.development.local`:

```env
VITE_USE_MOCK=false
```

Ele tem prioridade sobre o `.env.development` e já é ignorado pelo Git (regra `*.local` do `.gitignore`). **Reinicie o Vite** (`Ctrl+C` e `npm run dev` de novo) sempre que mudar um `.env`.

### Contas do mock

Todas com senha `123456`:

| E-mail | Tipo |
|--------|------|
| `ana@dampcode.dev` | participante |
| `bruno@dampcode.dev` | participante |
| `carla@dampcode.dev` | participante |
| `diego@dampcode.dev` | participante |
| `ifood@dampcode.dev` | empresa iFood |
| `nubank@dampcode.dev` | empresa Nubank |
| `admin@dampcode.dev` | admin (verifica empresas) |

Para zerar os dados do mock: DevTools (`F12`) → **Application** → **Local Storage** → `http://localhost:5173` → apague a chave `damp-mock-db` e recarregue.

### Outros comandos

| Comando | Para quê |
|---------|----------|
| `npm run build` | gera a versão de produção em `dist/` (o mock nunca entra no build) |
| `npm run preview` | serve o `dist/` localmente |
| `npm run lint` | ESLint |

> ⚠️ O front **precisa** rodar na porta **5173**: é a única origem liberada no CORS da API.
> Se o Vite disser que subiu na 5174, outro processo está usando a 5173 — feche-o.
> Para garantir: `npm run dev -- --port 5173 --strictPort`.

---

## 9. Primeiro uso com a API de verdade

O banco começa vazio, sem nenhuma conta.

1. Com a API (seção 7) e o front com `VITE_USE_MOCK=false` (seção 8) rodando, acesse `http://localhost:5173/escolha` e crie:
   - uma conta de **participante**;
   - uma conta de **empresa** (ela nasce com verificação pendente).
2. **Criar um Admin.** Não existe cadastro de Admin pela tela nem pela API. Cadastre uma conta comum e promova direto no banco, com o e-mail **em maiúsculas**:

   ```sql
   UPDATE accounts SET account_type = 'Admin' WHERE normalized_email = 'ADMIN@EXEMPLO.COM';
   ```

   No PowerShell, por exemplo:

   ```powershell
   & "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -d dampcode_dev -c "UPDATE accounts SET account_type = 'Admin' WHERE normalized_email = 'ADMIN@EXEMPLO.COM';"
   ```

3. **Saia e entre de novo** com essa conta: o papel vem no token, então o token antigo continua sem permissão de Admin.
4. Em `http://localhost:5173/admin`, aprove a empresa criada no passo 1. **Sem essa aprovação a empresa cria rascunhos, mas não consegue publicar** ("Sua empresa precisa ser verificada para publicar.").
5. Entre como empresa, crie e publique um Hackathon (a data de início precisa ser no futuro); entre como participante e se inscreva.

---

## 10. Checklist final

- [ ] `dotnet test DampCode_API.Tests` → todos aprovados
- [ ] `dotnet user-secrets list` (em `DampCode_API/DampCode_API`) mostra `ConnectionStrings:PostgreSql` e `Jwt:Secret`
- [ ] `dotnet ef database update ...` terminou com `Done.`
- [ ] `https://localhost:7092/swagger` abre sem aviso de certificado
- [ ] `npm run dev` sobe em `http://localhost:5173`
- [ ] Modo mock: login com `ana@dampcode.dev` / `123456` funciona
- [ ] Modo API real: cadastro de uma conta nova funciona e aparece em `GET /api/Auth/me`

---

## 11. Problemas comuns

| Sintoma | Causa provável | Como resolver |
|---------|----------------|---------------|
| API não sobe: `Configure Jwt:Secret com pelo menos 32 bytes por User Secrets ou pela variável Jwt__Secret.` | Segredo do JWT ausente ou curto | Seção 5, passo 2. Confira se rodou o `user-secrets set` dentro de `DampCode_API/DampCode_API`. |
| `Configure a connection string 'PostgreSql' por User Secrets ou pela variável ConnectionStrings__PostgreSql.` (na primeira requisição ou no `dotnet ef`) | Connection string ausente | Seção 5, passo 1. |
| Erro 500 e no log `28P01` (autenticação por senha falhou) | Senha errada na connection string | Grave de novo com a senha certa do `postgres`. |
| Erro 500 e no log `3D000` (banco não existe) ou `42P01` (relação não existe) | Migrations não aplicadas | Seção 6. Depois de todo `git pull` que trouxer migration nova, rode de novo. |
| Front mostra "Failed to fetch" / `ERR_CONNECTION_REFUSED` | API parada ou rodando sem `--launch-profile https` | Seção 7. |
| Erro de certificado (`ERR_CERT_AUTHORITY_INVALID`) ou fetch falhando só no navegador | Certificado de desenvolvimento não confiável | `dotnet dev-certs https --trust` e reinicie o navegador. Se persistir, abra `https://localhost:7092/swagger` uma vez e aceite. |
| Erro de CORS no console | Front fora da porta 5173 | Feche o que estiver na 5173 e suba o Vite nela. |
| Login com conta criada no banco não funciona / ainda aparece "Contas de teste" | Front em modo mock | Crie o `.env.development.local` com `VITE_USE_MOCK=false` e **reinicie** o Vite. |
| Entregas dão erro 500 (timeout do Mongo no log) | Serviço do MongoDB parado | Inicie o serviço `MongoDB` (Windows: `services.msc`) ou o container. |
| `dotnet ef` reclama que falta o .NET 8 | Só há runtime mais novo e o roll-forward não pegou | `DOTNET_ROLL_FORWARD=Major` (ver seção 6) ou instale o runtime 8. |
| `dotnet test` falha ao compilar com erro de `net10.0` | SDK 10 não instalado | Instale o .NET SDK 10. |
| Classe nova do Tailwind não aplica em arquivo novo | Vite ainda não enxergou o arquivo | Reinicie o `npm run dev`. |
| Ícone/aba de outro projeto aparece no `localhost:5173` | Cache do navegador de outro projeto que usou a mesma porta | `Ctrl+F5` ou limpe o cache do site. |
| Vite recusa iniciar por versão do Node | Node antigo | Atualize para 20.19+ ou 22.12+. |
