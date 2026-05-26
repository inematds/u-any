# Plano de Implementação da Homepage

> 🇬🇧 Versão original em inglês: [2026-03-15-homepage-implementation.md](./2026-03-15-homepage-implementation.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa por tarefa.

**Objetivo:** Construir uma homepage cinematográfica, com rolagem dirigida, para o Understand Anything usando Astro, publicada no GitHub Pages via branch `gh-pages`.

**Arquitetura:** Projeto Astro SSG em `homepage/` na main. Fontes auto-hospedadas (DM Serif Display, Inter, JetBrains Mono) com fallbacks robustos. Animações puramente em CSS disparadas por `IntersectionObserver`. Um workflow do GitHub Actions faz build e deploy para `gh-pages` a cada push.

**Tech Stack:** Astro 5, custom properties de CSS, JS vanilla, GitHub Actions

**Documento de design:** `docs/plans/2026-03-15-homepage-design.md`

---

### Tarefa 1: Inicializar o projeto Astro

**Arquivos:**
- Criar: `homepage/package.json`
- Criar: `homepage/astro.config.mjs`
- Criar: `homepage/tsconfig.json`
- Criar: `homepage/src/pages/index.astro` (placeholder)
- Criar: `homepage/src/layouts/Layout.astro` (placeholder)
- Criar: `homepage/public/.gitkeep`

**Passo 1: Inicializar projeto Astro**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything
mkdir -p homepage
cd homepage
pnpm create astro@latest . -- --template minimal --no-install --no-git --typescript strict
```

Se o prompt interativo travar, crie os arquivos manualmente.

**Passo 2: Configurar o Astro para o GitHub Pages**

Edite `homepage/astro.config.mjs`:

```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://lum1104.github.io',
  base: '/Understand-Anything',
});
```

**Passo 3: Verificar que o projeto compila**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm install
pnpm build
```

Esperado: build com sucesso, diretório `dist/` criado.

**Passo 4: Commit**

```bash
git add homepage/
git commit -m "feat(homepage): scaffold Astro project with GitHub Pages config"
```

---

### Tarefa 2: Fontes auto-hospedadas e CSS base

**Arquivos:**
- Criar: `homepage/public/fonts/DMSerifDisplay-Regular.woff2`
- Criar: `homepage/public/fonts/Inter-Regular.woff2`
- Criar: `homepage/public/fonts/Inter-SemiBold.woff2`
- Criar: `homepage/public/fonts/JetBrainsMono-Regular.woff2`
- Criar: `homepage/src/styles/global.css`

**Passo 1: Baixar os arquivos de fonte**

Baixe os arquivos WOFF2 da API do Google Fonts (ou do fontsource). Coloque-os em `homepage/public/fonts/`. Arquivos necessários:
- DM Serif Display Regular (woff2)
- Inter Regular + SemiBold (woff2)
- JetBrains Mono Regular (woff2)

Use curl para baixar do CDN do fontsource ou da API de CSS do Google Fonts. Exemplo:

```bash
mkdir -p homepage/public/fonts
# Baixar do fontsource (CDN confiável)
curl -L "https://cdn.jsdelivr.net/fontsource/fonts/dm-serif-display@latest/latin-400-normal.woff2" -o homepage/public/fonts/DMSerifDisplay-Regular.woff2
curl -L "https://cdn.jsdelivr.net/fontsource/fonts/inter@latest/latin-400-normal.woff2" -o homepage/public/fonts/Inter-Regular.woff2
curl -L "https://cdn.jsdelivr.net/fontsource/fonts/inter@latest/latin-600-normal.woff2" -o homepage/public/fonts/Inter-SemiBold.woff2
curl -L "https://cdn.jsdelivr.net/fontsource/fonts/jetbrains-mono@latest/latin-400-normal.woff2" -o homepage/public/fonts/JetBrainsMono-Regular.woff2
```

Se o download falhar, tente URLs alternativas ou use `npx fontsource` para instalar localmente.

**Passo 2: Criar CSS global com design tokens e declarações de font-face**

Crie `homepage/src/styles/global.css`:

```css
/* Font declarations — self-hosted, no external CDN dependency */
@font-face {
  font-family: 'DM Serif Display';
  src: url('/Understand-Anything/fonts/DMSerifDisplay-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: 'Inter';
  src: url('/Understand-Anything/fonts/Inter-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: 'Inter';
  src: url('/Understand-Anything/fonts/Inter-SemiBold.woff2') format('woff2');
  font-weight: 600;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: 'JetBrains Mono';
  src: url('/Understand-Anything/fonts/JetBrainsMono-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

/* Design tokens */
:root {
  --bg: #0a0a0a;
  --surface: #141414;
  --border: #1a1a1a;
  --accent: #d4a574;
  --accent-glow: rgba(212, 165, 116, 0.15);
  --text: #e8e2d8;
  --text-muted: #8a8578;

  --font-heading: 'DM Serif Display', Georgia, 'Times New Roman', serif;
  --font-body: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-code: 'JetBrains Mono', 'SF Mono', 'Cascadia Code', 'Fira Code', monospace;
}

/* Reset & base */
*, *::before, *::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html {
  scroll-behavior: smooth;
}

body {
  font-family: var(--font-body);
  background-color: var(--bg);
  color: var(--text);
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
  overflow-x: hidden;
}

/* Noise texture overlay */
body::before {
  content: '';
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 9999;
  opacity: 0.03;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)'/%3E%3C/svg%3E");
}

/* Scroll-reveal animation */
@keyframes fadeSlideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.reveal {
  opacity: 0;
}

.reveal.visible {
  animation: fadeSlideUp 0.8s ease-out forwards;
}

/* Stagger delays for feature cards */
.reveal-delay-1 { animation-delay: 0.1s; }
.reveal-delay-2 { animation-delay: 0.25s; }
.reveal-delay-3 { animation-delay: 0.4s; }

a {
  color: var(--accent);
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}
```

**Passo 3: Importar o CSS global no Layout**

Atualize `homepage/src/layouts/Layout.astro`:

```astro
---
interface Props {
  title: string;
}

const { title } = Astro.props;
---

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="Turn any codebase into an interactive knowledge graph you can explore, search, and learn from." />
    <link rel="icon" type="image/svg+xml" href="/Understand-Anything/favicon.svg" />
    <title>{title}</title>
  </head>
  <body>
    <slot />
  </body>
</html>

<style is:global>
  @import '../styles/global.css';
</style>
```

**Passo 4: Fazer build e verificar carregamento das fontes**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

Esperado: build com sucesso. Verifique se `dist/fonts/` contém os arquivos woff2.

**Passo 5: Commit**

```bash
git add homepage/public/fonts/ homepage/src/styles/global.css homepage/src/layouts/Layout.astro
git commit -m "feat(homepage): add self-hosted fonts and design token CSS"
```

---

### Tarefa 3: Componente de Nav Bar

**Arquivos:**
- Criar: `homepage/src/components/Nav.astro`

**Passo 1: Criar o componente de navegação**

Crie `homepage/src/components/Nav.astro`:

```astro
---
const githubUrl = 'https://github.com/Lum1104/Understand-Anything';
---

<nav class="nav" id="nav">
  <div class="nav-inner">
    <a href="/Understand-Anything/" class="nav-logo">Understand Anything</a>
    <div class="nav-links">
      <a href={githubUrl} target="_blank" rel="noopener noreferrer" class="nav-github">
        <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor" aria-hidden="true">
          <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0 0 24 12c0-6.63-5.37-12-12-12z"/>
        </svg>
        <span>GitHub</span>
      </a>
      <a href="#install" class="nav-cta">Get Started</a>
    </div>
  </div>
</nav>

<script>
  const nav = document.getElementById('nav');
  if (nav) {
    window.addEventListener('scroll', () => {
      nav.classList.toggle('scrolled', window.scrollY > 50);
    });
  }
</script>

<style>
  .nav {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 100;
    padding: 1rem 2rem;
    transition: background-color 0.3s ease, backdrop-filter 0.3s ease;
  }

  .nav.scrolled {
    background-color: rgba(10, 10, 10, 0.85);
    backdrop-filter: blur(12px);
    border-bottom: 1px solid var(--border);
  }

  .nav-inner {
    max-width: 1200px;
    margin: 0 auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .nav-logo {
    font-family: var(--font-heading);
    font-size: 1.25rem;
    color: var(--text);
    text-decoration: none;
  }

  .nav-logo:hover {
    text-decoration: none;
    color: var(--accent);
  }

  .nav-links {
    display: flex;
    align-items: center;
    gap: 1.5rem;
  }

  .nav-github {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    color: var(--text-muted);
    font-size: 0.9rem;
    text-decoration: none;
    transition: color 0.2s;
  }

  .nav-github:hover {
    color: var(--text);
    text-decoration: none;
  }

  .nav-cta {
    background: var(--accent);
    color: var(--bg);
    padding: 0.5rem 1.25rem;
    border-radius: 6px;
    font-weight: 600;
    font-size: 0.9rem;
    text-decoration: none;
    transition: box-shadow 0.3s ease;
  }

  .nav-cta:hover {
    text-decoration: none;
    box-shadow: 0 0 20px var(--accent-glow);
  }

  @media (max-width: 480px) {
    .nav { padding: 0.75rem 1rem; }
    .nav-github span { display: none; }
  }
</style>
```

**Passo 2: Adicionar Nav ao index.astro (teste temporário)**

Atualize `homepage/src/pages/index.astro`:

```astro
---
import Layout from '../layouts/Layout.astro';
import Nav from '../components/Nav.astro';
---

<Layout title="Understand Anything">
  <Nav />
  <main style="height: 200vh; padding-top: 100px;">
    <p>Nav test — scroll to see transparency change</p>
  </main>
</Layout>
```

**Passo 3: Fazer build e verificar**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

Esperado: build com sucesso.

**Passo 4: Commit**

```bash
git add homepage/src/components/Nav.astro homepage/src/pages/index.astro
git commit -m "feat(homepage): add floating nav bar with scroll effect"
```

---

### Tarefa 4: Componente da seção Hero

**Arquivos:**
- Criar: `homepage/src/components/Hero.astro`
- Copiar: `assets/hero.jpg` → `homepage/public/images/hero.jpg`

**Passo 1: Copiar a imagem da hero**

```bash
mkdir -p /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage/public/images
cp /Users/yuxianglin/Desktop/opensource/Understand-Anything/assets/hero.jpg /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage/public/images/hero.jpg
```

**Passo 2: Criar o componente Hero**

Crie `homepage/src/components/Hero.astro`:

```astro
---
const githubUrl = 'https://github.com/Lum1104/Understand-Anything';
---

<section class="hero">
  <div class="hero-bg">
    <img src="/Understand-Anything/images/hero.jpg" alt="" class="hero-bg-img" loading="eager" />
    <div class="hero-overlay"></div>
  </div>
  <div class="hero-content">
    <h1 class="hero-title">Understand Any Codebase</h1>
    <p class="hero-sub">
      Turn 200,000 lines of code into an interactive knowledge graph you can
      explore, search, and learn from — powered by multi-agent AI analysis.
    </p>
    <div class="hero-actions">
      <a href="#install" class="hero-cta">Get Started</a>
      <a href={githubUrl} target="_blank" rel="noopener noreferrer" class="hero-secondary">
        View on GitHub &rarr;
      </a>
    </div>
  </div>
</section>

<style>
  .hero {
    position: relative;
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    overflow: hidden;
  }

  .hero-bg {
    position: absolute;
    inset: 0;
    z-index: 0;
  }

  .hero-bg-img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0.25;
  }

  .hero-overlay {
    position: absolute;
    inset: 0;
    background: linear-gradient(
      to bottom,
      rgba(10, 10, 10, 0.3) 0%,
      rgba(10, 10, 10, 0.7) 60%,
      var(--bg) 100%
    );
  }

  .hero-content {
    position: relative;
    z-index: 1;
    max-width: 800px;
    padding: 2rem;
    animation: fadeSlideUp 1s ease-out;
  }

  .hero-title {
    font-family: var(--font-heading);
    font-size: clamp(2.5rem, 6vw, 4.5rem);
    color: var(--text);
    line-height: 1.1;
    margin-bottom: 1.5rem;
    text-shadow: 0 0 60px var(--accent-glow);
  }

  .hero-sub {
    font-size: clamp(1rem, 2vw, 1.25rem);
    color: var(--text-muted);
    line-height: 1.7;
    max-width: 600px;
    margin: 0 auto 2.5rem;
  }

  .hero-actions {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 2rem;
    flex-wrap: wrap;
  }

  .hero-cta {
    background: var(--accent);
    color: var(--bg);
    padding: 0.85rem 2.5rem;
    border-radius: 8px;
    font-weight: 600;
    font-size: 1.1rem;
    text-decoration: none;
    transition: box-shadow 0.3s ease, transform 0.2s ease;
  }

  .hero-cta:hover {
    text-decoration: none;
    box-shadow: 0 0 30px var(--accent-glow), 0 0 60px rgba(212, 165, 116, 0.08);
    transform: translateY(-2px);
  }

  .hero-secondary {
    color: var(--text-muted);
    font-size: 1rem;
    text-decoration: none;
    transition: color 0.2s;
  }

  .hero-secondary:hover {
    color: var(--accent);
    text-decoration: none;
  }

  @media (max-width: 480px) {
    .hero-actions { flex-direction: column; gap: 1rem; }
  }
</style>
```

**Passo 3: Fazer build e verificar**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

**Passo 4: Commit**

```bash
git add homepage/src/components/Hero.astro homepage/public/images/hero.jpg
git commit -m "feat(homepage): add full-viewport hero section with gradient overlay"
```

---

### Tarefa 5: Componente de vitrine do Dashboard

**Arquivos:**
- Criar: `homepage/src/components/Showcase.astro`
- Copiar: `assets/overview.png` → `homepage/public/images/overview.png`

**Passo 1: Copiar o screenshot do dashboard**

```bash
cp /Users/yuxianglin/Desktop/opensource/Understand-Anything/assets/overview.png /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage/public/images/overview.png
```

**Passo 2: Criar o componente Showcase**

Crie `homepage/src/components/Showcase.astro`:

```astro
<section class="showcase">
  <p class="showcase-label reveal">See your codebase come alive</p>
  <div class="showcase-frame reveal reveal-delay-1">
    <div class="showcase-titlebar">
      <span class="dot red"></span>
      <span class="dot yellow"></span>
      <span class="dot green"></span>
    </div>
    <img
      src="/Understand-Anything/images/overview.png"
      alt="Understand Anything dashboard showing an interactive knowledge graph of a codebase"
      class="showcase-img"
      loading="lazy"
    />
  </div>
</section>

<style>
  .showcase {
    padding: 4rem 2rem 6rem;
    max-width: 1100px;
    margin: 0 auto;
    text-align: center;
  }

  .showcase-label {
    font-family: var(--font-heading);
    font-size: clamp(1.25rem, 3vw, 1.75rem);
    color: var(--text-muted);
    margin-bottom: 2.5rem;
  }

  .showcase-frame {
    border-radius: 12px;
    overflow: hidden;
    background: var(--surface);
    border: 1px solid var(--border);
    box-shadow:
      0 0 40px var(--accent-glow),
      0 25px 50px rgba(0, 0, 0, 0.5);
  }

  .showcase-titlebar {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 10px 14px;
    background: rgba(20, 20, 20, 0.8);
    border-bottom: 1px solid var(--border);
  }

  .dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
  }

  .dot.red { background: #ff5f57; }
  .dot.yellow { background: #ffbd2e; }
  .dot.green { background: #28c840; }

  .showcase-img {
    width: 100%;
    height: auto;
    display: block;
  }

  @media (max-width: 768px) {
    .showcase { padding: 2rem 1rem 4rem; }
  }
</style>
```

**Passo 3: Fazer build e verificar**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

**Passo 4: Commit**

```bash
git add homepage/src/components/Showcase.astro homepage/public/images/overview.png
git commit -m "feat(homepage): add dashboard showcase with browser frame and gold glow"
```

---

### Tarefa 6: Componente de cards de feature

**Arquivos:**
- Criar: `homepage/src/components/Features.astro`

**Passo 1: Criar o componente Features**

Crie `homepage/src/components/Features.astro`:

```astro
---
const features = [
  {
    icon: '◈',
    title: 'Interactive Knowledge Graph',
    description: 'Visualize files, functions, and dependencies as an explorable graph with smart layout.',
  },
  {
    icon: '¶',
    title: 'Plain-English Summaries',
    description: 'Every node explained in language anyone can understand — from junior devs to product managers.',
  },
  {
    icon: '⟐',
    title: 'Guided Tours',
    description: 'AI-generated walkthroughs that teach you the codebase step by step.',
  },
];
---

<section class="features">
  <div class="features-grid">
    {features.map((f, i) => (
      <div class={`feature-card reveal reveal-delay-${i + 1}`}>
        <span class="feature-icon">{f.icon}</span>
        <h3 class="feature-title">{f.title}</h3>
        <p class="feature-desc">{f.description}</p>
      </div>
    ))}
  </div>
</section>

<style>
  .features {
    padding: 2rem 2rem 6rem;
    max-width: 1100px;
    margin: 0 auto;
  }

  .features-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1.5rem;
  }

  .feature-card {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 12px;
    padding: 2rem;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
    backdrop-filter: blur(8px);
  }

  .feature-card:hover {
    border-color: rgba(212, 165, 116, 0.3);
    box-shadow: 0 0 20px var(--accent-glow);
  }

  .feature-icon {
    display: block;
    font-size: 1.75rem;
    color: var(--accent);
    margin-bottom: 1rem;
  }

  .feature-title {
    font-family: var(--font-heading);
    font-size: 1.25rem;
    color: var(--text);
    margin-bottom: 0.75rem;
  }

  .feature-desc {
    font-size: 0.95rem;
    color: var(--text-muted);
    line-height: 1.6;
  }

  @media (max-width: 768px) {
    .features-grid {
      grid-template-columns: 1fr;
      gap: 1rem;
    }
  }
</style>
```

**Passo 2: Fazer build e verificar**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

**Passo 3: Commit**

```bash
git add homepage/src/components/Features.astro
git commit -m "feat(homepage): add feature cards with glass-morphism and staggered reveal"
```

---

### Tarefa 7: Componente de CTA de instalação

**Arquivos:**
- Criar: `homepage/src/components/Install.astro`

**Passo 1: Criar o componente Install**

Crie `homepage/src/components/Install.astro`:

```astro
<section class="install" id="install">
  <div class="install-inner reveal">
    <h2 class="install-title">Get started in 30 seconds</h2>
    <div class="install-code">
      <div class="install-code-header">
        <span class="install-code-dot"></span>
        <span class="install-code-label">Claude Code</span>
      </div>
      <pre><code><span class="cmd">/plugin marketplace add</span> Lum1104/Understand-Anything
<span class="cmd">/plugin install</span> understand-anything
<span class="cmd">/understand</span></code></pre>
    </div>
    <p class="install-note">Works with <strong>Claude Code</strong> — Anthropic's official CLI for Claude.</p>
  </div>
</section>

<style>
  .install {
    padding: 6rem 2rem;
    text-align: center;
  }

  .install-inner {
    max-width: 640px;
    margin: 0 auto;
  }

  .install-title {
    font-family: var(--font-heading);
    font-size: clamp(1.5rem, 3vw, 2.25rem);
    color: var(--text);
    margin-bottom: 2rem;
  }

  .install-code {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 12px;
    overflow: hidden;
    text-align: left;
    margin-bottom: 1.5rem;
  }

  .install-code-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.75rem 1rem;
    border-bottom: 1px solid var(--border);
  }

  .install-code-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--accent);
  }

  .install-code-label {
    font-size: 0.8rem;
    color: var(--text-muted);
    font-family: var(--font-code);
  }

  pre {
    padding: 1.25rem 1.5rem;
    margin: 0;
    overflow-x: auto;
  }

  code {
    font-family: var(--font-code);
    font-size: 0.95rem;
    line-height: 1.8;
    color: var(--text);
  }

  .cmd {
    color: var(--accent);
  }

  .install-note {
    font-size: 0.9rem;
    color: var(--text-muted);
  }

  .install-note strong {
    color: var(--text);
  }
</style>
```

**Passo 2: Fazer build e verificar**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

**Passo 3: Commit**

```bash
git add homepage/src/components/Install.astro
git commit -m "feat(homepage): add install CTA with styled code block"
```

---

### Tarefa 8: Componente de Footer

**Arquivos:**
- Criar: `homepage/src/components/Footer.astro`

**Passo 1: Criar o componente Footer**

Crie `homepage/src/components/Footer.astro`:

```astro
---
const githubUrl = 'https://github.com/Lum1104/Understand-Anything';
---

<footer class="footer">
  <div class="footer-inner">
    <span class="footer-logo">Understand Anything</span>
    <div class="footer-links">
      <a href={githubUrl} target="_blank" rel="noopener noreferrer">GitHub</a>
      <span class="footer-sep">·</span>
      <a href={`${githubUrl}/blob/main/LICENSE`} target="_blank" rel="noopener noreferrer">License</a>
    </div>
    <p class="footer-note">Built as a Claude Code plugin</p>
  </div>
</footer>

<style>
  .footer {
    padding: 3rem 2rem;
    border-top: 1px solid var(--border);
    text-align: center;
  }

  .footer-inner {
    max-width: 1200px;
    margin: 0 auto;
  }

  .footer-logo {
    font-family: var(--font-heading);
    font-size: 1.1rem;
    color: var(--text);
    display: block;
    margin-bottom: 0.75rem;
  }

  .footer-links {
    margin-bottom: 0.75rem;
  }

  .footer-links a {
    color: var(--text-muted);
    font-size: 0.85rem;
    text-decoration: none;
    transition: color 0.2s;
  }

  .footer-links a:hover {
    color: var(--accent);
  }

  .footer-sep {
    color: var(--border);
    margin: 0 0.5rem;
  }

  .footer-note {
    font-size: 0.8rem;
    color: var(--text-muted);
  }
</style>
```

**Passo 2: Fazer build e verificar**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

**Passo 3: Commit**

```bash
git add homepage/src/components/Footer.astro
git commit -m "feat(homepage): add minimal footer"
```

---

### Tarefa 9: Montar página completa + script de scroll-reveal

**Arquivos:**
- Modificar: `homepage/src/pages/index.astro`

**Passo 1: Montar todos os componentes na página index**

Substitua `homepage/src/pages/index.astro` por:

```astro
---
import Layout from '../layouts/Layout.astro';
import Nav from '../components/Nav.astro';
import Hero from '../components/Hero.astro';
import Showcase from '../components/Showcase.astro';
import Features from '../components/Features.astro';
import Install from '../components/Install.astro';
import Footer from '../components/Footer.astro';
---

<Layout title="Understand Anything — Turn any codebase into a knowledge graph">
  <Nav />
  <Hero />
  <Showcase />
  <Features />
  <Install />
  <Footer />
</Layout>

<script>
  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          entry.target.classList.add('visible');
          observer.unobserve(entry.target);
        }
      });
    },
    { threshold: 0.15 }
  );

  document.querySelectorAll('.reveal').forEach((el) => observer.observe(el));
</script>
```

**Passo 2: Fazer build e verificar a página completa**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm build
```

Esperado: build com sucesso. `dist/index.html` contém todas as seções.

**Passo 3: Pré-visualizar localmente**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm preview
```

Abra no navegador e verifique: Nav, Hero, Showcase, Features, Install CTA e Footer renderizam corretamente. Animações de scroll disparam. Nav fica sólida ao rolar.

**Passo 4: Commit**

```bash
git add homepage/src/pages/index.astro
git commit -m "feat(homepage): assemble full page with scroll-reveal observer"
```

---

### Tarefa 10: Workflow de deploy via GitHub Actions

**Arquivos:**
- Criar: `.github/workflows/deploy-homepage.yml`

**Passo 1: Criar o arquivo de workflow**

```bash
mkdir -p /Users/yuxianglin/Desktop/opensource/Understand-Anything/.github/workflows
```

Crie `.github/workflows/deploy-homepage.yml`:

```yaml
name: Deploy Homepage

on:
  push:
    branches: [main]
    paths:
      - 'homepage/**'
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 10

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
          cache-dependency-path: homepage/pnpm-lock.yaml

      - name: Install dependencies
        working-directory: homepage
        run: pnpm install

      - name: Build
        working-directory: homepage
        run: pnpm build

      - uses: actions/upload-pages-artifact@v3
        with:
          path: homepage/dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

**Passo 2: Commit**

```bash
git add .github/workflows/deploy-homepage.yml
git commit -m "ci: add GitHub Actions workflow for homepage deployment to Pages"
```

---

### Tarefa 11: Criar favicon

**Arquivos:**
- Criar: `homepage/public/favicon.svg`

**Passo 1: Criar um SVG favicon simples, dourado sobre preto**

Crie `homepage/public/favicon.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <rect width="32" height="32" rx="6" fill="#0a0a0a"/>
  <text x="16" y="23" font-family="Georgia, serif" font-size="20" fill="#d4a574" text-anchor="middle" font-weight="bold">U</text>
</svg>
```

**Passo 2: Commit**

```bash
git add homepage/public/favicon.svg
git commit -m "feat(homepage): add favicon"
```

---

### Tarefa 12: Verificação final de build + atualização do README

**Arquivos:**
- Modificar: `README.md` (adicionar link da homepage)

**Passo 1: Build limpo completo**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
rm -rf dist node_modules
pnpm install && pnpm build
```

Esperado: build com sucesso, sem warnings.

**Passo 2: Preview local e checagem manual**

```bash
cd /Users/yuxianglin/Desktop/opensource/Understand-Anything/homepage
pnpm preview
```

Verifique no navegador:
- [ ] A página carrega sem erros
- [ ] As fontes renderizam (ou caem em fallback graciosamente)
- [ ] A seção Hero ocupa toda a viewport com imagem de fundo
- [ ] O screenshot do dashboard aparece na moldura de navegador com brilho dourado
- [ ] Os feature cards aparecem em 3 colunas (1 coluna no mobile)
- [ ] O bloco de código de instalação mostra os comandos corretos
- [ ] Animações de scroll disparam ao rolar
- [ ] Nav fica sólida ao rolar
- [ ] Todos os links funcionam (GitHub, Get Started com scroll suave)
- [ ] Responsivo: teste em 480px e 768px

**Passo 3: Adicionar link da homepage ao README.md**

Adicione um link "Homepage" perto do topo do README, após a seção de badges. Adicione uma única linha:

```markdown
**[Homepage](https://understand-anything.com)** | **[GitHub](https://github.com/Lum1104/Understand-Anything)**
```

**Passo 4: Commit final**

```bash
git add README.md
git commit -m "docs: add homepage link to README"
```

**Passo 5: Configurar o GitHub Pages**

Após push para a main, vá em Settings → Pages do repositório no GitHub → Source: selecione "GitHub Actions". O workflow fará deploy automático no próximo push que tocar em `homepage/`.
