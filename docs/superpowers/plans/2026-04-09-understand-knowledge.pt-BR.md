# Plano de Implementação do /understand-knowledge

> 🇬🇧 Versão original em inglês: [2026-04-09-understand-knowledge.md](./2026-04-09-understand-knowledge.md)

> **Para workers agênticos:** SUB-SKILL OBRIGATÓRIA: Use superpowers:subagent-driven-development (recomendado) ou superpowers:executing-plans para implementar este plano tarefa a tarefa. Os passos usam sintaxe de checkbox (`- [ ]`) para acompanhamento.

**Objetivo:** Adicionar uma skill `/understand-knowledge` que recebe qualquer pasta de notas markdown (Obsidian, Logseq, Dendron, Foam, estilo Karpathy, Zettelkasten ou simples) e produz um grafo de conhecimento interativo com nodes e edges tipados, e visualização em dashboard.

**Arquitetura:** Estende o schema existente com 5 tipos de node de conhecimento e 6 tipos de edge de conhecimento. Um novo pipeline de 5 agentes (knowledge-scanner → format-detector → article-analyzer → relationship-builder → graph-reviewer) processa arquivos markdown. O dashboard renderiza grafos de conhecimento com layout vertical, uma sidebar específica de conhecimento e um painel de modo de leitura — tudo guiado por um novo campo `kind` no objeto raiz do grafo.

**Stack:** TypeScript, Zod (validação de schema), React + ReactFlow (dashboard), dagre (layout), TailwindCSS v4, Vitest (testes)

**Spec:** `docs/superpowers/specs/2026-04-09-understand-knowledge-design.md`

---

## Estrutura de Arquivos

### Alterações no pacote core
- Modificar: `understand-anything-plugin/packages/core/src/types.ts` — adicionar 5 tipos de node, 6 tipos de edge, interface `KnowledgeMeta`, campo `kind`
- Modificar: `understand-anything-plugin/packages/core/src/schema.ts` — adicionar novos tipos aos schemas Zod, adicionar aliases
- Modificar: `understand-anything-plugin/packages/core/src/types.test.ts` — adicionar testes para novos tipos
- Teste: `understand-anything-plugin/packages/core/src/__tests__/knowledge-schema.test.ts` — testes de validação para schema específico de conhecimento

### Alterações no dashboard
- Modificar: `understand-anything-plugin/packages/dashboard/src/store.ts` — adicionar tipos de node de conhecimento, categorias de edge, `ViewMode`, categorias de node
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/CustomNode.tsx` — adicionar cores para 5 novos tipos de node
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/NodeInfo.tsx` — adicionar cores de badge e labels de edge para novos tipos, adicionar seções da sidebar de conhecimento
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/ProjectOverview.tsx` — adicionar estatísticas específicas de conhecimento
- Modificar: `understand-anything-plugin/packages/dashboard/src/index.css` — adicionar variáveis CSS para 5 novas cores de node
- Modificar: `understand-anything-plugin/packages/dashboard/src/App.tsx` — detectar campo `kind`, definir view mode
- Criar: `understand-anything-plugin/packages/dashboard/src/components/KnowledgeInfo.tsx` — sidebar específica de conhecimento
- Criar: `understand-anything-plugin/packages/dashboard/src/components/ReadingPanel.tsx` — overlay de leitura de artigo completo

### Definições de skill & agentes
- Criar: `understand-anything-plugin/skills/understand-knowledge/SKILL.md` — ponto de entrada da skill
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/obsidian.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/logseq.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/dendron.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/foam.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/karpathy.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/zettelkasten.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/plain.md`
- Criar: `understand-anything-plugin/agents/knowledge-scanner.md`
- Criar: `understand-anything-plugin/agents/format-detector.md`
- Criar: `understand-anything-plugin/agents/article-analyzer.md`
- Criar: `understand-anything-plugin/agents/relationship-builder.md`

O agente existente `graph-reviewer.md` é reutilizado para a etapa final de validação.

---

## Tarefa 1: Estender Tipos do Core

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/types.ts`

- [ ] **Passo 1: Adicionar tipos de node de conhecimento à union NodeType**

Em `understand-anything-plugin/packages/core/src/types.ts`, adicione os 5 tipos de conhecimento após os tipos de domínio:

```typescript
// Node types (21 total: 5 code + 8 non-code + 3 domain + 5 knowledge)
export type NodeType =
  | "file" | "function" | "class" | "module" | "concept"
  | "config" | "document" | "service" | "table" | "endpoint"
  | "pipeline" | "schema" | "resource"
  | "domain" | "flow" | "step"
  | "article" | "entity" | "topic" | "claim" | "source";
```

- [ ] **Passo 2: Adicionar tipos de edge de conhecimento à union EdgeType**

```typescript
// Edge types (35 total in 8 categories)
export type EdgeType =
  | "imports" | "exports" | "contains" | "inherits" | "implements"
  | "calls" | "subscribes" | "publishes" | "middleware"
  | "reads_from" | "writes_to" | "transforms" | "validates"
  | "depends_on" | "tested_by" | "configures"
  | "related" | "similar_to"
  | "deploys" | "serves" | "provisions" | "triggers"
  | "migrates" | "documents" | "routes" | "defines_schema"
  | "contains_flow" | "flow_step" | "cross_domain"
  | "cites" | "contradicts" | "builds_on" | "exemplifies" | "categorized_under" | "authored_by";
```

- [ ] **Passo 3: Adicionar interface KnowledgeMeta**

Adicione após a interface `DomainMeta`:

```typescript
// Optional knowledge metadata for article/entity/topic/claim/source nodes
export interface KnowledgeMeta {
  format?: "obsidian" | "logseq" | "dendron" | "foam" | "karpathy" | "zettelkasten" | "plain";
  wikilinks?: string[];
  backlinks?: string[];
  frontmatter?: Record<string, unknown>;
  sourceUrl?: string;
  confidence?: number; // 0-1, for LLM-inferred relationships
}
```

- [ ] **Passo 4: Adicionar knowledgeMeta ao GraphNode**

```typescript
export interface GraphNode {
  id: string;
  type: NodeType;
  name: string;
  filePath?: string;
  lineRange?: [number, number];
  summary: string;
  tags: string[];
  complexity: "simple" | "moderate" | "complex";
  languageNotes?: string;
  domainMeta?: DomainMeta;
  knowledgeMeta?: KnowledgeMeta;
}
```

- [ ] **Passo 5: Adicionar campo kind ao KnowledgeGraph**

```typescript
export interface KnowledgeGraph {
  version: string;
  kind?: "codebase" | "knowledge"; // undefined defaults to "codebase" for backward compat
  project: ProjectMeta;
  nodes: GraphNode[];
  edges: GraphEdge[];
  layers: Layer[];
  tour: TourStep[];
}
```

- [ ] **Passo 6: Build do core e verificar ausência de erros de tipo**

Execute: `pnpm --filter @understand-anything/core build`
Esperado: Build limpo, sem erros

- [ ] **Passo 7: Commit**

```bash
git add understand-anything-plugin/packages/core/src/types.ts
git commit -m "feat(core): add knowledge node types, edge types, KnowledgeMeta, and graph kind field"
```

---

## Tarefa 2: Estender Validação de Schema

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/schema.ts`
- Criar: `understand-anything-plugin/packages/core/src/__tests__/knowledge-schema.test.ts`

- [ ] **Passo 1: Adicionar tipos de edge de conhecimento ao EdgeTypeSchema**

Em `understand-anything-plugin/packages/core/src/schema.ts`, atualize o `EdgeTypeSchema` z.enum para incluir os 6 novos tipos:

```typescript
export const EdgeTypeSchema = z.enum([
  "imports", "exports", "contains", "inherits", "implements",
  "calls", "subscribes", "publishes", "middleware",
  "reads_from", "writes_to", "transforms", "validates",
  "depends_on", "tested_by", "configures",
  "related", "similar_to",
  "deploys", "serves", "provisions", "triggers",
  "migrates", "documents", "routes", "defines_schema",
  "contains_flow", "flow_step", "cross_domain",
  // Knowledge
  "cites", "contradicts", "builds_on", "exemplifies", "categorized_under", "authored_by",
]);
```

- [ ] **Passo 2: Adicionar aliases de tipos de node de conhecimento**

Adicione a `NODE_TYPE_ALIASES`:

```typescript
  // Knowledge aliases
  note: "article",
  page: "article",
  wiki_page: "article",
  person: "entity",
  tool: "entity",
  paper: "entity",
  organization: "entity",
  org: "entity",
  category: "topic",
  theme: "topic",
  tag_topic: "topic",
  assertion: "claim",
  insight: "claim",
  takeaway: "claim",
  reference: "source",
  raw: "source",
  citation: "source",
```

- [ ] **Passo 3: Adicionar aliases de tipos de edge de conhecimento**

Adicione a `EDGE_TYPE_ALIASES`:

```typescript
  // Knowledge aliases
  references: "cites",
  cited_by: "cites",
  sourced_from: "cites",
  conflicts_with: "contradicts",
  disagrees_with: "contradicts",
  extends: "builds_on",  // Note: "extends" was already mapped to "inherits" — knowledge context will use builds_on via the relationship-builder agent prompt, so keep "extends" → "inherits" for code
  refines: "builds_on",
  deepens: "builds_on",
  example_of: "exemplifies",
  instance_of: "exemplifies",
  belongs_to: "categorized_under",
  tagged_with: "categorized_under",
  part_of: "categorized_under",
  written_by: "authored_by",
  created_by: "authored_by",
```

- [ ] **Passo 4: Escrever o teste que deve falhar para validação de grafo de conhecimento**

Crie `understand-anything-plugin/packages/core/src/__tests__/knowledge-schema.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { validateGraph } from "../schema";
import type { KnowledgeGraph } from "../types";

describe("knowledge graph schema validation", () => {
  const minimalKnowledgeGraph: KnowledgeGraph = {
    version: "1.0",
    kind: "knowledge",
    project: {
      name: "Test KB",
      languages: [],
      frameworks: [],
      description: "A test knowledge base",
      analyzedAt: new Date().toISOString(),
      gitCommitHash: "abc123",
    },
    nodes: [
      {
        id: "article:test-note",
        type: "article",
        name: "Test Note",
        summary: "A test article node",
        tags: ["test"],
        complexity: "simple",
      },
      {
        id: "entity:karpathy",
        type: "entity",
        name: "Andrej Karpathy",
        summary: "AI researcher",
        tags: ["person", "ai"],
        complexity: "simple",
      },
      {
        id: "topic:pkm",
        type: "topic",
        name: "Personal Knowledge Management",
        summary: "Tools and methods for managing personal knowledge",
        tags: ["knowledge", "productivity"],
        complexity: "moderate",
      },
    ],
    edges: [
      {
        source: "article:test-note",
        target: "entity:karpathy",
        type: "authored_by",
        direction: "forward",
        weight: 0.8,
      },
      {
        source: "article:test-note",
        target: "topic:pkm",
        type: "categorized_under",
        direction: "forward",
        weight: 0.7,
      },
    ],
    layers: [
      {
        id: "layer:pkm",
        name: "PKM",
        description: "Personal Knowledge Management topic cluster",
        nodeIds: ["article:test-note", "topic:pkm"],
      },
    ],
    tour: [],
  };

  it("validates a minimal knowledge graph", () => {
    const result = validateGraph(minimalKnowledgeGraph);
    const fatals = result.issues.filter((i) => i.level === "fatal");
    expect(fatals).toHaveLength(0);
  });

  it("accepts all knowledge node types", () => {
    const graph = {
      ...minimalKnowledgeGraph,
      nodes: [
        ...minimalKnowledgeGraph.nodes,
        { id: "claim:rag-bad", type: "claim", name: "RAG loses context", summary: "An assertion", tags: ["claim"], complexity: "simple" },
        { id: "source:paper1", type: "source", name: "Attention paper", summary: "A source", tags: ["paper"], complexity: "simple" },
      ],
    };
    const result = validateGraph(graph);
    const fatals = result.issues.filter((i) => i.level === "fatal");
    expect(fatals).toHaveLength(0);
  });

  it("accepts all knowledge edge types", () => {
    const graph = {
      ...minimalKnowledgeGraph,
      nodes: [
        ...minimalKnowledgeGraph.nodes,
        { id: "claim:c1", type: "claim", name: "Claim 1", summary: "c1", tags: [], complexity: "simple" },
        { id: "claim:c2", type: "claim", name: "Claim 2", summary: "c2", tags: [], complexity: "simple" },
        { id: "source:s1", type: "source", name: "Source 1", summary: "s1", tags: [], complexity: "simple" },
        { id: "article:a2", type: "article", name: "Article 2", summary: "a2", tags: [], complexity: "simple" },
      ],
      edges: [
        ...minimalKnowledgeGraph.edges,
        { source: "article:test-note", target: "source:s1", type: "cites", direction: "forward", weight: 0.7 },
        { source: "claim:c1", target: "claim:c2", type: "contradicts", direction: "forward", weight: 0.6 },
        { source: "article:a2", target: "article:test-note", type: "builds_on", direction: "forward", weight: 0.7 },
        { source: "entity:karpathy", target: "topic:pkm", type: "exemplifies", direction: "forward", weight: 0.5 },
      ],
    };
    const result = validateGraph(graph);
    const fatals = result.issues.filter((i) => i.level === "fatal");
    expect(fatals).toHaveLength(0);
  });

  it("resolves knowledge node type aliases", () => {
    const graph = {
      ...minimalKnowledgeGraph,
      nodes: [
        { id: "note:n1", type: "note", name: "A Note", summary: "note alias", tags: [], complexity: "simple" },
        { id: "person:p1", type: "person", name: "A Person", summary: "person alias", tags: [], complexity: "simple" },
      ],
      edges: [],
      layers: [],
    };
    const result = validateGraph(graph);
    const noteNode = result.graph.nodes.find((n) => n.id === "note:n1");
    const personNode = result.graph.nodes.find((n) => n.id === "person:p1");
    expect(noteNode?.type).toBe("article");
    expect(personNode?.type).toBe("entity");
  });

  it("resolves knowledge edge type aliases", () => {
    const graph = {
      ...minimalKnowledgeGraph,
      edges: [
        { source: "article:test-note", target: "entity:karpathy", type: "written_by", direction: "forward", weight: 0.8 },
      ],
    };
    const result = validateGraph(graph);
    const edge = result.graph.edges.find((e) => e.source === "article:test-note" && e.target === "entity:karpathy");
    expect(edge?.type).toBe("authored_by");
  });
});
```

- [ ] **Passo 5: Executar os testes para verificar que falham**

Execute: `pnpm --filter @understand-anything/core test -- --run src/__tests__/knowledge-schema.test.ts`
Esperado: Testes falham porque EdgeTypeSchema ainda não inclui tipos de conhecimento (se schema.ts não foi atualizado), ou passam se os Passos 1-3 foram feitos corretamente.

- [ ] **Passo 6: Executar todos os testes do core para verificar que nada quebrou**

Execute: `pnpm --filter @understand-anything/core test -- --run`
Esperado: Todos os testes existentes passam, novos testes de conhecimento passam

- [ ] **Passo 7: Commit**

```bash
git add understand-anything-plugin/packages/core/src/schema.ts understand-anything-plugin/packages/core/src/__tests__/knowledge-schema.test.ts
git commit -m "feat(core): add knowledge types to schema validation with aliases and tests"
```

---

## Tarefa 3: Dashboard — Variáveis CSS & Cores de Node

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/index.css`

- [ ] **Passo 1: Adicionar variáveis CSS para 5 tipos de node de conhecimento**

Em `understand-anything-plugin/packages/dashboard/src/index.css`, adicione após a linha existente `--color-node-resource`:

```css
  /* Knowledge node colors */
  --color-node-article: #d4a574;   /* warm amber */
  --color-node-entity: #7ba4c9;    /* soft blue */
  --color-node-topic: #c9b06c;     /* muted gold */
  --color-node-claim: #6fb07a;     /* soft green */
  --color-node-source: #8a8a8a;    /* gray */
```

- [ ] **Passo 2: Adicionar utilitários text-color do Tailwind para nodes de conhecimento**

Verifique se TailwindCSS v4 captura as variáveis CSS automaticamente. Se o padrão existente usa classes `text-node-*` definidas em outro lugar, adicione entradas correspondentes. Verifique se há um config do Tailwind ou se as variáveis CSS são consumidas diretamente.

Veja como `text-node-file` etc. existentes são definidos — se estiverem no arquivo CSS como utility classes, adicione:

```css
  .text-node-article { color: var(--color-node-article); }
  .text-node-entity { color: var(--color-node-entity); }
  .text-node-topic { color: var(--color-node-topic); }
  .text-node-claim { color: var(--color-node-claim); }
  .text-node-source { color: var(--color-node-source); }
```

E variantes `border-node-*` e `bg-node-*` correspondentes se o padrão exigir.

- [ ] **Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/index.css
git commit -m "feat(dashboard): add CSS variables and utility classes for knowledge node types"
```

---

## Tarefa 4: Dashboard — Store & Mapas de Tipo

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/store.ts`

- [ ] **Passo 1: Adicionar tipos de conhecimento à union NodeType**

Atualize o `NodeType` local em store.ts:

```typescript
export type NodeType = "file" | "function" | "class" | "module" | "concept" | "config" | "document" | "service" | "table" | "endpoint" | "pipeline" | "schema" | "resource" | "domain" | "flow" | "step" | "article" | "entity" | "topic" | "claim" | "source";
```

- [ ] **Passo 2: Adicionar categoria de edge de conhecimento**

Atualize `EdgeCategory` e `EDGE_CATEGORY_MAP`:

```typescript
export type EdgeCategory = "structural" | "behavioral" | "data-flow" | "dependencies" | "semantic" | "infrastructure" | "domain" | "knowledge";

export const EDGE_CATEGORY_MAP: Record<EdgeCategory, string[]> = {
  structural: ["imports", "exports", "contains", "inherits", "implements"],
  behavioral: ["calls", "subscribes", "publishes", "middleware"],
  "data-flow": ["reads_from", "writes_to", "transforms", "validates"],
  dependencies: ["depends_on", "tested_by", "configures"],
  semantic: ["related", "similar_to"],
  infrastructure: ["deploys", "serves", "provisions", "triggers"],
  domain: ["contains_flow", "flow_step", "cross_domain"],
  knowledge: ["cites", "contradicts", "builds_on", "exemplifies", "categorized_under", "authored_by"],
};
```

- [ ] **Passo 3: Adicionar knowledge a ALL_NODE_TYPES e ALL_EDGE_CATEGORIES**

```typescript
export const ALL_NODE_TYPES: NodeType[] = ["file", "function", "class", "module", "concept", "config", "document", "service", "table", "endpoint", "pipeline", "schema", "resource", "domain", "flow", "step", "article", "entity", "topic", "claim", "source"];

export const ALL_EDGE_CATEGORIES: EdgeCategory[] = ["structural", "behavioral", "data-flow", "dependencies", "semantic", "infrastructure", "domain", "knowledge"];
```

- [ ] **Passo 4: Adicionar "knowledge" a ViewMode e NodeCategory**

```typescript
export type ViewMode = "structural" | "domain" | "knowledge";

export type NodeCategory = "code" | "config" | "docs" | "infra" | "data" | "domain" | "knowledge";
```

Atualize o `NODE_CATEGORY_MAP` (encontre onde mapeia tipos de node para categorias) para incluir:

```typescript
  article: "knowledge",
  entity: "knowledge",
  topic: "knowledge",
  claim: "knowledge",
  source: "knowledge",
```

- [ ] **Passo 5: Adicionar padrão de filtro para tipo de node de conhecimento**

No estado inicial `nodeTypeFilters` do store, adicione:

```typescript
nodeTypeFilters: { code: true, config: true, docs: true, infra: true, data: true, domain: true, knowledge: true },
```

- [ ] **Passo 6: Build do dashboard e verificar ausência de erros**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo

- [ ] **Passo 7: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/store.ts
git commit -m "feat(dashboard): add knowledge types to store, edge categories, and view mode"
```

---

## Tarefa 5: Dashboard — Mapas de Tipo de CustomNode & NodeInfo

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/CustomNode.tsx`
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/NodeInfo.tsx`

- [ ] **Passo 1: Adicionar cores de node de conhecimento em CustomNode.tsx**

No mapa `typeColors`, adicione após a entrada `step`:

```typescript
  // Knowledge
  article: "var(--color-node-article)",
  entity: "var(--color-node-entity)",
  topic: "var(--color-node-topic)",
  claim: "var(--color-node-claim)",
  source: "var(--color-node-source)",
```

No mapa `typeTextColors`, adicione:

```typescript
  // Knowledge
  article: "text-node-article",
  entity: "text-node-entity",
  topic: "text-node-topic",
  claim: "text-node-claim",
  source: "text-node-source",
```

- [ ] **Passo 2: Adicionar cores de badge de node de conhecimento em NodeInfo.tsx**

No mapa `typeBadgeColors`, adicione:

```typescript
  // Knowledge
  article: "text-node-article border border-node-article/30 bg-node-article/10",
  entity: "text-node-entity border border-node-entity/30 bg-node-entity/10",
  topic: "text-node-topic border border-node-topic/30 bg-node-topic/10",
  claim: "text-node-claim border border-node-claim/30 bg-node-claim/10",
  source: "text-node-source border border-node-source/30 bg-node-source/10",
```

- [ ] **Passo 3: Adicionar labels de edge de conhecimento em NodeInfo.tsx**

No mapa `EDGE_LABELS`, adicione:

```typescript
  // Knowledge
  cites: { forward: "cites", backward: "cited by" },
  contradicts: { forward: "contradicts", backward: "contradicted by" },
  builds_on: { forward: "builds on", backward: "built upon by" },
  exemplifies: { forward: "exemplifies", backward: "exemplified by" },
  categorized_under: { forward: "categorized under", backward: "categorizes" },
  authored_by: { forward: "authored by", backward: "authored" },
```

- [ ] **Passo 4: Build do dashboard e verificar**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo, sem erros de tipo

- [ ] **Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/CustomNode.tsx understand-anything-plugin/packages/dashboard/src/components/NodeInfo.tsx
git commit -m "feat(dashboard): add knowledge node colors, badge colors, and edge labels"
```

---

## Tarefa 6: Dashboard — Componente de Sidebar de Conhecimento

**Arquivos:**
- Criar: `understand-anything-plugin/packages/dashboard/src/components/KnowledgeInfo.tsx`
- Modificar: `understand-anything-plugin/packages/dashboard/src/App.tsx`

- [ ] **Passo 1: Criar KnowledgeInfo.tsx**

Crie `understand-anything-plugin/packages/dashboard/src/components/KnowledgeInfo.tsx`:

```tsx
import { useDashboardStore } from "../store";
import type { GraphNode, GraphEdge, KnowledgeGraph } from "@understand-anything/core/types";

const KNOWLEDGE_NODE_TYPES = new Set(["article", "entity", "topic", "claim", "source"]);

function getBacklinks(nodeId: string, edges: GraphEdge[]): string[] {
  return edges
    .filter((e) => e.target === nodeId)
    .map((e) => e.source);
}

function getOutgoingLinks(nodeId: string, edges: GraphEdge[]): string[] {
  return edges
    .filter((e) => e.source === nodeId)
    .map((e) => e.target);
}

function NodeLink({ nodeId, nodes, onNavigate }: { nodeId: string; nodes: GraphNode[]; onNavigate: (id: string) => void }) {
  const node = nodes.find((n) => n.id === nodeId);
  if (!node) return <span className="text-text-muted text-xs">{nodeId}</span>;
  return (
    <button
      className="text-accent-dim hover:text-accent text-xs text-left truncate block w-full"
      onClick={() => onNavigate(nodeId)}
    >
      <span className="text-text-muted mr-1">[{node.type}]</span>
      {node.name}
    </button>
  );
}

export default function KnowledgeInfo() {
  const graph = useDashboardStore((s) => s.graph);
  const selectedNode = useDashboardStore((s) => s.selectedNode);
  const setSelectedNode = useDashboardStore((s) => s.setSelectedNode);

  if (!graph || !selectedNode) return null;

  const node = graph.nodes.find((n) => n.id === selectedNode);
  if (!node) return null;

  const backlinks = getBacklinks(node.id, graph.edges);
  const outgoing = getOutgoingLinks(node.id, graph.edges);
  const meta = node.knowledgeMeta;

  return (
    <div className="space-y-4 p-4">
      {/* Header */}
      <div>
        <div className="text-xs text-text-muted uppercase tracking-wider mb-1">{node.type}</div>
        <h2 className="text-lg font-serif text-text-primary">{node.name}</h2>
      </div>

      {/* Summary */}
      <p className="text-sm text-text-secondary leading-relaxed">{node.summary}</p>

      {/* Tags */}
      {node.tags.length > 0 && (
        <div className="flex flex-wrap gap-1">
          {node.tags.map((tag) => (
            <span key={tag} className="text-[10px] px-2 py-0.5 rounded-full bg-elevated border border-border-subtle text-text-muted">
              {tag}
            </span>
          ))}
        </div>
      )}

      {/* Knowledge-specific metadata */}
      {meta?.sourceUrl && (
        <div>
          <div className="text-xs text-text-muted uppercase tracking-wider mb-1">Source</div>
          <span className="text-xs text-accent-dim break-all">{meta.sourceUrl}</span>
        </div>
      )}

      {meta?.confidence !== undefined && (
        <div>
          <div className="text-xs text-text-muted uppercase tracking-wider mb-1">Confidence</div>
          <div className="flex items-center gap-2">
            <div className="flex-1 h-1.5 bg-elevated rounded-full overflow-hidden">
              <div
                className="h-full bg-accent-dim rounded-full"
                style={{ width: `${meta.confidence * 100}%` }}
              />
            </div>
            <span className="text-xs text-text-muted">{Math.round(meta.confidence * 100)}%</span>
          </div>
        </div>
      )}

      {/* Frontmatter */}
      {meta?.frontmatter && Object.keys(meta.frontmatter).length > 0 && (
        <div>
          <div className="text-xs text-text-muted uppercase tracking-wider mb-1">Frontmatter</div>
          <div className="space-y-1">
            {Object.entries(meta.frontmatter).map(([key, value]) => (
              <div key={key} className="text-xs">
                <span className="text-text-muted">{key}:</span>{" "}
                <span className="text-text-secondary">{String(value)}</span>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* Backlinks */}
      {backlinks.length > 0 && (
        <div>
          <div className="text-xs text-text-muted uppercase tracking-wider mb-1">
            Backlinks ({backlinks.length})
          </div>
          <div className="space-y-0.5">
            {backlinks.map((id) => (
              <NodeLink key={id} nodeId={id} nodes={graph.nodes} onNavigate={setSelectedNode} />
            ))}
          </div>
        </div>
      )}

      {/* Outgoing */}
      {outgoing.length > 0 && (
        <div>
          <div className="text-xs text-text-muted uppercase tracking-wider mb-1">
            Outgoing Links ({outgoing.length})
          </div>
          <div className="space-y-0.5">
            {outgoing.map((id) => (
              <NodeLink key={id} nodeId={id} nodes={graph.nodes} onNavigate={setSelectedNode} />
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Passo 2: Integrar KnowledgeInfo na renderização da sidebar do App.tsx**

Em `understand-anything-plugin/packages/dashboard/src/App.tsx`, encontre onde a sidebar renderiza `NodeInfo` e adicione uma condição: se `graph.kind === "knowledge"` e um node estiver selecionado, renderize `KnowledgeInfo` em vez de `NodeInfo`.

Importe no topo:
```typescript
import KnowledgeInfo from "./components/KnowledgeInfo";
```

Na seção da sidebar, envolva a renderização existente de NodeInfo:
```tsx
{graph?.kind === "knowledge" ? <KnowledgeInfo /> : <NodeInfo />}
```

- [ ] **Passo 3: Build do dashboard e verificar**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo

- [ ] **Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/KnowledgeInfo.tsx understand-anything-plugin/packages/dashboard/src/App.tsx
git commit -m "feat(dashboard): add KnowledgeInfo sidebar component for knowledge graphs"
```

---

## Tarefa 7: Dashboard — Painel de Leitura

**Arquivos:**
- Criar: `understand-anything-plugin/packages/dashboard/src/components/ReadingPanel.tsx`
- Modificar: `understand-anything-plugin/packages/dashboard/src/App.tsx`

- [ ] **Passo 1: Criar ReadingPanel.tsx**

Crie `understand-anything-plugin/packages/dashboard/src/components/ReadingPanel.tsx`:

```tsx
import { useState } from "react";
import { useDashboardStore } from "../store";

export default function ReadingPanel() {
  const graph = useDashboardStore((s) => s.graph);
  const selectedNode = useDashboardStore((s) => s.selectedNode);
  const setSelectedNode = useDashboardStore((s) => s.setSelectedNode);
  const [isExpanded, setIsExpanded] = useState(false);

  if (!graph || graph.kind !== "knowledge" || !selectedNode) return null;

  const node = graph.nodes.find((n) => n.id === selectedNode);
  if (!node || node.type !== "article") return null;

  // Get backlinks for this article
  const backlinks = graph.edges
    .filter((e) => e.target === node.id)
    .map((e) => {
      const sourceNode = graph.nodes.find((n) => n.id === e.source);
      return sourceNode ? { id: sourceNode.id, name: sourceNode.name, type: sourceNode.type } : null;
    })
    .filter(Boolean) as { id: string; name: string; type: string }[];

  return (
    <div
      className={`absolute bottom-0 left-0 right-0 bg-surface border-t border-border-subtle transition-all duration-300 z-50 ${
        isExpanded ? "h-[70vh]" : "h-[45vh]"
      }`}
    >
      {/* Header bar */}
      <div className="flex items-center justify-between px-4 py-2 border-b border-border-subtle bg-elevated">
        <div className="flex items-center gap-2">
          <span className="text-xs text-text-muted uppercase tracking-wider">Reading</span>
          <span className="text-sm font-serif text-text-primary">{node.name}</span>
        </div>
        <div className="flex items-center gap-2">
          <button
            onClick={() => setIsExpanded(!isExpanded)}
            className="text-xs text-text-muted hover:text-text-primary px-2 py-1"
          >
            {isExpanded ? "Collapse" : "Expand"}
          </button>
          <button
            onClick={() => setSelectedNode(null)}
            className="text-xs text-text-muted hover:text-text-primary px-2 py-1"
          >
            Close
          </button>
        </div>
      </div>

      <div className="flex h-[calc(100%-40px)] overflow-hidden">
        {/* Main content */}
        <div className="flex-1 overflow-y-auto p-6">
          <div className="max-w-3xl mx-auto">
            <h1 className="text-2xl font-serif text-text-primary mb-4">{node.name}</h1>

            {/* Tags */}
            {node.tags.length > 0 && (
              <div className="flex flex-wrap gap-1 mb-4">
                {node.tags.map((tag) => (
                  <span key={tag} className="text-[10px] px-2 py-0.5 rounded-full bg-elevated border border-border-subtle text-text-muted">
                    {tag}
                  </span>
                ))}
              </div>
            )}

            {/* Article content (summary for now — full markdown rendering is a future enhancement) */}
            <div className="prose prose-invert prose-sm max-w-none">
              <p className="text-text-secondary leading-relaxed">{node.summary}</p>
            </div>

            {/* Frontmatter metadata */}
            {node.knowledgeMeta?.frontmatter && Object.keys(node.knowledgeMeta.frontmatter).length > 0 && (
              <div className="mt-6 p-3 rounded-lg bg-elevated border border-border-subtle">
                <div className="text-xs text-text-muted uppercase tracking-wider mb-2">Metadata</div>
                {Object.entries(node.knowledgeMeta.frontmatter).map(([key, value]) => (
                  <div key={key} className="text-xs mb-1">
                    <span className="text-text-muted">{key}:</span>{" "}
                    <span className="text-text-secondary">{String(value)}</span>
                  </div>
                ))}
              </div>
            )}
          </div>
        </div>

        {/* Backlinks sidebar */}
        {backlinks.length > 0 && (
          <div className="w-56 border-l border-border-subtle overflow-y-auto p-3 bg-elevated">
            <div className="text-xs text-text-muted uppercase tracking-wider mb-2">
              Backlinks ({backlinks.length})
            </div>
            <div className="space-y-1">
              {backlinks.map((link) => (
                <button
                  key={link.id}
                  className="text-xs text-accent-dim hover:text-accent text-left truncate block w-full"
                  onClick={() => setSelectedNode(link.id)}
                >
                  <span className="text-text-muted mr-1">[{link.type}]</span>
                  {link.name}
                </button>
              ))}
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Passo 2: Adicionar ReadingPanel ao App.tsx**

Importe e renderize `ReadingPanel` no layout principal do dashboard, posicionado no fundo:

```typescript
import ReadingPanel from "./components/ReadingPanel";
```

Adicione `<ReadingPanel />` dentro do container do dashboard, após a área de visualização do grafo.

- [ ] **Passo 3: Build e verificar**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo

- [ ] **Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/ReadingPanel.tsx understand-anything-plugin/packages/dashboard/src/App.tsx
git commit -m "feat(dashboard): add ReadingPanel for article reading mode in knowledge graphs"
```

---

## Tarefa 8: Dashboard — Layout Vertical para Grafos de Conhecimento

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/GraphView.tsx`
- Modificar: `understand-anything-plugin/packages/dashboard/src/utils/layout.ts` (se a direção ainda não for configurável)

- [ ] **Passo 1: Verificar como a direção do layout é passada para o dagre**

Leia `GraphView.tsx` para encontrar onde `applyDagreLayout` é chamado. O `applyDagreLayout` em layout.ts já aceita um parâmetro `direction: "TB" | "LR"` (padrão `"TB"`).

Encontre onde GraphView chama essa função e verifique qual direção ele passa.

- [ ] **Passo 2: Passar o kind do grafo para a decisão de layout**

Em `GraphView.tsx`, onde o layout é aplicado, verifique o campo `kind` do grafo. Se `kind === "knowledge"`, use `"TB"` (top-to-bottom). Se `kind === "codebase"` ou undefined, mantenha o padrão existente.

O objeto graph está disponível via o store. Adicione:

```typescript
const graphKind = useDashboardStore((s) => s.graph?.kind);
const layoutDirection = graphKind === "knowledge" ? "TB" : "LR";
```

Passe `layoutDirection` para a chamada de layout.

- [ ] **Passo 3: Build e verificar**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo

- [ ] **Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/GraphView.tsx
git commit -m "feat(dashboard): use vertical top-down layout for knowledge graphs"
```

---

## Tarefa 9: Dashboard — Estilização de Edge de Conhecimento

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/GraphView.tsx`

- [ ] **Passo 1: Adicionar mapa de estilos de edge de conhecimento**

Em `GraphView.tsx`, adicione um mapa de estilos para tipos de edge de conhecimento. Siga o padrão existente de `DomainGraphView.tsx` que usa a prop `style` do ReactFlow:

```typescript
const KNOWLEDGE_EDGE_STYLES: Record<string, React.CSSProperties> = {
  cites: { strokeDasharray: "6 3", strokeWidth: 1.5 },
  contradicts: { stroke: "#c97070", strokeWidth: 2 },
  builds_on: { stroke: "var(--color-accent)", strokeWidth: 2 },
  categorized_under: { stroke: "rgba(150,150,150,0.5)", strokeWidth: 1 },
  authored_by: { strokeDasharray: "3 3", stroke: "var(--color-node-entity)", strokeWidth: 1.5 },
  exemplifies: { strokeDasharray: "3 3", stroke: "var(--color-node-claim)", strokeWidth: 1.5 },
};
```

- [ ] **Passo 2: Aplicar estilos ao construir edges do ReactFlow**

Onde edges são convertidos para o formato ReactFlow, verifique se o grafo é `kind === "knowledge"` e se o tipo de edge tem um estilo de conhecimento. Mescle o estilo:

```typescript
const knowledgeStyle = graph?.kind === "knowledge" ? KNOWLEDGE_EDGE_STYLES[edge.type] : undefined;
// Merge with existing edge style
const style = { ...baseEdgeStyle, ...knowledgeStyle };
```

- [ ] **Passo 3: Build e verificar**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo

- [ ] **Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/GraphView.tsx
git commit -m "feat(dashboard): add distinct edge styles for knowledge relationship types"
```

---

## Tarefa 10: Dashboard — ProjectOverview Ciente de Conhecimento

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/components/ProjectOverview.tsx`

- [ ] **Passo 1: Adicionar estatísticas específicas de conhecimento**

Em `ProjectOverview.tsx`, detecte `graph.kind === "knowledge"` e mostre estatísticas específicas de conhecimento:

- Total de artigos, entidades, tópicos, claims, sources (em vez de "code, config, docs, infra, data")
- Formato detectado (do `knowledgeMeta.format` do primeiro node)
- Remover seções "Languages" e "Frameworks" para grafos de conhecimento (ficarão vazias)

Adicione após o grid de estatísticas existente:

```tsx
{graph.kind === "knowledge" && (
  <div className="space-y-2">
    <div className="text-xs text-text-muted uppercase tracking-wider">Knowledge Stats</div>
    <div className="grid grid-cols-2 gap-2">
      <StatBox label="Articles" value={graph.nodes.filter(n => n.type === "article").length} />
      <StatBox label="Entities" value={graph.nodes.filter(n => n.type === "entity").length} />
      <StatBox label="Topics" value={graph.nodes.filter(n => n.type === "topic").length} />
      <StatBox label="Claims" value={graph.nodes.filter(n => n.type === "claim").length} />
      <StatBox label="Sources" value={graph.nodes.filter(n => n.type === "source").length} />
    </div>
  </div>
)}
```

Reutilize ou crie um componente `StatBox` correspondendo ao estilo existente.

- [ ] **Passo 2: Esconder seções específicas de código condicionalmente**

Envolva as seções "Languages", "Frameworks" e a discriminação por tipo de arquivo específicas de código em uma condição:

```tsx
{graph.kind !== "knowledge" && (
  <>
    {/* existing languages/frameworks/file-types sections */}
  </>
)}
```

- [ ] **Passo 3: Build e verificar**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo

- [ ] **Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/ProjectOverview.tsx
git commit -m "feat(dashboard): add knowledge-specific stats to ProjectOverview"
```

---

## Tarefa 11: Criar Definições de Agentes

**Arquivos:**
- Criar: `understand-anything-plugin/agents/knowledge-scanner.md`
- Criar: `understand-anything-plugin/agents/format-detector.md`
- Criar: `understand-anything-plugin/agents/article-analyzer.md`
- Criar: `understand-anything-plugin/agents/relationship-builder.md`

- [ ] **Passo 1: Criar agente knowledge-scanner**

Crie `understand-anything-plugin/agents/knowledge-scanner.md`:

```markdown
---
name: knowledge-scanner
description: Scans a directory for markdown files and produces a file manifest for knowledge base analysis
model: inherit
---

# Knowledge Scanner Agent

You scan a target directory to discover all markdown files for knowledge base analysis.

## Input

You receive a JSON block with:
- `targetDir` — absolute path to the knowledge base directory

## Task

1. Use Glob/Bash to find all `.md` files in the target directory (recursive)
2. Exclude common non-content directories: `.obsidian/`, `logseq/`, `.foam/`, `_meta/`, `node_modules/`, `.git/`
3. For each file, capture:
   - `path` — relative path from targetDir
   - `sizeLines` — number of lines
   - `preview` — first 20 lines of content
4. Detect directory structure signatures:
   - Check for `.obsidian/` directory
   - Check for `logseq/` + `pages/` directories
   - Check for `.dendron.yml` or `*.schema.yml`
   - Check for `.foam/` or `.vscode/foam.json`
   - Check for `raw/` + `wiki/` + `index.md`
   - Scan a sample of files for `[[wikilinks]]` and unique ID prefixes
5. Write results to `$PROJECT_ROOT/.understand-anything/intermediate/knowledge-manifest.json`

## Output Schema

```json
{
  "targetDir": "/absolute/path",
  "totalFiles": 342,
  "directorySignatures": {
    "hasObsidianDir": true,
    "hasLogseqDir": false,
    "hasDendronConfig": false,
    "hasFoamConfig": false,
    "hasKarpathyStructure": false,
    "hasWikilinks": true,
    "hasUniqueIdPrefixes": false
  },
  "files": [
    {
      "path": "notes/topic.md",
      "sizeLines": 45,
      "preview": "---\ntags: [ai, ml]\n---\n# Topic Name\n..."
    }
  ]
}
```

## Rules

- Do NOT read file contents beyond the 20-line preview
- Sort files by path alphabetically
- Report total count prominently
- Write output to `.understand-anything/intermediate/knowledge-manifest.json`
```

- [ ] **Passo 2: Criar agente format-detector**

Crie `understand-anything-plugin/agents/format-detector.md`:

```markdown
---
name: format-detector
description: Detects the knowledge base format from directory signatures and file samples
model: inherit
---

# Format Detector Agent

You analyze the knowledge-manifest.json to determine which knowledge base format is being used.

## Input

Read `.understand-anything/intermediate/knowledge-manifest.json` produced by the knowledge-scanner.

## Detection Priority

Apply these rules in order (first match wins):

| Priority | Signal | Format |
|----------|--------|--------|
| 1 | `hasObsidianDir === true` | `obsidian` |
| 2 | `hasLogseqDir === true` | `logseq` |
| 3 | `hasDendronConfig === true` | `dendron` |
| 4 | `hasFoamConfig === true` | `foam` |
| 5 | `hasKarpathyStructure === true` | `karpathy` |
| 6 | `hasWikilinks === true` AND `hasUniqueIdPrefixes === true` | `zettelkasten` |
| 7 | fallback | `plain` |

## Output

Write to `.understand-anything/intermediate/format-detection.json`:

```json
{
  "format": "obsidian",
  "confidence": 0.95,
  "parsingHints": {
    "linkStyle": "wikilink",
    "metadataLocation": "yaml-frontmatter",
    "folderSemantics": "none",
    "specialFiles": [".obsidian/app.json"],
    "tagSyntax": "hashtag-inline"
  }
}
```

## Rules

- Always produce exactly one format
- Set confidence based on how many signals matched
- Include parsing hints that will help the article-analyzer
```

- [ ] **Passo 3: Criar agente article-analyzer**

Crie `understand-anything-plugin/agents/article-analyzer.md`:

```markdown
---
name: article-analyzer
description: Analyzes individual markdown files to extract knowledge nodes and explicit edges
model: inherit
---

# Article Analyzer Agent

You analyze batches of markdown files from a knowledge base to extract structured knowledge graph data.

## Input

You receive a JSON block with:
- `projectRoot` — absolute path to the knowledge base
- `batchFiles` — array of file objects from the manifest (path, sizeLines, preview)
- `format` — detected format from format-detection.json
- `parsingHints` — format-specific parsing guidance

You also receive a **format guide** (injected by the skill) that describes how to parse this specific format.

## Task

For each file in the batch:

### 1. Read the full file content

### 2. Extract the article node

- **id**: `article:<relative-path-without-extension>` (e.g., `article:notes/topic`)
- **type**: `article`
- **name**: First heading, or frontmatter title, or filename
- **filePath**: relative path
- **summary**: 2-3 sentence summary of the article content
- **tags**: from frontmatter tags, inline #tags, or inferred from content (3-5 tags)
- **complexity**: `simple` (<50 lines), `moderate` (50-200 lines), `complex` (>200 lines)
- **knowledgeMeta**: `{ format, wikilinks, frontmatter }`

### 3. Extract entity nodes

Identify named entities mentioned in the article:
- People, organizations, tools, papers, projects, datasets
- **id**: `entity:<normalized-lowercase-name>` (e.g., `entity:andrej-karpathy`)
- **type**: `entity`
- **summary**: one-sentence description based on context in the article
- **tags**: entity category tags like `person`, `tool`, `paper`, `organization`

### 4. Extract claim nodes (for articles with strong assertions)

- Only extract claims that are significant takeaways or insights
- **id**: `claim:<article-path>:<short-slug>` (e.g., `claim:notes/topic:rag-loses-context`)
- **type**: `claim`
- **summary**: the assertion itself

### 5. Extract source nodes (for cited references)

- External URLs, paper references, book citations
- **id**: `source:<normalized-url-or-title>`
- **type**: `source`
- **knowledgeMeta**: `{ sourceUrl }`

### 6. Extract explicit edges

- `[[wikilinks]]` → find target article, create `related` edge
- Frontmatter references → `categorized_under` or `related` edges
- Inline citations/URLs → `cites` edges to source nodes
- Author mentions → `authored_by` edges

## Node ID Conventions

```
article:<relative-path-without-extension>
entity:<normalized-lowercase-name>
topic:<normalized-lowercase-name>
claim:<article-path>:<short-slug>
source:<normalized-url-or-title>
```

Normalize: lowercase, replace spaces with hyphens, remove special characters.

**Deduplicate entities**: If the same entity appears across multiple files in the batch, emit it only once. Use the most informative summary.

## Edge Weight Conventions

```
contains: 1.0
authored_by: 0.9
cites: 0.8
categorized_under: 0.7
builds_on: 0.7
related: 0.5
exemplifies: 0.5
contradicts: 0.6
```

## Output

Write per-batch results to `.understand-anything/intermediate/article-batch-<N>.json`:

```json
{
  "nodes": [...],
  "edges": [...]
}
```

## Rules

- One article node per file (always)
- Entity nodes only for clearly named entities (not generic concepts)
- Claim nodes only for significant assertions (not every sentence)
- Source nodes only for explicit external references
- Deduplicate entities within the batch
- Respect the format guide for parsing links and metadata
```

- [ ] **Passo 4: Criar agente relationship-builder**

Crie `understand-anything-plugin/agents/relationship-builder.md`:

```markdown
---
name: relationship-builder
description: Discovers implicit cross-file relationships and builds topic clusters from analyzed knowledge nodes
model: inherit
---

# Relationship Builder Agent

You analyze all extracted nodes and edges to discover implicit relationships that explicit links missed.

## Input

Read all `article-batch-*.json` files from `.understand-anything/intermediate/`. Merge all nodes and edges.

## Task

### 1. Deduplicate entities globally

Multiple batches may have emitted the same entity. Merge them:
- Keep the most detailed summary
- Union all tags
- Collapse duplicate IDs

### 2. Discover implicit relationships

For each pair of articles/entities, determine if there's an implicit relationship:

- **builds_on**: Article A extends or deepens ideas from Article B (similar topics, references same entities, but goes further)
- **contradicts**: Article A makes claims that conflict with Article B
- **categorized_under**: Group articles into topic clusters
- **exemplifies**: An entity is a concrete example of a concept/topic
- **related**: Articles share significant thematic overlap but aren't explicitly linked

Set `confidence` in knowledgeMeta for LLM-inferred edges (0.0-1.0).

### 3. Build topic nodes

Identify thematic clusters across all articles:
- **id**: `topic:<normalized-name>`
- **type**: `topic`
- **summary**: description of what this topic covers
- Create `categorized_under` edges from articles/entities to their topics

### 4. Build layers

Group nodes into layers by topic:
- Each topic becomes a layer
- Articles, entities, claims, and sources are assigned to their primary topic's layer
- Nodes not clearly belonging to any topic go into an "Uncategorized" layer

### 5. Build tour

Create a guided tour through the knowledge base:
- Start with the broadest topic overview
- Walk through key articles in a logical learning order
- Each step covers 1-3 related nodes
- 5-10 tour steps total

## Output

Write to `.understand-anything/intermediate/relationships.json`:

```json
{
  "nodes": [...],
  "edges": [...],
  "layers": [...],
  "tour": [...]
}
```

## Rules

- Only add edges with confidence > 0.4
- Don't duplicate edges that already exist from article-analyzer
- Topics should be meaningful clusters (3+ articles), not one-off categories
- Tour should be navigable by someone new to the knowledge base
- Keep layers balanced — no layer with 50%+ of all nodes
```

- [ ] **Passo 5: Commit**

```bash
git add understand-anything-plugin/agents/knowledge-scanner.md understand-anything-plugin/agents/format-detector.md understand-anything-plugin/agents/article-analyzer.md understand-anything-plugin/agents/relationship-builder.md
git commit -m "feat(agents): add knowledge-scanner, format-detector, article-analyzer, and relationship-builder agents"
```

---

## Tarefa 12: Criar Guias de Formato

**Arquivos:**
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/obsidian.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/logseq.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/dendron.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/foam.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/karpathy.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/zettelkasten.md`
- Criar: `understand-anything-plugin/skills/understand-knowledge/formats/plain.md`

**IMPORTANTE**: Cada guia de formato deve ser **baseado em pesquisa**. O agente implementador DEVE:
1. Usar WebSearch e WebFetch para ler a **documentação oficial** de cada formato
2. Estudar as regras reais de parsing, não suposições
3. Incluir exemplos específicos de sintaxe da documentação real

- [ ] **Passo 1: Criar guia de formato obsidian.md**

Pesquise a documentação oficial do Obsidian (https://help.obsidian.md/) e crie `understand-anything-plugin/skills/understand-knowledge/formats/obsidian.md`:

O guia deve cobrir:
- Detecção: existe diretório `.obsidian/`
- Sintaxe de link: `[[wikilink]]`, `[[note|alias]]`, `[[note#heading]]`, `![[embed]]`
- Metadados: YAML frontmatter entre delimitadores `---`
- Tags: `#tag` inline, `tags:` no frontmatter (tanto array quanto separados por espaço)
- Properties: Obsidian Properties (campos de frontmatter renderizados na UI)
- Semântica de pastas: Obsidian não atribui significado a pastas por padrão
- Arquivos especiais: `.obsidian/app.json`, `.obsidian/workspace.json` (ignorar)
- Canvas: arquivos `.canvas` (formato JSON, descrevem layouts espaciais — extraia referências de cartões)
- Dataview: campos inline `key:: value`, `[key:: value]`

- [ ] **Passo 2: Criar guia de formato logseq.md**

Pesquise a documentação do Logseq (https://docs.logseq.com/) e crie `understand-anything-plugin/skills/understand-knowledge/formats/logseq.md`:

Cobrir:
- Detecção: diretórios `logseq/` + `pages/`
- Estrutura: `journals/YYYY_MM_DD.md` (notas diárias), `pages/*.md` (páginas nomeadas)
- Sintaxe de link: `[[wikilinks]]`, `((block-references))` por UUID
- Baseado em blocos: conteúdo organizado em outlines com bullet points
- Properties: sintaxe `key:: value` em blocos
- Tags: `#tag` inline, page tags via properties
- Especial: `logseq/config.edn` para configuração

- [ ] **Passo 3: Criar guia de formato dendron.md**

Pesquise a wiki do Dendron (https://wiki.dendron.so/) e crie `understand-anything-plugin/skills/understand-knowledge/formats/dendron.md`:

Cobrir:
- Detecção: arquivos `.dendron.yml` ou `*.schema.yml`
- Hierarquia: nomes de arquivo delimitados por ponto (`a.b.c.md`)
- Sintaxe de link: `[[wikilinks]]` com consciência de hierarquia
- Schemas: arquivos `.schema.yml` definem estrutura hierárquica esperada
- Frontmatter: YAML com campos `id` e `title` obrigatórios
- Stubs: arquivos de hierarquia intermediária criados automaticamente

- [ ] **Passo 4: Criar guia de formato foam.md**

Pesquise a documentação do Foam (https://foambubble.github.io/foam/) e crie `understand-anything-plugin/skills/understand-knowledge/formats/foam.md`:

Cobrir:
- Detecção: diretório `.foam/` ou `.vscode/foam.json`
- Sintaxe de link: `[[wikilinks]]` mais link reference definitions no rodapé do arquivo
- Placeholder links: links para arquivos inexistentes
- Frontmatter: YAML padrão
- Auto-linking: Foam atualiza automaticamente links em rename/move de arquivo

- [ ] **Passo 5: Criar guia de formato karpathy.md**

Pesquise o gist do Karpathy (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) e crie `understand-anything-plugin/skills/understand-knowledge/formats/karpathy.md`:

Cobrir:
- Detecção: diretórios `raw/` + `wiki/` + `index.md`
- Estrutura: `raw/` (fontes imutáveis), `wiki/` (artigos compilados), `_meta/` (estado)
- Arquivos especiais: `index.md` (lista mestre de páginas), `log.md` (log append-only de operações)
- Estilo de link: links markdown padrão (não wikilinks)
- Log parsing: entradas `## [YYYY-MM-DD] operation | Title`
- Artigos da wiki: compilados por LLM, podem ter cross-references e backlinks

- [ ] **Passo 6: Criar guia de formato zettelkasten.md**

Pesquise zettelkasten.de e crie `understand-anything-plugin/skills/understand-knowledge/formats/zettelkasten.md`:

Cobrir:
- Detecção: `[[wikilinks]]` + prefixos únicos de ID em nomes de arquivo (timestamps como `202604091234`)
- Notas atômicas: uma ideia por nota
- IDs únicos: timestamp ou prefixo alfanumérico no nome do arquivo
- Links: `[[wikilinks]]` com links tipados opcionais
- Frontmatter: YAML com tags, data de criação
- Sem hierarquia de pastas: estrutura plana, conexões apenas via links

- [ ] **Passo 7: Criar guia de formato plain.md**

Crie `understand-anything-plugin/skills/understand-knowledge/formats/plain.md`:

Cobrir:
- Detecção: fallback quando nenhum outro formato é detectado
- Links: links markdown padrão `[text](relative/path.md)`
- Estrutura: hierarquia de pastas fornece categorização
- Headings: hierarquia `#` fornece estrutura dentro de arquivos
- Sem expectativas de metadados especiais
- Tags: não esperadas (LLM infere tópicos)

- [ ] **Passo 8: Commit**

```bash
git add understand-anything-plugin/skills/understand-knowledge/formats/
git commit -m "feat(skill): add 7 research-backed format guides for knowledge base parsing"
```

---

## Tarefa 13: Criar SKILL.md

**Arquivos:**
- Criar: `understand-anything-plugin/skills/understand-knowledge/SKILL.md`

- [ ] **Passo 1: Criar a definição da skill**

Crie `understand-anything-plugin/skills/understand-knowledge/SKILL.md`:

```markdown
---
name: understand-knowledge
description: Analyze a markdown knowledge base (Obsidian, Logseq, Dendron, Foam, Karpathy-style, Zettelkasten, or plain) to produce an interactive knowledge graph with typed relationships
argument-hint: [path/to/notes] [--ingest <file-or-folder>]
---

# /understand-knowledge

Analyze a personal knowledge base of markdown files and produce an interactive knowledge graph.

## Arguments

- `path/to/notes` — (optional) directory containing markdown files. Defaults to current working directory.
- `--ingest <path>` — (optional) incrementally add new file(s) to an existing knowledge graph.

## Phase 0: Pre-flight

1. Determine the target directory:
   - If a path argument is provided, use it
   - Otherwise use the current working directory
2. Create `.understand-anything/` and `.understand-anything/intermediate/` directories if they don't exist
3. If `--ingest` flag is present:
   - Verify `.understand-anything/knowledge-graph.json` exists (error if not — must run full scan first)
   - Read the existing graph
   - Skip to Phase 2 with only the new/changed files
4. Get the current git commit hash (if in a git repo, otherwise use "no-git")

## Phase 1: SCAN

Dispatch the **knowledge-scanner** agent:

```json
{
  "targetDir": "<absolute-path-to-target>"
}
```

Wait for the agent to write `.understand-anything/intermediate/knowledge-manifest.json`.

Report: "Scanned {totalFiles} markdown files."

## Phase 2: FORMAT DETECTION

Dispatch the **format-detector** agent.

Wait for `.understand-anything/intermediate/format-detection.json`.

Report: "Detected format: {format} (confidence: {confidence})"

## Phase 3: ANALYZE

Read the format detection result. Load the corresponding format guide:

- `obsidian` → inject `skills/understand-knowledge/formats/obsidian.md`
- `logseq` → inject `skills/understand-knowledge/formats/logseq.md`
- `dendron` → inject `skills/understand-knowledge/formats/dendron.md`
- `foam` → inject `skills/understand-knowledge/formats/foam.md`
- `karpathy` → inject `skills/understand-knowledge/formats/karpathy.md`
- `zettelkasten` → inject `skills/understand-knowledge/formats/zettelkasten.md`
- `plain` → inject `skills/understand-knowledge/formats/plain.md`

Batch the files from the manifest into groups of 15-25 files each.

For each batch, dispatch an **article-analyzer** agent with:

```json
{
  "projectRoot": "<absolute-path>",
  "batchFiles": [...],
  "format": "<detected-format>",
  "parsingHints": {...}
}
```

Inject the format guide content into each agent's context.

Run up to 5 batches concurrently.

Wait for all `article-batch-*.json` files.

Report: "Analyzed {totalFiles} files across {batchCount} batches."

## Phase 4: RELATIONSHIPS

Dispatch the **relationship-builder** agent.

Wait for `.understand-anything/intermediate/relationships.json`.

Report: "Discovered {topicCount} topics, {implicitEdgeCount} implicit relationships."

## Phase 5: ASSEMBLE

Merge all intermediate results into a single knowledge graph:

1. Read all `article-batch-*.json` files — collect all nodes and edges
2. Read `relationships.json` — merge in topic nodes, implicit edges, layers, and tour
3. Deduplicate nodes by ID (keep the most complete version)
4. Deduplicate edges by source+target+type
5. Assemble into `KnowledgeGraph` format:

```json
{
  "version": "1.0",
  "kind": "knowledge",
  "project": {
    "name": "<directory-name>",
    "languages": [],
    "frameworks": [],
    "description": "Knowledge base analyzed from <format> format",
    "analyzedAt": "<ISO-timestamp>",
    "gitCommitHash": "<hash>"
  },
  "nodes": [...],
  "edges": [...],
  "layers": [...],
  "tour": [...]
}
```

## Phase 6: REVIEW

Dispatch the existing **graph-reviewer** agent to validate:
- All edge source/target IDs reference existing nodes
- No orphan nodes (nodes with zero edges)
- No duplicate node IDs
- All layers reference existing nodes
- Tour steps reference existing nodes

Apply fixes from the reviewer.

## Phase 7: SAVE

1. Write `.understand-anything/knowledge-graph.json`
2. Write `.understand-anything/meta.json`:
   ```json
   {
     "lastAnalyzedAt": "<ISO-timestamp>",
     "gitCommitHash": "<hash>",
     "version": "1.0",
     "analyzedFiles": <count>,
     "knowledgeFormat": "<detected-format>"
   }
   ```
3. Clean up `.understand-anything/intermediate/` directory
4. Report: "Knowledge graph saved with {nodeCount} nodes and {edgeCount} edges."

## Phase 8: DASHBOARD

Auto-trigger `/understand-dashboard` to launch the visualization.

## Incremental Mode (--ingest)

When `--ingest <path>` is specified:

1. Read the existing `knowledge-graph.json`
2. Scan only the specified file(s) or folder
3. Skip format detection (reuse format from existing graph's metadata)
4. Run article-analyzer on only the new/changed files
5. Run relationship-builder on new nodes against the full existing graph
6. Merge new nodes/edges into the existing graph
7. Re-run graph-reviewer
8. Save updated graph
```

- [ ] **Passo 2: Commit**

```bash
git add understand-anything-plugin/skills/understand-knowledge/SKILL.md
git commit -m "feat(skill): add /understand-knowledge skill definition with 8-phase pipeline"
```

---

## Tarefa 14: Build, Testar & Verificar de Ponta a Ponta

**Arquivos:**
- Todos os arquivos modificados

- [ ] **Passo 1: Build do pacote core**

Execute: `pnpm --filter @understand-anything/core build`
Esperado: Build limpo, sem erros

- [ ] **Passo 2: Executar testes do core**

Execute: `pnpm --filter @understand-anything/core test -- --run`
Esperado: Todos os testes passam, incluindo os novos testes de knowledge-schema

- [ ] **Passo 3: Build do dashboard**

Execute: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build limpo, sem erros

- [ ] **Passo 4: Executar lint**

Execute: `pnpm lint`
Esperado: Sem erros de lint

- [ ] **Passo 5: Verificar que a skill é descoberta**

Verifique que o arquivo da skill existe e tem frontmatter válido:

Execute: `head -5 understand-anything-plugin/skills/understand-knowledge/SKILL.md`
Esperado: YAML delimitado por `---` válido com name, description, argument-hint

- [ ] **Passo 6: Verificar que todos os agentes estão presentes**

Execute: `ls understand-anything-plugin/agents/ | grep knowledge\|format\|article\|relationship`
Esperado: `knowledge-scanner.md`, `format-detector.md`, `article-analyzer.md`, `relationship-builder.md`

- [ ] **Passo 7: Verificar que todos os guias de formato estão presentes**

Execute: `ls understand-anything-plugin/skills/understand-knowledge/formats/`
Esperado: `obsidian.md`, `logseq.md`, `dendron.md`, `foam.md`, `karpathy.md`, `zettelkasten.md`, `plain.md`

- [ ] **Passo 8: Commit final**

```bash
git add -A
git commit -m "feat: complete /understand-knowledge implementation — knowledge base analysis skill"
```
