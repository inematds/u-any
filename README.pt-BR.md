<h1 align="center">Understand Anything</h1>

> 🇬🇧 Read this in [English](./README.md).

<p align="center">
  <strong>Transforme qualquer base de código, base de conhecimento ou documentação em um grafo de conhecimento interativo que você pode explorar, pesquisar e questionar.</strong>
  <br />
  <em>Funciona com Claude Code, Codex, Cursor, Copilot, Gemini CLI e mais.</em>
</p>

<p align="center">
  <a href="https://trendshift.io/repositories/23482" target="_blank"><img src="https://trendshift.io/api/badge/repositories/23482" alt="Lum1104%2FUnderstand-Anything | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>
</p>

<p align="center">
  <a href="README.md">English</a> | <a href="READMEs/README.zh-CN.md">简体中文</a> | <a href="READMEs/README.zh-TW.md">繁體中文</a> | <a href="READMEs/README.ja-JP.md">日本語</a> | <a href="READMEs/README.ko-KR.md">한국어</a> | <a href="READMEs/README.es-ES.md">Español</a> | <a href="READMEs/README.tr-TR.md">Türkçe</a> | <a href="READMEs/README.ru-RU.md">Русский</a> | <a href="README.pt-BR.md">Português (Brasil)</a>
</p>

<p align="center">
  <a href="#-início-rápido"><img src="https://img.shields.io/badge/Quick_Start-blue" alt="Início Rápido" /></a>
  <a href="https://github.com/Lum1104/Understand-Anything/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow" alt="License: MIT" /></a>
  <a href="https://docs.anthropic.com/en/docs/claude-code"><img src="https://img.shields.io/badge/Claude_Code-8A2BE2" alt="Claude Code" /></a>
  <a href="#codex"><img src="https://img.shields.io/badge/Codex-000000" alt="Codex" /></a>
  <a href="#vs-code--github-copilot"><img src="https://img.shields.io/badge/Copilot-24292e" alt="Copilot" /></a>
  <a href="#copilot-cli"><img src="https://img.shields.io/badge/Copilot_CLI-24292e" alt="Copilot CLI" /></a>
  <a href="#gemini-cli"><img src="https://img.shields.io/badge/Gemini_CLI-4285F4" alt="Gemini CLI" /></a>
  <a href="#opencode"><img src="https://img.shields.io/badge/OpenCode-38bdf8" alt="OpenCode" /></a>
  <a href="#mistral-vibe-cli"><img src="https://img.shields.io/badge/Vibe_CLI-7c3aed" alt="Vibe CLI" /></a>
  <a href="https://understand-anything.com"><img src="https://img.shields.io/badge/Homepage-d4a574" alt="Página inicial" /></a>
  <a href="https://understand-anything.com/demo/"><img src="https://img.shields.io/badge/Live_Demo-00c853" alt="Demo ao vivo" /></a>
</p>

<p align="center">
  <img src="assets/hero.png" alt="Understand Anything — Transforme qualquer base de código em um grafo de conhecimento interativo" width="800" />
</p>

<p align="center">
  <strong>💬 <a href="https://discord.gg/pydat66RY">Entre na comunidade do Discord &rarr;</a></strong>
  <br />
  <em>Tire dúvidas, compartilhe o que construiu, receba ajuda da comunidade.</em>
</p>

---

**Você acabou de entrar em um novo time. A base de código tem 200.000 linhas. Por onde começar?**

Understand Anything é um [Claude Code Plugin](https://code.claude.com/docs/en/plugins-reference#plugins-reference) que analisa seu projeto com um pipeline multi-agente, constrói um grafo de conhecimento de cada arquivo, função, classe e dependência, e então te oferece um dashboard interativo para explorar tudo visualmente. Pare de ler código no escuro. Comece a enxergar o panorama completo.

> **O objetivo não é um grafo que te impressione com o quão complexa sua base de código é — é um grafo que silenciosamente te ensina como cada peça se encaixa.**

---

## ✨ Funcionalidades

> [!NOTE]
> **Quer pular a leitura?** Experimente o [demo ao vivo](https://understand-anything.com/demo/) na nossa [página inicial](https://understand-anything.com/) — um dashboard totalmente interativo que você pode mover, dar zoom, pesquisar e explorar direto no navegador.

### Explore o grafo estrutural

Navegue sua base de código como um grafo de conhecimento interativo — cada arquivo, função e classe é um nó clicável, pesquisável e explorável. Selecione qualquer nó para ver resumos em linguagem natural, relacionamentos e tours guiados.

### Entenda a lógica de negócio

Mude para a visão de domínio e veja como seu código se mapeia em processos reais de negócio — domínios, fluxos e passos dispostos em um grafo horizontal.

### Analise bases de conhecimento

Aponte `/understand-knowledge` para uma [wiki LLM no padrão Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) e obtenha um grafo de conhecimento com layout force-directed e clusterização por comunidades. O parser determinístico extrai wikilinks e categorias do `index.md`, e então agentes LLM descobrem relacionamentos implícitos, extraem entidades e revelam afirmações — transformando sua wiki em um grafo navegável de ideias interconectadas.

<table>
  <tr>
    <td width="50%" valign="top">
      <h3>🧭 Tours Guiados</h3>
      <p>Walkthroughs auto-gerados da arquitetura, ordenados por dependência. Aprenda a base de código na ordem certa.</p>
    </td>
    <td width="50%" valign="top">
      <h3>🔍 Busca Fuzzy & Semântica</h3>
      <p>Encontre qualquer coisa por nome ou por significado. Pesquise "quais partes lidam com autenticação?" e obtenha resultados relevantes em todo o grafo.</p>
    </td>
  </tr>
  <tr>
    <td width="50%" valign="top">
      <h3>📊 Análise de Impacto de Diff</h3>
      <p>Veja quais partes do sistema suas alterações afetam antes do commit. Entenda os efeitos em cascata pela base de código.</p>
    </td>
    <td width="50%" valign="top">
      <h3>🎭 UI Adaptativa por Persona</h3>
      <p>O dashboard ajusta o nível de detalhe conforme quem você é — dev júnior, PM ou usuário avançado.</p>
    </td>
  </tr>
  <tr>
    <td width="50%" valign="top">
      <h3>🏗️ Visualização por Camadas</h3>
      <p>Agrupamento automático por camada arquitetural — API, Service, Data, UI, Utility — com legenda colorida.</p>
    </td>
    <td width="50%" valign="top">
      <h3>📚 Conceitos de Linguagem</h3>
      <p>12 padrões de programação (generics, closures, decorators, etc.) explicados em contexto, onde quer que apareçam.</p>
    </td>
  </tr>
</table>

---

## 🚀 Início Rápido

> Os nomes de skills, agents e slash commands (ex.: `/understand`, `/understand-dashboard`, `project-scanner`) permanecem em inglês porque são identificadores usados pelas ferramentas para acionar funcionalidades. Não traduza esses nomes ao executá-los.

### 1. Instale o plugin

```bash
/plugin marketplace add Lum1104/Understand-Anything
/plugin install understand-anything
```

### 2. Analise sua base de código

```bash
/understand
```

Um pipeline multi-agente varre seu projeto, extrai cada arquivo, função, classe e dependência, e então constrói um grafo de conhecimento salvo em `.understand-anything/knowledge-graph.json`.

**Saída localizada:** Use `--language` para gerar conteúdo no idioma de sua preferência:

```bash
# Gera conteúdo em chinês (知识图节点描述和 Dashboard UI)
/understand --language zh

# Idiomas suportados: en (padrão), zh, zh-TW, ja, ko, ru
```

O parâmetro `--language` afeta:
- Resumos e descrições de nós no grafo de conhecimento
- Rótulos, botões e tooltips da UI do dashboard
- Explicações dos tours guiados

### 3. Explore o dashboard

```bash
/understand-dashboard
```

Um dashboard web interativo abre com sua base de código visualizada como um grafo — colorida por camada arquitetural, pesquisável e clicável. Selecione qualquer nó para ver seu código, relacionamentos e uma explicação em linguagem natural.

### 4. Continue aprendendo

```bash
# Pergunte qualquer coisa sobre a base de código
/understand-chat How does the payment flow work?

# Analise o impacto das suas alterações atuais
/understand-diff

# Mergulhe fundo em um arquivo ou função específica
/understand-explain src/auth/login.ts

# Gere um guia de onboarding para novos membros do time
/understand-onboard

# Extraia conhecimento de domínio de negócio (domínios, fluxos, passos)
/understand-domain

# Analise uma wiki LLM no padrão Karpathy
/understand-knowledge ~/path/to/wiki

# Reexecute a qualquer hora — incremental por padrão (só reanalisa arquivos alterados)
/understand

# Atualize automaticamente a cada commit via post-commit hook
/understand --auto-update

# Limite a um subdiretório (útil em monorepos enormes)
/understand src/frontend
```

---

## 🌐 Instalação Multi-Plataforma

Understand-Anything funciona em múltiplas plataformas de codificação com IA.

### Claude Code (Nativo)

```bash
/plugin marketplace add Lum1104/Understand-Anything
/plugin install understand-anything
```

### Instalação em uma linha (Codex / OpenCode / OpenClaw / Antigravity / Gemini CLI / Pi Agent / Vibe CLI / VS Code Copilot / Hermes / Cline / KIMI CLI)

**macOS / Linux:**
```bash
curl -fsSL https://raw.githubusercontent.com/Lum1104/Understand-Anything/main/install.sh | bash
# ou pule o prompt passando a plataforma:
curl -fsSL https://raw.githubusercontent.com/Lum1104/Understand-Anything/main/install.sh | bash -s codex
```

**Windows (PowerShell):**
```powershell
iwr -useb https://raw.githubusercontent.com/Lum1104/Understand-Anything/main/install.ps1 | iex
```

O instalador clona o repositório em `~/.understand-anything/repo` e cria os symlinks corretos para a plataforma escolhida. Reinicie sua CLI/IDE depois disso.

- Valores suportados para `<platform>`: `gemini`, `codex`, `opencode`, `pi`, `openclaw`, `antigravity`, `vibe`, `vscode`, `hermes`, `cline`, `kimi`
- Atualizar depois: `./install.sh --update`
- Desinstalar: `./install.sh --uninstall <platform>`

### Cursor

O Cursor faz auto-discovery do plugin via `.cursor-plugin/plugin.json` quando este repositório é clonado. Não precisa instalar manualmente — basta clonar e abrir no Cursor.

Se o auto-discovery não detectar, instale manualmente: abra **Cursor Settings → Plugins**, cole `https://github.com/Lum1104/Understand-Anything` no campo de busca e adicione a partir daí.

### VS Code + GitHub Copilot

VS Code com GitHub Copilot (v1.108+) faz auto-discovery do plugin via `.copilot-plugin/plugin.json` quando este repositório é clonado. Não precisa instalar manualmente — basta clonar e abrir no VS Code.

Para skills pessoais (disponíveis em todos os projetos), execute o `install.sh` acima com a plataforma `vscode`.

### Copilot CLI

```bash
copilot plugin install Lum1104/Understand-Anything:understand-anything-plugin
```

### Compatibilidade entre Plataformas

| Plataforma | Status | Método de Instalação |
|----------|--------|----------------|
| Claude Code | ✅ Nativo | Plugin marketplace |
| Cursor | ✅ Suportado | Auto-discovery |
| VS Code + GitHub Copilot | ✅ Suportado | Auto-discovery |
| Copilot CLI | ✅ Suportado | Plugin install |
| Codex | ✅ Suportado | `install.sh codex` |
| OpenCode | ✅ Suportado | `install.sh opencode` |
| OpenClaw | ✅ Suportado | `install.sh openclaw` |
| Antigravity | ✅ Suportado | `install.sh antigravity` |
| Gemini CLI | ✅ Suportado | `install.sh gemini` |
| Pi Agent | ✅ Suportado | `install.sh pi` |
| Vibe CLI | ✅ Suportado | `install.sh vibe` |
| Hermes | ✅ Suportado | `install.sh hermes` |
| Cline | ✅ Suportado | `install.sh cline` |
| KIMI CLI | ✅ Suportado | `install.sh kimi` |

---

## 📦 Compartilhe o Grafo com Seu Time

O grafo é apenas JSON — **commite uma vez e seus colegas pulam o pipeline**. Ótimo para onboarding, revisões de PR e docs-as-code.

> **Exemplo:** [GoogleCloudPlatform/microservices-demo (fork)](https://github.com/Lum1104/microservices-demo) — referência em Go / Java / Python / Node com o grafo já commitado.

**O que commitar:** tudo em `.understand-anything/` *exceto* `intermediate/` e `diff-overlay.json` (são rascunhos locais).

```gitignore
.understand-anything/intermediate/
.understand-anything/diff-overlay.json
```

**Mantenha atualizado:** habilite `/understand --auto-update` — um post-commit hook aplica patches incrementais no grafo, de modo que cada commit entra com um grafo correspondente. Ou reexecute `/understand` manualmente antes de releases.

**Grafos grandes (10 MB+):** rastreie com **git-lfs**.

```bash
git lfs install
git lfs track ".understand-anything/*.json"
git add .gitattributes .understand-anything/
```

---

## 🔧 Por Baixo dos Panos

### Tree-sitter + LLM híbrido

Análise estática e LLMs fazem cada um o que faz de melhor:

- **Tree-sitter (determinístico)** — faz o parse do código-fonte em uma árvore sintática concreta e extrai fatos estruturais: imports, exports, definições de função/classe, sites de chamada, herança. Pré-resolvido em um `importMap` durante a fase de scan e passado aos file-analyzers, evitando re-deriva de imports do source. Mesma entrada → mesma saída, sempre. Também alimenta a detecção de mudanças baseada em fingerprint para updates incrementais.
- **LLM (semântico)** — lê a estrutura parseada junto com o source original para produzir o que parsers não conseguem: resumos em linguagem natural, tags, atribuição de camadas arquiteturais, mapeamento para domínios de negócio, tours guiados, destaques de conceitos de linguagem.

Essa divisão é o motivo de o grafo ser reprodutível no lado estrutural (o mesmo código sempre gera as mesmas arestas), mas ainda capturar intenção no lado semântico (para que um arquivo *serve*, não só o que ele importa).

### Pipeline Multi-Agente

O comando `/understand` orquestra 5 agentes especializados, e `/understand-domain` adiciona um 6º:

| Agente | Função |
|-------|------|
| `project-scanner` | Descobre arquivos, detecta linguagens e frameworks |
| `file-analyzer` | Extrai funções, classes, imports; produz nós e arestas do grafo |
| `architecture-analyzer` | Identifica camadas arquiteturais |
| `tour-builder` | Gera tours de aprendizado guiados |
| `graph-reviewer` | Valida completude do grafo e integridade referencial (roda inline por padrão; use `--review` para revisão completa por LLM) |
| `domain-analyzer` | Extrai domínios de negócio, fluxos e passos de processo (usado por `/understand-domain`) |
| `article-analyzer` | Extrai entidades, afirmações e relacionamentos implícitos de artigos de wiki (usado por `/understand-knowledge`) |

Os file-analyzers rodam em paralelo (até 5 concorrentes, 20-30 arquivos por lote). Suporta updates incrementais — só reanalisa arquivos que mudaram desde a última execução.

---

## 🎥 Comunidade

Um walkthrough feito pela comunidade pela **Better Stack**.

<p align="center">
  <a href="https://www.youtube.com/watch?v=VmIUXVlt7_I"><img src="https://img.youtube.com/vi/VmIUXVlt7_I/maxresdefault.jpg" alt="Walkthrough da comunidade por Better Stack — assista no YouTube" width="480" /></a>
  <br />
  <em><a href="https://www.youtube.com/watch?v=VmIUXVlt7_I">Assista no YouTube &rarr;</a></em>
</p>

Fez um vídeo, post de blog ou tutorial? Abra uma issue ou PR — vamos adorar destacar aqui.

---

## 🤝 Contribuindo

Contribuições são bem-vindas! Como começar:

1. Faça fork do repositório
2. Crie uma branch de feature (`git checkout -b feature/my-feature`)
3. Rode os testes (`pnpm --filter @understand-anything/core test`)
4. Faça commit das alterações e abra um pull request

Por favor, abra uma issue primeiro para mudanças grandes para que possamos discutir a abordagem.

---

<p align="center">
  <strong>Pare de ler código no escuro. Comece a entender tudo.</strong>
</p>

## Histórico de Estrelas

<a href="https://www.star-history.com/?repos=Lum1104%2FUnderstand-Anything&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/image?repos=Lum1104/Understand-Anything&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/image?repos=Lum1104/Understand-Anything&type=date&legend=top-left" />
   <img alt="Gráfico de Histórico de Estrelas" src="https://api.star-history.com/image?repos=Lum1104/Understand-Anything&type=date&legend=top-left" />
 </picture>
</a>

<p align="center">
  <em>Obrigado a todos que usaram e contribuíram — saber que isso economiza tempo das pessoas é o que faz valer a pena ter construído.</em>
</p>

<p align="center">
  Licença MIT &copy; <a href="https://github.com/Lum1104">Lum1104</a>
</p>
