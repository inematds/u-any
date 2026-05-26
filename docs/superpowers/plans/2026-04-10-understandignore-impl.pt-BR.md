# Plano de Implementação do .understandignore

> 🇬🇧 Versão original em inglês: [2026-04-10-understandignore-impl.md](./2026-04-10-understandignore-impl.md)

> **Para workers agênticos:** SUB-SKILL OBRIGATÓRIA: Use superpowers:subagent-driven-development (recomendado) ou superpowers:executing-plans para implementar este plano tarefa a tarefa. Os passos usam sintaxe de checkbox (`- [ ]`) para acompanhamento.

**Objetivo:** Adicionar exclusão de arquivos configurável pelo usuário via arquivos `.understandignore` usando sintaxe `.gitignore`, com arquivos iniciais gerados automaticamente e uma pausa de revisão pré-análise.

**Arquitetura:** Um módulo `IgnoreFilter` em `packages/core` usa o pacote npm `ignore` para fazer parse de arquivos `.understandignore` e filtrar paths. Um `IgnoreGenerator` companheiro escaneia o projeto em busca de padrões comuns e produz um arquivo inicial com sugestões comentadas. O agente `project-scanner` aplica o filtro como uma segunda passagem após suas exclusões hardcoded existentes. A skill `/understand` adiciona uma Fase 0.5 que gera o arquivo inicial e pausa para revisão do usuário.

**Stack:** TypeScript, pacote npm `ignore`, Vitest

**Spec:** `docs/superpowers/specs/2026-04-10-understandignore-design.md`

---

## Estrutura de Arquivos

### Pacote core
- Criar: `understand-anything-plugin/packages/core/src/ignore-filter.ts` — faz parse do .understandignore, mescla com defaults, filtra paths
- Criar: `understand-anything-plugin/packages/core/src/ignore-generator.ts` — gera .understandignore inicial escaneando o projeto
- Criar: `understand-anything-plugin/packages/core/src/__tests__/ignore-filter.test.ts` — testes do filtro
- Criar: `understand-anything-plugin/packages/core/src/__tests__/ignore-generator.test.ts` — testes do gerador
- Modificar: `understand-anything-plugin/packages/core/src/index.ts` — exporta novos módulos
- Modificar: `understand-anything-plugin/packages/core/package.json` — adiciona dependência `ignore`

### Agentes & skills
- Modificar: `understand-anything-plugin/agents/project-scanner.md` — adiciona etapa de filtragem da Camada 2
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` — adiciona Fase 0.5

---

## Tarefa 1: Adicionar dependência `ignore`

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/package.json`

- [ ] **Passo 1: Instalar o pacote npm `ignore`**

Execute:
```bash
cd understand-anything-plugin && pnpm add --filter @understand-anything/core ignore
```

- [ ] **Passo 2: Verificar que foi adicionado**

Execute: `grep ignore understand-anything-plugin/packages/core/package.json`
Esperado: `"ignore": "^7.x.x"` (ou similar) em dependencies

- [ ] **Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/core/package.json understand-anything-plugin/pnpm-lock.yaml
git commit -m "chore(core): add ignore package for .understandignore support"
```

---

## Tarefa 2: Criar módulo IgnoreFilter com testes (TDD)

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/ignore-filter.ts`
- Criar: `understand-anything-plugin/packages/core/src/__tests__/ignore-filter.test.ts`

- [ ] **Passo 1: Escrever os testes que devem falhar**

Crie `understand-anything-plugin/packages/core/src/__tests__/ignore-filter.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { createIgnoreFilter, DEFAULT_IGNORE_PATTERNS } from "../ignore-filter";
import { mkdirSync, writeFileSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";

describe("IgnoreFilter", () => {
  let testDir: string;

  beforeEach(() => {
    testDir = join(tmpdir(), `ignore-filter-test-${Date.now()}`);
    mkdirSync(testDir, { recursive: true });
    mkdirSync(join(testDir, ".understand-anything"), { recursive: true });
  });

  afterEach(() => {
    rmSync(testDir, { recursive: true, force: true });
  });

  describe("DEFAULT_IGNORE_PATTERNS", () => {
    it("contains node_modules", () => {
      expect(DEFAULT_IGNORE_PATTERNS).toContain("node_modules/");
    });

    it("contains .git", () => {
      expect(DEFAULT_IGNORE_PATTERNS).toContain(".git/");
    });

    it("contains bin and obj for .NET", () => {
      expect(DEFAULT_IGNORE_PATTERNS).toContain("bin/");
      expect(DEFAULT_IGNORE_PATTERNS).toContain("obj/");
    });

    it("contains build output directories", () => {
      expect(DEFAULT_IGNORE_PATTERNS).toContain("dist/");
      expect(DEFAULT_IGNORE_PATTERNS).toContain("build/");
      expect(DEFAULT_IGNORE_PATTERNS).toContain("out/");
      expect(DEFAULT_IGNORE_PATTERNS).toContain("coverage/");
    });
  });

  describe("createIgnoreFilter with no user file", () => {
    it("ignores files matching default patterns", () => {
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("node_modules/foo/bar.js")).toBe(true);
      expect(filter.isIgnored("dist/index.js")).toBe(true);
      expect(filter.isIgnored(".git/config")).toBe(true);
      expect(filter.isIgnored("bin/Debug/app.dll")).toBe(true);
      expect(filter.isIgnored("obj/Release/net8.0/app.dll")).toBe(true);
    });

    it("does not ignore source files", () => {
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("src/index.ts")).toBe(false);
      expect(filter.isIgnored("README.md")).toBe(false);
      expect(filter.isIgnored("package.json")).toBe(false);
    });

    it("ignores lock files", () => {
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("pnpm-lock.yaml")).toBe(true);
      expect(filter.isIgnored("package-lock.json")).toBe(true);
      expect(filter.isIgnored("yarn.lock")).toBe(true);
    });

    it("ignores binary/asset files", () => {
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("logo.png")).toBe(true);
      expect(filter.isIgnored("font.woff2")).toBe(true);
      expect(filter.isIgnored("doc.pdf")).toBe(true);
    });

    it("ignores generated files", () => {
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("bundle.min.js")).toBe(true);
      expect(filter.isIgnored("style.min.css")).toBe(true);
      expect(filter.isIgnored("source.map")).toBe(true);
    });

    it("ignores IDE directories", () => {
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored(".idea/workspace.xml")).toBe(true);
      expect(filter.isIgnored(".vscode/settings.json")).toBe(true);
    });
  });

  describe("createIgnoreFilter with user .understandignore", () => {
    it("reads patterns from .understand-anything/.understandignore", () => {
      writeFileSync(
        join(testDir, ".understand-anything", ".understandignore"),
        "# Exclude tests\n__tests__/\n*.test.ts\n"
      );
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("__tests__/foo.test.ts")).toBe(true);
      expect(filter.isIgnored("src/utils.test.ts")).toBe(true);
      expect(filter.isIgnored("src/utils.ts")).toBe(false);
    });

    it("reads patterns from project root .understandignore", () => {
      writeFileSync(
        join(testDir, ".understandignore"),
        "docs/\n"
      );
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("docs/README.md")).toBe(true);
      expect(filter.isIgnored("src/index.ts")).toBe(false);
    });

    it("handles # comments and blank lines", () => {
      writeFileSync(
        join(testDir, ".understand-anything", ".understandignore"),
        "# This is a comment\n\n\nfixtures/\n\n# Another comment\n"
      );
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("fixtures/data.json")).toBe(true);
      expect(filter.isIgnored("src/index.ts")).toBe(false);
    });

    it("supports ! negation to override defaults", () => {
      writeFileSync(
        join(testDir, ".understand-anything", ".understandignore"),
        "!dist/\n"
      );
      const filter = createIgnoreFilter(testDir);
      // dist/ is in defaults but negated by user
      expect(filter.isIgnored("dist/index.js")).toBe(false);
    });

    it("supports ** recursive matching", () => {
      writeFileSync(
        join(testDir, ".understand-anything", ".understandignore"),
        "**/snapshots/\n"
      );
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("src/components/snapshots/Button.snap")).toBe(true);
      expect(filter.isIgnored("snapshots/foo.snap")).toBe(true);
    });

    it("merges .understand-anything/ and root .understandignore", () => {
      writeFileSync(
        join(testDir, ".understand-anything", ".understandignore"),
        "__tests__/\n"
      );
      writeFileSync(
        join(testDir, ".understandignore"),
        "fixtures/\n"
      );
      const filter = createIgnoreFilter(testDir);
      expect(filter.isIgnored("__tests__/foo.ts")).toBe(true);
      expect(filter.isIgnored("fixtures/data.json")).toBe(true);
      expect(filter.isIgnored("src/index.ts")).toBe(false);
    });
  });
});
```

- [ ] **Passo 2: Executar os testes para verificar que falham**

Execute: `pnpm --filter @understand-anything/core test -- --run src/__tests__/ignore-filter.test.ts`
Esperado: FALHA — módulo não encontrado

- [ ] **Passo 3: Implementar IgnoreFilter**

Crie `understand-anything-plugin/packages/core/src/ignore-filter.ts`:

```typescript
import ignore, { type Ignore } from "ignore";
import { readFileSync, existsSync } from "node:fs";
import { join } from "node:path";

/**
 * Hardcoded default ignore patterns matching the project-scanner agent's
 * exclusion rules, plus bin/obj for .NET projects.
 */
export const DEFAULT_IGNORE_PATTERNS: string[] = [
  // Dependency directories
  "node_modules/",
  ".git/",
  "vendor/",
  "venv/",
  ".venv/",
  "__pycache__/",

  // Build output
  "dist/",
  "build/",
  "out/",
  "coverage/",
  ".next/",
  ".cache/",
  ".turbo/",
  "target/",
  "bin/",
  "obj/",

  // Lock files
  "*.lock",
  "package-lock.json",
  "yarn.lock",
  "pnpm-lock.yaml",

  // Binary/asset files
  "*.png",
  "*.jpg",
  "*.jpeg",
  "*.gif",
  "*.svg",
  "*.ico",
  "*.woff",
  "*.woff2",
  "*.ttf",
  "*.eot",
  "*.mp3",
  "*.mp4",
  "*.pdf",
  "*.zip",
  "*.tar",
  "*.gz",

  // Generated files
  "*.min.js",
  "*.min.css",
  "*.map",
  "*.generated.*",

  // IDE/editor
  ".idea/",
  ".vscode/",

  // Misc
  "LICENSE",
  ".gitignore",
  ".editorconfig",
  ".prettierrc",
  ".eslintrc*",
  "*.log",
];

export interface IgnoreFilter {
  /** Returns true if the given relative path should be excluded from analysis. */
  isIgnored(relativePath: string): boolean;
}

/**
 * Creates an IgnoreFilter that merges hardcoded defaults with user-defined
 * patterns from .understandignore files.
 *
 * Pattern load order (later entries can override earlier ones via ! negation):
 * 1. Hardcoded defaults
 * 2. .understand-anything/.understandignore (if exists)
 * 3. .understandignore at project root (if exists)
 */
export function createIgnoreFilter(projectRoot: string): IgnoreFilter {
  const ig: Ignore = ignore();

  // Layer 1: hardcoded defaults
  ig.add(DEFAULT_IGNORE_PATTERNS);

  // Layer 2: .understand-anything/.understandignore
  const projectIgnorePath = join(projectRoot, ".understand-anything", ".understandignore");
  if (existsSync(projectIgnorePath)) {
    const content = readFileSync(projectIgnorePath, "utf-8");
    ig.add(content);
  }

  // Layer 3: .understandignore at project root
  const rootIgnorePath = join(projectRoot, ".understandignore");
  if (existsSync(rootIgnorePath)) {
    const content = readFileSync(rootIgnorePath, "utf-8");
    ig.add(content);
  }

  return {
    isIgnored(relativePath: string): boolean {
      return ig.ignores(relativePath);
    },
  };
}
```

- [ ] **Passo 4: Executar os testes para verificar que passam**

Execute: `pnpm --filter @understand-anything/core test -- --run src/__tests__/ignore-filter.test.ts`
Esperado: Todos os testes PASSAM

- [ ] **Passo 5: Build para verificar que não há erros de tipo**

Execute: `pnpm --filter @understand-anything/core build`
Esperado: Build limpo

- [ ] **Passo 6: Commit**

```bash
git add understand-anything-plugin/packages/core/src/ignore-filter.ts understand-anything-plugin/packages/core/src/__tests__/ignore-filter.test.ts
git commit -m "feat(core): add IgnoreFilter module with .understandignore parsing and tests"
```

---

## Tarefa 3: Criar módulo IgnoreGenerator com testes (TDD)

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/ignore-generator.ts`
- Criar: `understand-anything-plugin/packages/core/src/__tests__/ignore-generator.test.ts`

- [ ] **Passo 1: Escrever os testes que devem falhar**

Crie `understand-anything-plugin/packages/core/src/__tests__/ignore-generator.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { generateStarterIgnoreFile } from "../ignore-generator";
import { mkdirSync, rmSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";

describe("generateStarterIgnoreFile", () => {
  let testDir: string;

  beforeEach(() => {
    testDir = join(tmpdir(), `ignore-gen-test-${Date.now()}`);
    mkdirSync(testDir, { recursive: true });
  });

  afterEach(() => {
    rmSync(testDir, { recursive: true, force: true });
  });

  it("includes a header comment explaining the file", () => {
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain(".understandignore");
    expect(content).toContain("same as .gitignore");
    expect(content).toContain("Built-in defaults");
  });

  it("all suggestions are commented out", () => {
    // Create some directories to trigger suggestions
    mkdirSync(join(testDir, "__tests__"), { recursive: true });
    mkdirSync(join(testDir, "docs"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    const lines = content.split("\n").filter((l) => l.trim() && !l.startsWith("#"));
    // No active (uncommented) patterns
    expect(lines).toHaveLength(0);
  });

  it("suggests __tests__ when __tests__ directory exists", () => {
    mkdirSync(join(testDir, "__tests__"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# __tests__/");
  });

  it("suggests docs when docs directory exists", () => {
    mkdirSync(join(testDir, "docs"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# docs/");
  });

  it("suggests test directories when they exist", () => {
    mkdirSync(join(testDir, "test"), { recursive: true });
    mkdirSync(join(testDir, "tests"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# test/");
    expect(content).toContain("# tests/");
  });

  it("suggests fixtures when fixtures directory exists", () => {
    mkdirSync(join(testDir, "fixtures"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# fixtures/");
  });

  it("suggests examples when examples directory exists", () => {
    mkdirSync(join(testDir, "examples"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# examples/");
  });

  it("suggests .storybook when .storybook directory exists", () => {
    mkdirSync(join(testDir, ".storybook"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# .storybook/");
  });

  it("suggests migrations when migrations directory exists", () => {
    mkdirSync(join(testDir, "migrations"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# migrations/");
  });

  it("suggests scripts when scripts directory exists", () => {
    mkdirSync(join(testDir, "scripts"), { recursive: true });
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# scripts/");
  });

  it("always includes generic suggestions", () => {
    const content = generateStarterIgnoreFile(testDir);
    expect(content).toContain("# *.snap");
    expect(content).toContain("# *.test.*");
    expect(content).toContain("# *.spec.*");
  });

  it("does not suggest directories that don't exist", () => {
    const content = generateStarterIgnoreFile(testDir);
    // __tests__ doesn't exist, so it shouldn't be in directory suggestions
    // (it may still be in generic test file patterns)
    expect(content).not.toContain("# __tests__/");
    expect(content).not.toContain("# .storybook/");
  });
});
```

- [ ] **Passo 2: Executar os testes para verificar que falham**

Execute: `pnpm --filter @understand-anything/core test -- --run src/__tests__/ignore-generator.test.ts`
Esperado: FALHA — módulo não encontrado

- [ ] **Passo 3: Implementar IgnoreGenerator**

Crie `understand-anything-plugin/packages/core/src/ignore-generator.ts`:

```typescript
import { existsSync } from "node:fs";
import { join } from "node:path";

const HEADER = `# .understandignore — patterns for files/dirs to exclude from analysis
# Syntax: same as .gitignore (globs, # comments, ! negation, trailing / for dirs)
# Lines below are suggestions — uncomment to activate.
# Use ! prefix to force-include something excluded by defaults.
#
# Built-in defaults (always excluded unless negated):
#   node_modules/, .git/, dist/, build/, bin/, obj/, *.lock, *.min.js, etc.
#
`;

/** Directories to check for and suggest excluding. */
const DETECTABLE_DIRS = [
  { dir: "__tests__", pattern: "__tests__/" },
  { dir: "test", pattern: "test/" },
  { dir: "tests", pattern: "tests/" },
  { dir: "fixtures", pattern: "fixtures/" },
  { dir: "testdata", pattern: "testdata/" },
  { dir: "docs", pattern: "docs/" },
  { dir: "examples", pattern: "examples/" },
  { dir: "scripts", pattern: "scripts/" },
  { dir: "migrations", pattern: "migrations/" },
  { dir: ".storybook", pattern: ".storybook/" },
];

/** Always-included generic suggestions. */
const GENERIC_SUGGESTIONS = [
  "*.test.*",
  "*.spec.*",
  "*.snap",
];

/**
 * Generates a starter .understandignore file by scanning the project root
 * for common directories and suggesting them as commented-out exclusions.
 *
 * All suggestions are commented out — the user must uncomment to activate.
 * Returns the file content as a string.
 */
export function generateStarterIgnoreFile(projectRoot: string): string {
  const sections: string[] = [HEADER];

  // Detected directory suggestions
  const detected: string[] = [];
  for (const { dir, pattern } of DETECTABLE_DIRS) {
    if (existsSync(join(projectRoot, dir))) {
      detected.push(pattern);
    }
  }

  if (detected.length > 0) {
    sections.push("# --- Detected directories (uncomment to exclude) ---\n");
    for (const pattern of detected) {
      sections.push(`# ${pattern}`);
    }
    sections.push("");
  }

  // Generic suggestions (always included)
  sections.push("# --- Test file patterns (uncomment to exclude) ---\n");
  for (const pattern of GENERIC_SUGGESTIONS) {
    sections.push(`# ${pattern}`);
  }
  sections.push("");

  return sections.join("\n");
}
```

- [ ] **Passo 4: Executar os testes para verificar que passam**

Execute: `pnpm --filter @understand-anything/core test -- --run src/__tests__/ignore-generator.test.ts`
Esperado: Todos os testes PASSAM

- [ ] **Passo 5: Build**

Execute: `pnpm --filter @understand-anything/core build`
Esperado: Build limpo

- [ ] **Passo 6: Commit**

```bash
git add understand-anything-plugin/packages/core/src/ignore-generator.ts understand-anything-plugin/packages/core/src/__tests__/ignore-generator.test.ts
git commit -m "feat(core): add IgnoreGenerator for starter .understandignore file creation"
```

---

## Tarefa 4: Exportar novos módulos do core

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/index.ts`

- [ ] **Passo 1: Adicionar exports**

Adicione ao final de `understand-anything-plugin/packages/core/src/index.ts`:

```typescript
export {
  createIgnoreFilter,
  DEFAULT_IGNORE_PATTERNS,
  type IgnoreFilter,
} from "./ignore-filter.js";
export { generateStarterIgnoreFile } from "./ignore-generator.js";
```

- [ ] **Passo 2: Build e executar todos os testes**

Execute: `pnpm --filter @understand-anything/core build && pnpm --filter @understand-anything/core test -- --run`
Esperado: Build limpo, todos os testes passam

- [ ] **Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/index.ts
git commit -m "feat(core): export IgnoreFilter and IgnoreGenerator from core index"
```

---

## Tarefa 5: Atualizar agente project-scanner

**Arquivos:**
- Modificar: `understand-anything-plugin/agents/project-scanner.md`

- [ ] **Passo 1: Ler o project-scanner.md atual**

Leia `understand-anything-plugin/agents/project-scanner.md` para entender a estrutura atual.

- [ ] **Passo 2: Adicionar bin/ e obj/ às exclusões hardcoded**

No Passo 2 (Exclusion Filtering), adicione `bin/` e `obj/` à linha "Build output":

Altere:
```
- **Build output:** paths with a directory segment matching `dist/`, `build/`, `out/`, `coverage/`, `.next/`, `.cache/`, `.turbo/`, `target/` (Rust)
```

Para:
```
- **Build output:** paths with a directory segment matching `dist/`, `build/`, `out/`, `coverage/`, `.next/`, `.cache/`, `.turbo/`, `target/` (Rust), `bin/` (.NET), `obj/` (.NET)
```

- [ ] **Passo 3: Adicionar etapa de filtragem da Camada 2**

Após o Passo 2 (Exclusion Filtering), adicione um novo passo:

```markdown
**Step 2.5 -- User-Configured Filtering (.understandignore)**

After applying the hardcoded exclusion filters above, apply user-configured patterns from `.understandignore`:

1. Check if `.understand-anything/.understandignore` exists in the project root. If so, read it.
2. Check if `.understandignore` exists in the project root. If so, read it.
3. Parse both files using `.gitignore` syntax (glob patterns, `#` comments, blank lines ignored, `!` prefix for negation, trailing `/` for directories, `**/` for recursive matching).
4. Filter the remaining file list through these patterns. Files matching any pattern are excluded.
5. `!` negation patterns override the hardcoded exclusions from Step 2 (e.g., `!dist/` force-includes dist/).
6. Track the count of files removed by this step as `filteredByIgnore`.

This filtering must be deterministic (not LLM-based). Use a Node.js script with the `ignore` npm package if implementing programmatically, or apply the patterns manually if the file list is small.
```

- [ ] **Passo 4: Atualizar o schema de saída do scan**

Encontre a seção do schema JSON de saída e adicione o campo `filteredByIgnore`:

```json
{
  "name": "...",
  "description": "...",
  "languages": ["..."],
  "frameworks": ["..."],
  "files": [...],
  "totalFiles": 123,
  "filteredByIgnore": 5,
  "estimatedComplexity": "moderate",
  "importMap": {}
}
```

- [ ] **Passo 5: Commit**

```bash
git add understand-anything-plugin/agents/project-scanner.md
git commit -m "feat(agent): add .understandignore support and bin/obj exclusions to project-scanner"
```

---

## Tarefa 6: Atualizar skill /understand com Fase 0.5

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md`

- [ ] **Passo 1: Ler a seção da Fase 0 atual do SKILL.md**

Leia `understand-anything-plugin/skills/understand/SKILL.md` linhas 22-80 para entender a Fase 0.

- [ ] **Passo 2: Adicionar Fase 0.5 após a Fase 0**

Após a seção da Fase 0 (depois do separador `---` antes da Fase 1), insira:

```markdown
## Phase 0.5 — Ignore Configuration

Set up and verify the `.understandignore` file before scanning.

1. Check if `$PROJECT_ROOT/.understand-anything/.understandignore` exists.
2. **If it does NOT exist**, generate a starter file:
   - Run a Node.js script (or inline logic) that scans `$PROJECT_ROOT` for common directories (`__tests__/`, `test/`, `tests/`, `fixtures/`, `testdata/`, `docs/`, `examples/`, `scripts/`, `migrations/`, `.storybook/`) and generates a `.understandignore` file with commented-out suggestions.
   - Write the generated content to `$PROJECT_ROOT/.understand-anything/.understandignore`.
   - Report to the user:
     > "Generated `.understand-anything/.understandignore` with suggested exclusions based on your project structure. Please review it and uncomment any patterns you'd like to exclude from analysis. When ready, confirm to continue."
   - **Wait for user confirmation before proceeding.**
3. **If it already exists**, report:
   > "Found `.understand-anything/.understandignore`. Review it if needed, then confirm to continue."
   - **Wait for user confirmation before proceeding.**
4. After confirmation, proceed to Phase 1.

**Note:** The `.understandignore` file uses `.gitignore` syntax. The user can add patterns to exclude files from analysis, or use `!` prefix to force-include files excluded by built-in defaults (e.g., `!dist/` to analyze dist/ files).

---
```

- [ ] **Passo 3: Atualizar o relato da Fase 1**

Na seção da Fase 1, após o gate check (~linha 114), adicione uma nota sobre relatar estatísticas de ignore:

```markdown
After scanning, if the scan result includes `filteredByIgnore > 0`, report:
> "Scanned {totalFiles} files ({filteredByIgnore} excluded by .understandignore)"
```

- [ ] **Passo 4: Commit**

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "feat(skill): add Phase 0.5 for .understandignore setup and review pause"
```

---

## Tarefa 7: Build, testar e verificar de ponta a ponta

**Arquivos:**
- Todos os arquivos modificados

- [ ] **Passo 1: Build do core**

Execute: `pnpm --filter @understand-anything/core build`
Esperado: Build limpo

- [ ] **Passo 2: Executar todos os testes do core**

Execute: `pnpm --filter @understand-anything/core test -- --run`
Esperado: Todos os testes passam (existentes + novos ignore-filter + ignore-generator)

- [ ] **Passo 3: Build do pacote skill**

Execute: `pnpm --filter @understand-anything/skill build`
Esperado: Build limpo

- [ ] **Passo 4: Verificar que os arquivos existem**

Execute:
```bash
ls understand-anything-plugin/packages/core/src/ignore-filter.ts understand-anything-plugin/packages/core/src/ignore-generator.ts
```
Esperado: Ambos os arquivos listados

- [ ] **Passo 5: Verificar que os exports funcionam**

Execute:
```bash
node -e "import('@understand-anything/core').then(m => { console.log('IgnoreFilter:', typeof m.createIgnoreFilter); console.log('Generator:', typeof m.generateStarterIgnoreFile); })"
```
Esperado: Ambos mostram `function`

- [ ] **Passo 6: Commit final (se houver alterações não-staged)**

```bash
git status
# If clean, skip. If changes exist:
git add -A && git commit -m "chore: final verification for .understandignore support"
```
