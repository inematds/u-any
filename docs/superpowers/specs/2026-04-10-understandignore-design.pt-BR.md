# Spec de Design do .understandignore

> 🇬🇧 Versão original em inglês: [2026-04-10-understandignore-design.md](./2026-04-10-understandignore-design.md)

## Visão Geral

Adicionar exclusão de arquivos configurável pelo usuário via arquivos `.understandignore`, usando sintaxe `.gitignore`. Isso torna a análise mais rápida pulando arquivos irrelevantes (código vendor, output gerado, fixtures de teste) sem modificar defaults hardcoded.

## Objetivos

- Permitir que usuários excluam arquivos/diretórios da análise via `.understandignore`
- Usar sintaxe `.gitignore` (familiar, sem curva de aprendizado)
- Manter defaults hardcoded como built-in — `.understandignore` adiciona padrões em cima
- Permitir negação `!` para forçar inclusão de arquivos excluídos pelos defaults
- Auto-gerar um arquivo starter comentado no primeiro run (código determinístico, não LLM)
- Pausar antes da análise para deixar o usuário revisar o arquivo ignore

## Não-Objetivos

- Substituir `.gitignore` — isso é específico de análise
- Arquivos `.understandignore` por diretório (apenas raiz do projeto e `.understand-anything/`)
- GUI para editar padrões de ignore

---

## Módulo IgnoreFilter

Novo arquivo: `packages/core/src/ignore-filter.ts`

Usa o pacote npm [`ignore`](https://www.npmjs.com/package/ignore) para casamento de padrões compatível com gitignore.

### API

```typescript
export interface IgnoreFilter {
  isIgnored(relativePath: string): boolean;
}

export function createIgnoreFilter(projectRoot: string): IgnoreFilter;
```

### Comportamento

`createIgnoreFilter` carrega padrões nesta ordem (entradas posteriores podem sobrepor as anteriores):

1. **Defaults hardcoded** — os padrões de exclusão existentes do project-scanner (node_modules/, .git/, dist/, build/, bin/, obj/, *.lock, *.min.js, etc.)
2. **`.understand-anything/.understandignore`** — em nível de projeto, vive junto ao output
3. **`.understandignore`** na raiz do projeto — localização alternativa para visibilidade

Padrões mesclam aditivamente. Negação `!` em arquivos do usuário pode sobrepor defaults hardcoded (ex.: `!dist/` força-inclui dist/).

### Padrões Default Hardcoded

Estes são os defaults built-in (batendo com o comportamento atual do project-scanner, mais bin/obj para .NET):

```
# Dependency directories
node_modules/
.git/
vendor/
venv/
.venv/
__pycache__/

# Build output
dist/
build/
out/
coverage/
.next/
.cache/
.turbo/
target/
bin/
obj/

# Lock files
*.lock
package-lock.json
yarn.lock
pnpm-lock.yaml

# Binary/asset files
*.png
*.jpg
*.jpeg
*.gif
*.svg
*.ico
*.woff
*.woff2
*.ttf
*.eot
*.mp3
*.mp4
*.pdf
*.zip
*.tar
*.gz

# Generated files
*.min.js
*.min.css
*.map
*.generated.*

# IDE/editor
.idea/
.vscode/

# Misc
LICENSE
.gitignore
.editorconfig
.prettierrc
.eslintrc*
*.log
```

---

## Gerador de Arquivo Starter

Novo arquivo: `packages/core/src/ignore-generator.ts`

### API

```typescript
export function generateStarterIgnoreFile(projectRoot: string): string;
```

### Comportamento

- Código determinístico — escaneia o diretório do projeto por padrões comuns
- Retorna o conteúdo do arquivo como string (o caller escreve no disco)
- Todas as sugestões vêm **comentadas** — o usuário precisa descomentar para ativar
- Comentário de cabeçalho explica o arquivo, sintaxe e defaults built-in

### Lógica de Detecção

| Se existir | Sugerir |
|-----------|---------|
| `__tests__/` ou arquivos `*.test.*` | `# __tests__/`, `# *.test.*`, `# *.spec.*` |
| `fixtures/` ou `testdata/` | `# fixtures/`, `# testdata/` |
| `test/` ou `tests/` | `# test/`, `# tests/` |
| `.storybook/` | `# .storybook/` |
| `docs/` | `# docs/` |
| `examples/` | `# examples/` |
| `scripts/` | `# scripts/` |
| `migrations/` | `# migrations/` |
| arquivos `*.snap` | `# *.snap` |
| `bin/` (não-.NET, ou seja, shell scripts) | `# bin/` |
| `obj/` | `# obj/` |

### Formato do Arquivo Gerado

```
# .understandignore — patterns for files/dirs to exclude from analysis
# Syntax: same as .gitignore (globs, # comments, ! negation, trailing / for dirs)
# Lines below are suggestions — uncomment to activate.
# Use ! prefix to force-include something excluded by defaults.
#
# Built-in defaults (always excluded unless negated):
#   node_modules/, .git/, dist/, build/, bin/, obj/, *.lock, *.min.js, etc.
#

# --- Suggested exclusions (uncomment to activate) ---

# Test files
# __tests__/
# *.test.*
# *.spec.*

# Test data
# fixtures/
# testdata/

# Documentation
# docs/

# ... (more suggestions based on detection)
```

Só gerado se `.understand-anything/.understandignore` ainda não existir.

---

## Integração com a Skill

### Fase 0.5: Setup de Ignore (nova fase no SKILL.md)

Adicionada entre Pre-flight (Fase 0) e SCAN (Fase 1):

1. Verifica se `.understand-anything/.understandignore` existe
2. Se não, roda `generateStarterIgnoreFile(projectRoot)` e escreve o resultado em `.understand-anything/.understandignore`
3. Reporta ao usuário:
   - **Primeiro run:** "Generated `.understand-anything/.understandignore` with suggested exclusions. Please review it and uncomment any patterns you'd like to exclude. When ready, confirm to continue."
   - **Runs subsequentes:** "Found `.understand-anything/.understandignore`. Review it if needed, then confirm to continue."
4. Espera confirmação do usuário antes de prosseguir

### Mudanças na Fase 1: SCAN

O script de scan do agente `project-scanner` é atualizado para:

1. Coletar arquivos via `git ls-files` (ou fallback)
2. Aplicar o filtro de padrão hardcoded do agente (Camada 1 — comportamento existente)
3. Aplicar `IgnoreFilter` do core (Camada 2 — padrões do usuário)
4. Adicionar contagem `filteredByIgnore` à saída do scan
5. Reportar: "Scanned {totalFiles} files ({filteredByIgnore} excluded by .understandignore)"

Filtragem em duas camadas:
- **Camada 1:** Padrões hardcoded do agente no prompt (filtro rápido, grosso)
- **Camada 2:** `IgnoreFilter` do core (código determinístico, configurável pelo usuário)

---

## Atualização do Agente Project Scanner

Mudanças em `understand-anything-plugin/agents/project-scanner.md`:

- Após a lista de arquivos ser construída e a filtragem de Camada 1 aplicada, o agente roda um script Node.js que importa `createIgnoreFilter` de `@understand-anything/core` e filtra os paths restantes
- O JSON de resultado do scan inclui um novo campo `filteredByIgnore: number`
- Padrões de exclusão hardcoded existentes no prompt do agente permanecem para compatibilidade retroativa

---

## Testes

### `packages/core/src/__tests__/ignore-filter.test.ts`

- Parseia padrões glob básicos (`*.log`, `dist/`)
- Lida com comentários `#` e linhas em branco
- Lida com negação `!` (force-include)
- Lida com casamento recursivo `**/`
- Lida com `/` final para padrões só de diretório
- Mescla defaults + padrões do usuário corretamente
- `!` em arquivo do usuário sobrepõe defaults hardcoded
- Retorna `false` para paths que não casam com nenhum padrão

### `packages/core/src/__tests__/ignore-generator.test.ts`

- Gera arquivo starter com comentário de cabeçalho
- Detecta diretórios existentes e sugere padrões relevantes
- Todas as sugestões vêm comentadas (prefixadas com `# `)
- Não sobrescreve arquivo existente
- Inclui sugestões de bin/obj quando relevantes

---

## Estrutura de Arquivos

| Arquivo | Propósito |
|------|---------|
| `packages/core/src/ignore-filter.ts` | Parsear .understandignore, mesclar com defaults, filtrar paths |
| `packages/core/src/ignore-generator.ts` | Gerar arquivo starter escaneando estrutura do projeto |
| `packages/core/src/__tests__/ignore-filter.test.ts` | Testes da lógica de filtro |
| `packages/core/src/__tests__/ignore-generator.test.ts` | Testes do gerador |
| `agents/project-scanner.md` | Adicionar filtragem Camada 2 via IgnoreFilter |
| `skills/understand/SKILL.md` | Adicionar Fase 0.5 (gerar + pausar para revisão) |
| `packages/core/package.json` | Adicionar dependência npm `ignore` |
