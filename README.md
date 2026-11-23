# DAMPCode-Base
Repositório central e fonte única de documentação da organização **DAMP-Code**: visão do produto, requisitos, regras de negócio, design, arquitetura e o passo a passo para instalar, configurar e rodar a **DampCode** em qualquer máquina.

# DampCode
**Projeto para aprendizado através de Hackathons**

<div>
    <span style="font-size:30px">Centro Paula Souza</span><br>
    <span style="font-size:30px">Faculdade de Tecnologia de Jahu</span><br>
    <h2>Curso de Tecnologia em Desenvolvimento de Software Multiplataforma</h2>
    <h2>Início: 1º Semestre de 2026</h2>
</div>
Documento da aplicação web

---

## 🚀 Comece por aqui

| Preciso... | Documento |
|------------|-----------|
| Instalar e rodar tudo numa máquina nova (API, front, bancos, `.env`, segredos) | **[Instalação em uma máquina nova](./docs/instalacao.md)** |
| Saber onde fica cada configuração, segredo e variável de ambiente | [Configuração e segredos](./docs/configuracao-e-segredos.md) |
| Entender o fluxo do time: branches, commits, migrations, testes, PRs | [Processos do time](./docs/processos.md) |
| Ver como front, API e bancos se conectam, tabelas, rotas e páginas | [Arquitetura](./docs/arquitetura.md) |
| Rodar tudo em containers, entender o CI/CD e publicar no Kubernetes | [Docker, CI/CD e Kubernetes](./docs/deploy-kubernetes.md) |

**Resumo rápido** (detalhes e solução de problemas na [instalação](./docs/instalacao.md)):

Front sozinho, com dados simulados (só precisa de Git e Node 20.19+):

```bash
git clone https://github.com/DAMP-Code/Damp-Code-Front.git
cd Damp-Code-Front
npm ci
npm run dev        # http://localhost:5173 — contas de teste com senha 123456
```

API (precisa de .NET SDK 10, PostgreSQL 13+ e MongoDB rodando):

```bash
git clone https://github.com/DAMP-Code/DampCode_API.git
cd DampCode_API/DampCode_API
dotnet user-secrets set "ConnectionStrings:PostgreSql" "Host=localhost;Port=5432;Database=dampcode_dev;Username=postgres;Password=<sua-senha>"
dotnet user-secrets set "Jwt:Secret" "<texto aleatório com 32+ caracteres>"
cd ..
dotnet tool restore
dotnet ef database update --project DampCode_API --startup-project DampCode_API
dotnet dev-certs https --trust
dotnet run --project DampCode_API --launch-profile https   # https://localhost:7092/swagger
```

Para o front usar a API real, crie `Damp-Code-Front/.env.development.local` com `VITE_USE_MOCK=false` e reinicie o Vite.

Tudo em containers (só precisa de Docker; os três repositórios clonados lado a lado):

```bash
cd DAMPCode-Base
cp .env.example .env          # preencha JWT_SECRET
docker compose up --build     # http://localhost:8080
```

---

## 📖 Sumário
- [Descrição da Aplicação Web](#-descrição-da-aplicação-web)  
- [Objetivos](#-objetivos)  
- [Documento de Requisitos](#-documento-de-requisitos)  
- [Regras de Negócio](#-regras-de-negócio)  
- [Design](#-design)  
- [Modelo de Navegação](#-modelo-de-navegação)  
- [Prototipagem](#️-prototipagem)  
- [Aplicação](#-aplicação)  
- [Considerações Finais](#-considerações-finais)  

---

## 🌐 Descrição da Aplicação Web

### 1.1 Descrição
Este projeto consiste no desenvolvimento de uma **aplicação web para programadores**, dinâmica e interativa através de Hackathons.  
Principais funcionalidades:  
- Permitir ao usuário a criação e a participação em Hackathons.  
- **Cadastro de usuários** e integração com a rede e a comunidade.
- **Cadastro de empresas** para elaboração de processos seletivos através dos Hackathons.  
- Diferencial: **personalização elevada** dos Hackathons e conexão direta **com o mercado de trabalho**, ampliando o repertório do usuário e auxiliando empresas.  

### 1.2 Tecnologias e Métodos Utilizados

**Front-End**
- **React 19 + Vite 8** → interfaces responsivas e dinâmicas
- **Tailwind CSS 4 + shadcn/ui** → estilo e componentes
- **React Router 7**, **ApexCharts** (gráficos), **GSAP/Motion** (animações)

**Back-End**
- **ASP.NET Core 8 (C#)** → API REST com autenticação **JWT**
- **PostgreSQL** (Entity Framework Core) → dados principais
- **MongoDB** → código das entregas dos participantes
- **xUnit** → testes automatizados

**Infraestrutura**
- **Docker** → imagens da API e do front, **Docker Compose** para rodar tudo localmente
- **GitHub Actions** → CI (testes, build, imagens) e CD
- **Kubernetes** → deploy com Kustomize, imagens no GitHub Container Registry

**Processo**
- **Figma** → prototipagem e design de telas  
- **Scrum** → metodologia ágil baseada em sprints curtos e iterativos  
- **Git + GitHub** → branches por funcionalidade e Pull Requests ([Processos do time](./docs/processos.md))

Detalhes e versões em [Arquitetura](./docs/arquitetura.md#-tecnologias).

### 1.3 Cronograma do Projeto
📌 O cronograma detalhado está disponível em:  
👉 **[DampCode | Trello](#)**  

---

## 🎯 Objetivos

### 2.1 Geral
Oferecer um ambiente que incentive o **aprendizado dinâmico e ativo** através dos Hackathons, fortalecendo o repertório e proporcionando **experiência prática ao usuário**.  

Recursos adicionais:  
- Níveis, experiência e ranking para promover a competitividade 
- Criação personalizada dos Hackathons  
- Contato com as empresas e suas necessidades  
- Contato com uma comunidade ativa 

---

## 📑 Documento de Requisitos

Um **documento de requisitos de sistema** registra as especificações que o sistema deve atender, servindo como guia para a equipe e os stakeholders.  

### 3.1 ✅ Requisitos Funcionais (RF)

| Código | Descrição |
|--------|------------|
| RF01 | Cadastrar usuário |
| RF02 | Login e logout de usuários |
| RF03 | Gerenciar perfil do usuário |
| RF04 | Pesquisar Hackathons |
| RF05 | Exibir Hackathons |
| RF06 | Criação de Hackathons personalizada |
| RF07 | Finalizar Hackathons |
| RF08 | Acompanhar progresso no perfil |
| RF09 | Gerenciar Hackathons |
| RF10 | Visualizar comunidade |
| RF11 | Interação com a comunidade |
| RF12 | Página institucional |
| RF13 | Página de contato |
| RF14 | Página “Quem somos” |
| RF15 | Gamificação do usuário |
| RF16 | Administração de Hackathons e usuários |
| RF17 | Poder verificar Hackathons concluídos e seus vencedores |

---

### 3.2 ⚙️ Requisitos Não Funcionais (RNF)

| Código | Descrição |
|--------|------------|
| RNF01 | Usabilidade (site intuitivo e responsivo) |
| RNF02 | Desempenho (carregamento rápido) |
| RNF03 | Acessibilidade (WCAG, navegação por teclado, etc.) |
| RNF04 | Compatibilidade (multiplataforma e navegadores) |
| RNF05 | Segurança (criptografia, HTTPS, autenticação) |
| RNF06 | Manutenibilidade (código modular e documentado) |
| RNF07 | Escalabilidade |
| RNF08 | Reutilização de componentes |
| RNF09 | Alta disponibilidade |
| RNF10 | Backup e recuperação de dados |

---

### 3.3 📌 Casos de Uso
*(Inserir tabela/diagrama de casos de uso)*  

Atores do sistema hoje: **participante**, **empresa** (e seus funcionários) e **admin** (verifica empresas). Os papéis e o que cada um acessa estão em [Processos do time › Contas e papéis](./docs/processos.md#11-contas-e-papéis).

---

## 📊 Regras de Negócio

### 4.1 O que será realizado?
- Usuários podem criar e participar de Hackathons, salvar suas participações recentes e divulgar suas capacidades.  
- Plataforma de aprendizado tecnológico intuitiva, através de prática e competitividade.

Regras já implementadas no sistema:
- **Empresas** nascem com verificação pendente e só **publicam** Hackathons depois de aprovadas pelo **Admin**.
- O Hackathon passa por **Rascunho → Publicado → Ativo → Em avaliação → Encerrado** (ou **Cancelado**); depois de publicado, as fases seguem as datas definidas pela empresa. Ver [Arquitetura › Ciclo de vida](./docs/arquitetura.md#-ciclo-de-vida-do-hackathon).
- Participação **individual** ou em **equipe**, com convites e pedidos para entrar.
- Entrega de **código** com histórico de versões; a empresa compara com uma solução de referência (com sugestão automática por semelhança).
- A empresa dá nota (0–100), escolhe os vencedores, define **prêmios por posição** e publica o resultado, que fica visível no ranking público.
- A empresa pode publicar **módulos de estudo** (texto, link ou vídeo) dentro do Hackathon.

### Parcerias-Chave
- Patrocinador PJ 
- Empresas de tecnologia
- Comunidade de Hackathons  
- Serviços de hospedagem
- Donos de cursos  

### Atividades-Chave
- Desenvolvimento e manutenção  
- Curadoria de conteúdo  
- Marketing digital 
- Parcerias 
- Análise de dados do usuário 

### Proposta de Valor
- Plataforma de aprendizado intuitiva e prática
- Usuários podem criar e participar de Hackathons para divulgar suas capacidades  
- Integração com o mercado de trabalho e processos seletivos  
- Comunidade ativa com engajamento contínuo através de novos desafios e propostas

### Segmentos de Clientes
- Estudantes de tecnologia  
- Programadores buscando experiência prática 
- Pessoas que buscam integração com o mercado   
- Empresas que precisam de mão de obra em tecnologia

### Canais
- Site Web
- Aplicação Mobile  

### Relacionamento com o Cliente
- Personalização por perfil
- Área do usuário com suas tecnologias e preferências
- Comunidade ativa nos Hackathons
- Gamificação da UX 

### Fontes de Receita
- Publicidade (Ads personalizados dentro da plataforma) 
- Parcerias com empresas  
- Planos premium com recursos avançados
- Espaço para empresas e semelhantes venderem e-books e cursos

### Recursos-Chave
- Plataforma Web  
- Banco de dados de tecnologias  
- Algoritmo de recomendação  
- Sistema de cadastro e moderação

### Estruturas de Custo
- Desenvolvimento e manutenção  
- Hospedagem e servidores  
- Marketing e publicidade
- Design e produção de conteúdo 
- Suporte técnico e moderação
- Eventual equipe de parceiros

---

### 4.2 BM Canvas
- Modelo de negócios Canvas, organizando visualmente os objetivos e as necessidades do projeto.

## 🎨 Design

- **Paleta de cores:** *Tema baseado em 4 tons de roxo*
`#0B051B`
`#1F0F46`
`#170A36`
`#22114D`

- **Cores destaque:**
`#6C48C5`
`#C68FE6`
`#FFEA00`

- As cores ficam como tokens do Tailwind em `src/style.css` do front (`--color-brand`, `--color-brand-light`, `--color-highlight`...).
- **Tipografia:** *no protótipo, Dangrek (títulos) / Exo (todo o restante). No front atual, **Geist** (variável) em todo o texto.*
- **Logo:** `src/assets/DAMPCode.svg` no repositório do front; o ícone da aba fica em `public/favicon.svg`.

### 4.4 Wireframe
Wireframe disponível no Figma:  
👉 [Acessar Wireframe](https://www.figma.com/proto/tUEgq9SX1vpvqGZdSOW9Pb/Untitled?node-id=225-169&p=f&t=Lvr8pQHNGuO7iOkf-0&scaling=min-zoom&content-scaling=fixed&page-id=0%3A1)  

---

## 🧭 Modelo de Navegação
*(Inserir imagem do modelo de navegação)*  

As páginas que existem hoje, com quem pode acessar cada uma, estão em [Arquitetura › Páginas do front](./docs/arquitetura.md#-páginas-do-front).

---

## 🖌️ Prototipagem
Protótipo criado no Figma:  
👉 [Acessar Protótipo](https://www.figma.com/design/vUViGgaIlrKPADdi3aWy3O/Untitled?node-id=0-1&t=ATtDGzXCgMVPFe7U-1)  

---

## 💻 Aplicação
Repositórios do projeto no GitHub ([organização DAMP-Code](https://github.com/DAMP-Code)):  

| Repositório | Conteúdo |
|-------------|----------|
| 👉 [DampCode_API](https://github.com/DAMP-Code/DampCode_API) | API ASP.NET Core, migrations, testes e specs das funcionalidades |
| 👉 [Damp-Code-Front](https://github.com/DAMP-Code/Damp-Code-Front) | Front-end React |
| 👉 [DAMPCode-Base](https://github.com/DAMP-Code/DAMPCode-Base) | Esta documentação |

Para rodar: [Instalação em uma máquina nova](./docs/instalacao.md).

---

## 📝 Considerações Finais
Durante o desenvolvimento da **DampCode**, utilizamos metodologias ágeis e iterativas, superando o desafio da aprendizagem.  
- Personalização avançada durante a criação dos Hackathons  
- Divisão de funcionalidades para usuário comum, empresa e administrador  
- Ciclos curtos de desenvolvimento com Scrum  
- Expansão contínua de funcionalidades, com banco relacional (PostgreSQL) e NoSQL (MongoDB)

➡️ O projeto visa oferecer uma **nova abordagem no nicho de Hackathons**, incentivando o desenvolvimento e o crescimento dos usuários.
