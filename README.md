# Sniparium

**Your personal code snippet bank — inside VS Code and on the web.**

> Available on [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ibiulabs.sniparium) · Web app at [sniparium.ibiulabs.com](https://sniparium.ibiulabs.com)

---

## Why Sniparium exists

Every developer has that folder. The one with files named `utils.txt`, `helpers_OLD.js`, `stuff_to_remember.md` — a graveyard of code fragments that were useful once and will be useful again, if only you could find them.

The workflow is always the same: you write a clever function for one project, copy it to a notes app, forget which app, rewrite it from scratch six months later. Or you keep a long Gist list that becomes impossible to navigate. Or you bookmark Stack Overflow answers that expire. The code that should save you time ends up costing it.

Sniparium was built to fix that. The idea is simple: a personal bank of code snippets that lives where you actually work — inside VS Code — and can also be reached from any browser. You write a piece of code you want to keep, select it, run a command, and it's saved. Later you search for it by title, language, or tag and paste it directly at your cursor. No context switching, no copy-pasting from another window, no re-googling what you already solved.

Beyond personal productivity, Sniparium lets you mark snippets as public and share them via a permanent URL. That means your own solutions become shareable references — for teammates, for your portfolio, or for anyone who finds them through the web app.

---

## How it was built

Sniparium is a full-stack monorepo with three independent packages: a web app, a REST API, and a VS Code extension. I built the entire thing with the help of **GitHub Copilot**, primarily using the **Claude Sonnet 4.6** and **Claude Opus 4.6** models for code generation, architecture decisions, security audits, and debugging.

### API — `sniparium-api`

The backend is a **Fastify** server written in **TypeScript**. Fastify was chosen over Express for its significantly better throughput and first-class TypeScript support with schema validation via Zod.

Authentication is handled by **Better Auth**, a self-hosted, open-source auth library that manages GitHub OAuth and session tokens. Being self-hosted means there are no per-user costs and no third-party dependency for something as critical as auth — all session data is stored in our own database.

The database is **PostgreSQL** hosted on **Neon** (serverless Postgres). Schema and queries are written with **Drizzle ORM**, a TypeScript-first ORM that generates fully typed queries from the schema definition. Full-text search across snippets is implemented natively in PostgreSQL with `tsvector` and `tsquery`, updated automatically by a database trigger on every insert and update.

The VS Code extension authenticates with the API using a Bearer token stored in `vscode.SecretStorage` (the OS keychain — Credential Manager on Windows, Keychain on macOS, libsecret on Linux). Web sessions use signed HTTP cookies managed by Better Auth.

The API is tested with **Vitest** and integration tests run against a real Neon database branch isolated from production. CI/CD is handled by **GitHub Actions**: a CI pipeline runs type checks, builds, and integration tests on every push; a CD pipeline deploys to **Render** (API) and **Vercel** (web) only after CI passes. Errors in production are captured in real time by **Sentry**. The Render free tier is kept alive with **UptimeRobot** pinging `/healthz` every 5 minutes.

### Web App — `sniparium-web`

The frontend is a **Next.js** application using the App Router. It serves two purposes: an authenticated dashboard for managing your own snippets, and public pages for shared snippets with server-side rendering for SEO.

The UI is built with **Tailwind CSS** and **shadcn/ui** — components that are copied directly into the project rather than installed as a closed dependency, giving full control over their code. The snippet creation and editing form uses **Monaco Editor** (the same editor that powers VS Code) so writing and editing code feels native. Snippet display on listing and public pages uses **Shiki** to render static syntax-highlighted HTML on the server, which is much lighter than loading Monaco for read-only code.

The web app connects to the API using **tRPC**, which creates a type-safe bridge between the Fastify backend and the Next.js frontend. This means all API calls — `trpc.snippets.list.useQuery()`, for example — are fully typed end to end, with no duplicated interface definitions.

### VS Code Extension — `sniparium-vscode`

The extension is built with the **VS Code Extension API** and compiled with **esbuild**, which is dramatically faster than the default webpack build used in VS Code extension templates. It registers four commands in the Command Palette and handles the GitHub OAuth flow through a URI handler (`vscode://ibiulabs.sniparium`), which receives the auth callback directly from the browser and stores the session token securely via `vscode.SecretStorage`.

The extension is published to the **Visual Studio Marketplace** via `vsce` and a Personal Access Token from **Azure DevOps**.

---

## Installation

### VS Code Extension

**Via the Marketplace (recommended):**

1. Open VS Code
2. Press `Ctrl+Shift+X` (or `Cmd+Shift+X` on macOS) to open the Extensions panel
3. Search for **Sniparium**
4. Click **Install**

**Direct link:**
[marketplace.visualstudio.com/items?itemName=ibiulabs.sniparium](https://marketplace.visualstudio.com/items?itemName=ibiulabs.sniparium)

**Via Quick Open:**

Press `Ctrl+P` / `Cmd+P`, paste the command below and press Enter:

```
ext install ibiulabs.sniparium
```

### Web App

No installation required. Access directly at [sniparium.ibiulabs.com](https://sniparium.ibiulabs.com).

---

## How to use

### VS Code

All Sniparium commands are available in the **Command Palette** (`Ctrl+Shift+P` / `Cmd+Shift+P`).

#### Sign in

1. Open the Command Palette
2. Run **`Sniparium: Search Snippet`** or **`Sniparium: Save Selection`**
3. If you are not signed in, a browser window will open to authorize with GitHub
4. After authorization, click **Open VS Code** in the browser — the extension connects automatically

#### Save a snippet

1. Select any code in the editor
2. Open the Command Palette → **`Sniparium: Save Selection`**
3. Fill in the title, language, description (optional), and tags (optional)
4. Press Enter — the snippet is saved to your account

#### Search and insert a snippet

1. Open the Command Palette → **`Sniparium: Search Snippet`**
2. Type to search by title, language, or tag
3. Select a snippet from the list
4. The snippet is pasted at your cursor position

#### Other commands

| Command                      | Description                              |
| ---------------------------- | ---------------------------------------- |
| `Sniparium: Save Selection`  | Save selected code as a new snippet      |
| `Sniparium: Search Snippet`  | Search your snippets and paste at cursor |
| `Sniparium: Sign Out`        | Sign out of your account                 |
| `Sniparium: Open in Browser` | Open Sniparium in your default browser   |

### Web App

1. Go to [sniparium.ibiulabs.com](https://sniparium.ibiulabs.com)
2. Sign in with your GitHub account
3. In the dashboard you can:
   - **Create snippets** using the Monaco Editor with full syntax highlighting
   - **Add tags** to organize your library
   - **Search** your snippets by title, language, or tag
   - **Edit and delete** existing snippets
   - **Mark snippets as public** to get a shareable permanent URL (e.g. `sniparium.vercel.app/s/<id>`)

---

## Current versions and codebase

These are the first functional versions of Sniparium, released for public use:

| Package            | Version |
| ------------------ | ------- |
| `sniparium-api`    | 1.4.5   |
| `sniparium-web`    | 1.7.0   |
| `sniparium-vscode` | 1.2.5   |

**Codebase size (across all three packages):**

| Category                                   | Lines      |
| ------------------------------------------ | ---------- |
| Source code (TypeScript, CSS, SQL, config) | 37,607     |
| Documentation (Markdown)                   | 3,237      |
| **Total**                                  | **40,844** |

---

## License

No license
---

---

# Sniparium

**Seu banco pessoal de snippets de código — dentro do VS Code e na web.**

> Disponível no [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ibiulabs.sniparium) · Web app em [sniparium.ibiulabs.com](https://sniparium.ibiulabs.com)

---

## Por que o Sniparium existe

Todo desenvolvedor tem aquela pasta. A dos arquivos chamados `utils.txt`, `helpers_ANTIGO.js`, `coisas_pra_lembrar.md` — um cemitério de fragmentos de código que foram úteis uma vez e serão úteis de novo, se você conseguir encontrá-los.

O fluxo é sempre o mesmo: você escreve uma função esperta num projeto, copia pra um app de notas, esquece qual app, reescreve a função do zero seis meses depois. Ou mantém uma lista enorme de Gists que fica impossível de navegar. Ou salva favoritos do Stack Overflow que expiram. O código que deveria economizar tempo acaba custando.

O Sniparium foi feito pra resolver isso. A ideia é simples: um banco pessoal de snippets que vive onde você realmente trabalha — dentro do VS Code — e que também pode ser acessado de qualquer navegador. Você escreve um trecho de código que quer guardar, seleciona, roda um comando e ele está salvo. Depois você busca por título, linguagem ou tag e cola diretamente na posição do cursor. Sem trocar de contexto, sem copiar de outra janela, sem pesquisar de novo o que você já solucionou.

Além da produtividade pessoal, o Sniparium permite marcar snippets como públicos e compartilhá-los via uma URL permanente. Isso transforma suas próprias soluções em referências compartilháveis — para o time, para seu portfólio ou para qualquer pessoa que as encontre pelo web app.

---

## Como foi construído

O Sniparium é um monorepo full-stack com três pacotes independentes: um web app, uma API REST e uma extensão para VS Code. Desenvolvi o projeto inteiro com o auxílio do **GitHub Copilot**, utilizando principalmente os modelos **Claude Sonnet 4.6** e **Claude Opus 4.6** para geração de código, decisões de arquitetura, auditorias de segurança e depuração.

### API — `sniparium-api`

O backend é um servidor **Fastify** escrito em **TypeScript**. O Fastify foi escolhido no lugar do Express pelo throughput significativamente melhor e pelo suporte de primeira classe a TypeScript com validação de schemas via Zod.

A autenticação é feita pelo **Better Auth**, uma biblioteca de auth open-source e self-hosted que gerencia o GitHub OAuth e os tokens de sessão. Por ser self-hosted, não há custos por usuário e nenhuma dependência de terceiros para algo tão crítico quanto auth — todos os dados de sessão ficam no nosso próprio banco.

O banco de dados é um **PostgreSQL** hospedado no **Neon** (Postgres serverless). O schema e as queries são escritos com **Drizzle ORM**, um ORM TypeScript-first que gera queries completamente tipadas a partir da definição do schema. A busca full-text nos snippets é implementada nativamente no PostgreSQL com `tsvector` e `tsquery`, atualizada automaticamente por um trigger do banco a cada insert e update.

A extensão VS Code se autentica na API usando um Bearer token armazenado no `vscode.SecretStorage` (o keychain do sistema operacional — Credential Manager no Windows, Keychain no macOS, libsecret no Linux). As sessões da web usam cookies HTTP assinados gerenciados pelo Better Auth.

A API é testada com **Vitest** e os testes de integração rodam contra uma branch real do Neon isolada do ambiente de produção. O CI/CD é feito pelo **GitHub Actions**: um pipeline de CI roda type check, build e testes a cada push; um pipeline de CD faz o deploy no **Render** (API) e na **Vercel** (web) apenas depois que o CI passa. Erros em produção são capturados em tempo real pelo **Sentry**. O free tier do Render é mantido ativo pelo **UptimeRobot**, que faz ping no `/healthz` a cada 5 minutos.

### Web App — `sniparium-web`

O frontend é uma aplicação **Next.js** com App Router. Tem dois propósitos: um dashboard autenticado para gerenciar seus próprios snippets, e páginas públicas para snippets compartilhados com server-side rendering para SEO.

A interface é construída com **Tailwind CSS** e **shadcn/ui** — componentes copiados diretamente para dentro do projeto ao invés de instalados como dependência fechada, o que garante controle total sobre o código. O formulário de criação e edição de snippets usa o **Monaco Editor** (o mesmo editor que roda dentro do VS Code) para que escrever e editar código seja uma experiência nativa. A exibição de snippets nas páginas de listagem e nas páginas públicas usa o **Shiki** para renderizar HTML com syntax highlighting estático no servidor, muito mais leve do que carregar o Monaco para código somente-leitura.

O web app se conecta à API via **tRPC**, que cria uma ponte type-safe entre o backend Fastify e o frontend Next.js. Isso significa que todas as chamadas à API — como `trpc.snippets.list.useQuery()` — são completamente tipadas de ponta a ponta, sem definições de interface duplicadas.

### Extensão VS Code — `sniparium-vscode`

A extensão é construída com a **VS Code Extension API** e compilada com **esbuild**, que é dramaticamente mais rápido do que o webpack padrão usado nos templates de extensão do VS Code. Ela registra quatro comandos na Command Palette e gerencia o fluxo de OAuth do GitHub por um URI handler (`vscode://ibiulabs.sniparium`), que recebe o callback direto do navegador e armazena o token de sessão com segurança via `vscode.SecretStorage`.

A extensão é publicada no **Visual Studio Marketplace** via `vsce` e um Personal Access Token do **Azure DevOps**.

---

## Instalação

### Extensão VS Code

**Pelo Marketplace (recomendado):**

1. Abra o VS Code
2. Pressione `Ctrl+Shift+X` (ou `Cmd+Shift+X` no macOS) para abrir o painel de Extensões
3. Pesquise por **Sniparium**
4. Clique em **Install**

**Link direto:**
[marketplace.visualstudio.com/items?itemName=ibiulabs.sniparium](https://marketplace.visualstudio.com/items?itemName=ibiulabs.sniparium)

**Via Quick Open:**

Pressione `Ctrl+P` / `Cmd+P`, cole o comando abaixo e pressione Enter:

```
ext install ibiulabs.sniparium
```

### Web App

Sem instalação necessária. Acesse diretamente em [sniparium.ibiulabs.com](https://sniparium.ibiulabs.com).

---

## Como usar

### VS Code

Todos os comandos do Sniparium estão disponíveis na **Command Palette** (`Ctrl+Shift+P` / `Cmd+Shift+P`).

#### Entrar na conta

1. Abra a Command Palette
2. Execute **`Sniparium: Search Snippet`** ou **`Sniparium: Save Selection`**
3. Se não estiver autenticado, uma janela do navegador vai abrir para autorizar com o GitHub
4. Após a autorização, clique em **Open VS Code** no navegador — a extensão conecta automaticamente

#### Salvar um snippet

1. Selecione qualquer trecho de código no editor
2. Abra a Command Palette → **`Sniparium: Save Selection`**
3. Preencha o título, a linguagem, a descrição (opcional) e as tags (opcional)
4. Pressione Enter — o snippet é salvo na sua conta

#### Buscar e inserir um snippet

1. Abra a Command Palette → **`Sniparium: Search Snippet`**
2. Digite para buscar por título, linguagem ou tag
3. Selecione um snippet da lista
4. O snippet é colado na posição do cursor

#### Outros comandos

| Comando                      | Descrição                                       |
| ---------------------------- | ----------------------------------------------- |
| `Sniparium: Save Selection`  | Salva o código selecionado como um novo snippet |
| `Sniparium: Search Snippet`  | Busca seus snippets e cola na posição do cursor |
| `Sniparium: Sign Out`        | Sai da conta                                    |
| `Sniparium: Open in Browser` | Abre o Sniparium no navegador padrão            |

### Web App

1. Acesse [sniparium.ibiulabs.com](https://sniparium.ibiulabs.com)
2. Entre com sua conta do GitHub
3. No dashboard você pode:
   - **Criar snippets** usando o Monaco Editor com syntax highlighting completo
   - **Adicionar tags** para organizar sua biblioteca
   - **Buscar** seus snippets por título, linguagem ou tag
   - **Editar e excluir** snippets existentes
   - **Marcar snippets como públicos** para obter uma URL permanente e compartilhável (ex: `sniparium.vercel.app/s/<id>`)

---

## Versões atuais e base de código

Estas são as primeiras versões funcionais do Sniparium, liberadas para uso público:

| Pacote             | Versão |
| ------------------ | ------ |
| `sniparium-api`    | 1.4.5  |
| `sniparium-web`    | 1.7.0  |
| `sniparium-vscode` | 1.2.5  |

**Tamanho da base de código (todos os três pacotes):**

| Categoria                                   | Linhas     |
| ------------------------------------------- | ---------- |
| Código-fonte (TypeScript, CSS, SQL, config) | 37.607     |
| Documentação (Markdown)                     | 3.237      |
| **Total**                                   | **40.844** |

---

## Licença

Sem licença
