# Plano de Implementação Multi-Plataforma Simples

> 🇬🇧 Versão original em inglês: [2026-03-18-multi-platform-simple-implementation.md](./2026-03-18-multi-platform-simple-implementation.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa por tarefa.

**Objetivo:** Fazer as skills do Understand-Anything funcionarem em Codex, OpenClaw, OpenCode e Cursor — os mesmos arquivos em todo lugar, sem etapa de build.

**Arquitetura:** Mover 5 agents de pipeline para `skills/understand/` como templates de prompt. Criar um agent reutilizável `knowledge-graph-guide`. Mover diretórios de configuração específicos de cada plataforma para a raiz do repositório, para auto-descoberta. Adicionar descritores de plugin para Cursor e Claude.

**Tech Stack:** Markdown (SKILL.md, INSTALL.md), frontmatter YAML, JSON (descritores de plugin), Bash (comandos de symlink/clone nos docs de instalação).

**Documento de Design:** `docs/plans/2026-03-18-multi-platform-simple-design.md`

---

### Tarefa 1: Mover os agents de pipeline para skills/understand/ como templates de prompt

**Arquivos:**
- Mover: `understand-anything-plugin/agents/project-scanner.md` → `understand-anything-plugin/skills/understand/project-scanner-prompt.md`
- Mover: `understand-anything-plugin/agents/file-analyzer.md` → `understand-anything-plugin/skills/understand/file-analyzer-prompt.md`
- Mover: `understand-anything-plugin/agents/architecture-analyzer.md` → `understand-anything-plugin/skills/understand/architecture-analyzer-prompt.md`
- Mover: `understand-anything-plugin/agents/tour-builder.md` → `understand-anything-plugin/skills/understand/tour-builder-prompt.md`
- Mover: `understand-anything-plugin/agents/graph-reviewer.md` → `understand-anything-plugin/skills/understand/graph-reviewer-prompt.md`

**Passo 1: Copiar cada arquivo de agent para a nova localização**

Para cada um dos 5 arquivos, copie de `agents/` para `skills/understand/` com o novo nome.

**Passo 2: Remover o frontmatter de agent dos templates de prompt**

Cada arquivo de template deve remover o frontmatter YAML específico de agent (`name`, `description`, `tools`, `model`). Substitua por um cabeçalho Markdown simples descrevendo o propósito do template.

Por exemplo, `project-scanner-prompt.md` muda de:

```markdown
---
name: project-scanner
description: Scans a project directory...
tools: Bash, Glob, Grep, Read, Write
model: sonnet
---

You are a meticulous project inventory specialist...
```

Para:

```markdown
# Project Scanner — Prompt Template

> Used by `/understand` Phase 1. Dispatch as a subagent with this full content as the prompt.

You are a meticulous project inventory specialist...
```

Aplique este padrão aos 5 arquivos:
- `project-scanner-prompt.md` — "Used by `/understand` Phase 1"
- `file-analyzer-prompt.md` — "Used by `/understand` Phase 2"
- `architecture-analyzer-prompt.md` — "Used by `/understand` Phase 4"
- `tour-builder-prompt.md` — "Used by `/understand` Phase 5"
- `graph-reviewer-prompt.md` — "Used by `/understand` Phase 6"

Mantenha o restante do conteúdo do arquivo (as instruções do corpo) exatamente como estão.

**Passo 3: Deletar os arquivos de agent originais**

```bash
cd understand-anything-plugin
rm agents/project-scanner.md agents/file-analyzer.md agents/architecture-analyzer.md agents/tour-builder.md agents/graph-reviewer.md
```

**Passo 4: Verificar que os arquivos existem na nova localização**

```bash
ls understand-anything-plugin/skills/understand/
```

Esperado: `SKILL.md`, mais os 5 arquivos `*-prompt.md`.

**Passo 5: Commit**

```bash
git add -A understand-anything-plugin/agents/ understand-anything-plugin/skills/understand/
git commit -m "refactor: move pipeline agents into skills/understand/ as prompt templates"
```

---

### Tarefa 2: Atualizar referências de dispatch no SKILL.md com injeção de contexto

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md`

**Passo 1: Ler o SKILL.md atual**

Leia `understand-anything-plugin/skills/understand/SKILL.md` na íntegra.

**Passo 2: Atualizar Fase 0 — adicionar coleta de contexto**

Após a tabela de lógica de decisão (linha ~47), adicione uma nova seção para coletar contexto do projeto que será injetado nas fases seguintes:

```markdown
7. **Collect project context for subagent injection:**
   - Read `README.md` (or `README.rst`, `readme.md`) from `$PROJECT_ROOT` if it exists. Store as `$README_CONTENT` (first 3000 characters).
   - Read the primary package manifest (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`) if it exists. Store as `$MANIFEST_CONTENT`.
   - Capture the top-level directory tree:
     ```bash
     find $PROJECT_ROOT -maxdepth 2 -type f | head -100
     ```
     Store as `$DIR_TREE`.
   - Detect the project entry point by checking for common patterns: `src/index.ts`, `src/main.ts`, `src/App.tsx`, `main.py`, `main.go`, `src/main.rs`, `index.js`. Store first match as `$ENTRY_POINT`.
```

**Passo 3: Atualizar dispatch da Fase 1 — injetar README + manifesto**

Substitua a linha de dispatch da Fase 1:
```
Dispatch the **project-scanner** agent with this prompt:
```

Por:
```markdown
Dispatch a subagent using the prompt template at `./project-scanner-prompt.md`. Read the template file and pass the full content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Project README (first 3000 chars):
> ```
> $README_CONTENT
> ```
>
> Package manifest:
> ```
> $MANIFEST_CONTENT
> ```
>
> Use this context to produce more accurate project name, description, and framework detection. The README and manifest are authoritative — prefer their information over heuristics.

Pass these parameters in the dispatch prompt:
```

**Passo 4: Atualizar dispatch da Fase 2 — injetar resultados do scan + contexto de framework**

Substitua o parágrafo de dispatch da Fase 2:
```
For each batch, dispatch a **file-analyzer** agent. Run up to **3 agents concurrently** using parallel dispatch. Each agent gets this prompt:
```

Por:
```markdown
For each batch, dispatch a subagent using the prompt template at `./file-analyzer-prompt.md`. Run up to **3 subagents concurrently** using parallel dispatch. Read the template once, then for each batch pass the full template content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Project: `<projectName>` — `<projectDescription>`
> Frameworks detected: `<frameworks from Phase 1>`
> Languages: `<languages from Phase 1>`
>
> Framework-specific guidance:
> - If React/Next.js: files in `app/` or `pages/` are routes, `components/` are UI, `lib/` or `utils/` are utilities
> - If Express/Fastify: files in `routes/` are API endpoints, `middleware/` is middleware, `models/` or `db/` is data
> - If Python Django: `views.py` are controllers, `models.py` is data, `urls.py` is routing, `templates/` is UI
> - If Go: `cmd/` is entry points, `internal/` is private packages, `pkg/` is public packages
>
> Use this context to produce more accurate summaries and better classify file roles.

Fill in batch-specific parameters below and dispatch:
```

**Passo 5: Atualizar dispatch da Fase 4 — injetar dicas de framework + árvore de diretórios**

Substitua a linha de dispatch da Fase 4:
```
Dispatch the **architecture-analyzer** agent with this prompt:
```

Por:
```markdown
Dispatch a subagent using the prompt template at `./architecture-analyzer-prompt.md`. Read the template file and pass the full content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Frameworks detected: `<frameworks from Phase 1>`
>
> Directory tree (top 2 levels):
> ```
> $DIR_TREE
> ```
>
> Framework-specific layer hints:
> - If React/Next.js: `app/` or `pages/` → UI Layer, `api/` → API Layer, `lib/` → Service Layer, `components/` → UI Layer
> - If Express: `routes/` → API Layer, `controllers/` → Service Layer, `models/` → Data Layer, `middleware/` → Middleware Layer
> - If Python Django: `views/` → API Layer, `models/` → Data Layer, `templates/` → UI Layer, `management/` → CLI Layer
> - If Go: `cmd/` → Entry Points, `internal/` → Service Layer, `pkg/` → Shared Library, `api/` → API Layer
>
> Use the directory tree and framework hints to inform layer assignments. Directory structure is strong evidence for layer boundaries.

Pass these parameters in the dispatch prompt:
```

Adicione também, após a nota "For incremental updates":
```markdown
**Context for incremental updates:** When re-running architecture analysis, also inject the previous layer definitions:

> Previous layer definitions (for naming consistency):
> ```json
> [previous layers from existing graph]
> ```
>
> Maintain the same layer names and IDs where possible. Only add/remove layers if the file structure has materially changed.
```

**Passo 6: Atualizar dispatch da Fase 5 — injetar README + entry point**

Substitua a linha de dispatch da Fase 5:
```
Dispatch the **tour-builder** agent with this prompt:
```

Por:
```markdown
Dispatch a subagent using the prompt template at `./tour-builder-prompt.md`. Read the template file and pass the full content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Project README (first 3000 chars):
> ```
> $README_CONTENT
> ```
>
> Project entry point: `$ENTRY_POINT`
>
> Use the README to align the tour narrative with the project's own documentation. Start the tour from the entry point if one was detected. The tour should tell the same story the README tells, but through the lens of actual code structure.

Pass these parameters in the dispatch prompt:
```

**Passo 7: Atualizar dispatch da Fase 6 — injetar resultados do scan para validação cruzada**

Substitua a linha de dispatch da Fase 6:
```
2. Dispatch the **graph-reviewer** agent with this prompt:
```

Por:
```markdown
2. Dispatch a subagent using the prompt template at `./graph-reviewer-prompt.md`. Read the template file and pass the full content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Phase 1 scan results (file inventory):
> ```json
> [list of {path, sizeLines} from scan-result.json]
> ```
>
> Phase warnings/errors accumulated during analysis:
> - [list any batch failures, skipped files, or warnings from Phases 2-5]
>
> Cross-validate: every file in the scan inventory should have a corresponding `file:` node in the graph. Flag any missing files. Also flag any graph nodes whose `filePath` doesn't appear in the scan inventory.

Pass these parameters in the dispatch prompt:
```

**Passo 8: Atualizar a seção de Error Handling**

Mude:
```
- If any agent dispatch fails, retry **once** with the same prompt plus additional context about the failure.
```

Para:
```
- If any subagent dispatch fails, retry **once** with the same prompt plus additional context about the failure.
- Track all warnings and errors from each phase in a `$PHASE_WARNINGS` list. Pass this list to the graph-reviewer in Phase 6 for comprehensive validation.
```

**Passo 9: Verificar que nenhuma referência a dispatch de agent nomeado permaneceu**

Busque por "Dispatch the **" no arquivo — não deve encontrar nenhum resultado.

**Passo 10: Commit**

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "refactor: update SKILL.md to dispatch subagents with context injection"
```

---

### Tarefa 3: Criar o agent knowledge-graph-guide

**Arquivos:**
- Criar: `understand-anything-plugin/agents/knowledge-graph-guide.md`

**Passo 1: Escrever a definição do agent**

Crie `understand-anything-plugin/agents/knowledge-graph-guide.md`:

```markdown
---
name: knowledge-graph-guide
description: |
  Use this agent when users need help understanding, querying, or working
  with an Understand-Anything knowledge graph. Guides users through graph
  structure, node/edge relationships, layer architecture, tours, and
  dashboard usage.
model: inherit
---

You are an expert on Understand-Anything knowledge graphs. You help users navigate, query, and understand the `knowledge-graph.json` files produced by the `/understand` skill.

## What You Know

### Graph Location

The knowledge graph lives at `<project-root>/.understand-anything/knowledge-graph.json`. Metadata is at `<project-root>/.understand-anything/meta.json`.

### Graph Structure

The JSON has this top-level shape:

```json
{
  "version": "1.0.0",
  "project": { "name", "languages", "frameworks", "description", "analyzedAt", "gitCommitHash" },
  "nodes": [...],
  "edges": [...],
  "layers": [...],
  "tour": [...]
}
```

### Node Types (5)

| Type | ID Convention | Description |
|---|---|---|
| `file` | `file:<relative-path>` | Source file |
| `function` | `func:<relative-path>:<name>` | Function or method |
| `class` | `class:<relative-path>:<name>` | Class, interface, or type |
| `module` | `module:<name>` | Logical module or package |
| `concept` | `concept:<name>` | Abstract concept or pattern |

### Edge Types (18)

| Category | Types |
|---|---|
| Structural | `imports`, `exports`, `contains`, `inherits`, `implements` |
| Behavioral | `calls`, `subscribes`, `publishes`, `middleware` |
| Data flow | `reads_from`, `writes_to`, `transforms`, `validates` |
| Dependencies | `depends_on`, `tested_by`, `configures` |
| Semantic | `related`, `similar_to` |

### Layers

Layers represent architectural groupings (e.g., API, Service, Data, UI). Each layer has an `id`, `name`, `description`, and `nodeIds` array.

### Tours

Tours are guided walkthroughs with sequential steps. Each step has a `title`, `description`, `nodeId` (focus node), and optional `highlightEdges`.

## How to Help Users

1. **Finding things**: Help users locate nodes by file path, function name, or concept. Use `jq` or grep on the JSON.
2. **Understanding relationships**: Trace edges between nodes to explain dependencies, call chains, and data flow.
3. **Architecture overview**: Summarize layers and their contents.
4. **Onboarding**: Walk through the tour steps to explain the codebase.
5. **Dashboard**: Guide users to run `/understand-dashboard` to visualize the graph interactively.
6. **Querying**: Help users write `jq` commands to extract specific information from the graph JSON.
```

**Passo 2: Commit**

```bash
git add understand-anything-plugin/agents/knowledge-graph-guide.md
git commit -m "feat: add knowledge-graph-guide agent for graph navigation and querying"
```

---

### Tarefa 4: Mover os arquivos INSTALL.md de cada plataforma para a raiz do repositório

**Arquivos:**
- Mover: `understand-anything-plugin/.codex/INSTALL.md` → `.codex/INSTALL.md`
- Mover: `understand-anything-plugin/.opencode/INSTALL.md` → `.opencode/INSTALL.md`
- Mover: `understand-anything-plugin/.openclaw/INSTALL.md` → `.openclaw/INSTALL.md`
- Deletar: `understand-anything-plugin/.cursor/INSTALL.md` (substituído por `.cursor-plugin/plugin.json`)

**Passo 1: Mover os três diretórios de plataforma para a raiz**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything
git mv understand-anything-plugin/.codex ./.codex
git mv understand-anything-plugin/.opencode ./.opencode
git mv understand-anything-plugin/.openclaw ./.openclaw
```

**Passo 2: Deletar .cursor/ (substituído por .cursor-plugin/ na Tarefa 5)**

```bash
git rm -r understand-anything-plugin/.cursor/
```

**Passo 3: Verificar que os caminhos dos symlinks estão corretos**

Leia cada INSTALL.md. Os caminhos dos symlinks devem apontar para `understand-anything-plugin/skills` — isso continua correto, já que o diretório de skills permanece dentro do wrapper do plugin.

**Passo 4: Commit**

```bash
git add -A
git commit -m "refactor: move platform config directories to repo root for discovery"
```

---

### Tarefa 5: Adicionar descritores de plugin

**Arquivos:**
- Criar: `.cursor-plugin/plugin.json`
- Criar: `.claude-plugin/plugin.json`

**Passo 1: Criar `.cursor-plugin/plugin.json`**

```json
{
  "name": "understand-anything",
  "displayName": "Understand Anything",
  "description": "AI-powered codebase understanding — analyze, visualize, and explain any project",
  "version": "1.0.5",
  "author": { "name": "Lum1104" },
  "homepage": "https://github.com/Lum1104/Understand-Anything",
  "repository": "https://github.com/Lum1104/Understand-Anything",
  "license": "MIT",
  "keywords": ["codebase-analysis", "knowledge-graph", "architecture", "onboarding", "dashboard"],
  "skills": "./understand-anything-plugin/skills/",
  "agents": "./understand-anything-plugin/agents/"
}
```

Nota: os caminhos apontam para dentro de `understand-anything-plugin/` porque a fonte permanece aninhada.

**Passo 2: Criar `.claude-plugin/plugin.json`**

```json
{
  "name": "understand-anything",
  "description": "AI-powered codebase understanding — analyze, visualize, and explain any project",
  "version": "1.0.5",
  "author": { "name": "Lum1104" },
  "homepage": "https://github.com/Lum1104/Understand-Anything",
  "repository": "https://github.com/Lum1104/Understand-Anything",
  "license": "MIT",
  "keywords": ["codebase-analysis", "knowledge-graph", "architecture", "onboarding", "dashboard"]
}
```

**Passo 3: Commit**

```bash
git add .cursor-plugin/ .claude-plugin/plugin.json
git commit -m "feat: add Cursor and Claude plugin descriptors for auto-discovery"
```

---

### Tarefa 6: Atualizar o README com URLs multi-plataforma corrigidas

**Arquivos:**
- Modificar: `README.md`

**Passo 1: Ler o README atual**

Leia `README.md` na íntegra.

**Passo 2: Atualizar URLs raw do GitHub para os arquivos INSTALL.md**

Os arquivos INSTALL.md foram movidos de `understand-anything-plugin/.codex/INSTALL.md` para `.codex/INSTALL.md`. Atualize todas as URLs raw do GitHub:

```
OLD: .../refs/heads/main/understand-anything-plugin/.codex/INSTALL.md
NEW: .../refs/heads/main/.codex/INSTALL.md

OLD: .../refs/heads/main/understand-anything-plugin/.openclaw/INSTALL.md
NEW: .../refs/heads/main/.openclaw/INSTALL.md

OLD: .../refs/heads/main/understand-anything-plugin/.opencode/INSTALL.md
NEW: .../refs/heads/main/.opencode/INSTALL.md
```

**Passo 3: Substituir a seção do Cursor**

Substitua a seção de instalação dirigida por IA do Cursor por:

```markdown
### Cursor

Cursor auto-discovers the plugin via `.cursor-plugin/plugin.json` when this repo is cloned. No manual installation needed — just clone and open in Cursor.
```

**Passo 4: Commit**

```bash
git add README.md
git commit -m "docs: update multi-platform URLs after moving configs to root"
```

---

### Tarefa 7: Verificar que tudo funciona

**Passo 1: Verificar configs de plataforma na raiz**

```bash
ls .codex/INSTALL.md .opencode/INSTALL.md .openclaw/INSTALL.md
ls .cursor-plugin/plugin.json .claude-plugin/plugin.json
```

Todos devem existir.

**Passo 2: Verificar que a fonte do plugin está intacta**

```bash
ls understand-anything-plugin/skills/understand/
ls understand-anything-plugin/agents/
ls understand-anything-plugin/packages/
```

Skills, agents e packages devem continuar existindo dentro do wrapper.

**Passo 3: Verificar que nenhuma config de plataforma permanece dentro do wrapper**

```bash
ls understand-anything-plugin/.codex/ 2>/dev/null    # should fail
ls understand-anything-plugin/.cursor/ 2>/dev/null   # should fail
ls understand-anything-plugin/.opencode/ 2>/dev/null # should fail
ls understand-anything-plugin/.openclaw/ 2>/dev/null # should fail
```

**Passo 4: Rodar os testes**

```bash
pnpm --filter @understand-anything/core build && pnpm --filter @understand-anything/core test
```

Todos os testes devem passar — só arquivos de config foram movidos, não código-fonte.

**Passo 5: Verificar que o marketplace.json está inalterado**

```bash
cat .claude-plugin/marketplace.json | grep source
```

Esperado: `"source": "./understand-anything-plugin"` — inalterado, ainda correto.

**Passo 6: Verificar que não há URLs raw do GitHub obsoletas**

```bash
grep -r "understand-anything-plugin/\." README.md
```

Esperado: 0 resultados (nenhuma URL apontando para localizações antigas de config de plataforma aninhadas).
