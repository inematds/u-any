# Understand Anything — Design da Homepage do Projeto

> 🇬🇧 Versão original em inglês: [2026-03-15-homepage-design.md](./2026-03-15-homepage-design.md)

**Data**: 2026-03-15
**Objetivo**: Atrair novos usuários para o plugin Understand Anything do Claude Code
**Abordagem**: "The Reveal" — site cinematográfico de página única dirigido por scroll

## Stack Tecnológica

- **Astro** (gerador de site estático, zero overhead de framework JS)
- **Fontes auto-hospedadas** (sem dependência do CDN do Google Fonts — funciona na China)
- **CSS** com variáveis batendo com o tema do dashboard
- **Vanilla JS** para animações de scroll com `IntersectionObserver`
- **GitHub Actions** para CI/CD para a branch `gh-pages`

## Código-fonte e Deploy

- Fonte: diretório `homepage/` na branch `main`
- Saída do build: deployada para a branch `gh-pages` via GitHub Actions
- URL: `understand-anything.com`

## Estrutura da Página (ordem de scroll)

### 1. Nav Bar
Nav flutuante minimalista. Logo/wordmark à esquerda, botão de star do GitHub + CTA "Get Started" à direita. Transparente, fica sólida ao rolar.

### 2. Hero (viewport inteira)
- Headline: **"Understand Any Codebase"**
- Sub-headline: "Turn 200,000 lines of code into an interactive knowledge graph you can explore, search, and learn from — powered by multi-agent AI analysis."
- CTA: "Get Started" (botão dourado, rola até a seção de install)
- Secundário: "View on GitHub" (link de texto)
- Fundo: `hero.jpg` com overlay gradiente escuro

### 3. Showcase do Dashboard
- Label: "See your codebase come alive"
- `overview.png` em um frame estilizado de browser com sombra de brilho dourado
- Fade-in ao rolar

### 4. Cards de Features (3 colunas)
Animação de fade-in escalonada:
1. **Interactive Knowledge Graph** — "Visualize files, functions, and dependencies as an explorable graph with smart layout."
2. **Plain-English Summaries** — "Every node explained in language anyone can understand — from junior devs to product managers."
3. **Guided Tours** — "AI-generated walkthroughs that teach you the codebase step by step."

### 5. CTA de Instalação
- Headline: "Get started in 30 seconds"
- Bloco de código:
  ```
  /plugin marketplace add Lum1104/Understand-Anything
  /plugin install understand-anything
  /understand
  ```
- Nota "Works with Claude Code"

### 6. Footer
- Wordmark "Understand Anything"
- Link do GitHub, licença
- "Built as a Claude Code plugin"

## Sistema de Design Visual

### Cores (batendo com o dashboard)
| Token | Valor | Uso |
|-------|-------|-------|
| `--bg` | `#0a0a0a` | Fundo da página |
| `--surface` | `#141414` | Fundo dos cards |
| `--border` | `#1a1a1a` | Bordas, divisores |
| `--accent` | `#d4a574` | Acento primário dourado/âmbar |
| `--text` | `#e8e2d8` | Texto primário (branco quente) |
| `--text-muted` | `#8a8578` | Texto secundário |

### Tipografia (auto-hospedada, com fallbacks)
- **Headings**: DM Serif Display → Georgia, "Times New Roman", serif
- **Body**: Inter → -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif
- **Code**: JetBrains Mono → "SF Mono", "Cascadia Code", "Fira Code", monospace
- Headline do hero: ~4rem serif com text-shadow sutil de brilho

### Efeitos
- Brilho dourado no frame do screenshot do dashboard (`box-shadow` com dourado em baixa opacidade)
- Overlay sutil de textura de ruído (SVG, batendo com o dashboard)
- Animações de fade+slide-up disparadas por scroll (CSS `@keyframes` + `IntersectionObserver`)
- Botão CTA: fundo dourado com pulso de brilho no hover
- Cards: glass-morphism com `backdrop-filter: blur`
- Responsivo: 768px (tablet), 480px (mobile)
