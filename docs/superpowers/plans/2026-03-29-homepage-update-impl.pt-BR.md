# Plano de Implementação — Atualização de Funcionalidades da Homepage

> 🇬🇧 Versão original em inglês: [2026-03-29-homepage-update-impl.md](./2026-03-29-homepage-update-impl.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa a tarefa.

**Objetivo:** Atualizar a homepage em Astro para refletir funcionalidades das releases v1.2.0–v2.0.0.

**Arquitetura:** Três edições de arquivo — expandir `Features.astro` de 3 para 6 cards, atualizar a nota de plataforma em `Install.astro` e atualizar o tagline em `Footer.astro`. Sem novos arquivos nem mudanças estruturais.

**Stack:** Astro 6, CSS grid

---

### Tarefa 1: Atualizar `Features.astro` — Substituir 3 Cards por 6

**Arquivos:**
- Modificar: `homepage/src/components/Features.astro`

**Passo 1: Substituir o array de features (linhas 2–18)**

Substitua todo o array `features` do frontmatter por:

```astro
---
const features = [
  {
    icon: '◈',
    title: 'Interactive Knowledge Graph',
    description: 'Visualize files, functions, and dependencies as an explorable graph with hierarchical drill-down and smart layout.',
  },
  {
    icon: '⬡',
    title: 'Beyond Code Analysis',
    description: 'Analyze your entire project — Dockerfiles, Terraform, SQL, Markdown, and 26+ file types mapped into one unified graph.',
  },
  {
    icon: '⊘',
    title: 'Smart Filtering & Search',
    description: 'Filter by node type, complexity, layer, or edge category. Fuzzy and semantic search to find anything instantly.',
  },
  {
    icon: '⎙',
    title: 'Export & Share',
    description: 'Export your knowledge graph as high-quality PNG, SVG, or filtered JSON — ready for docs, presentations, or further analysis.',
  },
  {
    icon: '⟿',
    title: 'Dependency Path Finder',
    description: 'Find the shortest path between any two components. Understand how parts of your system connect at a glance.',
  },
  {
    icon: '⟐',
    title: 'Guided Tours & Onboarding',
    description: 'AI-generated walkthroughs that teach the codebase step by step, plus onboarding guides for new team members.',
  },
];
---
```

**Passo 2: Atualizar a lógica de delay do reveal (linha 24)**

O atual `reveal-delay-${i + 1}` só tem CSS para delays 1–3. Com 6 cards em 2 linhas, use módulo para que cada linha escalone 1/2/3:

```astro
<div class={`feature-card reveal reveal-delay-${(i % 3) + 1}`}>
```

**Passo 3: Atualizar o CSS do grid para tratar 2 linhas corretamente**

Sem mudança — `grid-template-columns: repeat(3, 1fr)` já quebra para a segunda linha. O breakpoint mobile `1fr` também funciona. Nenhuma alteração de CSS necessária.

**Passo 4: Verificar build**

Rode: `cd homepage && npx astro build`
Esperado: build completa sem erros.

**Passo 5: Commit**

```bash
git add homepage/src/components/Features.astro
git commit -m "feat(homepage): expand features section to 6 cards for v2.0.0"
```

---

### Tarefa 2: Atualizar `Install.astro` — Nota Multi-Plataforma

**Arquivos:**
- Modificar: `homepage/src/components/Install.astro`

**Passo 1: Substituir a nota de plataforma (linha 13)**

Mudar:
```html
<p class="install-note">Works with <strong>Claude Code</strong> — Anthropic's official CLI for Claude.</p>
```

Para:
```html
<p class="install-note">Works with <strong>Claude Code</strong>, <strong>Codex</strong>, <strong>OpenCode</strong>, <strong>Gemini CLI</strong>, and more.</p>
```

**Passo 2: Commit**

```bash
git add homepage/src/components/Install.astro
git commit -m "feat(homepage): update install note for multi-platform support"
```

---

### Tarefa 3: Atualizar `Footer.astro` — Tagline

**Arquivos:**
- Modificar: `homepage/src/components/Footer.astro`

**Passo 1: Substituir o tagline (linha 13)**

Mudar:
```html
<p class="footer-note">Built as a Claude Code plugin</p>
```

Para:
```html
<p class="footer-note">Built for AI coding assistants</p>
```

**Passo 2: Verificar build completo**

Rode: `cd homepage && npx astro build`
Esperado: build limpa, sem erros.

**Passo 3: Commit**

```bash
git add homepage/src/components/Footer.astro
git commit -m "feat(homepage): update footer tagline for multi-platform"
```
