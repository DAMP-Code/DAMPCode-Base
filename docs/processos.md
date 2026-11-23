# 🔄 Processos do time

Como trabalhamos no dia a dia depois que a máquina está pronta: atualizar o código, criar branch, commitar, mexer no banco, testar e abrir PR.

> Ainda não instalou? Comece por [Instalação em uma máquina nova](./instalacao.md).

---

## 📖 Sumário
1. [Repositórios e branches principais](#1-repositórios-e-branches-principais)
2. [Rotina para começar o dia](#2-rotina-para-começar-o-dia)
3. [Do pedido ao PR](#3-do-pedido-ao-pr)
4. [Branches](#4-branches)
5. [Commits](#5-commits)
6. [Pull Requests](#6-pull-requests)
7. [Banco de dados e migrations](#7-banco-de-dados-e-migrations)
8. [Testes e verificações antes do PR](#8-testes-e-verificações-antes-do-pr)
9. [Mock do front](#9-mock-do-front)
10. [Specs e planos](#10-specs-e-planos)
11. [Contas e papéis](#11-contas-e-papéis)
12. [Pendências conhecidas](#12-pendências-conhecidas)

---

## 1. Repositórios e branches principais

| Repositório | Branch principal | O que tem |
|-------------|------------------|-----------|
| [`DampCode_API`](https://github.com/DAMP-Code/DampCode_API) | `master` | API, testes, specs e planos em `docs/superpowers/` |
| [`Damp-Code-Front`](https://github.com/DAMP-Code/Damp-Code-Front) | `main` | Front React |
| [`DAMPCode-Base`](https://github.com/DAMP-Code/DAMPCode-Base) | `main` | Documentação central (este repositório) |

Ninguém commita direto em `master`/`main` dos repositórios de código: tudo entra por Pull Request.

---

## 2. Rotina para começar o dia

```bash
# API
cd DampCode_API
git checkout master
git pull
dotnet ef database update --project DampCode_API --startup-project DampCode_API   # só se veio migration nova
dotnet run --project DampCode_API --launch-profile https

# Front (outro terminal)
cd Damp-Code-Front
git checkout main
git pull
npm ci        # só se o package-lock.json mudou
npm run dev
```

Como saber se precisa dos passos opcionais:

- **Migration nova**: o `git pull` trouxe arquivos em `DampCode_API/Data/Migrations/`. Na dúvida, rode o `database update`; se não houver nada novo ele só avisa que está atualizado.
- **Dependência nova**: o `git pull` alterou o `package-lock.json` (API: o `dotnet run` já restaura os pacotes NuGet sozinho).

---

## 3. Do pedido ao PR

```
pedido / tarefa
   │
   ▼
spec curta em docs/superpowers/specs (funcionalidade grande)
   │
   ▼
branch feat/... nos repositórios envolvidos
   │
   ▼
código + testes + migration (se mexer no banco) + mock (se o front consumir rota nova)
   │
   ▼
verificações da seção 8
   │
   ▼
push + PR (um por repositório) ──► revisão ──► merge na branch principal
```

Funcionalidades que mexem em API e front andam **juntas**: mesma branch com o mesmo nome nos dois repositórios e um PR em cada um, citando o outro.

---

## 4. Branches

Criadas a partir da branch principal atualizada, com nome em português e em minúsculas, separado por hífen:

| Prefixo | Uso | Exemplos reais |
|---------|-----|----------------|
| `feat/` | funcionalidade nova | `feat/inscricao-e-equipes`, `feat/ranking-dos-hackathons` |
| `fix/` | correção | `fix/responsividade`, `fix/telas-de-visitante` |
| `docs/` | só documentação | `docs/instalacao-e-processos` |

```bash
git checkout master        # main no front
git pull
git checkout -b feat/nome-da-funcionalidade
```

Quando uma funcionalidade depende de outra que ainda não entrou, a branch nova pode sair da anterior (branches empilhadas). Nesse caso, abra o PR do topo da pilha ou ajuste a base do PR depois do merge da anterior.

---

## 5. Commits

Formato: **`Tipo: descrição curta em português`**, dizendo o que a mudança faz.

| Tipo | Quando |
|------|--------|
| `Feat:` | funcionalidade ou tela nova |
| `Fix:` | correção de bug ou ajuste |
| `Docs:` | documentação, specs e planos |
| `Test:` | só testes |

Exemplos do histórico:

```
Feat: rota pública do resultado do Hackathon
Feat: módulos e conteúdos do Hackathon (modelo e migration)
Fix: salvar o Hackathon sem os campos de referência mantém a referência
Docs: design de avaliação e premiação
```

Boas práticas:

- Um commit por passo que faça sentido sozinho (modelo + migration, serviço, rotas, tela...).
- Não misture refatoração com funcionalidade no mesmo commit.
- Confira o `git status` antes: **nunca** commite `secrets.json`, `.env.*.local`, `bin/`, `obj/`, `node_modules/` ou `dist/`.

---

## 6. Pull Requests

- **Base**: `master` na API, `main` no front.
- **Título**: no mesmo padrão dos commits, curto. Ex.: `Feat: módulos e conteúdos, configurações da conta e resultado público`.
- **Descrição**: o que muda, como testar, se tem migration, e o link do PR do outro repositório quando a funcionalidade for dos dois lados.
- Antes de pedir revisão, rode as verificações da [seção 8](#8-testes-e-verificações-antes-do-pr).
- Quando a funcionalidade tem PR nos dois repositórios, faça o merge dos dois juntos para `master`/`main` não ficarem descasados.
- Depois do merge, apague a branch e volte para a principal: `git checkout master && git pull`.

---

## 7. Banco de dados e migrations

O banco principal é o **PostgreSQL**, via Entity Framework Core. O **MongoDB** guarda só o código de cada versão das entregas (coleção `SubmissionVersions`) e não tem migration.

### Criar uma migration

Depois de alterar um model em `DampCode_API/Models/` ou uma configuração em `DampCode_API/Data/Configurations/`, na raiz da API:

```bash
dotnet ef migrations add NomeEmPascalCase --project DampCode_API --startup-project DampCode_API --output-dir Data/Migrations
dotnet ef database update --project DampCode_API --startup-project DampCode_API
```

Regras:

- Nome em PascalCase descrevendo a mudança: `HackathonModules`, `AccountAvatar`.
- Commite **juntos** os três arquivos gerados: a migration, o `.Designer.cs` e o `DampCodeDbContextModelSnapshot.cs` atualizado.
- **Nunca** edite uma migration que já está em `master`: crie outra corrigindo.
- Desfazer a última migration **antes** do push: `dotnet ef migrations remove --project DampCode_API --startup-project DampCode_API` (se já aplicou, rode antes `dotnet ef database update <MigrationAnterior> ...`).
- Mantenha compatível com PostgreSQL 13+.
- Nomes de tabela e coluna seguem `snake_case` (configurado em `Data/Configurations`).

### Recriar o banco local do zero

Útil quando o banco local ficou bagunçado:

```bash
dotnet ef database drop --project DampCode_API --startup-project DampCode_API
dotnet ef database update --project DampCode_API --startup-project DampCode_API
```

Isso apaga **todos** os dados locais (inclusive a conta Admin, que precisa ser promovida de novo).

---

## 8. Testes e verificações antes do PR

### API

```bash
dotnet build
dotnet test DampCode_API.Tests
```

- Todos os testes precisam passar. Eles não usam PostgreSQL nem MongoDB: `ApiFactory` sobe a API em ambiente `Testing` com EF Core InMemory e `InMemorySubmissionVersionStore`.
- Funcionalidade nova vem com teste de serviço (`*ServiceTests.cs`) e, quando cria rota, teste de endpoint (`*EndpointTests.cs`).
- O banco em memória **não** aplica índices únicos, check constraints nem transações: regra que depende disso precisa ser conferida também no PostgreSQL local.

### Front

```bash
npm run build
npm run lint
```

- O `build` precisa passar. O aviso de *chunk* maior que 500 kB é conhecido.
- O `lint` ainda acusa erros antigos (18 em 23/11/2026). Não adicione novos: compare o número antes e depois da sua mudança.
- Teste a tela no navegador nos dois modos quando mexer em chamada à API: com mock e com a API real.
- Confira a tela também em largura de celular (DevTools → modo responsivo).

---

## 9. Mock do front

O modo mock (`VITE_USE_MOCK=true`) permite trabalhar no front sem API. Para ele continuar útil:

- Quando o front passar a consumir uma **rota nova** da API, implemente a mesma rota em `src/mock/installMockApi.js`, com o mesmo formato de resposta e os mesmos códigos de erro.
- Dados iniciais ficam em `src/mock/seed.js`. Se adicionar uma lista nova ao banco simulado, inclua a chave na migração de bancos antigos dentro de `loadDatabase()` (quem já tem `damp-mock-db` salvo recebe a lista nova sem precisar apagar).
- Contas de teste novas entram em `MOCK_ACCOUNTS` (aparecem na caixa "Contas de teste" do login).

---

## 10. Specs e planos

Funcionalidades grandes começam por um documento curto, versionado na API:

| Pasta | Nome do arquivo | Conteúdo |
|-------|-----------------|----------|
| `docs/superpowers/specs/` | `AAAA-MM-DD-tema-design.md` | o que vai ser feito, regras de negócio, decisões |
| `docs/superpowers/plans/` | `AAAA-MM-DD-tema.md` | passo a passo da implementação, em tarefas |

Exemplos: `2026-10-29-inscricao-e-equipes-design.md`, `2026-11-04-avaliacao-e-premiacao-design.md`. O commit da spec usa `Docs:`.

---

## 11. Contas e papéis

| Papel | Como nasce | Valor no token (`role`) | Área no front |
|-------|------------|-------------------------|---------------|
| Participante | cadastro em `/cadastroUser` | `participante` | `/dashboard/participante`, `/perfil` |
| Empresa | cadastro em `/cadastroEmpresa` (fica com verificação **pendente**) | `empresa` | `/dashboard/Empresa` |
| Admin | promovido direto no banco (ver [instalação, seção 9](./instalacao.md#9-primeiro-uso-com-a-api-de-verdade)) | `admin` | `/admin` (verificação de empresas) |

O papel vem dentro do token: depois de mudar o tipo de uma conta no banco, é preciso **sair e entrar de novo**. Tokens deixam de valer se a conta for suspensa, desativada, excluída ou trocar a senha.

---

## 12. Pendências conhecidas

Registradas para ninguém estranhar:

- O front tem a URL da API (`https://localhost:7092`) fixa em vários arquivos e a API libera só `http://localhost:5173` no CORS (ver [Configuração e segredos](./configuracao-e-segredos.md#-valores-fixos-no-código)).
- Não há CI nem deploy automatizado.
- O e-mail oficial nos Termos e na Política de Privacidade ainda é placeholder (`src/app-components/legal/contato.js` no front).
- O `README.md` do front ainda é a cópia antiga do documento acadêmico; a documentação atual é a deste repositório.
- O `DampCode_API.http` ainda tem o exemplo `weatherforecast` do template; use o Swagger para testar rotas.
