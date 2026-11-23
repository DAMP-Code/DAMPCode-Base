# 🔐 Configuração e segredos

Referência de **tudo** que a API e o front leem de configuração: o que é segredo, onde cada valor fica e como trocar em desenvolvimento e em produção.

> Passo a passo de instalação: [Instalação em uma máquina nova](./instalacao.md).

---

## 📖 Sumário
- [Regras de ouro](#-regras-de-ouro)
- [API](#-api)
  - [Chaves de configuração](#chaves-de-configuração)
  - [De onde a API lê (ordem de prioridade)](#de-onde-a-api-lê-ordem-de-prioridade)
  - [User Secrets (desenvolvimento)](#user-secrets-desenvolvimento)
  - [Variáveis de ambiente (produção)](#variáveis-de-ambiente-produção)
  - [Secrets do GitHub Actions](#secrets-do-github-actions)
  - [Portas e perfis](#portas-e-perfis)
- [Front](#-front)
  - [Arquivos .env](#arquivos-env)
  - [O que fica no navegador](#o-que-fica-no-navegador)
- [Valores padrão no código](#-valores-padrão-no-código)
- [Trocar ou vazar um segredo](#-trocar-ou-vazar-um-segredo)

---

## ✅ Regras de ouro

1. **Nenhum segredo no Git.** Senhas e chaves ficam em User Secrets (desenvolvimento) ou variáveis de ambiente (produção). O `appsettings.json` só guarda valores que podem ser públicos.
2. **Nada secreto no front.** Toda variável `VITE_*` vai parar no JavaScript que o navegador baixa: qualquer pessoa consegue ler.
3. **Cada pessoa tem os seus segredos.** Não compartilhe o `secrets.json`, a senha do seu PostgreSQL nem a chave do JWT por chat ou commit.
4. Arquivos que **não** podem ser commitados: `secrets.json`, `.env.*.local`, qualquer `appsettings.*.json` com senha.

---

## 🔧 API

### Chaves de configuração

| Chave | Obrigatória | Segredo | Onde fica por padrão | Valor de desenvolvimento |
|-------|:-----------:|:-------:|----------------------|--------------------------|
| `ConnectionStrings:PostgreSql` | ✅ | ✅ | User Secrets | `Host=localhost;Port=5432;Database=dampcode_dev;Username=postgres;Password=<senha>` |
| `Jwt:Secret` | ✅ | ✅ | User Secrets | texto aleatório com **≥ 32 bytes** |
| `ConnectionStrings:DbConnection` (MongoDB) | ✅ | ⚠️ se tiver senha | `appsettings.json` | `mongodb://localhost/damp_code` |
| `Jwt:Issuer` | ✅ | — | `appsettings.json` | `Api` |
| `Jwt:Audience` | ✅ | — | `appsettings.json` | `Front` |
| `Jwt:ExpirationMinutes` | — | — | padrão no código | `1440` (24h). Aceita de 1 a 43200. |
| `Cors:Origins` (lista) | — | — | padrão no código | `["http://localhost:5173"]` quando não configurado |
| `ASPNETCORE_ENVIRONMENT` | — | — | `launchSettings.json` | `Development` |
| `Logging:LogLevel:*` | — | — | `appsettings*.json` | `Information` / `Warning` para `Microsoft.AspNetCore` |

O que acontece quando falta algo:

| Falta | Efeito |
|-------|--------|
| `Jwt:Secret` (ou menor que 32 bytes) | A API **não inicia**: `Configure Jwt:Secret com pelo menos 32 bytes por User Secrets ou pela variável Jwt__Secret.` |
| `ConnectionStrings:PostgreSql` | A API inicia, mas a primeira requisição que usa o banco (e o `dotnet ef`) falha com `Configure a connection string 'PostgreSql'...` |
| MongoDB fora do ar | A API inicia; só salvar/abrir código de entrega falha. |
| `ASPNETCORE_ENVIRONMENT` diferente de `Development` | Swagger desligado **e** User Secrets não são carregados. |

### De onde a API lê (ordem de prioridade)

O ASP.NET Core junta várias fontes; a de baixo vence a de cima:

1. `appsettings.json`
2. `appsettings.{Ambiente}.json` (ex.: `appsettings.Development.json`)
3. **User Secrets** — só quando o ambiente é `Development`
4. **Variáveis de ambiente**
5. Argumentos de linha de comando (`dotnet run -- --Jwt:Secret=...`)

Por isso dá para sobrescrever o MongoDB do `appsettings.json` pelos User Secrets sem editar arquivo versionado.

### User Secrets (desenvolvimento)

O `DampCode_API.csproj` já tem o `UserSecretsId` (`DampCode_API-59e4d42a-d7ba-4372-bdf2-68b6902e4314`), então basta usar os comandos **dentro de `DampCode_API/DampCode_API`**:

```bash
dotnet user-secrets list                                   # ver o que está gravado
dotnet user-secrets set "Jwt:Secret" "<valor>"             # gravar/alterar
dotnet user-secrets remove "ConnectionStrings:DbConnection" # remover uma chave
dotnet user-secrets clear                                  # apagar tudo
```

O arquivo gerado fica em:

- Windows: `%APPDATA%\Microsoft\UserSecrets\DampCode_API-59e4d42a-d7ba-4372-bdf2-68b6902e4314\secrets.json`
- Linux/macOS: `~/.microsoft/usersecrets/DampCode_API-59e4d42a-d7ba-4372-bdf2-68b6902e4314/secrets.json`

Formato do arquivo (se quiser editar à mão):

```json
{
  "ConnectionStrings:PostgreSql": "Host=localhost;Port=5432;Database=dampcode_dev;Username=postgres;Password=<senha>",
  "Jwt:Secret": "<texto aleatório com pelo menos 32 caracteres>"
}
```

Gerar uma chave de JWT:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('base64'))"
```

### Variáveis de ambiente (produção)

Em variável de ambiente, o `:` vira `__` (dois sublinhados):

| Chave | Variável de ambiente |
|-------|----------------------|
| `ConnectionStrings:PostgreSql` | `ConnectionStrings__PostgreSql` |
| `ConnectionStrings:DbConnection` | `ConnectionStrings__DbConnection` |
| `Jwt:Secret` | `Jwt__Secret` |
| `Jwt:ExpirationMinutes` | `Jwt__ExpirationMinutes` |
| `Cors:Origins` | `Cors__Origins__0`, `Cors__Origins__1`... (uma por origem) |

Exemplo (bash):

```bash
export ASPNETCORE_ENVIRONMENT=Production
export ConnectionStrings__PostgreSql="Host=meu-servidor;Port=5432;Database=dampcode;Username=dampcode_app;Password=<senha>"
export ConnectionStrings__DbConnection="mongodb://usuario:senha@meu-mongo:27017/damp_code?authSource=admin"
export Jwt__Secret="<chave de produção, diferente da de desenvolvimento>"
```

PowerShell:

```powershell
$env:Jwt__Secret = "<chave>"
```

Em **produção** (Docker e Kubernetes) a API recebe tudo por variável de ambiente. No compose os valores vêm do `.env` do DAMPCode-Base; no cluster, do Secret `dampcode-secrets`. Detalhes em [Docker, CI/CD e Kubernetes](./deploy-kubernetes.md).

### Secrets do GitHub Actions

O CI **não usa segredo nenhum** para testar: o `ApiFactory` sobe a API em ambiente `Testing`, com banco em memória e uma chave de JWT própria. A publicação no GHCR usa o `GITHUB_TOKEN` automático. Só o deploy precisa de configuração, nos dois repositórios de código:

| Nome | Tipo | Conteúdo |
|------|------|----------|
| `KUBE_CONFIG` | Secret | kubeconfig do cluster em base64 |
| `K8S_DEPLOY` | Variable | `true` para liberar o deploy |

As senhas dos bancos e a chave do JWT de produção **não** passam pelo GitHub: ficam só no Secret do Kubernetes.

### Portas e perfis

Definidos em `DampCode_API/Properties/launchSettings.json`:

| Perfil | URLs | Como usar |
|--------|------|-----------|
| `https` ✅ | `https://localhost:7092` e `http://localhost:5062` | `dotnet run --project DampCode_API --launch-profile https` — **é o que o front espera** |
| `http` | `http://localhost:5062` | perfil padrão do `dotnet run` sem parâmetro; o front **não** funciona com ele |
| `IIS Express` | `http://localhost:22441` / `https://localhost:44327` | só Visual Studio |

O certificado do `https` local vem de `dotnet dev-certs https --trust` (uma vez por máquina).

---

## 🎨 Front

### Arquivos .env

O Vite carrega, nesta ordem (o de baixo vence):

| Arquivo | Versionado? | Uso |
|---------|:-----------:|-----|
| `.env.development` | ✅ | padrão do time: `VITE_USE_MOCK=true` |
| `.env.development.local` | ❌ (ignorado por `*.local`) | **sua** escolha local, ex.: `VITE_USE_MOCK=false` |

Variáveis existentes:

| Variável | Valores | Efeito |
|----------|---------|--------|
| `VITE_USE_MOCK` | `true` (padrão) / `false` | Com qualquer valor diferente de `"false"`, em `npm run dev`, o `src/mock/installMockApi.js` intercepta as chamadas para a API e responde com dados do `src/mock/seed.js`. No `npm run build` o mock nunca é instalado. |
| `VITE_API_URL` | ausente (padrão) / URL / vazio | Endereço da API, lido em `src/app-components/utilities/apiUrl.js`. **Ausente**: `https://localhost:7092` (desenvolvimento). **Vazio**: chamadas relativas `/api/...` — é o que a imagem Docker usa, com o nginx repassando para a API. **URL**: API em outro domínio (aí libere esse domínio em `Cors:Origins`). |

Depois de mudar um `.env`, **reinicie o Vite**.

### O que fica no navegador

| Chave do `localStorage` | Conteúdo |
|-------------------------|----------|
| `user` | Usuário logado **com o token JWT**. Sair da conta apaga. |
| `damp-mock-db` | Banco simulado do modo mock. Apague para voltar aos dados iniciais. |

---

## 📌 Valores padrão no código

Padrões que valem quando nada é configurado:

| Valor | Onde | Como mudar |
|-------|------|------------|
| Origem do CORS `http://localhost:5173` | `DampCode_API/Program.cs` (`AllowReact`) | `Cors:Origins` (ou `Cors__Origins__0`...). Em desenvolvimento o front tem que rodar nessa porta. |
| URL da API `https://localhost:7092` | `src/app-components/utilities/apiUrl.js` (todas as páginas e o mock importam `API_URL` daqui) | `VITE_API_URL` |
| Coleção do MongoDB `SubmissionVersions` | `MongoSubmissionVersionStore.cs` | só pelo código. O **banco** (`damp_code`) vem da connection string. |
| Porta dos containers `8080` | `Dockerfile` da API e do front | `ASPNETCORE_HTTP_PORTS` (API); no front, só pelo `nginx/default.conf.template` |

---

## 🚨 Trocar ou vazar um segredo

| Segredo | Se vazou / precisa trocar | Efeito da troca |
|---------|---------------------------|-----------------|
| `Jwt:Secret` | Gere outro e grave com `user-secrets set` (ou na variável de ambiente) e reinicie a API | Todos os tokens emitidos param de valer: todo mundo precisa entrar de novo. |
| Senha do PostgreSQL | Troque no banco (`ALTER USER postgres WITH PASSWORD '...'`) e atualize `ConnectionStrings:PostgreSql` | Nenhum, além de reiniciar a API. |
| Senha do MongoDB | Troque no Mongo e atualize `ConnectionStrings:DbConnection` | Nenhum, além de reiniciar a API. |

Se um segredo foi parar num commit, **trocar o valor é obrigatório**: apagar o arquivo num commit novo não tira o valor do histórico do Git.
