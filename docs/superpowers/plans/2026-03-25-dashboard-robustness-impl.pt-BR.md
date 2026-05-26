# Plano de Implementação da Robustez do Dashboard

> 🇬🇧 Versão original em inglês: [2026-03-25-dashboard-robustness-impl.md](./2026-03-25-dashboard-robustness-impl.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa a tarefa.

**Objetivo:** Tornar o dashboard resiliente a grafos de conhecimento imperfeitos gerados por LLM, auto-corrigindo issues recuperáveis, descartando itens quebrados e mostrando warnings amber amigáveis com relatórios de erro fáceis de copiar e colar.

**Arquitetura:** Pipeline de três camadas em `schema.ts`: sanitize (Tier 1 silencioso) → auto-fix (Tier 2 rastreado) → validate por-item (Tier 3 drop) → fatal gate (Tier 4). Novo componente `WarningBanner` no dashboard exibe issues categorizadas com botão de copy.

**Stack Técnico:** Zod (validation), React + TailwindCSS (UI do dashboard), Vitest (testing)

---

### Tarefa 1: Adicionar tipo GraphIssue e sanitizeGraph (Tier 1)

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/schema.ts:95-99`
- Teste: `understand-anything-plugin/packages/core/src/__tests__/schema.test.ts`

**Passo 1: Escrever os testes que falham para sanitizeGraph**

Adicionar ao final de `schema.test.ts`, antes do `});` de fechamento:

```typescript
describe("sanitizeGraph", () => {
  it("converts null optional node fields to undefined", () => {
    const graph = structuredClone(validGraph);
    (graph.nodes[0] as any).filePath = null;
    (graph.nodes[0] as any).lineRange = null;
    (graph.nodes[0] as any).languageNotes = null;

    const result = sanitizeGraph(graph as any);
    const node = (result as any).nodes[0];
    expect(node.filePath).toBeUndefined();
    expect(node.lineRange).toBeUndefined();
    expect(node.languageNotes).toBeUndefined();
  });

  it("converts null optional edge fields to undefined", () => {
    const graph = structuredClone(validGraph);
    (graph.edges[0] as any).description = null;

    const result = sanitizeGraph(graph as any);
    const edge = (result as any).edges[0];
    expect(edge.description).toBeUndefined();
  });

  it("lowercases enum-like strings on nodes", () => {
    const graph = structuredClone(validGraph);
    (graph.nodes[0] as any).type = "FILE";
    (graph.nodes[0] as any).complexity = "Simple";

    const result = sanitizeGraph(graph as any);
    const node = (result as any).nodes[0];
    expect(node.type).toBe("file");
    expect(node.complexity).toBe("simple");
  });

  it("lowercases enum-like strings on edges", () => {
    const graph = structuredClone(validGraph);
    (graph.edges[0] as any).type = "IMPORTS";
    (graph.edges[0] as any).direction = "Forward";

    const result = sanitizeGraph(graph as any);
    const edge = (result as any).edges[0];
    expect(edge.type).toBe("imports");
    expect(edge.direction).toBe("forward");
  });

  it("converts null tour/layers to empty arrays", () => {
    const graph = structuredClone(validGraph);
    (graph as any).tour = null;
    (graph as any).layers = null;

    const result = sanitizeGraph(graph as any);
    expect((result as any).tour).toEqual([]);
    expect((result as any).layers).toEqual([]);
  });

  it("converts null optional tour step fields to undefined", () => {
    const graph = structuredClone(validGraph);
    (graph.tour[0] as any).languageLesson = null;

    const result = sanitizeGraph(graph as any);
    expect((result as any).tour[0].languageLesson).toBeUndefined();
  });
});
```

**Passo 2: Rodar os testes para confirmar que falham**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: FAIL — `sanitizeGraph` não está exportado

**Passo 3: Adicionar tipo GraphIssue e atualizar ValidationResult**

Em `schema.ts`, substituir a interface `ValidationResult` (linhas 95-99) por:

```typescript
export interface GraphIssue {
  level: "auto-corrected" | "dropped" | "fatal";
  category: string;
  message: string;
  path?: string;
}

export interface ValidationResult {
  success: boolean;
  data?: z.infer<typeof KnowledgeGraphSchema>;
  issues: GraphIssue[];
  fatal?: string;
  /** @deprecated Use issues/fatal instead */
  errors?: string[];
}
```

**Passo 4: Implementar sanitizeGraph**

Adicionar após os mapas de alias (após linha 39), antes de `GraphNodeSchema`:

```typescript
export function sanitizeGraph(data: Record<string, unknown>): Record<string, unknown> {
  const result = { ...data };

  // Null → empty array for top-level collections
  if (data.tour === null || data.tour === undefined) result.tour = [];
  if (data.layers === null || data.layers === undefined) result.layers = [];

  // Sanitize nodes
  if (Array.isArray(data.nodes)) {
    result.nodes = (data.nodes as Record<string, unknown>[]).map((node) => {
      if (typeof node !== "object" || node === null) return node;
      const n = { ...node };
      // Null → undefined for optional fields
      if (n.filePath === null) delete n.filePath;
      if (n.lineRange === null) delete n.lineRange;
      if (n.languageNotes === null) delete n.languageNotes;
      // Lowercase enum-like strings
      if (typeof n.type === "string") n.type = n.type.toLowerCase();
      if (typeof n.complexity === "string") n.complexity = n.complexity.toLowerCase();
      return n;
    });
  }

  // Sanitize edges
  if (Array.isArray(data.edges)) {
    result.edges = (data.edges as Record<string, unknown>[]).map((edge) => {
      if (typeof edge !== "object" || edge === null) return edge;
      const e = { ...edge };
      if (e.description === null) delete e.description;
      if (typeof e.type === "string") e.type = e.type.toLowerCase();
      if (typeof e.direction === "string") e.direction = e.direction.toLowerCase();
      return e;
    });
  }

  // Sanitize tour steps
  if (Array.isArray(result.tour)) {
    result.tour = (result.tour as Record<string, unknown>[]).map((step) => {
      if (typeof step !== "object" || step === null) return step;
      const s = { ...step };
      if (s.languageLesson === null) delete s.languageLesson;
      return s;
    });
  }

  return result;
}
```

**Passo 5: Atualizar imports no arquivo de teste**

Atualizar a linha de import em `schema.test.ts`:

```typescript
import {
  validateGraph,
  normalizeGraph,
  sanitizeGraph,
  NODE_TYPE_ALIASES,
  EDGE_TYPE_ALIASES,
} from "../schema.js";
```

**Passo 6: Rodar os testes para confirmar que passam**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: Todos os testes de sanitizeGraph PASSAM. Testes existentes continuam PASSANDO.

**Passo 7: Commit**

```bash
git add understand-anything-plugin/packages/core/src/schema.ts understand-anything-plugin/packages/core/src/__tests__/schema.test.ts
git commit -m "feat(core): add GraphIssue type and sanitizeGraph (Tier 1 silent fixes)"
```

---

### Tarefa 2: Adicionar mapas de auto-fix e autoFixGraph (Tier 2)

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/schema.ts`
- Teste: `understand-anything-plugin/packages/core/src/__tests__/schema.test.ts`

**Passo 1: Escrever os testes que falham**

Adicionar a `schema.test.ts`, antes do `});` de fechamento:

```typescript
describe("autoFixGraph", () => {
  it("defaults missing complexity to moderate with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.nodes[0] as any).complexity;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).nodes[0].complexity).toBe("moderate");
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "nodes[0].complexity" })
    );
  });

  it("maps complexity aliases with issue", () => {
    const graph = structuredClone(validGraph);
    (graph.nodes[0] as any).complexity = "low";

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).nodes[0].complexity).toBe("simple");
    expect(issues.length).toBe(1);
    expect(issues[0].level).toBe("auto-corrected");
  });

  it("maps all complexity aliases correctly", () => {
    const mapping: Record<string, string> = {
      low: "simple", easy: "simple",
      medium: "moderate", intermediate: "moderate",
      high: "complex", hard: "complex", difficult: "complex",
    };
    for (const [alias, expected] of Object.entries(mapping)) {
      const graph = structuredClone(validGraph);
      (graph.nodes[0] as any).complexity = alias;
      const { data } = autoFixGraph(graph as any);
      expect((data as any).nodes[0].complexity).toBe(expected);
    }
  });

  it("defaults missing tags to empty array with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.nodes[0] as any).tags;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).nodes[0].tags).toEqual([]);
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "nodes[0].tags" })
    );
  });

  it("defaults missing summary to node name with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.nodes[0] as any).summary;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).nodes[0].summary).toBe("index.ts");
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "nodes[0].summary" })
    );
  });

  it("defaults missing node type to file with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.nodes[0] as any).type;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).nodes[0].type).toBe("file");
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "nodes[0].type" })
    );
  });

  it("defaults missing direction to forward with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.edges[0] as any).direction;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).edges[0].direction).toBe("forward");
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "edges[0].direction" })
    );
  });

  it("maps direction aliases with issue", () => {
    const mapping: Record<string, string> = {
      to: "forward", outbound: "forward",
      from: "backward", inbound: "backward",
      both: "bidirectional", mutual: "bidirectional",
    };
    for (const [alias, expected] of Object.entries(mapping)) {
      const graph = structuredClone(validGraph);
      (graph.edges[0] as any).direction = alias;
      const { data } = autoFixGraph(graph as any);
      expect((data as any).edges[0].direction).toBe(expected);
    }
  });

  it("defaults missing weight to 0.5 with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.edges[0] as any).weight;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).edges[0].weight).toBe(0.5);
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "edges[0].weight" })
    );
  });

  it("coerces string weight to number with issue", () => {
    const graph = structuredClone(validGraph);
    (graph.edges[0] as any).weight = "0.8";

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).edges[0].weight).toBe(0.8);
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "type-coercion", path: "edges[0].weight" })
    );
  });

  it("clamps out-of-range weight with issue", () => {
    const graph = structuredClone(validGraph);
    (graph.edges[0] as any).weight = 1.5;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).edges[0].weight).toBe(1);
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "out-of-range", path: "edges[0].weight" })
    );
  });

  it("defaults missing edge type to depends_on with issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.edges[0] as any).type;

    const { data, issues } = autoFixGraph(graph as any);
    expect((data as any).edges[0].type).toBe("depends_on");
    expect(issues).toContainEqual(
      expect.objectContaining({ level: "auto-corrected", category: "missing-field", path: "edges[0].type" })
    );
  });

  it("returns no issues for a valid graph", () => {
    const { issues } = autoFixGraph(validGraph as any);
    expect(issues).toEqual([]);
  });
});
```

**Passo 2: Rodar os testes para confirmar que falham**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: FAIL — `autoFixGraph` não está exportado

**Passo 3: Implementar mapas de alias e autoFixGraph**

Adicionar a `schema.ts` após o mapa `EDGE_TYPE_ALIASES` existente (após linha 39):

```typescript
export const COMPLEXITY_ALIASES: Record<string, string> = {
  low: "simple",
  easy: "simple",
  medium: "moderate",
  intermediate: "moderate",
  high: "complex",
  hard: "complex",
  difficult: "complex",
};

export const DIRECTION_ALIASES: Record<string, string> = {
  to: "forward",
  outbound: "forward",
  from: "backward",
  inbound: "backward",
  both: "bidirectional",
  mutual: "bidirectional",
};
```

Adicionar a função `autoFixGraph` após `sanitizeGraph`:

```typescript
export function autoFixGraph(data: Record<string, unknown>): {
  data: Record<string, unknown>;
  issues: GraphIssue[];
} {
  const issues: GraphIssue[] = [];
  const result = { ...data };

  if (Array.isArray(data.nodes)) {
    result.nodes = (data.nodes as Record<string, unknown>[]).map((node, i) => {
      if (typeof node !== "object" || node === null) return node;
      const n = { ...node };
      const name = (n.name as string) || (n.id as string) || `index ${i}`;

      // Missing or empty type
      if (!n.type || typeof n.type !== "string") {
        n.type = "file";
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `nodes[${i}] ("${name}"): missing "type" — defaulted to "file"`,
          path: `nodes[${i}].type`,
        });
      }

      // Missing or empty complexity
      if (!n.complexity || n.complexity === "") {
        n.complexity = "moderate";
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `nodes[${i}] ("${name}"): missing "complexity" — defaulted to "moderate"`,
          path: `nodes[${i}].complexity`,
        });
      } else if (typeof n.complexity === "string" && n.complexity in COMPLEXITY_ALIASES) {
        const original = n.complexity;
        n.complexity = COMPLEXITY_ALIASES[n.complexity];
        issues.push({
          level: "auto-corrected",
          category: "alias",
          message: `nodes[${i}] ("${name}"): complexity "${original}" — mapped to "${n.complexity}"`,
          path: `nodes[${i}].complexity`,
        });
      }

      // Missing tags
      if (!Array.isArray(n.tags)) {
        n.tags = [];
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `nodes[${i}] ("${name}"): missing "tags" — defaulted to []`,
          path: `nodes[${i}].tags`,
        });
      }

      // Missing summary
      if (!n.summary || typeof n.summary !== "string") {
        n.summary = (n.name as string) || "No summary";
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `nodes[${i}] ("${name}"): missing "summary" — defaulted to name`,
          path: `nodes[${i}].summary`,
        });
      }

      return n;
    });
  }

  if (Array.isArray(data.edges)) {
    result.edges = (data.edges as Record<string, unknown>[]).map((edge, i) => {
      if (typeof edge !== "object" || edge === null) return edge;
      const e = { ...edge };

      // Missing type
      if (!e.type || typeof e.type !== "string") {
        e.type = "depends_on";
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `edges[${i}]: missing "type" — defaulted to "depends_on"`,
          path: `edges[${i}].type`,
        });
      }

      // Missing direction
      if (!e.direction || typeof e.direction !== "string") {
        e.direction = "forward";
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `edges[${i}]: missing "direction" — defaulted to "forward"`,
          path: `edges[${i}].direction`,
        });
      } else if (e.direction in DIRECTION_ALIASES) {
        const original = e.direction;
        e.direction = DIRECTION_ALIASES[e.direction as string];
        issues.push({
          level: "auto-corrected",
          category: "alias",
          message: `edges[${i}]: direction "${original}" — mapped to "${e.direction}"`,
          path: `edges[${i}].direction`,
        });
      }

      // Missing weight
      if (e.weight === undefined || e.weight === null) {
        e.weight = 0.5;
        issues.push({
          level: "auto-corrected",
          category: "missing-field",
          message: `edges[${i}]: missing "weight" — defaulted to 0.5`,
          path: `edges[${i}].weight`,
        });
      } else if (typeof e.weight === "string") {
        const parsed = parseFloat(e.weight as string);
        if (!isNaN(parsed)) {
          const original = e.weight;
          e.weight = parsed;
          issues.push({
            level: "auto-corrected",
            category: "type-coercion",
            message: `edges[${i}]: weight was string "${original}" — coerced to number`,
            path: `edges[${i}].weight`,
          });
        }
      }

      // Clamp weight to [0, 1]
      if (typeof e.weight === "number" && (e.weight < 0 || e.weight > 1)) {
        const original = e.weight;
        e.weight = Math.max(0, Math.min(1, e.weight));
        issues.push({
          level: "auto-corrected",
          category: "out-of-range",
          message: `edges[${i}]: weight ${original} clamped to ${e.weight}`,
          path: `edges[${i}].weight`,
        });
      }

      return e;
    });
  }

  return { data: result, issues };
}
```

**Passo 4: Atualizar imports no arquivo de teste**

```typescript
import {
  validateGraph,
  normalizeGraph,
  sanitizeGraph,
  autoFixGraph,
  NODE_TYPE_ALIASES,
  EDGE_TYPE_ALIASES,
} from "../schema.js";
```

**Passo 5: Rodar os testes para confirmar que passam**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: Todos os novos testes de autoFixGraph PASSAM. Testes existentes continuam PASSANDO.

**Passo 6: Commit**

```bash
git add understand-anything-plugin/packages/core/src/schema.ts understand-anything-plugin/packages/core/src/__tests__/schema.test.ts
git commit -m "feat(core): add autoFixGraph with complexity/direction aliases and default values (Tier 2)"
```

---

### Tarefa 3: Reescrever validateGraph para ser permissivo (Tier 3 + 4)

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/schema.ts:138-151`
- Teste: `understand-anything-plugin/packages/core/src/__tests__/schema.test.ts`

**Passo 1: Escrever os testes que falham para validação permissiva**

Adicionar a `schema.test.ts`:

```typescript
describe("permissive validation", () => {
  it("drops nodes missing id with dropped issue", () => {
    const graph = structuredClone(validGraph);
    delete (graph.nodes[0] as any).id;
    // Add a second valid node so graph isn't fatal
    graph.nodes.push({
      id: "node-2", type: "file", name: "other.ts",
      summary: "Other file", tags: ["util"], complexity: "simple",
    });

    const result = validateGraph(graph);
    expect(result.success).toBe(true);
    expect(result.data!.nodes.length).toBe(1);
    expect(result.data!.nodes[0].id).toBe("node-2");
    expect(result.issues).toContainEqual(
      expect.objectContaining({ level: "dropped", category: "invalid-node" })
    );
  });

  it("drops edges referencing non-existent nodes with dropped issue", () => {
    const graph = structuredClone(validGraph);
    graph.edges[0].target = "non-existent-node";

    const result = validateGraph(graph);
    expect(result.success).toBe(true);
    expect(result.data!.edges.length).toBe(0);
    expect(result.issues).toContainEqual(
      expect.objectContaining({ level: "dropped", category: "invalid-reference" })
    );
  });

  it("returns fatal when 0 valid nodes remain", () => {
    const graph = structuredClone(validGraph);
    delete (graph.nodes[0] as any).id;

    const result = validateGraph(graph);
    expect(result.success).toBe(false);
    expect(result.fatal).toContain("No valid nodes");
  });

  it("returns fatal when project metadata is missing", () => {
    const graph = structuredClone(validGraph);
    delete (graph as any).project;

    const result = validateGraph(graph);
    expect(result.success).toBe(false);
    expect(result.fatal).toContain("project metadata");
  });

  it("returns fatal when input is not an object", () => {
    const result = validateGraph("not an object");
    expect(result.success).toBe(false);
    expect(result.fatal).toContain("Invalid input");
  });

  it("loads graph with mixed good and bad nodes", () => {
    const graph = structuredClone(validGraph);
    // Add a good node
    graph.nodes.push({
      id: "node-2", type: "function", name: "doThing",
      summary: "Does a thing", tags: ["util"], complexity: "moderate",
    });
    // Add a bad node (missing id AND name — unrecoverable)
    (graph.nodes as any[]).push({ type: "file", summary: "broken" });

    const result = validateGraph(graph);
    expect(result.success).toBe(true);
    expect(result.data!.nodes.length).toBe(2);
    expect(result.issues.some((i) => i.level === "dropped")).toBe(true);
  });

  it("filters dangling nodeIds from layers", () => {
    const graph = structuredClone(validGraph);
    graph.layers[0].nodeIds.push("non-existent-node");

    const result = validateGraph(graph);
    expect(result.success).toBe(true);
    expect(result.data!.layers[0].nodeIds).toEqual(["node-1"]);
  });

  it("filters dangling nodeIds from tour steps", () => {
    const graph = structuredClone(validGraph);
    graph.tour[0].nodeIds.push("non-existent-node");

    const result = validateGraph(graph);
    expect(result.success).toBe(true);
    expect(result.data!.tour[0].nodeIds).toEqual(["node-1"]);
  });

  it("returns empty issues array for a perfect graph", () => {
    const result = validateGraph(validGraph);
    expect(result.success).toBe(true);
    expect(result.issues).toEqual([]);
  });

  it("auto-corrects and loads graph that would have failed strict validation", () => {
    // Graph with many Tier 2 issues: missing complexity, weight as string, null filePath
    const messy = {
      version: "1.0.0",
      project: validGraph.project,
      nodes: [{
        id: "n1", type: "FILE", name: "app.ts",
        filePath: null, summary: "App entry",
        tags: null, complexity: "HIGH",
      }],
      edges: [{
        source: "n1", target: "n1", type: "CALLS",
        direction: "TO", weight: "0.9",
      }],
      layers: [{ id: "l1", name: "Core", description: "Core", nodeIds: ["n1"] }],
      tour: [],
    };

    const result = validateGraph(messy);
    expect(result.success).toBe(true);
    expect(result.data!.nodes[0].complexity).toBe("complex");
    expect(result.data!.nodes[0].tags).toEqual([]);
    expect(result.data!.edges[0].weight).toBe(0.9);
    expect(result.data!.edges[0].direction).toBe("forward");
    expect(result.issues.length).toBeGreaterThan(0);
    expect(result.issues.every((i) => i.level === "auto-corrected")).toBe(true);
  });
});
```

**Passo 2: Rodar os testes para confirmar que falham**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: FAIL — `validateGraph` não retorna `issues` nem `fatal`

**Passo 3: Reescrever validateGraph**

Substituir a função `validateGraph` existente em `schema.ts` (linhas 138-151) por:

```typescript
export function validateGraph(data: unknown): ValidationResult {
  // Tier 4: Fatal — not even an object
  if (typeof data !== "object" || data === null) {
    return { success: false, issues: [], fatal: "Invalid input: not an object" };
  }

  const raw = data as Record<string, unknown>;

  // Tier 1: Sanitize
  const sanitized = sanitizeGraph(raw);

  // Existing: Normalize type aliases
  const normalized = normalizeGraph(sanitized) as Record<string, unknown>;

  // Tier 2: Auto-fix defaults and coercion
  const { data: fixed, issues } = autoFixGraph(
    normalized as Record<string, unknown>,
  );

  // Tier 4: Fatal — missing project metadata
  const projectResult = ProjectMetaSchema.safeParse(fixed.project);
  if (!projectResult.success) {
    return {
      success: false,
      issues,
      fatal: "Missing or invalid project metadata",
    };
  }

  // Tier 3: Validate nodes individually, drop broken
  const validNodes: z.infer<typeof GraphNodeSchema>[] = [];
  if (Array.isArray(fixed.nodes)) {
    for (let i = 0; i < fixed.nodes.length; i++) {
      const node = fixed.nodes[i] as Record<string, unknown>;
      const result = GraphNodeSchema.safeParse(node);
      if (result.success) {
        validNodes.push(result.data);
      } else {
        const name = node?.name || node?.id || `index ${i}`;
        issues.push({
          level: "dropped",
          category: "invalid-node",
          message: `nodes[${i}] ("${name}"): ${result.error.issues[0]?.message ?? "validation failed"} — removed`,
          path: `nodes[${i}]`,
        });
      }
    }
  }

  // Tier 4: Fatal — no valid nodes
  if (validNodes.length === 0) {
    return {
      success: false,
      issues,
      fatal: "No valid nodes found in knowledge graph",
    };
  }

  // Tier 3: Validate edges + referential integrity
  const nodeIds = new Set(validNodes.map((n) => n.id));
  const validEdges: z.infer<typeof GraphEdgeSchema>[] = [];
  if (Array.isArray(fixed.edges)) {
    for (let i = 0; i < fixed.edges.length; i++) {
      const edge = fixed.edges[i] as Record<string, unknown>;
      const result = GraphEdgeSchema.safeParse(edge);
      if (!result.success) {
        issues.push({
          level: "dropped",
          category: "invalid-edge",
          message: `edges[${i}]: ${result.error.issues[0]?.message ?? "validation failed"} — removed`,
          path: `edges[${i}]`,
        });
        continue;
      }
      if (!nodeIds.has(result.data.source)) {
        issues.push({
          level: "dropped",
          category: "invalid-reference",
          message: `edges[${i}]: source "${result.data.source}" does not exist in nodes — removed`,
          path: `edges[${i}].source`,
        });
        continue;
      }
      if (!nodeIds.has(result.data.target)) {
        issues.push({
          level: "dropped",
          category: "invalid-reference",
          message: `edges[${i}]: target "${result.data.target}" does not exist in nodes — removed`,
          path: `edges[${i}].target`,
        });
        continue;
      }
      validEdges.push(result.data);
    }
  }

  // Validate layers (drop broken, filter dangling nodeIds)
  const validLayers: z.infer<typeof LayerSchema>[] = [];
  if (Array.isArray(fixed.layers)) {
    for (let i = 0; i < (fixed.layers as unknown[]).length; i++) {
      const result = LayerSchema.safeParse((fixed.layers as unknown[])[i]);
      if (result.success) {
        validLayers.push({
          ...result.data,
          nodeIds: result.data.nodeIds.filter((id) => nodeIds.has(id)),
        });
      } else {
        issues.push({
          level: "dropped",
          category: "invalid-layer",
          message: `layers[${i}]: ${result.error.issues[0]?.message ?? "validation failed"} — removed`,
          path: `layers[${i}]`,
        });
      }
    }
  }

  // Validate tour steps (drop broken, filter dangling nodeIds)
  const validTour: z.infer<typeof TourStepSchema>[] = [];
  if (Array.isArray(fixed.tour)) {
    for (let i = 0; i < (fixed.tour as unknown[]).length; i++) {
      const result = TourStepSchema.safeParse((fixed.tour as unknown[])[i]);
      if (result.success) {
        validTour.push({
          ...result.data,
          nodeIds: result.data.nodeIds.filter((id) => nodeIds.has(id)),
        });
      } else {
        issues.push({
          level: "dropped",
          category: "invalid-tour-step",
          message: `tour[${i}]: ${result.error.issues[0]?.message ?? "validation failed"} — removed`,
          path: `tour[${i}]`,
        });
      }
    }
  }

  const graph = {
    version: typeof fixed.version === "string" ? fixed.version : "1.0.0",
    project: projectResult.data,
    nodes: validNodes,
    edges: validEdges,
    layers: validLayers,
    tour: validTour,
  };

  return { success: true, data: graph, issues };
}
```

**Passo 4: Rodar os testes para confirmar que os novos testes passam**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: Novos testes permissivos PASSAM. Alguns testes antigos podem agora falhar (esperado — tratado na Tarefa 4).

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/schema.ts understand-anything-plugin/packages/core/src/__tests__/schema.test.ts
git commit -m "feat(core): rewrite validateGraph for permissive per-item validation (Tier 3+4)"
```

---

### Tarefa 4: Atualizar testes existentes para o novo comportamento permissivo

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/__tests__/schema.test.ts`

A nova validação permissiva muda o comportamento de vários testes existentes. Veja o que muda:

| Teste | Comportamento antigo | Novo comportamento |
|------|-------------|-------------|
| "validates a correct graph" | `success: true, errors: undefined` | `success: true, issues: []` |
| "rejects missing required fields" | `success: false, errors` | `success: false, fatal` (project ausente) |
| "rejects node with invalid type" | `success: false, errors` | `success: false, fatal` (0 nodes válidos após drop) |
| "rejects edge with invalid EdgeType" | `success: false, errors` | `success: true` (edge dropado, node válido) |
| "rejects weight >1" | `success: false, errors` | `success: true` (weight clampado) |
| "rejects weight <0" | `success: false, errors` | `success: true` (weight clampado) |
| "rejects 'tests' edge type" | `success: false` | `success: true` (edge dropado) |
| "rejects truly invalid edge types" | `success: false` | `success: true` (edge dropado) |

**Passo 1: Atualizar os testes afetados**

Substituir os testes a seguir no bloco describe `"schema validation"`:

```typescript
it("validates a correct knowledge graph", () => {
  const result = validateGraph(validGraph);
  expect(result.success).toBe(true);
  expect(result.data).toBeDefined();
  expect(result.data!.version).toBe("1.0.0");
  expect(result.issues).toEqual([]);
});

it("rejects graph with missing required fields", () => {
  const incomplete = { version: "1.0.0" };
  const result = validateGraph(incomplete);
  expect(result.success).toBe(false);
  expect(result.fatal).toBeDefined();
});

it("rejects node with invalid type — drops node, fatal if none remain", () => {
  const graph = structuredClone(validGraph);
  (graph.nodes[0] as any).type = "invalid_type";

  const result = validateGraph(graph);
  expect(result.success).toBe(false);
  expect(result.fatal).toContain("No valid nodes");
  expect(result.issues).toContainEqual(
    expect.objectContaining({ level: "dropped", category: "invalid-node" })
  );
});

it("drops edge with invalid EdgeType but loads graph", () => {
  const graph = structuredClone(validGraph);
  (graph.edges[0] as any).type = "not_a_real_edge_type";

  const result = validateGraph(graph);
  expect(result.success).toBe(true);
  expect(result.data!.edges.length).toBe(0);
  expect(result.issues).toContainEqual(
    expect.objectContaining({ level: "dropped", category: "invalid-edge" })
  );
});

it("auto-corrects weight >1 by clamping", () => {
  const graph = structuredClone(validGraph);
  graph.edges[0].weight = 1.5;

  const result = validateGraph(graph);
  expect(result.success).toBe(true);
  expect(result.issues).toContainEqual(
    expect.objectContaining({ level: "auto-corrected", category: "out-of-range" })
  );
});

it("auto-corrects weight <0 by clamping", () => {
  const graph = structuredClone(validGraph);
  graph.edges[0].weight = -0.1;

  const result = validateGraph(graph);
  expect(result.success).toBe(true);
  expect(result.issues).toContainEqual(
    expect.objectContaining({ level: "auto-corrected", category: "out-of-range" })
  );
});
```

Também atualizar o teste de tipo de edge "tests" e o teste de "truly invalid edge types":

```typescript
it('drops "tests" edge type — direction-inverting alias is unsafe', () => {
  const graph = structuredClone(validGraph);
  (graph.edges[0] as any).type = "tests";

  const result = validateGraph(graph);
  expect(result.success).toBe(true);
  expect(result.data!.edges.length).toBe(0);
  expect(result.issues).toContainEqual(
    expect.objectContaining({ level: "dropped" })
  );
});

it("drops truly invalid edge types after normalization", () => {
  const graph = structuredClone(validGraph);
  (graph.edges[0] as any).type = "totally_bogus";

  const result = validateGraph(graph);
  expect(result.success).toBe(true);
  expect(result.data!.edges.length).toBe(0);
  expect(result.issues).toContainEqual(
    expect.objectContaining({ level: "dropped" })
  );
});
```

**Passo 2: Rodar todos os testes**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: TODOS os testes PASSAM

**Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/__tests__/schema.test.ts
git commit -m "test(core): update existing tests for permissive validation behavior"
```

---

### Tarefa 5: Criar componente WarningBanner do dashboard

**Arquivos:**
- Criar: `understand-anything-plugin/packages/dashboard/src/components/WarningBanner.tsx`

**Passo 1: Buildar pacote core para import do dashboard**

Rodar: `pnpm --filter @understand-anything/core build`
Esperado: Build conclui com novos exports

**Passo 2: Criar componente WarningBanner**

Criar `understand-anything-plugin/packages/dashboard/src/components/WarningBanner.tsx`:

```tsx
import { useState } from "react";
import type { GraphIssue } from "@understand-anything/core/schema";

interface WarningBannerProps {
  issues: GraphIssue[];
}

export default function WarningBanner({ issues }: WarningBannerProps) {
  const [expanded, setExpanded] = useState(false);
  const [copied, setCopied] = useState(false);

  const autoCorrected = issues.filter((i) => i.level === "auto-corrected");
  const dropped = issues.filter((i) => i.level === "dropped");

  const summaryParts: string[] = [];
  if (autoCorrected.length > 0) {
    summaryParts.push(
      `${autoCorrected.length} auto-correction${autoCorrected.length > 1 ? "s" : ""}`,
    );
  }
  if (dropped.length > 0) {
    summaryParts.push(
      `${dropped.length} dropped item${dropped.length > 1 ? "s" : ""}`,
    );
  }

  const copyText = [
    "The following issues were found in your knowledge-graph.json.",
    "These are LLM generation errors — not a system bug.",
    "You can ask your agent to fix these specific issues in the knowledge-graph.json file:",
    "",
    ...issues.map(
      (i) =>
        `[${i.level === "auto-corrected" ? "Auto-corrected" : "Dropped"}] ${i.message}`,
    ),
  ].join("\n");

  const handleCopy = async () => {
    await navigator.clipboard.writeText(copyText);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <div className="px-5 py-3 border-b text-sm" style={{
      backgroundColor: "rgba(212, 165, 116, 0.08)",
      borderColor: "rgba(212, 165, 116, 0.25)",
    }}>
      <div className="flex items-center justify-between">
        <button
          onClick={() => setExpanded(!expanded)}
          className="flex items-center gap-2 text-left transition-colors"
          style={{ color: "var(--color-gold-dim)" }}
        >
          <svg
            className={`w-4 h-4 shrink-0 transition-transform ${expanded ? "rotate-90" : ""}`}
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              strokeWidth={2}
              d="M9 5l7 7-7 7"
            />
          </svg>
          <span>
            Knowledge graph loaded with {summaryParts.join(" and ")}
          </span>
        </button>
        <button
          onClick={handleCopy}
          className="shrink-0 flex items-center gap-1 text-xs transition-colors"
          style={{ color: copied ? "var(--color-gold)" : "var(--color-text-muted)" }}
        >
          <svg
            className="w-3.5 h-3.5"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            {copied ? (
              <path
                strokeLinecap="round"
                strokeLinejoin="round"
                strokeWidth={2}
                d="M5 13l4 4L19 7"
              />
            ) : (
              <path
                strokeLinecap="round"
                strokeLinejoin="round"
                strokeWidth={2}
                d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z"
              />
            )}
          </svg>
          {copied ? "Copied!" : "Copy issues"}
        </button>
      </div>

      {expanded && (
        <div className="mt-3 space-y-1 font-mono text-xs">
          {autoCorrected.length > 0 && (
            <>
              <div className="text-text-muted font-sans text-xs font-medium mt-2 mb-1">
                Auto-corrected ({autoCorrected.length})
              </div>
              {autoCorrected.map((issue, i) => (
                <div key={`ac-${i}`} className="text-text-secondary pl-3">
                  {issue.message}
                </div>
              ))}
            </>
          )}
          {dropped.length > 0 && (
            <>
              <div className="font-sans text-xs font-medium mt-2 mb-1" style={{ color: "var(--color-gold)" }}>
                Dropped ({dropped.length})
              </div>
              {dropped.map((issue, i) => (
                <div key={`dr-${i}`} className="pl-3" style={{ color: "var(--color-gold-dim)" }}>
                  {issue.message}
                </div>
              ))}
            </>
          )}
          <p className="mt-3 text-text-muted font-sans text-xs">
            These are LLM generation issues, not system bugs. Copy the issues
            above and ask your agent to fix them in the knowledge-graph.json, or
            re-run{" "}
            <code className="text-gold-dim">/understand</code> for a fresh
            generation.
          </p>
        </div>
      )}
    </div>
  );
}
```

**Passo 3: Verificar que o dashboard buila**

Rodar: `pnpm --filter @understand-anything/dashboard build`
Esperado: Build conclui (componente ainda não conectado, mas deve compilar)

**Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/components/WarningBanner.tsx
git commit -m "feat(dashboard): add WarningBanner component for graph validation issues"
```

---

### Tarefa 6: Conectar WarningBanner em App.tsx

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/dashboard/src/App.tsx`

**Passo 1: Atualizar App.tsx**

Adicionar import no topo do arquivo (após outros imports de componentes):

```typescript
import WarningBanner from "./components/WarningBanner";
import type { GraphIssue } from "@understand-anything/core/schema";
```

Adicionar state para issues (após o state `loadError`, linha 26):

```typescript
const [graphIssues, setGraphIssues] = useState<GraphIssue[]>([]);
```

Substituir o `useEffect` de carregamento de grafo (linhas 119-136) por:

```typescript
useEffect(() => {
  fetch("/knowledge-graph.json")
    .then((res) => res.json())
    .then((data: unknown) => {
      const result = validateGraph(data);
      if (result.success && result.data) {
        setGraph(result.data);
        setGraphIssues(result.issues);
        if (result.issues.length > 0) {
          const autoCorrected = result.issues.filter((i) => i.level === "auto-corrected");
          const dropped = result.issues.filter((i) => i.level === "dropped");
          if (autoCorrected.length > 0) console.warn(`[understand-anything] Auto-corrected ${autoCorrected.length} graph issues`);
          if (dropped.length > 0) console.error(`[understand-anything] Dropped ${dropped.length} broken graph items`);
        }
      } else if (result.fatal) {
        console.error("Knowledge graph fatal error:", result.fatal);
        setLoadError(result.fatal);
      } else {
        setLoadError("Unknown validation error");
      }
    })
    .catch((err) => {
      console.error("Failed to load knowledge graph:", err);
      setLoadError(
        `Failed to load knowledge graph: ${err instanceof Error ? err.message : String(err)}`,
      );
    });
}, [setGraph]);
```

Substituir a seção do banner de erro (linhas 213-218) por:

```tsx
{/* Warning banner for graph issues */}
{graphIssues.length > 0 && !loadError && (
  <WarningBanner issues={graphIssues} />
)}

{/* Fatal error banner */}
{loadError && (
  <div className="px-5 py-3 bg-red-900/30 border-b border-red-700 text-red-200 text-sm">
    {loadError}
  </div>
)}
```

**Passo 2: Buildar e verificar**

Rodar: `pnpm --filter @understand-anything/core build && pnpm --filter @understand-anything/dashboard build`
Esperado: Ambos os builds concluem

**Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/dashboard/src/App.tsx
git commit -m "feat(dashboard): wire WarningBanner to display graph validation issues"
```

---

### Tarefa 7: Verificação final

**Passo 1: Rodar todos os testes do core**

Rodar: `pnpm --filter @understand-anything/core test`
Esperado: TODOS os testes passam

**Passo 2: Buildar pipeline completo**

Rodar: `pnpm --filter @understand-anything/core build && pnpm --filter @understand-anything/dashboard build`
Esperado: Ambos os builds concluem sem erros

**Passo 3: Lint**

Rodar: `pnpm lint`
Esperado: Sem erros de lint nos arquivos alterados

**Passo 4: Commit final (se precisar de fixes do linter)**

```bash
git add -A
git commit -m "chore: lint fixes for dashboard robustness feature"
```
