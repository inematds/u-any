# Suporte Multi-Plataforma de Skills — Design Simplificado

> 🇬🇧 Versão original em inglês: [2026-03-18-multi-platform-simple-design.md](./2026-03-18-multi-platform-simple-design.md)

**Data**: 2026-03-18
**Status**: Aprovado
**Objetivo**: Fazer as skills do Understand-Anything funcionarem em Codex, OpenClaw, OpenCode e Cursor sem nenhuma etapa de build — os mesmos arquivos em todo lugar.

## Princípios de Design

Segue o padrão de [obra/superpowers](https://github.com/obra/superpowers):
1. **Mesmos arquivos, todas as plataformas** — sem marcadores de template, sem etapa de build, sem variantes específicas de plataforma
2. **`model: inherit`** — agentes usam o modelo da sessão pai, ficando agnósticos de plataforma
3. **Instalação orientada por IA** — arquivos `.{platform}/INSTALL.md` que o agente de IA lê e executa
4. **Skills auto-contidas** — templates de prompt do pipeline vivem dentro do diretório da skill, não em uma pasta `agents/` separada

## Mudança 1: Mover os Agentes do Pipeline Para Dentro da Skill

Os 5 agentes do pipeline (project-scanner, file-analyzer, architecture-analyzer, tour-builder, graph-reviewer) são usados exclusivamente pela skill `/understand`. Eles viram templates de prompt co-localizados com a skill:

**Antes:**
```
agents/
  project-scanner.md          # agent definition
  file-analyzer.md
  architecture-analyzer.md
  tour-builder.md
  graph-reviewer.md
skills/understand/
  SKILL.md                    # dispatches named agents
```

**Depois:**
```
skills/understand/
  SKILL.md                           # dispatches subagents using templates
  project-scanner-prompt.md          # prompt template (no agent frontmatter)
  file-analyzer-prompt.md
  architecture-analyzer-prompt.md
  tour-builder-prompt.md
  graph-reviewer-prompt.md
```

Os arquivos de template do prompt mantêm o conteúdo completo das instruções mas removem o frontmatter de agente (`name`, `tools`, `model`). O dispatch no `SKILL.md` muda de "Dispatch the **project-scanner** agent" para "Dispatch a subagent using the template at `./project-scanner-prompt.md`".

### Custo de Contexto

Ler templates pela sessão principal adiciona ~11K tokens no total (~5.5% de um contexto de 200K). Isso é sequencial (um template por vez), e a compressão de contexto recupera conteúdo anterior. Trade-off aceitável pela portabilidade.

## Mudança 2: Novo Agente Registrado — knowledge-graph-guide

Criar um agente reutilizável que qualquer skill ou usuário pode invocar para trabalhar com grafos de conhecimento:

```yaml
# agents/knowledge-graph-guide.md
---
name: knowledge-graph-guide
description: |
  Use this agent when users need help understanding, querying, or working
  with an Understand-Anything knowledge graph. Guides users through graph
  structure, node/edge relationships, layer architecture, tours, and
  dashboard usage.
model: inherit
---
```

Esse agente conhece:
- O esquema JSON do KnowledgeGraph (nodes, edges, layers, tours)
- Os 5 tipos de nó e 18 tipos de aresta
- Como navegar e consultar o grafo
- Como usar o dashboard interativo
- Como interpretar camadas arquiteturais e tours guiados

## Mudança 3: Arquivos de Instalação por Plataforma

Cada plataforma ganha um `INSTALL.md` que o agente de IA pode buscar e seguir:

| Arquivo | Plataforma | Mecanismo de Instalação |
|------|----------|-------------------|
| `.codex/INSTALL.md` | Codex | `git clone` + symlink para `~/.agents/skills/` |
| `.opencode/INSTALL.md` | OpenCode | Config de plugin em `opencode.json` |
| `.openclaw/INSTALL.md` | OpenClaw | `git clone` + symlink para `~/.openclaw/skills/` |
| `.cursor/INSTALL.md` | Cursor | `git clone` + symlink para `.cursor/plugins/` |

O usuário diz ao agente uma linha:
```
Fetch and follow instructions from https://raw.githubusercontent.com/Lum1104/Understand-Anything/refs/heads/main/understand-anything-plugin/.codex/INSTALL.md
```

O agente executa o clone + symlink/config automaticamente.

## Mudança 4: Atualização do README

Adicionar uma seção "Multi-Platform Installation" ao README.md com um one-liner por plataforma.

## Resumo de Arquivos

| Ação | Arquivos |
|--------|-------|
| Deletar | `agents/project-scanner.md`, `agents/file-analyzer.md`, `agents/architecture-analyzer.md`, `agents/tour-builder.md`, `agents/graph-reviewer.md` |
| Criar | `skills/understand/project-scanner-prompt.md`, `skills/understand/file-analyzer-prompt.md`, `skills/understand/architecture-analyzer-prompt.md`, `skills/understand/tour-builder-prompt.md`, `skills/understand/graph-reviewer-prompt.md` |
| Criar | `agents/knowledge-graph-guide.md` |
| Criar | `.codex/INSTALL.md`, `.opencode/INSTALL.md`, `.openclaw/INSTALL.md`, `.cursor/INSTALL.md` |
| Modificar | `skills/understand/SKILL.md` (referências de dispatch) |
| Modificar | `README.md` (seção multi-plataforma) |

## O Que Não Precisamos

- ~~`platforms/platform-config.json`~~ — mesmos arquivos em todo lugar
- ~~`platforms/build.mjs`~~ — sem etapa de build
- ~~marcadores de template `{{MARKER}}`~~ — sem templating
- ~~`scripts/install-*.sh`~~ — o agente de IA segue o INSTALL.md
- ~~`dist-platforms/`~~ — sem saída gerada

## Compatibilidade de Plataformas

| Plataforma | Método de Install | Descoberta de Agente | Descoberta de Skill |
|----------|---------------|-----------------|-----------------|
| Claude Code | Marketplace (existente) | dir `agents/` | dir `skills/` |
| Codex | INSTALL.md → symlink | N/A (templates na skill) | `~/.agents/skills/` |
| OpenCode | INSTALL.md → config de plugin | N/A (templates na skill) | Plugin se auto-registra |
| OpenClaw | INSTALL.md → symlink | N/A (templates na skill) | `~/.openclaw/skills/` |
| Cursor | INSTALL.md → symlink | dir `agents/` | `.cursor/plugins/` |
