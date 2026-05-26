# Design da Atualização da Homepage — 2026-03-29

> 🇬🇧 Versão original em inglês: [2026-03-29-homepage-update-design.md](./2026-03-29-homepage-update-design.md)

## Objetivo

Atualizar a homepage Astro (`homepage/`) para refletir features adicionadas nas releases v1.2.0, v1.3.0 e v2.0.0. A estrutura/layout do README e da homepage permanecem inalterados.

## Escopo

Três áreas para atualizar:

### 1. Seção de Features — Expandir de 3 para 6 Cards

Atual (3 cards):
- Interactive Knowledge Graph
- Plain-English Summaries
- Guided Tours

Atualizado (6 cards, 2 linhas de 3):

| # | Título | Ícone | Descrição |
|---|-------|------|-------------|
| 1 | Interactive Knowledge Graph | `◈` | Visualize files, functions, and dependencies as an explorable graph with hierarchical drill-down and smart layout. |
| 2 | Beyond Code Analysis | `⬡` | Analyze your entire project — Dockerfiles, Terraform, SQL, Markdown, and 26+ file types mapped into one unified graph. |
| 3 | Smart Filtering & Search | `⊘` | Filter by node type, complexity, layer, or edge category. Fuzzy and semantic search to find anything instantly. |
| 4 | Export & Share | `⎙` | Export your knowledge graph as high-quality PNG, SVG, or filtered JSON — ready for docs, presentations, or further analysis. |
| 5 | Dependency Path Finder | `⟿` | Find the shortest path between any two components. Understand how parts of your system connect at a glance. |
| 6 | Guided Tours & Onboarding | `⟐` | AI-generated walkthroughs that teach the codebase step by step, plus onboarding guides for new team members. |

### 2. Seção de Install

Atualizar a nota de "só Claude Code" para multi-plataforma:
- Antes: "Works with Claude Code — Anthropic's official CLI for Claude."
- Depois: "Works with Claude Code, Codex, OpenCode, Gemini CLI, and more."

### 3. Footer

Atualizar a tagline:
- Antes: "Built as a Claude Code plugin"
- Depois: "Built for AI coding assistants"

## Arquivos para Modificar

- `homepage/src/components/Features.astro` — substituir 3 cards por 6
- `homepage/src/components/Install.astro` — atualizar nota de plataformas
- `homepage/src/components/Footer.astro` — atualizar tagline

## Fora de Escopo

- Atualizações do README.md
- Seção de showcase / screenshot
- Componente Nav
- Seção hero
- Mudanças de layout / estrutura global de CSS
