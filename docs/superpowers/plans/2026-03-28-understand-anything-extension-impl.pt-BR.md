# Suporte Universal a Tipos de Arquivo — Plano de Implementação

> 🇬🇧 Versão original em inglês: [2026-03-28-understand-anything-extension-impl.md](./2026-03-28-understand-anything-extension-impl.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa por tarefa.

**Objetivo:** Estender o Understand Anything para analisar 26+ tipos de arquivo não-código (Markdown, Dockerfile, YAML, SQL, Terraform, etc.) com novos tipos de node/edge no grafo, parsers customizados, prompts de agente atualizados e visualização no dashboard.

**Arquitetura:** Estender o pipeline existente LanguageConfig + AnalyzerPlugin. Adicionar 8 novos tipos de node e 8 novos tipos de edge ao schema. Construir 12 analyzers leves baseados em regex/parser para formatos estruturados, somente LLM para não-estruturados. Atualizar todos os 5 prompts de agente para lidar com arquivos não-código. Adicionar novas cores de node e renderização no sidebar do dashboard.

**Stack Técnica:** TypeScript, Zod, Vitest, React, React Flow, Zustand, TailwindCSS v4, pacote npm `yaml`, `@iarna/toml`, `jsonc-parser`

**Doc de design:** `docs/plans/2026-03-28-understand-anything-extension-design.md`

---

## Tarefa 1: Estender os tipos do core — Node & Edge Types

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/types.ts:1-116`
- Test: `understand-anything-plugin/packages/core/src/types.test.ts`

**Passo 1: Escrever o teste que falha**

Em `types.test.ts`, adicione um teste que importa e verifica que os novos tipos de node existem em GraphNode e os tipos de edge existem em EdgeType:

```typescript
import { describe, it, expect } from "vitest";
import type { GraphNode, GraphEdge, EdgeType, StructuralAnalysis } from "../types.js";

describe("Extended types", () => {
  it("accepts all 13 node types", () => {
    const nodeTypes: GraphNode["type"][] = [
      "file", "function", "class", "module", "concept",
      "config", "document", "service", "table", "endpoint",
      "pipeline", "schema", "resource",
    ];
    expect(nodeTypes).toHaveLength(13);
  });

  it("accepts all 26 edge types", () => {
    const edgeTypes: EdgeType[] = [
      "imports", "exports", "contains", "inherits", "implements",
      "calls", "subscribes", "publishes", "middleware",
      "reads_from", "writes_to", "transforms", "validates",
      "depends_on", "tested_by", "configures",
      "related", "similar_to",
      "deploys", "serves", "migrates", "documents",
      "provisions", "routes", "defines_schema", "triggers",
    ];
    expect(edgeTypes).toHaveLength(26);
  });

  it("StructuralAnalysis has optional non-code fields", () => {
    const analysis: StructuralAnalysis = {
      functions: [], classes: [], imports: [], exports: [],
      sections: [{ name: "Introduction", level: 1, lineRange: [1, 10] }],
      definitions: [{ name: "users", kind: "table", lineRange: [1, 20], fields: ["id", "name"] }],
      services: [{ name: "web", image: "node:22", ports: [3000] }],
      endpoints: [{ method: "GET", path: "/api/users", lineRange: [5, 15] }],
      steps: [{ name: "build", lineRange: [1, 5] }],
      resources: [{ name: "aws_s3_bucket.main", kind: "aws_s3_bucket", lineRange: [1, 10] }],
    };
    expect(analysis.sections).toHaveLength(1);
    expect(analysis.definitions).toHaveLength(1);
    expect(analysis.services).toHaveLength(1);
    expect(analysis.endpoints).toHaveLength(1);
    expect(analysis.steps).toHaveLength(1);
    expect(analysis.resources).toHaveLength(1);
  });
});
```

**Passo 2: Rodar o teste para verificar que falha**

Rode: `pnpm --filter @understand-anything/core test -- --run types.test`
Esperado: FAIL — erros de compilação TypeScript para tipos novos que ainda não existem

**Passo 3: Implementar as extensões de tipo**

Em `types.ts`, atualize o union `GraphNode.type` (linha 12):

```typescript
type: "file" | "function" | "class" | "module" | "concept"
  | "config" | "document" | "service" | "table" | "endpoint"
  | "pipeline" | "schema" | "resource";
```

Atualize o tipo `EdgeType` (linhas 1-7) para adicionar 8 novos tipos de edge:

```typescript
export type EdgeType =
  | "imports" | "exports" | "contains" | "inherits" | "implements"  // Structural
  | "calls" | "subscribes" | "publishes" | "middleware"              // Behavioral
  | "reads_from" | "writes_to" | "transforms" | "validates"         // Data flow
  | "depends_on" | "tested_by" | "configures"                       // Dependencies
  | "related" | "similar_to"                                         // Semantic
  | "deploys" | "serves" | "migrates" | "documents"                 // Infrastructure
  | "provisions" | "routes" | "defines_schema" | "triggers";        // Infrastructure
```

Estenda `StructuralAnalysis` (após a linha 95) com novos campos opcionais:

```typescript
export interface SectionInfo {
  name: string;
  level: number;
  lineRange: [number, number];
}

export interface DefinitionInfo {
  name: string;
  kind: string; // "table", "message", "type", "schema"
  lineRange: [number, number];
  fields: string[];
}

export interface ServiceInfo {
  name: string;
  image?: string;
  ports: number[];
}

export interface EndpointInfo {
  method?: string;
  path: string;
  lineRange: [number, number];
}

export interface StepInfo {
  name: string;
  lineRange: [number, number];
}

export interface ResourceInfo {
  name: string;
  kind: string;
  lineRange: [number, number];
}

export interface ReferenceResolution {
  source: string;
  target: string;
  referenceType: string; // "file", "image", "schema", "service"
  line?: number;
}

export interface StructuralAnalysis {
  functions: Array<{ name: string; lineRange: [number, number]; params: string[]; returnType?: string }>;
  classes: Array<{ name: string; lineRange: [number, number]; methods: string[]; properties: string[] }>;
  imports: Array<{ source: string; specifiers: string[]; lineNumber: number }>;
  exports: Array<{ name: string; lineNumber: number }>;
  // Non-code structural data (all optional for backward compat)
  sections?: SectionInfo[];
  definitions?: DefinitionInfo[];
  services?: ServiceInfo[];
  endpoints?: EndpointInfo[];
  steps?: StepInfo[];
  resources?: ResourceInfo[];
}
```

Atualize a interface `AnalyzerPlugin` (linhas 109-115) — torne `resolveImports` opcional, adicione `extractReferences`:

```typescript
export interface AnalyzerPlugin {
  name: string;
  languages: string[];
  analyzeFile(filePath: string, content: string): StructuralAnalysis;
  resolveImports?(filePath: string, content: string): ImportResolution[];
  extractCallGraph?(filePath: string, content: string): CallGraphEntry[];
  extractReferences?(filePath: string, content: string): ReferenceResolution[];
}
```

**Passo 4: Rodar o teste para verificar que passa**

Rode: `pnpm --filter @understand-anything/core test -- --run types.test`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/types.ts understand-anything-plugin/packages/core/src/types.test.ts
git commit -m "feat(core): extend GraphNode/EdgeType/StructuralAnalysis for non-code file types"
```

---

## Tarefa 2: Estender Validação de Schema — Schemas Zod & Aliases

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/schema.ts:1-554`
- Test: `understand-anything-plugin/packages/core/src/__tests__/schema.test.ts`

**Passo 1: Escrever os testes que falham**

Adicione a `schema.test.ts`:

```typescript
describe("Extended node/edge types", () => {
  it("validates nodes with new types: config, document, service, table, endpoint, pipeline, schema, resource", () => {
    const newTypes = ["config", "document", "service", "table", "endpoint", "pipeline", "schema", "resource"];
    for (const type of newTypes) {
      const graph = structuredClone(validGraph);
      graph.nodes[0].type = type;
      const result = validateGraph(graph);
      expect(result.success).toBe(true);
      expect(result.data!.nodes[0].type).toBe(type);
    }
  });

  it("validates edges with new types: deploys, serves, migrates, documents, provisions, routes, defines_schema, triggers", () => {
    const newTypes = ["deploys", "serves", "migrates", "documents", "provisions", "routes", "defines_schema", "triggers"];
    for (const type of newTypes) {
      const graph = structuredClone(validGraph);
      graph.edges[0].type = type;
      const result = validateGraph(graph);
      expect(result.success).toBe(true);
      expect(result.data!.edges[0].type).toBe(type);
    }
  });

  it("auto-fixes new node type aliases: container→service, doc→document, workflow→pipeline, etc.", () => {
    const aliases = { container: "service", doc: "document", workflow: "pipeline", route: "endpoint", setting: "config", infra: "resource", migration: "table" };
    for (const [alias, canonical] of Object.entries(aliases)) {
      const graph = structuredClone(validGraph);
      (graph.nodes[0] as any).type = alias;
      const result = validateGraph(graph);
      expect(result.success).toBe(true);
      expect(result.data!.nodes[0].type).toBe(canonical);
    }
  });

  it("auto-fixes new edge type aliases: describes→documents, creates→provisions, exposes→serves", () => {
    const aliases = { describes: "documents", creates: "provisions", exposes: "serves" };
    for (const [alias, canonical] of Object.entries(aliases)) {
      const graph = structuredClone(validGraph);
      (graph.edges[0] as any).type = alias;
      const result = validateGraph(graph);
      expect(result.success).toBe(true);
      expect(result.data!.edges[0].type).toBe(canonical);
    }
  });
});
```

**Passo 2: Rodar o teste para verificar que falha**

Rode: `pnpm --filter @understand-anything/core test -- --run schema.test`
Esperado: FAIL — o enum Zod rejeita os novos tipos

**Passo 3: Implementar extensões de schema**

Em `schema.ts`:

1. Atualize `EdgeTypeSchema` (linha 4-10) — adicione 8 novos tipos de edge:
```typescript
export const EdgeTypeSchema = z.enum([
  "imports", "exports", "contains", "inherits", "implements",
  "calls", "subscribes", "publishes", "middleware",
  "reads_from", "writes_to", "transforms", "validates",
  "depends_on", "tested_by", "configures",
  "related", "similar_to",
  "deploys", "serves", "migrates", "documents",
  "provisions", "routes", "defines_schema", "triggers",
]);
```

2. Atualize `NODE_TYPE_ALIASES` (linha 13-22) — adicione novos aliases:
```typescript
export const NODE_TYPE_ALIASES: Record<string, string> = {
  func: "function", fn: "function", method: "function",
  interface: "class", struct: "class",
  mod: "module", pkg: "module", package: "module",
  // New non-code aliases
  container: "service", deployment: "service", pod: "service",
  doc: "document", readme: "document", docs: "document",
  workflow: "pipeline", job: "pipeline", ci: "pipeline", action: "pipeline",
  route: "endpoint", api: "endpoint", query: "endpoint", mutation: "endpoint",
  setting: "config", env: "config", configuration: "config",
  infra: "resource", infrastructure: "resource", terraform: "resource",
  migration: "table", database: "table", db: "table", view: "table",
  proto: "schema", protobuf: "schema", definition: "schema", typedef: "schema",
};
```

3. Atualize `EDGE_TYPE_ALIASES` (linha 25-39) — adicione novos aliases:
```typescript
// Add these entries:
  describes: "documents",
  documented_by: "documents",
  creates: "provisions",
  exposes: "serves",
  listens: "serves",
  deploys_to: "deploys",
  migrates_to: "migrates",
  routes_to: "routes",
  triggers_on: "triggers",
  fires: "triggers",
  defines: "defines_schema",
```

4. Atualize `GraphNodeSchema` (linha 267-277) — estenda o enum de type:
```typescript
export const GraphNodeSchema = z.object({
  id: z.string(),
  type: z.enum([
    "file", "function", "class", "module", "concept",
    "config", "document", "service", "table", "endpoint",
    "pipeline", "schema", "resource",
  ]),
  name: z.string(),
  filePath: z.string().optional(),
  lineRange: z.tuple([z.number(), z.number()]).optional(),
  summary: z.string(),
  tags: z.array(z.string()),
  complexity: z.enum(["simple", "moderate", "complex"]),
  languageNotes: z.string().optional(),
});
```

**Passo 4: Rodar o teste para verificar que passa**

Rode: `pnpm --filter @understand-anything/core test -- --run schema.test`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/schema.ts understand-anything-plugin/packages/core/src/__tests__/schema.test.ts
git commit -m "feat(core): extend Zod schemas and aliases for 8 new node/edge types"
```

---

## Tarefa 3: Atualizar PluginRegistry — resolveImports opcional

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/plugins/registry.ts:1-76`
- Test: `understand-anything-plugin/packages/core/src/__tests__/plugin-registry.test.ts`

**Passo 1: Escrever o teste que falha**

Adicione a `plugin-registry.test.ts`:

```typescript
it("handles plugins with optional resolveImports (non-code plugins)", () => {
  const markdownPlugin: AnalyzerPlugin = {
    name: "markdown",
    languages: ["markdown"],
    analyzeFile: () => ({ functions: [], classes: [], imports: [], exports: [] }),
    // No resolveImports — optional
  };
  registry.register(markdownPlugin);
  const result = registry.resolveImports("README.md", "# Hello");
  expect(result).toBeNull(); // Returns null for plugins without resolveImports
});
```

**Passo 2: Rodar o teste para verificar que falha**

Rode: `pnpm --filter @understand-anything/core test -- --run plugin-registry.test`
Esperado: FAIL — o registry atual chama `plugin.resolveImports(...)` incondicionalmente

**Passo 3: Atualizar PluginRegistry**

Em `registry.ts`, atualize `resolveImports` (linha 62-66):

```typescript
resolveImports(filePath: string, content: string): ImportResolution[] | null {
  const plugin = this.getPluginForFile(filePath);
  if (!plugin || !plugin.resolveImports) return null;
  return plugin.resolveImports(filePath, content);
}
```

**Passo 4: Rodar o teste para verificar que passa**

Rode: `pnpm --filter @understand-anything/core test -- --run plugin-registry.test`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/registry.ts understand-anything-plugin/packages/core/src/__tests__/plugin-registry.test.ts
git commit -m "feat(core): make resolveImports optional on AnalyzerPlugin"
```

---

## Tarefa 4: Adicionar Configs de Linguagens Não-Código (26 configs)

**Arquivos:**
- Create: `understand-anything-plugin/packages/core/src/languages/configs/markdown.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/yaml.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/json-config.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/toml.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/env.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/xml.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/dockerfile.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/sql.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/graphql.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/protobuf.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/terraform.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/github-actions.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/makefile.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/shell.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/html.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/css.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/openapi.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/kubernetes.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/docker-compose.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/json-schema.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/csv.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/restructuredtext.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/powershell.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/batch.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/jenkinsfile.ts`
- Create: `understand-anything-plugin/packages/core/src/languages/configs/plaintext.ts`
- Modify: `understand-anything-plugin/packages/core/src/languages/configs/index.ts`
- Test: `understand-anything-plugin/packages/core/src/__tests__/language-registry.test.ts`

**Passo 1: Escrever o teste que falha**

Adicione a `language-registry.test.ts`:

```typescript
describe("Non-code language configs", () => {
  it("detects all non-code file types via extension", () => {
    const registry = LanguageRegistry.createDefault();
    const expectations: [string, string][] = [
      ["README.md", "markdown"],
      ["config.yaml", "yaml"],
      ["package.json", "json"],
      ["config.toml", "toml"],
      [".env", "env"],
      ["pom.xml", "xml"],
      ["Dockerfile", "dockerfile"],
      ["schema.sql", "sql"],
      ["schema.graphql", "graphql"],
      ["types.proto", "protobuf"],
      ["main.tf", "terraform"],
      ["Makefile", "makefile"],
      ["deploy.sh", "shell"],
      ["index.html", "html"],
      ["styles.css", "css"],
      ["data.csv", "csv"],
      ["deploy.ps1", "powershell"],
    ];
    for (const [file, expectedId] of expectations) {
      const config = registry.getForFile(file);
      expect(config?.id, `${file} should be detected as ${expectedId}`).toBe(expectedId);
    }
  });
});
```

**Passo 2: Rodar o teste para verificar que falha**

Rode: `pnpm --filter @understand-anything/core test -- --run language-registry.test`
Esperado: FAIL — nenhum config registrado para extensões não-código

**Passo 3: Criar todos os arquivos de config**

Cada config segue o mesmo padrão de `typescript.ts`. Exemplo para markdown:

```typescript
// markdown.ts
import type { LanguageConfig } from "../types.js";

export const markdownConfig = {
  id: "markdown",
  displayName: "Markdown",
  extensions: [".md", ".mdx"],
  concepts: ["headings", "links", "code blocks", "front matter", "lists", "tables", "images"],
  filePatterns: {
    entryPoints: ["README.md"],
    barrels: [],
    tests: [],
    config: [],
  },
} satisfies LanguageConfig;
```

Crie configs similares para os 26 tipos. Mapeamentos-chave de extensão:
- yaml: `.yaml`, `.yml`
- json: `.json`, `.jsonc`
- toml: `.toml`
- env: `.env` (Nota: LanguageRegistry precisa de correspondência por filename, não apenas extensão)
- xml: `.xml`
- dockerfile: `Dockerfile` (detecção por filename — precisa de tratamento especial)
- sql: `.sql`
- graphql: `.graphql`, `.gql`
- protobuf: `.proto`
- terraform: `.tf`, `.tfvars`
- github-actions: (detectado pelo path `.github/workflows/*.yml` — adiar para o scanner)
- makefile: `Makefile` (por filename — precisa de tratamento especial)
- shell: `.sh`, `.bash`, `.zsh`
- html: `.html`, `.htm`
- css: `.css`, `.scss`, `.less`
- csv: `.csv`, `.tsv`
- powershell: `.ps1`, `.psm1`
- batch: `.bat`, `.cmd`
- plaintext: `.txt`
- restructuredtext: `.rst`
- jenkinsfile: (por filename — `Jenkinsfile`)

**Importante:** Para detecção por filename (Dockerfile, Makefile, Jenkinsfile), estenda o LanguageRegistry para suportar um array `filenames` além de `extensions`. Adicione um campo `filenames?: string[]` ao `LanguageConfig` e atualize `getForFile()` para checar basename contra filenames quando o lookup por extensão falhar.

Atualize `configs/index.ts` para importar e registrar todos os novos configs em `builtinLanguageConfigs`.

**Passo 4: Rodar o teste para verificar que passa**

Rode: `pnpm --filter @understand-anything/core test -- --run language-registry.test`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/languages/
git commit -m "feat(core): add 26 non-code language configs with filename-based detection"
```

---

## Tarefa 5: Construir Parsers Customizados (12 parsers)

**Arquivos:**
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/markdown-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/yaml-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/json-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/toml-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/env-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/dockerfile-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/sql-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/graphql-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/protobuf-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/terraform-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/makefile-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/shell-parser.ts`
- Create: `understand-anything-plugin/packages/core/src/plugins/parsers/index.ts`
- Test: `understand-anything-plugin/packages/core/src/__tests__/parsers.test.ts`

Cada parser implementa `AnalyzerPlugin`. Construa-os em estilo TDD, um de cada vez.

**Passo 1: Escrever testes que falham para todos os 12 parsers**

Crie `parsers.test.ts` com suítes de teste para cada parser. Exemplo para MarkdownParser:

```typescript
import { describe, it, expect } from "vitest";
import { MarkdownParser } from "../plugins/parsers/markdown-parser.js";

describe("MarkdownParser", () => {
  const parser = new MarkdownParser();

  it("extracts heading sections", () => {
    const content = "# Title\n\nIntro\n\n## Section A\n\nContent A\n\n### Subsection\n\nContent B";
    const result = parser.analyzeFile("README.md", content);
    expect(result.sections).toHaveLength(3);
    expect(result.sections![0]).toMatchObject({ name: "Title", level: 1 });
    expect(result.sections![1]).toMatchObject({ name: "Section A", level: 2 });
    expect(result.sections![2]).toMatchObject({ name: "Subsection", level: 3 });
  });

  it("extracts YAML front matter as imports", () => {
    const content = "---\ntitle: Test\ntags: [a, b]\n---\n# Content";
    const result = parser.analyzeFile("post.md", content);
    expect(result.imports).toHaveLength(0); // Front matter is metadata, not imports
  });

  it("extracts file references", () => {
    const parser2 = new MarkdownParser();
    const content = "See [guide](./docs/guide.md) and ![img](./assets/logo.png)";
    const refs = parser2.extractReferences!("README.md", content);
    expect(refs).toHaveLength(2);
    expect(refs[0]).toMatchObject({ target: "./docs/guide.md", referenceType: "file" });
    expect(refs[1]).toMatchObject({ target: "./assets/logo.png", referenceType: "image" });
  });
});
```

Suítes de teste similares para:
- **DockerfileParser**: Extrair estágios FROM, portas EXPOSE, fontes COPY
- **SQLParser**: Extrair CREATE TABLE, colunas, foreign keys
- **YAMLParser**: Extrair hierarquia de chaves de nível superior
- **JSONParser**: Extrair estrutura de chaves, `$ref`/`$defs`
- **TerraformParser**: Extrair blocos resource/module/variable
- **GraphQLParser**: Extrair definições type/query/mutation/subscription
- **ProtobufParser**: Extrair definições message/service/enum
- **MakefileParser**: Extrair targets e dependências
- **ShellParser**: Extrair definições de funções e comandos source
- **TOMLParser**: Extrair estrutura de seções
- **EnvParser**: Extrair nomes de variáveis

**Passo 2: Rodar os testes para verificar que falham**

Rode: `pnpm --filter @understand-anything/core test -- --run parsers.test`
Esperado: FAIL — módulos de parser não existem

**Passo 3: Implementar todos os 12 parsers**

Cada parser segue este padrão:

```typescript
import type { AnalyzerPlugin, StructuralAnalysis, ReferenceResolution } from "../../types.js";

export class MarkdownParser implements AnalyzerPlugin {
  name = "markdown-parser";
  languages = ["markdown"];

  analyzeFile(filePath: string, content: string): StructuralAnalysis {
    const sections = this.extractSections(content);
    return {
      functions: [], classes: [], imports: [], exports: [],
      sections,
    };
  }

  extractReferences(filePath: string, content: string): ReferenceResolution[] {
    const refs: ReferenceResolution[] = [];
    // Match [text](path) and ![alt](path)
    const linkRegex = /!?\[([^\]]*)\]\(([^)]+)\)/g;
    let match;
    while ((match = linkRegex.exec(content)) !== null) {
      const target = match[2];
      if (target.startsWith("http")) continue; // Skip external URLs
      const line = content.slice(0, match.index).split("\n").length;
      refs.push({
        source: filePath,
        target,
        referenceType: match[0].startsWith("!") ? "image" : "file",
        line,
      });
    }
    return refs;
  }

  private extractSections(content: string): SectionInfo[] {
    const sections: SectionInfo[] = [];
    const lines = content.split("\n");
    for (let i = 0; i < lines.length; i++) {
      const match = lines[i].match(/^(#{1,6})\s+(.+)/);
      if (match) {
        sections.push({
          name: match[2].trim(),
          level: match[1].length,
          lineRange: [i + 1, i + 1],
        });
      }
    }
    // Fix lineRange end for each section (extends to next heading or EOF)
    for (let i = 0; i < sections.length; i++) {
      const next = sections[i + 1];
      sections[i].lineRange[1] = next ? next.lineRange[0] - 1 : lines.length;
    }
    return sections;
  }
}
```

Crie `parsers/index.ts` que exporta todos os parsers e um helper `registerAllParsers(registry: PluginRegistry)`.

**Instalar novas dependências:**
```bash
cd understand-anything-plugin/packages/core
pnpm add yaml @iarna/toml jsonc-parser
```

**Passo 4: Rodar os testes para verificar que passam**

Rode: `pnpm --filter @understand-anything/core test -- --run parsers.test`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/parsers/ understand-anything-plugin/packages/core/src/__tests__/parsers.test.ts understand-anything-plugin/packages/core/package.json understand-anything-plugin/packages/core/pnpm-lock.yaml
git commit -m "feat(core): add 12 custom parsers for non-code file types"
```

---

## Tarefa 6: Atualizar GraphBuilder — Suportar Novos Tipos de Node

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/analyzer/graph-builder.ts:1-207`
- Test: `understand-anything-plugin/packages/core/src/analyzer/graph-builder.test.ts`

**Passo 1: Escrever o teste que falha**

Adicione a `graph-builder.test.ts`:

```typescript
describe("Non-code file support", () => {
  it("adds non-code file nodes with correct types", () => {
    const builder = new GraphBuilder("test", "abc123");
    builder.addNonCodeFile("README.md", {
      nodeType: "document",
      summary: "Project documentation",
      tags: ["documentation"],
      complexity: "simple",
    });
    const graph = builder.build();
    expect(graph.nodes).toHaveLength(1);
    expect(graph.nodes[0].type).toBe("document");
    expect(graph.nodes[0].id).toBe("file:README.md");
  });

  it("adds non-code child nodes (sections, definitions, services)", () => {
    const builder = new GraphBuilder("test", "abc123");
    builder.addNonCodeFileWithAnalysis("schema.sql", {
      nodeType: "file",
      summary: "Database schema",
      tags: ["database"],
      complexity: "moderate",
      definitions: [{ name: "users", kind: "table", lineRange: [1, 20] as [number, number], fields: ["id", "name", "email"] }],
    });
    const graph = builder.build();
    // File node + table child node
    expect(graph.nodes).toHaveLength(2);
    expect(graph.nodes[1].type).toBe("table");
    expect(graph.nodes[1].name).toBe("users");
    // Contains edge
    expect(graph.edges.some(e => e.type === "contains" && e.target.includes("users"))).toBe(true);
  });

  it("detects non-code languages from EXTENSION_LANGUAGE map", () => {
    const builder = new GraphBuilder("test", "abc123");
    builder.addFile("config.yaml", { summary: "Config", tags: [], complexity: "simple" });
    const graph = builder.build();
    expect(graph.project.languages).toContain("yaml");
  });
});
```

**Passo 2: Rodar o teste para verificar que falha**

Rode: `pnpm --filter @understand-anything/core test -- --run graph-builder.test`
Esperado: FAIL — os métodos `addNonCodeFile` e `addNonCodeFileWithAnalysis` não existem

**Passo 3: Implementar extensões do GraphBuilder**

Adicione novos métodos ao GraphBuilder:

```typescript
interface NonCodeFileMeta extends FileMeta {
  nodeType: GraphNode["type"];
}

interface NonCodeFileAnalysisMeta extends NonCodeFileMeta {
  definitions?: DefinitionInfo[];
  services?: ServiceInfo[];
  endpoints?: EndpointInfo[];
  steps?: StepInfo[];
  resources?: ResourceInfo[];
  sections?: SectionInfo[];
}

addNonCodeFile(filePath: string, meta: NonCodeFileMeta): void {
  const lang = detectLanguage(filePath);
  if (lang !== "unknown") this.languages.add(lang);
  const name = filePath.split("/").pop() ?? filePath;
  this.nodes.push({
    id: `file:${filePath}`,
    type: meta.nodeType,
    name,
    filePath,
    summary: meta.summary,
    tags: meta.tags,
    complexity: meta.complexity,
  });
}

addNonCodeFileWithAnalysis(filePath: string, meta: NonCodeFileAnalysisMeta): void {
  this.addNonCodeFile(filePath, meta);
  const fileId = `file:${filePath}`;

  // Create child nodes for definitions (tables, schemas, etc.)
  for (const def of meta.definitions ?? []) {
    const childId = `${def.kind}:${filePath}:${def.name}`;
    this.nodes.push({
      id: childId,
      type: this.mapKindToNodeType(def.kind),
      name: def.name,
      filePath,
      lineRange: def.lineRange,
      summary: `${def.kind}: ${def.name} (${def.fields.length} fields)`,
      tags: [],
      complexity: meta.complexity,
    });
    this.edges.push({ source: fileId, target: childId, type: "contains", direction: "forward", weight: 1 });
  }

  // Create child nodes for services
  for (const svc of meta.services ?? []) {
    const childId = `service:${filePath}:${svc.name}`;
    this.nodes.push({
      id: childId, type: "service", name: svc.name, filePath,
      summary: `Service ${svc.name}${svc.image ? ` (image: ${svc.image})` : ""}`,
      tags: [], complexity: meta.complexity,
    });
    this.edges.push({ source: fileId, target: childId, type: "contains", direction: "forward", weight: 1 });
  }

  // Similar for endpoints, steps, resources
}

private mapKindToNodeType(kind: string): GraphNode["type"] {
  const mapping: Record<string, GraphNode["type"]> = {
    table: "table", view: "table", index: "table",
    message: "schema", type: "schema", enum: "schema",
    resource: "resource", module: "resource",
    service: "service", deployment: "service",
    job: "pipeline", stage: "pipeline", target: "pipeline",
    route: "endpoint", query: "endpoint", mutation: "endpoint",
  };
  return mapping[kind] ?? "concept";
}
```

**Passo 4: Rodar o teste para verificar que passa**

Rode: `pnpm --filter @understand-anything/core test -- --run graph-builder.test`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/analyzer/graph-builder.ts understand-anything-plugin/packages/core/src/analyzer/graph-builder.test.ts
git commit -m "feat(core): add non-code file support to GraphBuilder"
```

---

## Tarefa 7: Atualizar Exports do Core

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/index.ts`

**Passo 1: Atualizar exports para incluir novos tipos e parsers**

Adicione a `index.ts`:

```typescript
// New structural analysis types
export type {
  SectionInfo,
  DefinitionInfo,
  ServiceInfo,
  EndpointInfo,
  StepInfo,
  ResourceInfo,
  ReferenceResolution,
} from "./types.js";

// Non-code parsers
export {
  MarkdownParser,
  DockerfileParser,
  SQLParser,
  YAMLConfigParser,
  JSONConfigParser,
  TOMLParser,
  EnvParser,
  GraphQLParser,
  ProtobufParser,
  TerraformParser,
  MakefileParser,
  ShellParser,
  registerAllParsers,
} from "./plugins/parsers/index.js";
```

**Passo 2: Buildar para verificar que os exports funcionam**

Rode: `pnpm --filter @understand-anything/core build`
Esperado: Sucesso, sem erros

**Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/index.ts
git commit -m "feat(core): export new types and parsers from core"
```

---

## Tarefa 8: Atualizar Prompts de Agente — Project Scanner

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/project-scanner-prompt.md`

**Passo 1: Atualizar o scanner para descobrir TODOS os tipos de arquivo**

Mudanças-chave no prompt:
1. Remover o filtro code-only — escanear `.md`, `.yaml`, `.json`, `.sql`, `.tf`, `Dockerfile`, etc.
2. Adicionar um campo `fileCategory` a cada arquivo descoberto: `"code" | "config" | "docs" | "infra" | "data" | "script" | "markup"`
3. Atualizar a lista de exclusão — ainda excluir `node_modules/`, `.git/`, binários, mas incluir arquivos não-código
4. Adicionar lógica de detecção de categoria no script de descoberta:
   - `.md`, `.rst`, `.txt` → `"docs"`
   - `.yaml`, `.yml`, `.json`, `.toml`, `.env`, `.xml` → `"config"`
   - `Dockerfile`, `docker-compose.*`, `.tf`, `.github/workflows/*`, `Makefile`, `Jenkinsfile` → `"infra"`
   - `.sql`, `.graphql`, `.proto`, `.schema.json`, `.csv` → `"data"`
   - `.sh`, `.bash`, `.ps1`, `.bat` → `"script"`
   - `.html`, `.css`, `.scss` → `"markup"`
   - Tudo mais → `"code"`
5. Atualizar o output schema para incluir `fileCategory` por arquivo

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/project-scanner-prompt.md
git commit -m "feat(agents): update project-scanner to discover all file types"
```

---

## Tarefa 9: Atualizar Prompts de Agente — File Analyzer

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/file-analyzer-prompt.md`

**Passo 1: Adicionar prompts de análise type-aware**

Mudanças-chave:
1. Adicionar uma seção no topo explicando categorias de arquivo e como analisar cada uma:
   - **Arquivos de código** (comportamento atual): Extrair funções, classes, imports, call graph
   - **Arquivos de config**: Extrair configurações-chave, o que configuram, qual código afetam
   - **Arquivos de documentação**: Extrair seções/headings, conceitos-chave, componentes de código referenciados
   - **Arquivos de infraestrutura**: Extrair serviços, portas, volumes, deployments, qual código eles deploiam
   - **Arquivos de Data/Schema**: Extrair tabelas, colunas, tipos, relacionamentos, código consumidor
   - **Arquivos de Pipeline**: Extrair jobs, steps, triggers, targets deploiados

2. Atualizar o output JSON schema para incluir novos campos:
   - `sections` (para docs)
   - `definitions` (para data/schema)
   - `services` (para infra)
   - `endpoints` (para schemas de API)
   - `steps` (para pipelines)
   - `resources` (para IaC)

3. Adicionar campo `nodeType` na saída: que tipo de GraphNode cada arquivo deve se tornar (file, config, document, service, etc.)

4. Atualizar a orientação de geração de edges:
   - Arquivos de config: gerar edges `configures` para arquivos de código que afetam
   - Arquivos de doc: gerar edges `documents` para o código descrito
   - Dockerfiles: gerar edges `deploys` para diretórios de código
   - Migrações SQL: gerar edges `migrates` para tabelas
   - Configs de CI: gerar edges `triggers` para pipelines
   - Schemas de API: gerar edges `defines_schema` para endpoints

5. Atualizar orientação de tagging com novas tags: `documentation`, `configuration`, `infrastructure`, `database`, `api-schema`, `ci-cd`, `deployment`, `migration`

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/file-analyzer-prompt.md
git commit -m "feat(agents): add type-aware analysis prompts for non-code files"
```

---

## Tarefa 10: Atualizar Prompts de Agente — Architecture Analyzer

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/architecture-analyzer-prompt.md`

**Passo 1: Adicionar detecção de padrões não-código**

Mudanças-chave:
1. Adicionar novos padrões arquiteturais a detectar:
   - **Topologia de deploy**: Dockerfile → docker-compose → manifestos K8s
   - **Pipeline de dados**: Definição de schema → migração → endpoint de API → código cliente
   - **Cobertura de documentação**: Quais módulos têm docs correspondentes
   - **Grafo de configuração**: Quais arquivos de config afetam quais caminhos de código
2. Atualizar hints de layer para incluir camadas não-código:
   - Layer `"infrastructure"` para Dockerfiles, K8s, Terraform
   - Layer `"documentation"` para docs
   - Layer `"data"` para SQL, schemas
   - Layer `"ci-cd"` para GitHub Actions, Jenkinsfiles
3. Atualizar o script para computar análise de dependência cross-category (code→infra, code→config, etc.)

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/architecture-analyzer-prompt.md
git commit -m "feat(agents): add non-code pattern detection to architecture analyzer"
```

---

## Tarefa 11: Atualizar Prompts de Agente — Tour Builder

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/tour-builder-prompt.md`

**Passo 1: Adicionar paradas de tour para não-código**

Mudanças-chave:
1. Atualizar a orientação de step do tour para incluir arquivos não-código:
   - Step 1 poderia ser README.md (visão geral do projeto)
   - Paradas de infraestrutura: "Como a app é containerizada"
   - Paradas de dados: "O schema do banco"
   - Paradas de CI/CD: "Como o código é deploiado"
2. Atualizar `languageLesson` para cobrir também conceitos não-código:
   - Dockerfile: multi-stage builds, layer caching
   - SQL: normalização, foreign keys
   - YAML: anchors, merge keys
   - Terraform: state management, modules

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/tour-builder-prompt.md
git commit -m "feat(agents): extend tour builder for non-code file stops"
```

---

## Tarefa 12: Atualizar Prompts de Agente — Graph Reviewer

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/graph-reviewer-prompt.md`

**Passo 1: Atualizar validação para novos tipos de node/edge**

Mudanças-chave:
1. Adicionar novos tipos de node à lista de tipos válidos no script de validação
2. Adicionar novos tipos de edge à lista de tipos válidos
3. Adicionar checks de qualidade para nodes não-código:
   - Config nodes devem ter edges `configures`
   - Document nodes devem ter edges `documents`
   - Service nodes devem ter edges `deploys`
   - Table nodes devem referenciar colunas

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/graph-reviewer-prompt.md
git commit -m "feat(agents): update graph reviewer for new node/edge types"
```

---

## Tarefa 13: Adicionar Snippets de Contexto de Linguagem

**Arquivos:**
- Create: `understand-anything-plugin/skills/understand/languages/markdown.md`
- Create: `understand-anything-plugin/skills/understand/languages/yaml.md`
- Create: `understand-anything-plugin/skills/understand/languages/json.md`
- Create: `understand-anything-plugin/skills/understand/languages/sql.md`
- Create: `understand-anything-plugin/skills/understand/languages/dockerfile.md`
- Create: `understand-anything-plugin/skills/understand/languages/terraform.md`
- Create: `understand-anything-plugin/skills/understand/languages/graphql.md`
- Create: `understand-anything-plugin/skills/understand/languages/protobuf.md`
- Create: `understand-anything-plugin/skills/understand/languages/shell.md`
- Create: `understand-anything-plugin/skills/understand/languages/html.md`
- Create: `understand-anything-plugin/skills/understand/languages/css.md`

Cada snippet segue o padrão dos existentes `typescript.md` / `python.md`:

```markdown
# Markdown

## Key Concepts
- Heading hierarchy (# through ######)
- Front matter (YAML metadata between --- delimiters)
- Code blocks (fenced with ``` or indented)
- Reference-style links
- Tables (pipe-delimited)

## Notable File Patterns
- `README.md` — Project overview (high-value entry point)
- `CONTRIBUTING.md` — Contribution guidelines
- `CHANGELOG.md` — Version history
- `docs/**/*.md` — Documentation directory

## Edge Patterns
- Markdown files `documents` the code components they describe
- Links to other .md files create `related` edges
- Code block references may imply `depends_on` edges

## Summary Style
> "Comprehensive guide document with N sections covering [topics]"
```

**Passo 1: Criar todos os 11 snippets de linguagem**

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/languages/
git commit -m "feat(agents): add language context snippets for 11 non-code file types"
```

---

## Tarefa 14: Atualizar SKILL.md — Pipeline Principal

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md`

**Passo 1: Atualizar o pipeline para tratar arquivos não-código**

Mudanças-chave:
1. **Phase 1 (SCAN)**: Atualizar o batching de arquivos para incluir não-código. Adicionar `fileCategory` aos metadados de batch.
2. **Phase 2 (ANALYZE)**: Atualizar a construção de batches para agrupar arquivos não-código relacionados (ex.: Dockerfile + docker-compose.yml). Passar `fileCategory` ao prompt do file-analyzer.
3. **Phase 4 (ARCHITECTURE)**: Injetar snippets de linguagem não-código para linguagens não-código detectadas.
4. **Phase 5 (TOUR)**: Incluir nodes não-código no pool de candidatos a tour.
5. **Phase 7 (SAVE)**: Sem mudanças necessárias (o schema lida com novos tipos).
6. **Tabela de referência de Node/Edge**: Adicionar os 8 novos tipos de node e 8 novos tipos de edge.

**Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "feat(pipeline): update main skill pipeline for non-code file analysis"
```

---

## Tarefa 15: Dashboard — Adicionar Cores de Tipo de Node aos Presets de Tema

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/themes/presets.ts:1-143`

**Passo 1: Adicionar 8 novas cores de tipo de node a todos os 5 presets**

Adicione essas entradas de cor ao objeto `colors` de cada preset:

Para presets escuros:
```typescript
"node-config": "#5eead4",    // Teal
"node-document": "#7dd3fc",  // Sky blue
"node-service": "#a78bfa",   // Violet
"node-table": "#6ee7b7",     // Emerald
"node-endpoint": "#fdba74",  // Orange
"node-pipeline": "#fda4af",  // Rose
"node-schema": "#fcd34d",    // Amber
"node-resource": "#a5b4fc",  // Indigo
```

Para preset claro, use versões levemente mais escuras:
```typescript
"node-config": "#14b8a6",
"node-document": "#38bdf8",
"node-service": "#8b5cf6",
"node-table": "#34d399",
"node-endpoint": "#fb923c",
"node-pipeline": "#fb7185",
"node-schema": "#facc15",
"node-resource": "#818cf8",
```

**Passo 2: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/themes/presets.ts
git commit -m "feat(dashboard): add 8 new node type colors to all theme presets"
```

---

## Tarefa 16: Dashboard — Atualizar o Componente CustomNode

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/CustomNode.tsx:1-137`

**Passo 1: Adicionar novas entradas aos maps typeColors e typeTextColors**

```typescript
const typeColors: Record<string, string> = {
  file: "var(--color-node-file)",
  function: "var(--color-node-function)",
  class: "var(--color-node-class)",
  module: "var(--color-node-module)",
  concept: "var(--color-node-concept)",
  config: "var(--color-node-config)",
  document: "var(--color-node-document)",
  service: "var(--color-node-service)",
  table: "var(--color-node-table)",
  endpoint: "var(--color-node-endpoint)",
  pipeline: "var(--color-node-pipeline)",
  schema: "var(--color-node-schema)",
  resource: "var(--color-node-resource)",
};

const typeTextColors: Record<string, string> = {
  file: "text-node-file",
  function: "text-node-function",
  class: "text-node-class",
  module: "text-node-module",
  concept: "text-node-concept",
  config: "text-node-config",
  document: "text-node-document",
  service: "text-node-service",
  table: "text-node-table",
  endpoint: "text-node-endpoint",
  pipeline: "text-node-pipeline",
  schema: "text-node-schema",
  resource: "text-node-resource",
};
```

**Passo 2: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/CustomNode.tsx
git commit -m "feat(dashboard): add new node type colors to CustomNode"
```

---

## Tarefa 17: Dashboard — Atualizar o Sidebar NodeInfo

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/NodeInfo.tsx:1-312`

**Passo 1: Adicionar cores de badge para novos tipos de node**

Adicione a `typeBadgeColors`:
```typescript
config: "text-node-config border border-node-config/30 bg-node-config/10",
document: "text-node-document border border-node-document/30 bg-node-document/10",
service: "text-node-service border border-node-service/30 bg-node-service/10",
table: "text-node-table border border-node-table/30 bg-node-table/10",
endpoint: "text-node-endpoint border border-node-endpoint/30 bg-node-endpoint/10",
pipeline: "text-node-pipeline border border-node-pipeline/30 bg-node-pipeline/10",
schema: "text-node-schema border border-node-schema/30 bg-node-schema/10",
resource: "text-node-resource border border-node-resource/30 bg-node-resource/10",
```

**Passo 2: Adicionar labels direcionais para novos tipos de edge**

Adicione a `getDirectionalLabel()`:
```typescript
case "deploys":
  return isSource ? "deploys" : "deployed by";
case "serves":
  return isSource ? "serves" : "served by";
case "migrates":
  return isSource ? "migrates" : "migrated by";
case "documents":
  return isSource ? "documents" : "documented by";
case "provisions":
  return isSource ? "provisions" : "provisioned by";
case "routes":
  return isSource ? "routes to" : "routed from";
case "defines_schema":
  return isSource ? "defines schema for" : "schema defined by";
case "triggers":
  return isSource ? "triggers" : "triggered by";
```

**Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/NodeInfo.tsx
git commit -m "feat(dashboard): add new node/edge type support to NodeInfo sidebar"
```

---

## Tarefa 18: Dashboard — Atualizar ProjectOverview com Breakdown por Tipo de Arquivo

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/ProjectOverview.tsx`

**Passo 1: Adicionar distribuição de tipo de arquivo**

Adicione uma seção "File Types" após o grid de stats que mostra contagem por categoria de tipo de node:
- Code: file + function + class
- Config: config
- Docs: document
- Infra: service + resource + pipeline
- Data: table + endpoint + schema

Use bolinhas coloridas correspondentes às cores dos tipos de node.

**Passo 2: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/ProjectOverview.tsx
git commit -m "feat(dashboard): add file type breakdown to ProjectOverview"
```

---

## Tarefa 19: Dashboard — Adicionar Controles de Filtro

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/store.ts`
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/GraphView.tsx`
- Modificar: `understand-anything-plugin/packages/dashboard/src/App.tsx`

**Passo 1: Adicionar estado de filtro ao store**

Adicione ao store Zustand:
```typescript
nodeTypeFilters: Record<string, boolean>; // { code: true, config: true, docs: true, infra: true, data: true }
toggleNodeTypeFilter: (category: string) => void;
```

Padrão de todas as categorias para `true` (visíveis).

**Passo 2: Aplicar filtros na computação de topologia do GraphView**

Em `useLayerDetailTopology`, filtre nodes baseado em `nodeTypeFilters` antes do layout.

**Passo 3: Adicionar checkboxes de filtro ao header do App.tsx**

Adicione pequenos toggles de checkbox próximos à legenda de layer para cada categoria.

**Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/store.ts understand-anything-plugin/packages/dashboard/src/components/GraphView.tsx understand-anything-plugin/packages/dashboard/src/App.tsx
git commit -m "feat(dashboard): add node type category filter controls"
```

---

## Tarefa 20: Verificação de Build do Dashboard

**Passo 1: Buildar o dashboard**

Rode: `pnpm --filter @understand-anything/dashboard build`
Esperado: Sucesso, sem erros TypeScript

**Passo 2: Buildar o pacote core**

Rode: `pnpm --filter @understand-anything/core build`
Esperado: Sucesso

**Passo 3: Rodar todos os testes do core**

Rode: `pnpm --filter @understand-anything/core test`
Esperado: Todos os testes passam

**Passo 4: Rodar lint**

Rode: `pnpm lint`
Esperado: Sem erros

**Passo 5: Commit das correções de lint, se houver**

```bash
git add -A
git commit -m "fix: lint and build fixes for universal file type support"
```

---

## Tarefa 21: Teste de Integração — Verificação Ponta a Ponta

**Passo 1: Smoke test no dev server**

Rode: `pnpm dev:dashboard`
- Carregue um grafo de conhecimento que inclua nodes não-código
- Verifique que os novos tipos de node renderizam com as cores corretas
- Verifique que o sidebar NodeInfo mostra os novos labels de edge
- Verifique que os controles de filtro funcionam

**Passo 2: Gerar grafo de teste com nodes não-código**

Atualize `scripts/generate-large-graph.mjs` para incluir tipos de node não-código na geração aleatória, depois gere um grafo de teste e carregue-o no dashboard.

**Passo 3: Commit**

```bash
git add scripts/generate-large-graph.mjs
git commit -m "feat(scripts): include non-code node types in test graph generator"
```

---

## Tarefa 22: Version Bump & Commit Final

**Arquivos:**
- Modificar: `understand-anything-plugin/package.json` → bump da versão
- Modificar: `.claude-plugin/marketplace.json` → bump da versão
- Modificar: `.claude-plugin/plugin.json` → bump da versão
- Modificar: `.cursor-plugin/plugin.json` → bump da versão

**Passo 1: Bump da versão em todos os 4 arquivos** (ex.: 1.3.0 → 1.4.0)

**Passo 2: Commit final**

```bash
git add understand-anything-plugin/package.json .claude-plugin/marketplace.json .claude-plugin/plugin.json .cursor-plugin/plugin.json
git commit -m "chore: bump version to 1.4.0 for universal file type support"
```

---

## Resumo de Todas as Tarefas

| # | Tarefa | Arquivos | Depende de |
|---|------|-------|------------|
| 1 | Estender tipos do core | types.ts | — |
| 2 | Estender validação de schema | schema.ts | 1 |
| 3 | Atualizar PluginRegistry | registry.ts | 1 |
| 4 | Adicionar 26 language configs | languages/configs/ | 1 |
| 5 | Construir 12 parsers customizados | plugins/parsers/ | 1, 3 |
| 6 | Atualizar GraphBuilder | graph-builder.ts | 1 |
| 7 | Atualizar exports do core | index.ts | 1-6 |
| 8 | Atualizar prompt do project-scanner | project-scanner-prompt.md | — |
| 9 | Atualizar prompt do file-analyzer | file-analyzer-prompt.md | — |
| 10 | Atualizar prompt do architecture-analyzer | architecture-analyzer-prompt.md | — |
| 11 | Atualizar prompt do tour-builder | tour-builder-prompt.md | — |
| 12 | Atualizar prompt do graph-reviewer | graph-reviewer-prompt.md | — |
| 13 | Adicionar snippets de contexto de linguagem | languages/*.md | — |
| 14 | Atualizar pipeline em SKILL.md | SKILL.md | 8-13 |
| 15 | Cores de tema do dashboard | presets.ts | — |
| 16 | CustomNode do dashboard | CustomNode.tsx | 15 |
| 17 | NodeInfo do dashboard | NodeInfo.tsx | 15 |
| 18 | ProjectOverview do dashboard | ProjectOverview.tsx | 15 |
| 19 | Controles de filtro do dashboard | store.ts, GraphView.tsx, App.tsx | 15-18 |
| 20 | Verificação de build | — | 1-19 |
| 21 | Teste de integração | — | 20 |
| 22 | Version bump | package.json × 4 | 21 |

**Grupos paralelizáveis:**
- Tarefas 1-7 (core) são sequenciais
- Tarefas 8-14 (prompts de agente) podem rodar em paralelo entre si, e em paralelo com as Tarefas 15-19 (dashboard)
- Tarefas 20-22 são sequenciais e dependem de todas as tarefas anteriores
