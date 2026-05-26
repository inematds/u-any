# Design do Sistema de Temas

> 🇬🇧 Versão original em inglês: [2026-03-26-theme-system-design.md](./2026-03-26-theme-system-design.md)

## Visão Geral

Adicionar um sistema de presets de tema curado com customização de cor de acento ao dashboard. Usuários selecionam entre 5 presets de tema desenhados à mão e, opcionalmente, trocam a cor de acento dentro de cada preset a partir de um conjunto de 8-10 swatches testados.

### Objetivos
- Suportar 5 presets de tema: Dark Gold (atual), Dark Ocean, Dark Forest, Dark Rose, Light Minimal
- Permitir customização da cor de acento dentro de cada preset (apenas swatches curados, sem picker livre)
- Persistir a preferência de tema tanto no `localStorage` (pessoal) quanto no `meta.json` (nível de projeto)
- Manter coerência visual — sem combinações de cor que o usuário possa quebrar
- Troca de tema sem reload via injeção de variável CSS em runtime

### Não-Objetivos
- Color picker livre (risco de combos feias/ilegíveis)
- Overrides de cor por componente
- Múltiplos temas simultâneos

---

## 1. Presets de Tema e Sistema de Cores

### 1.1 Definições de Preset

Cada preset é um mapeamento completo de nomes de variável CSS para valores. Os 5 presets:

| Token | Dark Gold | Dark Ocean | Dark Forest | Dark Rose | Light Minimal |
|-------|-----------|------------|-------------|-----------|---------------|
| `--color-root` | `#0a0a0a` | `#0a0e14` | `#0a100a` | `#100a0a` | `#f5f3f0` |
| `--color-surface` | `#111111` | `#111820` | `#111811` | `#181111` | `#eae7e3` |
| `--color-elevated` | `#1a1a1a` | `#1a222c` | `#1a241a` | `#221a1a` | `#ffffff` |
| `--color-panel` | `#141414` | `#141c24` | `#141c14` | `#1c1414` | `#f0ede9` |
| `--color-gold`* | `#d4a574` | `#5ba4cf` | `#5ea67a` | `#cf7a8a` | `#4a6fa5` |
| `--color-gold-dim`* | `#c9a96e` | `#4e93ba` | `#4e9468` | `#b96e7e` | `#3d5f8f` |
| `--color-gold-bright`* | `#e8c49a` | `#7abce0` | `#78c492` | `#e094a4` | `#6088bf` |
| `--color-text-primary` | `#f5f0eb` | `#e8edf2` | `#ebf0eb` | `#f2e8ea` | `#1a1a1a` |
| `--color-text-secondary` | `#a39787` | `#87939f` | `#87a38f` | `#9f8790` | `#6b6b6b` |
| `--color-text-muted` | `#6b5f53` | `#536b7a` | `#536b5a` | `#6b535a` | `#a0a0a0` |
| `--color-border-subtle` | `rgba(212,165,116,0.12)` | `rgba(91,164,207,0.12)` | `rgba(94,166,122,0.12)` | `rgba(207,122,138,0.12)` | `rgba(74,111,165,0.10)` |
| `--color-border-medium` | `rgba(212,165,116,0.25)` | `rgba(91,164,207,0.25)` | `rgba(94,166,122,0.25)` | `rgba(207,122,138,0.25)` | `rgba(74,111,165,0.18)` |

*\* Os nomes das variáveis CSS permanecem `--color-gold`, `--color-gold-dim`, `--color-gold-bright` mesmo para temas não-dourados. Eles representam "a cor de acento" de forma genérica. Renomeá-los para `--color-accent` é um refactor que podemos fazer, mas não é obrigatório — o nome da variável é um detalhe de implementação invisível ao usuário.*

**Decisão: Renomear `--color-gold*` para `--color-accent*`** para evitar confusão. Isso é um find-and-replace pela base de código sem mudança de comportamento.

### 1.2 Efeitos de Vidro (Glass)

Efeitos de vidro derivam das cores base e precisam de valores por preset:

| Token | Temas escuros | Light Minimal |
|-------|-------------|---------------|
| `--glass-bg` | `rgba(20,20,20,0.8)` | `rgba(255,255,255,0.8)` |
| `--glass-bg-heavy` | `rgba(20,20,20,0.95)` | `rgba(255,255,255,0.95)` |
| `--glass-border` | `rgba(accent,0.1)` | `rgba(accent,0.08)` |
| `--glass-border-heavy` | `rgba(accent,0.15)` | `rgba(accent,0.12)` |

As classes CSS `.glass` e `.glass-heavy` vão referenciar essas variáveis em vez de valores hardcoded.

### 1.3 Cores de Scrollbar e Glow

Estas também derivam da cor de acento e precisam virar variáveis CSS:

| Token | Propósito |
|-------|---------|
| `--scrollbar-thumb` | `rgba(accent, 0.2)` |
| `--scrollbar-thumb-hover` | `rgba(accent, 0.35)` |
| `--glow-color` | `rgba(accent, 0.4)` para glow de seleção de nó |
| `--glow-pulse` | `rgba(accent, 0.6)` para pulso de destaque no tour |

### 1.4 Cores de Tipo de Nó e de Diff

Estas são **semânticas** e ficam fixas em todos os temas escuros:

| Variável | Valor | Propósito |
|----------|-------|---------|
| `--color-node-file` | `#4a7c9b` | Nós de arquivo |
| `--color-node-function` | `#5a9e6f` | Nós de função |
| `--color-node-class` | `#8b6fb0` | Nós de classe |
| `--color-node-module` | `#c9a06c` | Nós de módulo |
| `--color-node-concept` | `#b07a8a` | Nós de conceito |
| `--color-diff-changed` | `#e05252` | Nós alterados |
| `--color-diff-affected` | `#d4a030` | Nós afetados |

Para **Light Minimal apenas**, estes são levemente dessaturados/escurecidos para manter legibilidade em fundos claros:

| Variável | Valor Light Minimal |
|----------|-------------------|
| `--color-node-file` | `#3a6a87` |
| `--color-node-function` | `#488a5b` |
| `--color-node-class` | `#755d99` |
| `--color-node-module` | `#a88a56` |
| `--color-node-concept` | `#966674` |

### 1.5 Swatches de Acento

Cada preset oferece 8 opções de cor de acento. A primeira é o default "nativo" daquele preset. Cada swatch fornece 3 valores (accent, accent-dim, accent-bright) mais as opacidades de borda e glass auto-derivadas.

**Swatches de acento para temas escuros** (compartilhados pelos 4 presets escuros):

| Nome | Accent | Dim | Bright |
|------|--------|-----|--------|
| Gold | `#d4a574` | `#c9a96e` | `#e8c49a` |
| Ocean | `#5ba4cf` | `#4e93ba` | `#7abce0` |
| Emerald | `#5ea67a` | `#4e9468` | `#78c492` |
| Rose | `#cf7a8a` | `#b96e7e` | `#e094a4` |
| Purple | `#9b7abf` | `#876bb0` | `#b494d4` |
| Amber | `#c9963a` | `#b5862e` | `#ddb05c` |
| Teal | `#4aab9a` | `#3d9686` | `#68c4b4` |
| Silver | `#a0a8b0` | `#8e959c` | `#b8bfc6` |

**Swatches de acento para Light Minimal:**

| Nome | Accent | Dim | Bright |
|------|--------|-----|--------|
| Indigo | `#4a6fa5` | `#3d5f8f` | `#6088bf` |
| Ocean | `#3a8ab5` | `#2e7aa0` | `#55a0cc` |
| Emerald | `#3a8a5c` | `#2e7a4e` | `#55a878` |
| Rose | `#a5566a` | `#8f4a5c` | `#bf6e82` |
| Purple | `#6b5a9e` | `#5c4d8a` | `#8474b5` |
| Amber | `#9e7a30` | `#8a6a28` | `#b5923e` |
| Teal | `#2e8a7a` | `#267a6c` | `#45a595` |
| Slate | `#5a6570` | `#4e5860` | `#6e7a85` |

### 1.6 Derivação de Borda e Glass

Quando um swatch de acento é selecionado, bordas e efeitos de glass são auto-derivados:

```typescript
function deriveFromAccent(accentHex: string, isDark: boolean) {
  return {
    borderSubtle: `rgba(${hexToRgb(accentHex)}, ${isDark ? 0.12 : 0.10})`,
    borderMedium: `rgba(${hexToRgb(accentHex)}, ${isDark ? 0.25 : 0.18})`,
    glassBorder: `rgba(${hexToRgb(accentHex)}, ${isDark ? 0.1 : 0.08})`,
    glassBorderHeavy: `rgba(${hexToRgb(accentHex)}, ${isDark ? 0.15 : 0.12})`,
    scrollbarThumb: `rgba(${hexToRgb(accentHex)}, 0.2)`,
    scrollbarThumbHover: `rgba(${hexToRgb(accentHex)}, 0.35)`,
    glowColor: `rgba(${hexToRgb(accentHex)}, 0.4)`,
    glowPulse: `rgba(${hexToRgb(accentHex)}, 0.6)`,
  };
}
```

---

## 2. Arquitetura e Fluxo de Dados

### 2.1 Estrutura de Arquivos

```
packages/dashboard/src/
  themes/
    types.ts          # ThemePreset, AccentSwatch, ThemeConfig types
    presets.ts         # 5 preset definitions + accent swatch arrays
    theme-engine.ts   # applyTheme(), deriveFromAccent(), hexToRgb()
    ThemeContext.tsx    # React context + provider + useTheme() hook
  components/
    ThemePicker.tsx    # Popover UI for preset + accent selection
```

### 2.2 Definições de Tipo

```typescript
// themes/types.ts

export type PresetId = 'dark-gold' | 'dark-ocean' | 'dark-forest' | 'dark-rose' | 'light-minimal';

export interface ThemePreset {
  id: PresetId;
  name: string;           // Display name: "Dark Gold"
  isDark: boolean;         // true for dark themes, false for light
  colors: Record<string, string>;  // CSS variable name -> value (without --)
  accentSwatches: AccentSwatch[];
  defaultAccentId: string; // Which swatch is the native default
}

export interface AccentSwatch {
  id: string;              // e.g. 'gold', 'ocean'
  name: string;            // Display name: "Gold"
  accent: string;          // Primary accent hex
  accentDim: string;       // Dimmed accent hex
  accentBright: string;    // Bright accent hex
}

export interface ThemeConfig {
  presetId: PresetId;
  accentId: string;        // Selected accent swatch ID
}
```

### 2.3 Theme Engine

O theme engine é uma camada de função pura (sem dependência de React):

```typescript
// themes/theme-engine.ts

export function applyTheme(config: ThemeConfig): void {
  const preset = getPreset(config.presetId);
  const accent = getAccent(preset, config.accentId);

  // 1. Apply base preset colors
  for (const [key, value] of Object.entries(preset.colors)) {
    document.documentElement.style.setProperty(`--color-${key}`, value);
  }

  // 2. Override accent colors from swatch
  document.documentElement.style.setProperty('--color-accent', accent.accent);
  document.documentElement.style.setProperty('--color-accent-dim', accent.accentDim);
  document.documentElement.style.setProperty('--color-accent-bright', accent.accentBright);

  // 3. Apply derived values (borders, glass, scrollbar, glow)
  const derived = deriveFromAccent(accent.accent, preset.isDark);
  for (const [key, value] of Object.entries(derived)) {
    document.documentElement.style.setProperty(`--${key}`, value);
  }

  // 4. Set data-theme attribute for any CSS-only selectors needed
  document.documentElement.setAttribute('data-theme', preset.isDark ? 'dark' : 'light');
}
```

### 2.4 Contexto React

```typescript
// themes/ThemeContext.tsx

interface ThemeContextValue {
  config: ThemeConfig;
  preset: ThemePreset;
  setPreset: (presetId: PresetId) => void;
  setAccent: (accentId: string) => void;
}
```

O provider:
1. Ao montar: resolve o tema a partir de `localStorage` > campo `meta.json` no grafo carregado > default (`dark-gold`)
2. Chama `applyTheme()` em toda mudança de config
3. Persiste em `localStorage` a cada mudança
4. NÃO escreve em `meta.json` a partir do dashboard (o dashboard é somente leitura para `meta.json`; `meta.json` é escrito pelo lado do CLI/plugin)

### 2.5 Integração com Zustand Store

O sistema de temas é **separado da Zustand store** — usa seu próprio contexto React. Justificativa:
- Estado de tema é ortogonal ao estado de grafo/UI
- Tema precisa ser aplicado antes do grafo carregar (evita flash de tema errado)
- Mantém a store focada em interação com o grafo

A store NÃO ganha nenhum campo relacionado a tema.

---

## 3. Componentes de UI

### 3.1 Botão de Theme Picker (Header)

Um pequeno botão com ícone de paleta na barra superior, posicionado depois dos controles existentes (PersonaSelector, DiffToggle, etc.).

- Clique abre um popover/dropdown
- O popover tem duas seções:
  - **Presets**: 5 cards/botões mostrando nome do preset + pequenos círculos de preview de cor
  - **Cores de Acento**: linha de 8 círculos de cor para o preset ativo
- Preset e acento ativos são destacados com um ring/check
- Selecionar um preset aplica instantaneamente; selecionar um acento aplica instantaneamente
- Clicar fora ou apertar Escape fecha o popover

### 3.2 Preview de Preset

Cada card de preset mostra:
- Nome (ex.: "Dark Gold")
- 3-4 círculos pequenos mostrando root, surface e accent como preview visual
- Check ou ring no ativo

### 3.3 Linha de Swatches de Acento

- 8 círculos pequenos preenchidos em linha horizontal
- Tooltip ou label no hover mostrando o nome do acento
- O ativo tem um indicador de ring/borda

### 3.4 Transições

Ao trocar de tema:
- Variáveis CSS atualizam instantaneamente (sem transição necessária para a maioria das propriedades)
- Opcionalmente adicionar um `transition: background-color 0.2s, color 0.2s` sutil em `html` para uma sensação suave
- Sem reload de página

---

## 4. Persistência e Resolução

### 4.1 Locais de Armazenamento

| Local | Formato | Escrito por | Lido por |
|----------|--------|-----------|---------|
| chave `localStorage`: `ua-theme` | `JSON.stringify(ThemeConfig)` | Dashboard (a cada mudança) | Dashboard (ao montar) |
| `.understand-anything/meta.json` | `{ ..., theme?: ThemeConfig }` | CLI/plugin (durante análise ou set explícito) | Dashboard (ao montar, como fallback) |

### 4.2 Ordem de Resolução

```
1. localStorage('ua-theme')     → user's personal preference (wins)
2. meta.json.theme              → project-level default (fallback)
3. { presetId: 'dark-gold', accentId: 'gold' }  → hard default
```

### 4.3 Extensão do Esquema do meta.json

Estender `AnalysisMeta` em `packages/core/src/types.ts`:

```typescript
export interface AnalysisMeta {
  lastAnalyzedAt: string;
  gitCommitHash: string;
  version: string;
  analyzedFiles: number;
  theme?: ThemeConfig;      // NEW — optional, project-level theme preference
}
```

### 4.4 Dashboard Lê Tema do meta.json

O dashboard atualmente carrega `/knowledge-graph.json` ao montar. Ele também precisa carregar `/meta.json` (ou o campo de tema pode ser embutido em `knowledge-graph.json`).

**Decisão:** Carregar `/meta.json` separadamente — é um arquivo pequeno e mantém as preocupações separadas. O dashboard busca `/meta.json` ao montar, extrai o campo `theme` se presente, e o usa como fallback quando `localStorage` não tem tema.

---

## 5. Consolidação de Cores Hardcoded

### 5.1 Problema

Muitos componentes usam valores RGBA hardcoded em vez de variáveis CSS:
- `rgba(212,165,116,0.3)` espalhado em GraphView, CustomNode, etc.
- `rgba(20,20,20,0.8)` em efeitos de glass
- `rgba(224,82,82,0.25)` em overlays de diff

Esses não vão responder a mudanças de tema.

### 5.2 Solução

Antes de implementar a troca de tema, consolidar todas as referências de cor hardcoded:

1. **Auditoria**: grep por valores hex/rgba hardcoded em arquivos de componente
2. **Substituir por variáveis CSS**: criar novas variáveis onde necessário (ex.: `--edge-color`, `--edge-color-dim`)
3. **Classes de glass**: atualizar `.glass` e `.glass-heavy` em `index.css` para usar variáveis
4. **Scrollbar**: atualizar estilos de scrollbar para usar variáveis
5. **Efeitos de glow**: atualizar `.node-glow`, `.diff-changed-glow`, `.diff-affected-glow` para usar variáveis

Padrões hardcoded principais a consolidar:

| Valor Hardcoded | Substituir Por |
|-----------------|-------------|
| `rgba(212,165,116,X)` | `var(--color-accent)` com modificador de opacidade ou variável dedicada |
| `rgba(20,20,20,0.8)` | `var(--glass-bg)` |
| `rgba(20,20,20,0.95)` | `var(--glass-bg-heavy)` |
| `color="rgba(212,165,116,0.15)"` em React Flow | Referência a variável |
| Cores âmbar em WarningBanner | Manter como está (cor semântica de aviso, independente de tema) |

### 5.3 Rename de Variável CSS

Renomear pela base de código:
- `--color-gold` -> `--color-accent`
- `--color-gold-dim` -> `--color-accent-dim`
- `--color-gold-bright` -> `--color-accent-bright`
- Todos os usos de classe Tailwind: `text-gold` -> `text-accent`, `bg-gold` -> `bg-accent`, etc.

---

## 6. Considerações para o Tema Light

O tema Light Minimal requer atenção especial:

### 6.1 Contraste Invertido

- Texto é escuro sobre fundos claros (invertido em relação aos temas escuros)
- Bordas precisam de opacidade menor para não parecerem duras
- Efeitos de glass usam rgba baseado em branco em vez de preto

### 6.2 Cores de Nó

Variantes levemente mais escuras/dessaturadas para legibilidade em fundos claros (ver Seção 1.4).

### 6.3 Atributo data-theme

Setar `data-theme="light"` em `<html>` para quaisquer estilos que não dão para tratar puramente com variáveis CSS (ex.: overrides de componente de terceiros, direções de box-shadow).

### 6.4 React Flow

Cores de fundo, minimap e arestas do React Flow precisam respeitar o tema. O override existente com `!important` em `.react-flow__background` já usa `var(--color-root)`, o que é bom. Cores do MiniMap em `GraphView.tsx` estão atualmente hardcoded e precisam ser atualizadas.

---

## 7. Resumo das Mudanças por Pacote

### packages/core
- Estender o tipo `AnalysisMeta` com `theme?: ThemeConfig` opcional
- Exportar os tipos `ThemeConfig` e `PresetId` do subpath `./types`

### packages/dashboard
- Novo diretório `themes/` com types, presets, engine e contexto
- Novo componente `ThemePicker` no header
- Renomear `--color-gold*` para `--color-accent*` em todos os arquivos
- Consolidar valores RGBA hardcoded em variáveis CSS
- Atualizar `index.css`: classes de glass, scrollbar, efeitos de glow para usar variáveis
- Atualizar `App.tsx`: envolver com ThemeProvider, adicionar ThemePicker ao header, buscar meta.json
- Atualizar componentes com cores hardcoded: GraphView, CustomNode, LayerLegend, etc.

---

## 8. Fora de Escopo

- Import/export de tema
- UI de criação de tema customizado
- Customização de cor por nó
- Transições animadas de tema além de transições CSS simples
- Sincronização de tema entre abas do navegador (nice-to-have para depois)
