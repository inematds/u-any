# Plano de Implementação — Redução de Tokens

> 🇬🇧 Versão original em inglês: [2026-03-27-token-reduction-impl.md](./2026-03-27-token-reduction-impl.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa por tarefa.

**Objetivo:** Reduzir o custo de tokens do `/understand` em ~85% em codebases grandes através de pré-resolução de imports, consolidação de batches, remoção de adendos, enxugamento de payload e gating do reviewer LLM.

**Arquitetura:** Cinco mudanças (C5 → C4 → C3 → C1+C2) aplicadas em ordem de rollout — primeiro as de menor risco. Todas as mudanças são em arquivos markdown de prompt/skill em `understand-anything-plugin/skills/understand/`. Nenhuma alteração de código TypeScript é necessária.

**Stack Técnica:** Arquivos markdown de skill, scripts inline Node.js embutidos em SKILL.md, pipeline JSON do grafo de conhecimento.

**Doc de design:** `docs/plans/2026-03-27-token-reduction-design.md`

---

## Tarefa 1: C5 — Aplicar gating no graph-reviewer atrás da flag `--review`

Substitui o subagente LLM graph-reviewer (sempre ativo) por um script determinístico inline de validação. O reviewer LLM só roda quando `--review` está em `$ARGUMENTS`. Economiza ~58.500 tokens por execução padrão.

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` (Phase 6, linhas 330–362)

### Passo 1: Abrir SKILL.md e localizar a Phase 6

Leia o arquivo e encontre "## Phase 6 — REVIEW" (linha 297). Identifique os passos 3–6 (linhas 330–362) que atualmente sempre despacham o subagente LLM graph-reviewer.

### Passo 2: Substituir os passos 3–6 da Phase 6 pela lógica condicional do reviewer

Substitua as linhas 330–362 (de "3. Dispatch a subagent using the prompt template" até "6. **If `approved: true`:** Proceed to Phase 7.") por:

```markdown
3. **Check `$ARGUMENTS` for `--review` flag.** Then run the appropriate validation path:

---

#### Default path (no `--review`): inline deterministic validation

Write the following Node.js script to `$PROJECT_ROOT/.understand-anything/tmp/ua-inline-validate.js`:

```javascript
#!/usr/bin/env node
const fs = require('fs');
const graphPath = process.argv[2];
const outputPath = process.argv[3];
try {
  const graph = JSON.parse(fs.readFileSync(graphPath, 'utf8'));
  const issues = [], warnings = [];
  const nodeIds = new Set();
  const seen = new Map();
  graph.nodes.forEach((n, i) => {
    if (!n.id) { issues.push(`Node[${i}] missing id`); return; }
    if (!n.type) issues.push(`Node[${i}] '${n.id}' missing type`);
    if (!n.name) issues.push(`Node[${i}] '${n.id}' missing name`);
    if (!n.summary) issues.push(`Node[${i}] '${n.id}' missing summary`);
    if (!n.tags || !n.tags.length) issues.push(`Node[${i}] '${n.id}' missing tags`);
    if (seen.has(n.id)) issues.push(`Duplicate node ID '${n.id}' at indices ${seen.get(n.id)} and ${i}`);
    else seen.set(n.id, i);
    nodeIds.add(n.id);
  });
  graph.edges.forEach((e, i) => {
    if (!nodeIds.has(e.source)) issues.push(`Edge[${i}] source '${e.source}' not found`);
    if (!nodeIds.has(e.target)) issues.push(`Edge[${i}] target '${e.target}' not found`);
  });
  const fileNodes = graph.nodes.filter(n => n.type === 'file').map(n => n.id);
  const assigned = new Map();
  (graph.layers || []).forEach(layer => {
    (layer.nodeIds || []).forEach(id => {
      if (!nodeIds.has(id)) issues.push(`Layer '${layer.id}' refs missing node '${id}'`);
      if (assigned.has(id)) issues.push(`Node '${id}' appears in multiple layers`);
      assigned.set(id, layer.id);
    });
  });
  fileNodes.forEach(id => {
    if (!assigned.has(id)) issues.push(`File node '${id}' not in any layer`);
  });
  (graph.tour || []).forEach((step, i) => {
    (step.nodeIds || []).forEach(id => {
      if (!nodeIds.has(id)) issues.push(`Tour step[${i}] refs missing node '${id}'`);
    });
  });
  const withEdges = new Set([
    ...graph.edges.map(e => e.source),
    ...graph.edges.map(e => e.target)
  ]);
  graph.nodes.forEach(n => {
    if (!withEdges.has(n.id)) warnings.push(`Node '${n.id}' has no edges (orphan)`);
  });
  const stats = {
    totalNodes: graph.nodes.length,
    totalEdges: graph.edges.length,
    totalLayers: (graph.layers || []).length,
    tourSteps: (graph.tour || []).length,
    nodeTypes: graph.nodes.reduce((a, n) => { a[n.type] = (a[n.type]||0)+1; return a; }, {}),
    edgeTypes: graph.edges.reduce((a, e) => { a[e.type] = (a[e.type]||0)+1; return a; }, {})
  };
  fs.writeFileSync(outputPath, JSON.stringify({ issues, warnings, stats }, null, 2));
  process.exit(0);
} catch (err) { process.stderr.write(err.message + '\n'); process.exit(1); }
```

Execute it:
```bash
node $PROJECT_ROOT/.understand-anything/tmp/ua-inline-validate.js \
  "$PROJECT_ROOT/.understand-anything/intermediate/assembled-graph.json" \
  "$PROJECT_ROOT/.understand-anything/intermediate/review.json"
```

If the script exits non-zero, read stderr, fix the script, and retry once.

---

#### `--review` path: full LLM reviewer

If `--review` IS in `$ARGUMENTS`, dispatch the LLM graph-reviewer subagent as follows:

Dispatch a subagent using the prompt template at `./graph-reviewer-prompt.md`. Read the template file and pass the full content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Phase 1 scan results (file inventory):
> ```json
> [list of {path, sizeLines} from scan-result.json]
> ```
>
> Phase warnings/errors accumulated during analysis:
> - [list any batch failures, skipped files, or warnings from Phases 2-5]
>
> Cross-validate: every file in the scan inventory should have a corresponding `file:` node in the graph. Flag any missing files. Also flag any graph nodes whose `filePath` doesn't appear in the scan inventory.

Pass these parameters in the dispatch prompt:

> Validate the knowledge graph at `$PROJECT_ROOT/.understand-anything/intermediate/assembled-graph.json`.
> Project root: `$PROJECT_ROOT`
> Read the file and validate it for completeness and correctness.
> Write output to: `$PROJECT_ROOT/.understand-anything/intermediate/review.json`

---

4. Read `$PROJECT_ROOT/.understand-anything/intermediate/review.json`.

5. **If `issues` array is non-empty:**
   - Review the `issues` list
   - Apply automated fixes where possible:
     - Remove edges with dangling references
     - Fill missing required fields with sensible defaults (e.g., empty `tags` -> `["untagged"]`, empty `summary` -> `"No summary available"`)
     - Remove nodes with invalid types
   - Re-run the final graph validation after automated fixes
   - If critical issues remain after one fix attempt, save the graph anyway but include the warnings in the final report and mark dashboard auto-launch as skipped

6. **If `issues` array is empty:** Proceed to Phase 7.
```

### Passo 3: Verificar a edição

Releia SKILL.md linhas 297–380 e confirme:
- O passo 3 da Phase 6 agora checa a flag `--review`
- O script de validação inline está presente e completo
- O caminho `--review` ainda despacha o subagente LLM de forma idêntica à anterior
- Os passos 4–6 tratam a saída `review.json` da mesma forma que antes

### Passo 4: Commit

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "perf(understand): gate LLM graph-reviewer behind --review flag, add inline deterministic validation"
```

---

## Tarefa 2: C4a — Enxugar payload de nós da Phase 4 (architecture)

Remove `name` e `languageNotes` do formato de nó de arquivo injetado no subagente architecture-analyzer. Esses campos não são necessários para atribuição de camadas arquiteturais e adicionam tokens desnecessários.

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` (Phase 4, em torno das linhas 188–196)

### Passo 1: Localizar o prompt de dispatch da Phase 4 em SKILL.md

Encontre o bloco que começa com "Pass these parameters in the dispatch prompt:" sob a Phase 4 (em torno da linha 181). Procure por:

```
> File nodes:
> ```json
> [list of {id, name, filePath, summary, tags} for all file-type nodes]
> ```
```

### Passo 2: Atualizar o formato dos nós de arquivo

Mude a linha dos file nodes de:
```
> [list of {id, name, filePath, summary, tags} for all file-type nodes]
```

Para:
```
> [list of {id, filePath, summary, tags} for all file-type nodes — omit name, complexity, languageNotes]
```

### Passo 3: Verificar

Releia a Phase 4 e confirme que a linha do formato dos nós foi atualizada. A linha de import edges abaixo (`[list of edges with type "imports"]`) permanece inalterada.

### Passo 4: Commit

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "perf(understand): slim Phase 4 architecture payload — drop redundant node fields"
```

---

## Tarefa 3: C4b — Enxugar payload da Phase 5 (tour builder)

A Phase 5 atualmente injeta todos os nós (incluindo function/class), todos os tipos de edge e objetos completos de layer (com arrays nodeIds). Apenas file nodes, edges imports+calls e layers enxutos são necessários para o design do tour. Esta é a maior mudança individual de payload, economizando ~105.000 tokens em um projeto de 500 arquivos.

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` (Phase 5, linhas 257–270)
- Modificar: `understand-anything-plugin/skills/understand/tour-builder-prompt.md` (input schema)

### Passo 1: Localizar o prompt de dispatch da Phase 5 em SKILL.md

Encontre o bloco que começa com (em torno da linha 257):
```
> Nodes (summarized):
> ```json
> [list of {id, name, filePath, summary, type} for key nodes]
> ```
>
> Layers:
> ```json
> [layers from Phase 4]
> ```
>
> Key edges:
> ```json
> [imports and calls edges]
> ```
```

### Passo 2: Substituir as três seções de payload

Substitua essas linhas por:

```markdown
> Nodes (file nodes only):
> ```json
> [list of {id, name, filePath, summary, type} for file-type nodes ONLY — do NOT include function or class nodes]
> ```
>
> Layers:
> ```json
> [list of {id, name, description} for each layer — omit nodeIds]
> ```
>
> Edges (imports and calls only):
> ```json
> [list of edges where type is "imports" or "calls" only — exclude all other edge types]
> ```
```

### Passo 3: Atualizar input schema de tour-builder-prompt.md

Abra `tour-builder-prompt.md` e encontre a seção "Script Requirements" (em torno das linhas 18–35). O input schema atual mostra:
```json
{
  "nodes": [...],
  "edges": [...],
  "layers": [
    {"id": "layer:core", "name": "Core", "nodeIds": ["file:src/index.ts"]}
  ]
}
```

Atualize o exemplo de layers para refletir o formato enxuto:
```json
{
  "nodes": [
    {"id": "file:src/index.ts", "type": "file", "name": "index.ts", "filePath": "src/index.ts", "summary": "..."}
  ],
  "edges": [
    {"source": "file:src/index.ts", "target": "file:src/utils.ts", "type": "imports"}
  ],
  "layers": [
    {"id": "layer:core", "name": "Core", "description": "Core application logic"}
  ]
}
```

Atualize também a descrição "G. Node Summary Index" (em torno da linha 84) para refletir que os nós de entrada são apenas do tipo file:

Encontre:
```
**G. Node Summary Index**

Create a lookup of each node ID to its `summary`, `type`, `tags` (default to empty array `[]` if not present in input), and `name` for easy reference.
```

Adicione uma nota depois:
```
Note: input nodes are file-type only. The nodeSummaryIndex will contain only file nodes.
```

### Passo 4: Verificar

- Releia o bloco de payload da Phase 5 em SKILL.md: confirma apenas file nodes, layers enxutos (sem nodeIds), edges apenas imports+calls
- Releia o input schema de tour-builder-prompt.md: layers não têm mais nodeIds

### Passo 5: Commit

```bash
git add understand-anything-plugin/skills/understand/SKILL.md \
        understand-anything-plugin/skills/understand/tour-builder-prompt.md
git commit -m "perf(understand): slim Phase 5 tour payload — file nodes only, imports+calls edges, slim layers"
```

---

## Tarefa 4: C3 — Remover adendos de linguagem/framework dos batches do file-analyzer

Os adendos (`languages/typescript.md`, `frameworks/react.md`, etc.) são atualmente injetados em todo prompt de batch do file-analyzer. Eles custam ~1.300 tokens × N batches. O modelo já conhece essas linguagens. Substitua por uma tabela de referência inline compacta (~150 tokens, paga uma vez, embutida no template base).

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` (Phase 2, linhas 104–117)
- Modificar: `understand-anything-plugin/skills/understand/file-analyzer-prompt.md` (adicionar seção de quick reference)

### Passo 1: Atualizar o bloco "Build the combined prompt template" em SKILL.md Phase 2

Encontre o bloco nas linhas 104–117:
```
**Build the combined prompt template:**
1. Read the base template at `./file-analyzer-prompt.md`.
2. **Language context injection:** ...
3. **Framework addendum injection:** ...

Then for each batch pass the combined template content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Project: `<projectName>` — `<projectDescription>`
> Frameworks detected: `<frameworks from Phase 1>`
> Languages: `<languages from Phase 1>`
>
> Use the language context and framework addendums (appended above) to produce more accurate summaries and better classify file roles.
```

Substitua por:
```markdown
**Build the prompt for each batch:**
1. Read the base template at `./file-analyzer-prompt.md`. (Language and framework hints are embedded in the template — do NOT append addendum files for Phase 2 batches. Addendums are reserved for Phase 4.)

Then for each batch pass the template content as the subagent's prompt, appending the following additional context:

> **Additional context from main session:**
>
> Project: `<projectName>` — `<projectDescription>`
> Languages: `<languages from Phase 1>`
```

Isso remove os passos 2 e 3 (os loops de injeção de adendos) inteiramente da Phase 2.

### Passo 2: Adicionar Language and Framework Quick Reference em file-analyzer-prompt.md

Abra `file-analyzer-prompt.md`. Encontre a seção "## Critical Constraints" perto do fim (em torno da linha 299). Insira a seguinte nova seção **antes** de "## Critical Constraints":

```markdown
## Language and Framework Quick Reference

Use these hints to improve tag and edge accuracy for common patterns. Your training knowledge covers these — this is a fast lookup for the most impactful signals.

**Tag signals:**

| Signal | Tags to apply |
|---|---|
| File in `hooks/`, exports a function starting with `use` | `hook`, `service` |
| File in `contexts/` or `context/`, exports a Provider component | `service`, `state` |
| File in `pages/` or `views/` | `ui`, `routing` |
| File in `store/`, `slices/`, `reducers/`, `state/` | `state` |
| File in `services/`, `api/`, `client/` | `service` |
| `__init__.py` at a package root with re-exports | `entry-point`, `barrel` |
| `manage.py` at the project root | `entry-point` |
| `mod.rs` in a directory | `barrel` |
| `main.go` in a `cmd/` subdirectory | `entry-point` |

**Edge signals:**

| Pattern | Edge to create |
|---|---|
| React component renders another component in its JSX | `contains` from parent to child |
| Component/hook calls a custom hook (`useX`) | `depends_on` from consumer to hook file |
| Context provider wraps components | `publishes` from provider to context definition |
| Component calls `useContext` or custom context hook | `subscribes` from consumer to context definition |
| Python file uses `from x import y` where x is a project file | `imports` edge (same rule as JS/TS) |
| Go file `import`s an internal package path | `imports` edge to the resolved file |

```

### Passo 3: Verificar

- Releia o bloco "Build the prompt" da Phase 2 em SKILL.md: os passos 2 e 3 (loops de adendo) desapareceram; a linha "Frameworks detected" no additional context desapareceu
- Releia file-analyzer-prompt.md: a nova seção "Language and Framework Quick Reference" aparece antes de Critical Constraints; sem referência a arquivos de adendo
- Confirme que "Build the combined prompt template" da Phase 4 (linhas 163–167) está **inalterado** — adendos ainda se aplicam ali

### Passo 4: Commit

```bash
git add understand-anything-plugin/skills/understand/SKILL.md \
        understand-anything-plugin/skills/understand/file-analyzer-prompt.md
git commit -m "perf(understand): remove addendum injection from Phase 2 batches, add compact inline hints to file-analyzer"
```

---

## Tarefa 5: C1a — Estender o scanner para pré-resolver imports

Adiciona um novo Passo 8 ao script do project scanner: parsear declarações de import de cada arquivo-fonte e resolver imports relativos contra a lista de arquivos descobertos. O mapa resolvido é gravado em `scan-result.json` como `importMap`. Esses são os dados que nos permitem eliminar `allProjectFiles` de cada batch na Tarefa 7.

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/project-scanner-prompt.md`

### Passo 1: Adicionar o Passo 8 aos requisitos do script do scanner

Abra `project-scanner-prompt.md`. Encontre "**Step 7 -- Project Name**" (em torno da linha 100). Após seu conteúdo (a lista de prioridades), adicione um novo passo:

```markdown
**Step 8 -- Import Resolution**

For each file in the discovered source list, extract and resolve relative import statements. The goal is to produce a map from each file's path to the list of project-internal files it imports. External package imports are ignored.

For each file, read its content and extract import paths using language-appropriate patterns:

| Language | Import patterns to match |
|---|---|
| TypeScript/JavaScript | `import ... from './...'` or `'../'`, `require('./...')` or `require('../...')` |
| Python | `from .x import y`, `from ..x import y`, `import .x` (relative only) |
| Go | Paths in `import (...)` blocks that start with the module path from `go.mod` |
| Rust | `use crate::`, `use super::`, `mod x` (within the same crate) |
| Java/Kotlin | Not resolvable by path — skip import resolution for these languages |
| Ruby | `require_relative '...'` paths |

For each extracted import path:
1. Compute the resolved file path relative to project root:
   - For relative imports (`./x`, `../x`): resolve from the importing file's directory
   - Try these extension variants in order if the import has no extension: `.ts`, `.tsx`, `.js`, `.jsx`, `/index.ts`, `/index.js`, `/index.tsx`, `/index.jsx`, `.py`, `.go`, `.rs`, `.rb`
2. Check if the resolved path exists in the discovered file list
3. If yes: add to this file's resolved imports list
4. If no: skip (external, unresolvable, or dynamic import)

Output format in the script result:
```json
"importMap": {
  "src/index.ts": ["src/utils.ts", "src/config.ts"],
  "src/utils.ts": [],
  "src/components/App.tsx": ["src/hooks/useAuth.ts", "src/store/index.ts"]
}
```

Keys are project-relative paths. Values are arrays of resolved project-relative paths. Every key in the file list must appear in `importMap` (use an empty array `[]` if no imports were resolved). External packages and unresolvable imports are omitted entirely.
```

### Passo 2: Atualizar o formato de saída do script do scanner

Encontre a seção "### Script Output Format" (em torno da linha 109) e atualize o exemplo JSON para incluir `importMap`:

Encontre isto no exemplo:
```json
{
  "scriptCompleted": true,
  "name": "project-name",
  ...
  "estimatedComplexity": "moderate"
}
```

Adicione `importMap` ao exemplo:
```json
{
  "scriptCompleted": true,
  "name": "project-name",
  "rawDescription": "...",
  "readmeHead": "...",
  "languages": ["javascript", "typescript"],
  "frameworks": ["React", "Vite"],
  "files": [
    {"path": "src/index.ts", "language": "typescript", "sizeLines": 150}
  ],
  "totalFiles": 42,
  "estimatedComplexity": "moderate",
  "importMap": {
    "src/index.ts": ["src/utils.ts", "src/config.ts"],
    "src/utils.ts": []
  }
}
```

Atualize também a lista de documentação de campos abaixo do exemplo para adicionar:
```
- `importMap` (object) — map from every source file path to its list of resolved project-internal import paths; empty array if no resolved imports; external packages excluded
```

### Passo 3: Atualizar a seção de assembly final para preservar importMap

Encontre "## Phase 2 -- Description and Final Assembly" (em torno da linha 153). Encontre a nota IMPORTANT:
```
**IMPORTANT:** The final output must NOT contain the `scriptCompleted`, `rawDescription`, or `readmeHead` fields.
```

Atualize para:
```
**IMPORTANT:** The final output must NOT contain the `scriptCompleted`, `rawDescription`, or `readmeHead` fields. All other fields — including `importMap` — MUST be preserved exactly as output by the script.
```

Atualize também o exemplo de saída final para incluir `importMap`:
```json
{
  "name": "project-name",
  "description": "...",
  "languages": ["typescript"],
  "frameworks": ["React"],
  "files": [...],
  "totalFiles": 42,
  "estimatedComplexity": "moderate",
  "importMap": {
    "src/index.ts": ["src/utils.ts"]
  }
}
```

### Passo 4: Verificar

Releia `project-scanner-prompt.md` e confirme:
- O Passo 8 está presente com a lógica completa de resolução de imports
- O formato de saída do script inclui `importMap`
- A documentação de campos inclui `importMap`
- A seção de assembly final preserva `importMap` na saída

### Passo 5: Commit

```bash
git add understand-anything-plugin/skills/understand/project-scanner-prompt.md
git commit -m "perf(understand): extend scanner to pre-resolve imports, output importMap in scan-result.json"
```

---

## Tarefa 6: C1b — Atualizar file-analyzer para usar batchImportData

Remove `allProjectFiles` do input schema do file-analyzer e o substitui por `batchImportData` (imports pré-resolvidos apenas para os arquivos deste batch). Atualiza a seção do script de extração para pular a resolução de imports inteiramente (já feita pelo scanner). Atualiza o passo de criação de edges para usar `batchImportData` diretamente.

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/file-analyzer-prompt.md`

### Passo 1: Atualizar o input JSON schema (Script Requirements, passo 1)

Encontre o bloco de input schema em torno da linha 19:
```json
{
  "projectRoot": "/path/to/project",
  "allProjectFiles": ["src/index.ts", "src/utils.ts", "..."],
  "batchFiles": [
    {"path": "src/index.ts", "language": "typescript", "sizeLines": 150},
    {"path": "src/utils.ts", "language": "typescript", "sizeLines": 80}
  ]
}
```

Substitua por:
```json
{
  "projectRoot": "/path/to/project",
  "batchFiles": [
    {"path": "src/index.ts", "language": "typescript", "sizeLines": 150},
    {"path": "src/utils.ts", "language": "typescript", "sizeLines": 80}
  ],
  "batchImportData": {
    "src/index.ts": ["src/utils.ts", "src/config.ts"],
    "src/utils.ts": []
  }
}
```

Atualize as descrições de campo:
- Remova: descrição de `allProjectFiles`
- Adicione: `batchImportData` (object) — map from each batch file's project-relative path to its list of pre-resolved project-internal imports. Produced by the project scanner. Use this directly for import edge creation — do NOT attempt to re-resolve imports yourself.

### Passo 2: Remover a extração de imports de "What the Script Must Extract"

Encontre a subseção "**Imports:**" sob "What the Script Must Extract" (em torno das linhas 49–53):
```
**Imports:**
- Source module path (exactly as written in the import statement)
- Imported specifiers (named imports, default import, namespace import)
- Line number
- For relative imports (starting with `./` or `../`), compute the resolved path...
```

Substitua toda essa subseção por:
```markdown
**Imports:**
- Do NOT extract imports in the script. Import resolution has already been performed by the project scanner.
- The pre-resolved imports for each file are provided in `batchImportData` in the input JSON.
- Do not include an `imports` field in the script output — import edges will be created in Phase 2 using `batchImportData` directly.
```

### Passo 3: Atualizar o formato de saída do script para remover imports

Encontre o array `results` no formato de saída do script (em torno da linha 67). O array `imports` atual na saída:
```json
"imports": [
  {"source": "./utils", "resolvedPath": "src/utils.ts", "specifiers": ["formatDate"], "line": 1, "isExternal": false},
  {"source": "express", "resolvedPath": null, "specifiers": ["default"], "line": 2, "isExternal": true}
],
```

Remova o array `imports` do formato de saída do script inteiramente. O resultado para cada arquivo deve ser:
```json
{
  "path": "src/index.ts",
  "language": "typescript",
  "totalLines": 150,
  "nonEmptyLines": 120,
  "functions": [...],
  "classes": [...],
  "exports": [...],
  "metrics": {
    "importCount": 5,
    "exportCount": 3,
    "functionCount": 4,
    "classCount": 1
  }
}
```

Mantenha `metrics.importCount` (derivado de `batchImportData[path].length`) como métrica útil.

Atualize a descrição de metrics para dizer:
```
- `importCount` (integer) — use `batchImportData[file.path].length` from the input JSON
```

### Passo 4: Atualizar a seção "Preparing the Script Input"

Encontre o comando `cat` em torno da linha 113 que cria o input JSON:
```bash
cat > $PROJECT_ROOT/.understand-anything/tmp/ua-file-analyzer-input-<batchIndex>.json << 'ENDJSON'
{
  "projectRoot": "<project-root>",
  "allProjectFiles": [<full file list from scan>],
  "batchFiles": [<this batch's files>]
}
ENDJSON
```

Substitua por:
```bash
cat > $PROJECT_ROOT/.understand-anything/tmp/ua-file-analyzer-input-<batchIndex>.json << 'ENDJSON'
{
  "projectRoot": "<project-root>",
  "batchFiles": [<this batch's files>],
  "batchImportData": <batchImportData JSON object — provided in your dispatch prompt>
}
ENDJSON
```

### Passo 5: Atualizar o Passo 3 (Create Edges) — Regra de criação de import edges

Encontre o "**Import edge creation rule:**" na seção "Step 3 -- Create Edges" (em torno da linha 213):
```
**Import edge creation rule:** For each import in the script output where `isExternal` is `false` and `resolvedPath` is non-null, create an `imports` edge from the current file node to `file:<resolvedPath>`. Do NOT create edges for external package imports.
```

Substitua por:
```markdown
**Import edge creation rule:** For each resolved path in `batchImportData[filePath]` (provided in the input JSON), create an `imports` edge from the current file node to `file:<resolvedPath>`. The `batchImportData` values contain only resolved project-internal paths — external packages have already been filtered out. Do NOT attempt to re-resolve imports from source.
```

### Passo 6: Remover referências a `allProjectFiles` de Critical Constraints

Encontre o último bullet em "## Critical Constraints" (em torno da linha 304):
```
- For import edges, use the script's `resolvedPath` field directly. Do NOT attempt to resolve import paths yourself -- the script already did this deterministically.
```

Substitua por:
```markdown
- For import edges, use `batchImportData[filePath]` directly from the input JSON. Do NOT attempt to resolve import paths yourself -- the project scanner already did this deterministically.
```

### Passo 7: Verificar

Releia `file-analyzer-prompt.md` e confirme:
- Input schema tem `batchImportData`, sem `allProjectFiles`
- Seção "What to Extract" do script: extração de imports substituída por "do not extract"
- Formato de saída do script: sem array `imports` por arquivo
- Preparing the Script Input: comando cat sem `allProjectFiles`
- Regra de criação de import edge: usa `batchImportData` e não a saída do script
- Critical Constraints: sem referência a `resolvedPath` do script

### Passo 8: Commit

```bash
git add understand-anything-plugin/skills/understand/file-analyzer-prompt.md
git commit -m "perf(understand): replace allProjectFiles with batchImportData in file-analyzer — import resolution now done by scanner"
```

---

## Tarefa 7: C1c + C2 — Atualizar orquestração da Phase 2 em SKILL.md

Conecta o `importMap` da Phase 1 nas fatias `batchImportData` por batch. Aumenta o tamanho do batch de 5-10 para 20-30 arquivos. Aumenta a concorrência de 3 para 5. Remove `allProjectFiles` do prompt de dispatch.

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` (Phase 0, Phase 1, Phase 2)

### Passo 1: Atualizar a Phase 1 para mencionar que importMap agora está em scan-result.json

Encontre a Phase 1 (em torno da linha 62) onde diz:
```
After the subagent completes, read `$PROJECT_ROOT/.understand-anything/intermediate/scan-result.json` to get:
- Project name, description
- Languages, frameworks
- File list with line counts
- Complexity estimate
```

Adicione um item à lista:
```
- Import map (`importMap`): pre-resolved project-internal imports per file
```

Adicione também uma nota:
```
Store `importMap` in memory as `$IMPORT_MAP` for use in Phase 2 batch construction.
```

### Passo 2: Mudar tamanho de batch e concorrência na Phase 2

Encontre a linha 100:
```
Batch the file list from Phase 1 into groups of **5-10 files each** (aim for balanced batch sizes).
```

Substitua por:
```
Batch the file list from Phase 1 into groups of **20-30 files each** (aim for ~25 files per batch for balanced sizes).
```

Encontre a linha 102:
```
For each batch, dispatch a subagent using the prompt template at `./file-analyzer-prompt.md`. Run up to **3 subagents concurrently** using parallel dispatch.
```

Substitua por:
```
For each batch, dispatch a subagent using the prompt template at `./file-analyzer-prompt.md`. Run up to **5 subagents concurrently** using parallel dispatch.
```

### Passo 3: Adicionar a construção de batchImportData ao bloco de dispatch

Encontre o bloco do prompt de dispatch (em torno das linhas 119–134):
```
Fill in batch-specific parameters below and dispatch:

> Analyze these source files and produce GraphNode and GraphEdge objects.
> Project root: `$PROJECT_ROOT`
> Project: `<projectName>`
> Languages: `<languages>`
> Batch index: `<batchIndex>`
> Write output to: `$PROJECT_ROOT/.understand-anything/intermediate/batch-<batchIndex>.json`
>
> All project files (for import resolution):
> `<full file path list from scan>`
>
> Files to analyze in this batch:
> 1. `<path>` (<sizeLines> lines)
> ...
```

Substitua por:
```markdown
Before dispatching each batch, construct `batchImportData` from `$IMPORT_MAP`:
```json
batchImportData = {}
for each file in this batch:
  batchImportData[file.path] = $IMPORT_MAP[file.path] ?? []
```

Fill in batch-specific parameters below and dispatch:

> Analyze these source files and produce GraphNode and GraphEdge objects.
> Project root: `$PROJECT_ROOT`
> Project: `<projectName>`
> Languages: `<languages>`
> Batch index: `<batchIndex>`
> Write output to: `$PROJECT_ROOT/.understand-anything/intermediate/batch-<batchIndex>.json`
>
> Pre-resolved import data for this batch (use this for all import edge creation — do NOT re-resolve imports from source):
> ```json
> <batchImportData JSON>
> ```
>
> Files to analyze in this batch:
> 1. `<path>` (<sizeLines> lines)
> 2. `<path>` (<sizeLines> lines)
> ...
```

### Passo 4: Atualizar o caminho de update incremental

Encontre "### Incremental update path" (em torno da linha 140):
```
Use the changed files list from Phase 0. Batch and dispatch file-analyzer subagents using the same process as above, but only for changed files.
```

Atualize para esclarecer que batchImportData ainda se aplica:
```
Use the changed files list from Phase 0. Batch and dispatch file-analyzer subagents using the same process as above (20-30 files per batch, up to 5 concurrent, with batchImportData constructed from $IMPORT_MAP), but only for changed files.
```

### Passo 5: Verificar todas as mudanças da Phase 2

Releia a Phase 2 inteira em SKILL.md e confirme:
- O tamanho de batch diz "20-30 files"
- A concorrência diz "5 subagents concurrently"
- O bloco "Build the prompt": apenas o passo 1 (read base template), sem passos de adendo
- Bloco de contexto adicional: sem linha "Frameworks detected", sem referência a adendo
- Prompt de dispatch: tem injeção de `batchImportData`, sem `allProjectFiles`
- Caminho incremental: menciona batchImportData

### Passo 6: Commit

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "perf(understand): wire importMap into batchImportData per batch, increase batch size 5-10→20-30, concurrency 3→5"
```

---

## Tarefa 8: Version bump

Por convenção do projeto, todos os quatro arquivos de versão devem permanecer sincronizados quando mudanças são enviadas.

**Arquivos:**
- Modificar: `understand-anything-plugin/package.json`
- Modificar: `.claude-plugin/marketplace.json`
- Modificar: `.claude-plugin/plugin.json`
- Modificar: `.cursor-plugin/plugin.json`

### Passo 1: Ler a versão atual

```bash
node -e "const p = require('./understand-anything-plugin/package.json'); console.log(p.version)"
```

Esperado: `1.2.1` (ou qualquer que seja a versão atual).

### Passo 2: Bump da versão patch em todos os quatro arquivos

Nova versão: `1.2.2` (bump patch — otimização interna, sem mudanças de API).

Atualize cada arquivo:
- `understand-anything-plugin/package.json`: `"version": "1.2.2"`
- `.claude-plugin/marketplace.json`: `"version": "1.2.2"` em `plugins[0]`
- `.claude-plugin/plugin.json`: `"version": "1.2.2"`
- `.cursor-plugin/plugin.json`: `"version": "1.2.2"`

### Passo 3: Verificar se todos os quatro arquivos batem

```bash
grep -r '"version"' understand-anything-plugin/package.json .claude-plugin/marketplace.json .claude-plugin/plugin.json .cursor-plugin/plugin.json
```

Todos os quatro devem mostrar `"version": "1.2.2"`.

### Passo 4: Commit

```bash
git add understand-anything-plugin/package.json \
        .claude-plugin/marketplace.json \
        .claude-plugin/plugin.json \
        .cursor-plugin/plugin.json
git commit -m "chore: bump version to 1.2.2"
```

---

## Tarefa 9: Build e smoke test

Verifica que todas as mudanças funcionam de ponta a ponta rodando `/understand --full` contra um projeto real.

**Arquivos:** Nenhum (apenas testes)

### Passo 1: Build dos pacotes

```bash
pnpm --filter @understand-anything/core build
pnpm --filter @understand-anything/skill build
```

Esperado: ambos os builds sem erros.

### Passo 2: Encontrar a versão instalada do plugin e copiar para o cache

```bash
ls ~/.claude/plugins/cache/understand-anything/understand-anything/
```

Anote a versão (ex.: `1.0.1`). Copie o build local para o cache:

```bash
VERSION=$(node -e "const p = require('./understand-anything-plugin/package.json'); console.log(p.version)")
rm -rf ~/.claude/plugins/cache/understand-anything/understand-anything/$VERSION
cp -R ./understand-anything-plugin ~/.claude/plugins/cache/understand-anything/understand-anything/$VERSION
```

### Passo 3: Smoke test em um projeto pequeno (~20 arquivos)

Abra uma sessão nova do Claude Code em um projeto TypeScript pequeno. Rode:
```
/understand --full
```

Verifique:
- Phases 0–7 completam sem erros
- `knowledge-graph.json` é criado
- A contagem de nodes e edges é razoável
- Layers e tour estão presentes
- Sem erros de "allProjectFiles" ou adendos na saída

### Passo 4: Smoke test em um projeto maior (~100+ arquivos)

Rode `/understand --full` em um projeto TypeScript+React médio/grande.

Verifique:
- Contagem de batches é ~4-6 (a 20-30 arquivos por batch para 100 arquivos), não 10-20
- Sem erros sobre resolução de imports faltante
- `importMap` está presente em `scan-result.json` (cheque `.understand-anything/intermediate/` antes da limpeza, ou adicione um debug log temporário)
- Qualidade do grafo é comparável à anterior (summaries descritivos, layers corretos)

### Passo 5: Testar a flag `--review`

Rode `/understand --full --review` no mesmo projeto.

Verifique:
- A Phase 6 agora despacha o subagente LLM graph-reviewer (não o script inline)
- `review.json` é produzido com campo `approved`
- O pipeline completa normalmente

### Passo 6: Commit final (se houve correções no smoke test)

```bash
git add -A
git commit -m "fix(understand): smoke test fixes for token reduction changes"
```

---

## Resumo

| Tarefa | Mudança | Risco |
|---|---|---|
| 1 | C5: Gate do reviewer | Baixo |
| 2 | C4a: Slim do payload da Phase 4 | Baixo |
| 3 | C4b: Slim do payload da Phase 5 | Baixo |
| 4 | C3: Remoção de adendos dos batches | Baixo |
| 5 | C1a: Resolução de imports no scanner | Médio |
| 6 | C1b: File-analyzer usa batchImportData | Médio |
| 7 | C1c+C2: Orquestração de SKILL.md + tamanho de batch | Médio |
| 8 | Version bump | Baixo |
| 9 | Smoke test | — |

Tarefas 1–4 são independentes das Tarefas 5–7. Podem ser enviadas separadamente se necessário. Tarefas 5, 6 e 7 são fortemente acopladas (scanner produz importMap → SKILL.md passa batchImportData → file-analyzer consome) e devem ser enviadas juntas.
