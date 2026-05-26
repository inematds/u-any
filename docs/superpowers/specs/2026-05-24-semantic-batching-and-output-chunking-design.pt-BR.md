# Design de Batching Semântico e Chunking de Saída

> 🇬🇧 Versão original em inglês: [2026-05-24-semantic-batching-and-output-chunking-design.md](./2026-05-24-semantic-batching-and-output-chunking-design.md)

**Data:** 2026-05-24
**Status:** Rascunho
**Branch:** `feat/semantic-batching-and-output-chunking`
**Issue:** [#159](https://github.com/Lum1104/Understand-Anything/issues/159) — Frequentemente vendo output limit exceeded

---

## Problema

A Fase 2 da skill `/understand` despacha subagentes `file-analyzer` em batches de 20-30 arquivos cada (`skills/understand/SKILL.md:282`). Dois problemas se acumulam em backends de LLM com restrição de saída (notavelmente Bedrock OPUS com max_tokens default de 4096-8192):

1. **Pressão no teto de saída.** Cada `file-analyzer` escreve um `batch-<N>.json` contendo todos os nós (arquivo + funções + classes) e arestas para seu batch. Para 25 arquivos densos o conteúdo JSON facilmente excede o orçamento de tokens de `Write(content=...)` por turno. O agente improvisa entrando em um "minimal output mode" indefinido e descarta nós/arestas silenciosamente. A issue #159 reporta isso para OPUS no Bedrock na escala de 100 arquivos.

2. **Batching baseado em contagem quebra a semântica de módulo.** Arquivos são batch-ados por contagem, não por relação lógica. Arquivos que importam uns aos outros (e que juntos formariam um módulo `auth`, um módulo `api`, etc.) ficam separados entre batches. O file-analyzer só vê arestas dentro do batch com confiança; arestas `calls`/`related`/`inherits`/`implements` entre módulos são descartadas nas fronteiras de batch.

O `recover_imports_from_scan` existente em `merge-batch-graphs.py:913` é uma rede de segurança determinística para arestas `imports` — mas não consegue recuperar arestas semânticas (calls / related / inherits / implements). Essas são perdidas.

---

## Objetivos

- Eliminar "Batch X failed (output limit)" de runs do `/understand` no Bedrock OPUS para projetos até 500 arquivos.
- Melhorar cobertura de arestas semânticas cross-batch substituindo batching baseado em contagem por detecção de comunidades Louvain no grafo de imports.
- Manter paridade de cobertura de arestas `imports` (sem regressão na rede de segurança existente).
- Permanecer dentro de um PR — adiar refactors mais amplos para follow-ups (Seção "Out of scope").

## Não-objetivos

- Refatorar o uso de tree-sitter da Fase 1 / 2 para deduplicar a extração por batch.
- Adicionar resumos de arquivo gerados por LLM ao neighborMap.
- Auto-tunar thresholds de saída por provider.

---

## Arquitetura

Pipeline antes:

```
Phase 1   project-scanner          → scan-result.json (files + importMap)
Phase 2   file-analyzer (×N concur) → batch-<i>.json (one per batch; SKILL.md prose batching)
Phase 2末 merge-batch-graphs.py    → assembled-graph.json
```

Pipeline depois:

```
Phase 1   project-scanner          → scan-result.json (unchanged)
Phase 1.5 compute-batches.mjs      → batches.json (NEW — semantic batching + neighborMap)
Phase 2   file-analyzer (×N concur) → batch-<i>.json (single) OR batch-<i>-part-<k>.json (split)
Phase 2末 merge-batch-graphs.py    → assembled-graph.json (verified, no code change)
```

**Responsabilidade única da Fase 1.5:** decisão de topologia + construção do neighborMap. Algoritmo puro — lê `scan-result.json`, escreve `batches.json`, sem chamadas LLM.

**Mudanças na Fase 2:** SKILL.md para de fazer batching em prosa; itera `batches.json` e despacha um file-analyzer por batch.

**Mudanças no file-analyzer:** consome o neighborMap; auto-verifica o tamanho da saída antes de escrever; faz split em `batch-<i>-part-<k>.json` quando acima dos thresholds.

**merge-batch-graphs.py:** sem mudanças de código — o glob `batch-*.json` e a regex de sort key já aceitam nomenclatura multi-part. Fixture de teste e melhoria de relatório no stderr adicionados.

---

## Componente 1 — `compute-batches.mjs`

**Localização:** `understand-anything-plugin/skills/understand/compute-batches.mjs`

**Invocação:** `node <SKILL_DIR>/compute-batches.mjs $PROJECT_ROOT [--changed-files=<path>]`

**Input:** `$PROJECT_ROOT/.understand-anything/intermediate/scan-result.json`

**Output:** `$PROJECT_ROOT/.understand-anything/intermediate/batches.json`

### Dependências

Adicionadas a `understand-anything-plugin/package.json`:

- `graphology` (~10KB)
- `graphology-communities-louvain` (~30KB)

Reusa o `TreeSitterPlugin` e `PluginRegistry` de `@understand-anything/core` (já importados por `extract-structure.mjs`).

### Algoritmo

```
1. Load scan-result.json.

2. Partition files by fileCategory:
   - codeFiles = files where fileCategory === "code"
   - nonCodeFiles = the rest

3. Code batching (Louvain on import graph):
   a. Build undirected graph: nodes = codeFiles, edges = importMap relations
      (weight=1, undirected so import and imported-by both count).
   b. Run graphology-communities-louvain → community assignment per file.
   c. For any community with size > 35 (max): split via edge-betweenness greedy
      cut (or simpler weakly-connected-component partition) until each
      sub-community ≤ 35. Log warning per split.
      (Whether this branch fires is decided by the implementation prototype
      step — see "Prototype-first implementation" below.)
   d. Communities with size < 5 are kept as-is. Wasted dispatches are
      bounded by the 5-concurrent cap, and the alternative ("merge small")
      adds edge cases without proportional value.

4. Non-code batching (hardcoded heuristics, moved from SKILL.md prose):
   - Group A: For each directory containing a `Dockerfile`, bundle that
     directory's `Dockerfile` + any `docker-compose.*` + any
     `.dockerignore` → one batch per such directory (so multi-service
     repos with several Dockerfiles get one batch per service).
   - Group B: `.github/workflows/*.yml` files → one batch.
   - Group C: `.gitlab-ci.yml` + files under `.circleci/` → one batch.
   - Group D: SQL files under any `migrations/` or `migration/` directory,
     sorted by filename → one batch per directory.
   - Group E: All other non-code files grouped by their immediate parent
     directory, max 20 per batch.

5. Assign batchIndex: code communities first (1..N), non-code groups
   second (N+1..M).

6. Exports extraction:
   - For each code file, run TreeSitterPlugin.extract() and collect
     top-level exports (function names, class names, exported const names).
   - Per-file failures: catch, set exports = [], emit warning.
   - Non-code files: exports = [].

7. Construct neighborMap (1-hop):
   For each file F in batch B:
     neighborMap[F.path] = [
       { path: G.path, batchIndex: G.batch, symbols: G.exports }
       for G in importMap[F.path] ∪ reverseImportMap[F.path]
       where G.batch ≠ B
     ]
   If neighborMap[F.path].length > 50, truncate to top 50 by neighbor
   degree (highest-imported neighbors kept), emit warning.

8. Construct batchImportData:
   For each batch B:
     batchImportData[F.path] = importMap[F.path]  for F in B.files

9. Write batches.json.

Fallback (script-internal): If steps 3a-3c throw, catch → emit warning
→ assign batches by alphabetical chunking (12 files per code batch).
Steps 4, 6, 7, 8 still run normally. Set `algorithm: "count-fallback"`
in the output.
```

### Implementação Louvain

Usar o algoritmo default modularity-greedy do `graphology-communities-louvain`:

```js
import Graph from 'graphology';
import louvain from 'graphology-communities-louvain';

const graph = new Graph({ type: 'undirected' });
for (const file of codeFiles) graph.addNode(file.path);
for (const [src, targets] of Object.entries(importMap)) {
  for (const tgt of targets) {
    if (graph.hasNode(src) && graph.hasNode(tgt) && !graph.hasEdge(src, tgt)) {
      graph.addEdge(src, tgt);
    }
  }
}
const communities = louvain(graph); // { nodeId: communityId }
```

### Schema de saída (`batches.json`)

```json
{
  "schemaVersion": 1,
  "algorithm": "louvain",
  "totalFiles": 100,
  "totalBatches": 7,
  "batches": [
    {
      "batchIndex": 1,
      "files": [
        { "path": "src/auth/login.ts", "language": "typescript",
          "sizeLines": 120, "fileCategory": "code" }
      ],
      "batchImportData": {
        "src/auth/login.ts": ["src/auth/session.ts", "src/db/users.ts"]
      },
      "neighborMap": {
        "src/auth/login.ts": [
          { "path": "src/db/users.ts", "batchIndex": 3,
            "symbols": ["User", "findById", "createUser"] }
        ]
      }
    }
  ]
}
```

`algorithm` é `"louvain"` no happy path, `"count-fallback"` quando o branch Louvain crashou.

### Modo `--changed-files`

Quando invocado com `--changed-files=<path>`, o script:

- Carrega paths de arquivo de `<path>` (um por linha).
- Ainda constrói o grafo de imports completo do projeto (para construção precisa do neighborMap).
- Só emite batches contendo arquivos alterados.
- Entradas do neighborMap referenciam arquivos não-alterados com seu batchIndex do re-run Louvain determinístico de grafo completo. A seed é fixa para que a atribuição seja reprodutível entre invocações incrementais.

### Implementação prototype-first

Antes de escrever o script completo, construir um esqueleto mínimo:

1. Carregar `scan-result.json` do diretório `.understand-anything/` deste repo (se ausente, gerar via `/understand --full`).
2. Rodar apenas Louvain — sem enforcement de tamanho, sem neighborMap.
3. Imprimir a distribuição de tamanho de comunidades.
4. Decidir: as comunidades do mundo real se agrupam em [5, 35]? Se sim, o branch de enforcement de tamanho pode ser desnecessário ou trivialmente defensivo. Se não, implementar o split por edge-betweenness.

Isso condiciona o código mais especulativo (enforcement de tamanho) a observação empírica em vez de design upfront.

---

## Componente 2 — Mudanças em `skills/understand/SKILL.md`

### Adicionar — seção Fase 1.5 (após Fase 1)

```markdown
## Phase 1.5 — BATCH

Report: `[Phase 1.5/7] Computing semantic batches...`

Run the bundled batching script:
\`\`\`bash
node <SKILL_DIR>/compute-batches.mjs $PROJECT_ROOT
\`\`\`

Reads `.understand-anything/intermediate/scan-result.json`, writes
`.understand-anything/intermediate/batches.json`.

Capture stderr. Append any line starting with `Warning:` to
$PHASE_WARNINGS for the final report.

If the script exits non-zero, the failure is hard — relay the full
stderr to the user as a Phase 1.5 failure. Do not attempt to recover;
the script's internal fallback (count-based) already handles recoverable
issues. A non-zero exit means a fundamental problem (missing input file,
malformed JSON, etc.).
```

### Substituir — seção Fase 2 ANALYZE (SKILL.md atual:280-332)

Deletar a prosa existente "Batch the file list from Phase 1 into groups of 20-30 files each", a prosa de agrupamento não-código (agora em compute-batches) e a prosa de construção de `batchImportData` no momento do dispatch (agora fornecida em batches.json). Substituir por:

```markdown
## Phase 2 — ANALYZE

### Full analysis path

Load `.understand-anything/intermediate/batches.json` (produced by
Phase 1.5). Iterate the `batches[]` array.

Report: `[Phase 2/7] Analyzing files — <totalFiles> files in
<totalBatches> batches (up to 5 concurrent)...`

For each batch, dispatch a `file-analyzer` subagent (up to 5
concurrent). Dispatch prompt template:

> Analyze these files and produce GraphNode and GraphEdge objects.
> Project root: `$PROJECT_ROOT`
> Project: `<projectName>`
> Languages: `<languages>`
> Batch: `<batchIndex>/<totalBatches>`
> Skill directory: `<SKILL_DIR>`
> Output: write to
> `$PROJECT_ROOT/.understand-anything/intermediate/batch-<batchIndex>.json`
> (single-file mode) OR `batch-<batchIndex>-part-<k>.json` (split mode,
> per Step B of your output protocol).
>
> Pre-resolved import data (use directly — do NOT re-resolve from source):
> \`\`\`json
> <batchImportData JSON inline from batches.json[i].batchImportData>
> \`\`\`
>
> Cross-batch neighbors with their exported symbols (confidence boost
> for cross-batch edges):
> \`\`\`json
> <neighborMap JSON inline from batches.json[i].neighborMap>
> \`\`\`
>
> Files to analyze:
> 1. `<path>` (<sizeLines> lines, language: `<language>`,
>    fileCategory: `<fileCategory>`)
> ...

$LANGUAGE_DIRECTIVE

After ALL batches complete, run the merge-and-normalize script:
\`\`\`bash
python <SKILL_DIR>/merge-batch-graphs.py $PROJECT_ROOT
\`\`\`

(Rest of Phase 2 unchanged.)
```

### Substituir — caminho de atualização incremental (SKILL.md atual:355-366)

```markdown
### Incremental update path

Run compute-batches.mjs with `--changed-files=<path>`, where `<path>`
is a temp file listing changed file paths (one per line). The script
reuses the full project's import graph for neighborMap computation
but only emits batches containing changed files. Dispatch file-analyzer
subagents per the same template as the full path.
```

### Orçamento de linhas

Prosa de contexto LLM líquida adicionada: Fase 1.5 (~12 linhas) + clarificações no template da Fase 2 (~5 linhas) − prosa de batching removida (~15 linhas) − prosa de construção de batchImportData removida (~6 linhas) ≈ **−4 linhas**.

---

## Componente 3 — Mudanças em `agents/file-analyzer.md`

### Adicionar — seção de contexto cross-batch

Inserir depois de "Step 1: Input file construction":

```markdown
### Cross-batch context (neighborMap)

Your dispatch prompt includes a `neighborMap` — for each file in your
batch, it lists project-internal neighbors in OTHER batches (files that
import yours or that you import), with their exported symbols.

Use neighborMap as a confidence boost for cross-batch edges (`calls`,
`related`, `inherits`, `implements` to nodes outside your batch):

- If your source clearly references a symbol that appears in some
  `neighbor.symbols`, emit the edge to
  `function:<neighbor.path>:<symbol>` or
  `class:<neighbor.path>:<symbol>` with confidence.
- If your source references a cross-batch symbol that is NOT in
  neighborMap (the project-scanner may not have extracted it), you may
  still emit the edge if you saw it explicitly in the imported file's
  surface — but prefer matching neighborMap symbols when available.
- Imports continue to use `batchImportData` (fully resolved), not
  neighborMap.

The merge script's dangling-edge dropper is the safety net for
genuinely unresolvable targets.
```

### Substituir — seção Writing Results (file-analyzer.md atual:467-475)

```markdown
## Writing Results — single or multi-part

**Step A — Compute totals.**
\`\`\`
nodeCount = nodes.length
edgeCount = edges.length
\`\`\`

**Step B — Decide split.**
- If `nodeCount ≤ 60` AND `edgeCount ≤ 120`: write ONE file to
  `.understand-anything/intermediate/batch-<batchIndex>.json`. Done.
  Skip to Step E.
- Otherwise: `parts = ceil(max(nodeCount / 60, edgeCount / 120))`.

**Step C — Partition.**
Sort files in your batch alphabetically by path. Chunk them sequentially
into `parts` groups of size `ceil(N / parts)`. For each part:
- All nodes whose `filePath` is in this part's files (for non-file
  nodes like `module`/`concept`, use the file they belong to).
- All edges whose `source` is in this part's nodes (target may be
  anywhere — same part, different part of same batch, different batch).

**Step D — Write each part.**
Write part `k` (1-indexed) to
`.understand-anything/intermediate/batch-<batchIndex>-part-<k>.json`.
Each part is a valid GraphFragment: `{ "nodes": [...], "edges": [...] }`.

**Step E — Self-validate.**
For each file written, verify:
- Valid JSON.
- `nodes` array exists and is well-formed.
- For every edge: `source` and `target` both appear as either (a) a
  node `id` in this part's nodes, OR (b) a `file:<path>` reference
  where `<path>` is in `neighborMap` or `batchImportData`, OR (c) a
  `function:<path>:<symbol>` / `class:<path>:<symbol>` reference where
  `<symbol>` is in some `neighbor.symbols`.

If validation fails on a part, do NOT silently rebuild. Respond with
an explicit error stating which part failed, which edge(s) failed
validation, and why. The dispatching session can then retry.

**Step F — Respond.**
Respond with ONLY a brief text summary: parts written (1 or more),
total nodes/edges across all parts, any files skipped. Do NOT include
JSON content in the response.
```

### Racional do threshold

`60 nós / 120 arestas por part` deriva de:

- JSON de nó de arquivo serializado ≈ 150-300 chars; function/class ≈ 80-150 chars; aresta ≈ 100-150 chars.
- 60 nós + 120 arestas ≈ 25-35KB de JSON ≈ 7000-9000 tokens de saída (tokenização de JSON é densa).
- Default `max_tokens` do Bedrock OPUS 4096-8192 → margem de segurança de ~10%.

Essas constantes vivem como prosa em file-analyzer.md por enquanto. Auto-tuning por provider é adiado para follow-up.

---

## Componente 4 — `merge-batch-graphs.py` (apenas verificação)

### Compatibilidade confirmada

O glob e a sort key existentes já lidam com arquivos multi-part de forma transparente:

- `intermediate_dir.glob("batch-*.json")` casa `batch-3-part-1.json`.
- `re.search(r"batch-(\d+)", p.stem)` extrai `3` de `batch-3-part-1`, dando a mesma sort key que `batch-3.json`. O `sorted` do Python é estável, então parts carregam na ordem de desempate lexicográfica.
- `merge_and_normalize` percorre `all_nodes.extend(...)` / `all_edges.extend(...)`; ordem de carga não afeta correção de dedup.
- `recover_imports_from_scan` opera no grafo mesclado — transparente a inputs multi-part.
- `link_tests` opera no pool de nós mesclados — transparente.

Sem mudança de código necessária para correção.

### Adicionar — consciência multi-part no relatório do stderr

`merge-batch-graphs.py:1026` atualmente imprime `Found {N} batch files:`. Melhorar:

```python
from collections import defaultdict
by_batch = defaultdict(list)
for f in batch_files:
    m = re.match(r"batch-(\d+)(?:-part-(\d+))?\.json", f.name)
    if m:
        by_batch[int(m.group(1))].append(f.name)

logical_count = len(by_batch)
multi_part = sum(1 for files in by_batch.values() if len(files) > 1)
print(
    f"Found {len(batch_files)} batch files "
    f"({logical_count} logical batches, {multi_part} multi-part)",
    file=sys.stderr,
)
```

### Adicionar — aviso de part faltando

Após o agrupamento, detectar batches lógicos com números de part não-contíguos (ex.: parts `{2, 3}` presentes mas `1` faltando) e emitir:

```
Warning: merge: batch <i> has parts {<set>} but missing part {<missing>}
  — possible truncated write — affected nodes/edges may be lost
```

---

## Modos de falha e observabilidade

| Ponto de falha | Comportamento | Rede de segurança | Texto de warning obrigatório |
|---|---|---|---|
| Biblioteca Louvain lança | exceção | Script-interno: catch → fallback baseado em contagem (12 arquivos/batch); neighborMap ainda construído | `Warning: compute-batches: Louvain failed (<msg>) — falling back to count-based grouping (12 files/batch) — module semantic boundaries lost` |
| Falha por arquivo na extração de exports do tree-sitter | exports vazios | symbols=[] no neighborMap | `Warning: compute-batches: exports extraction failed for <path> (<msg>) — symbols=[] in neighborMap — cross-batch edges to this file limited to file-level` |
| Louvain produz comunidade superdimensionada | tamanho > 35 | Split por edge-betweenness | `Warning: compute-batches: community size <N> > max 35 — splitting via edge-betweenness — modularity may decrease` |
| compute-batches crash total | exit não-zero, sem batches.json | SKILL.md mostra stderr completo ao usuário; sem fallback na Fase 2 | (erro próprio do script para stderr; SKILL.md relaya verbatim) |
| Truncamento do neighborMap | > 50 vizinhos | Top-50 por grau mantido | `Warning: compute-batches: neighborMap for <path> truncated from <N> to top 50 (by neighbor degree)` |
| JSON de part do file-analyzer malformado | `load_batch` pula | `load_batch:139` existente avisa e pula | (existente — verificar que o warning não é engolido) |
| Part faltando em batch multi-part | gap em parts | merge detecta e avisa | `Warning: merge: batch <i> has parts {<set>} but missing part {<missing>} — possible truncated write — affected nodes/edges may be lost` |
| Arestas penduradas do file-analyzer | source/target ausente | merge descarta, adiciona a `unfixable` (existente) | (existente) |
| Dispatch do file-analyzer falha | erro de subagente | mecanismo de retry-once existente | (existente) |

### Invariante de observabilidade

Todo fallback / degrade / drop DEVE:

1. Escrever uma linha no stderr no formato `Warning: <component>: <what happened> — <why> — <impact>`.
2. Bubble up para `$PHASE_WARNINGS` (mecanismo existente do SKILL.md) → relatório final da Fase 7 voltado ao usuário.
3. Nunca usar `catch {}` / `except: pass` silencioso. Code review trata isso como bloqueador.

### Invariantes

1. **scan-result.json é fonte da verdade.** Qualquer mudança de batching/topologia preserva o importMap; `recover_imports_from_scan` sempre restaura arestas `imports`.
2. **Dangling-edge dropper é defesa final.** Nenhuma aresta gerada por batch pode conectar a um nó inexistente no grafo montado.
3. **Sem fallback silencioso.** `batches.json` faltando → falha barulhenta. Fallback interno do compute-batches → warning barulhento que sobe ao usuário.

---

## Testes

### Testes unitários — `compute-batches.mjs`

Novo arquivo: `understand-anything-plugin/skills/understand/test_compute_batches.test.mjs` (Vitest).

Casos obrigatórios:

- **Louvain básico:** 3 cliques disjuntos → 3 batches.
- **importMap vazio:** arquivos independentes → batches count-fallback por chunking alfabético.
- **Comunidade superdimensionada:** grafo completo de 50 nós → split disparado, todos os sub-batches ≤ 35.
- **Agrupamento não-código A:** `Dockerfile` + `docker-compose.yml` + `.dockerignore` siblings → um batch por cluster de diretório.
- **Agrupamento não-código B:** `.github/workflows/*.yml` → um batch.
- **Agrupamento não-código C:** migrations SQL sob `migrations/` → um batch por diretório.
- **Mix código + não-código:** batchIndex de não-código segue batches de código.
- **Corretude do neighborMap:** arquivo A importa arquivo B entre batches → `neighborMap[A]` contém `{path: B, batchIndex: B's, symbols: B's exports}`.
- **neighborMap exclui mesmo-batch:** A e C no mesmo batch → `neighborMap[A]` não contém C.
- **Tolerância a falha de exports:** mockar TreeSitter para lançar em um arquivo → `exports = []` para aquele arquivo, outros não afetados.
- **`--changed-files`:** subset de input → output contém só batches com arquivos alterados; neighborMap pode referenciar arquivos não-alterados.
- **Triggers de fallback:** mockar throw do Louvain → campo `algorithm` = `"count-fallback"`, warning no stderr.
- **Assertion de warning por fallback:** para cada um de {crash do Louvain, falha de exports, split de oversize, truncamento do neighborMap}, assertar que a string exata de warning aparece no stderr.

### Testes unitários — `merge-batch-graphs.py`

Nova classe de teste `TestMultiPart` em `test_merge_batch_graphs.py`:

- Duas parts de um batch lógico: `batch-1-part-1.json` + `batch-1-part-2.json` → montado contém todos os nós/arestas dos dois.
- Três parts de um batch lógico.
- Arestas cross-part: aresta com source na part-1, nó target na part-2 → conectada após merge.
- part-1 malformada + part-2 válida: part-1 pulada com warning, conteúdo da part-2 presente.
- Mix de inputs single-batch e multi-part.
- Detecção de part faltando: `batch-1-part-2.json` + `batch-1-part-3.json` (sem part-1) → warning emitido com texto exato.
- Formato do stderr: assertar que `"X logical batches, Y multi-part"` aparece.

### Integração — gate de aceite do PR (manual)

Documentado no Test plan do PR:

- [ ] `pnpm install` (graphology instala limpo).
- [ ] `pnpm --filter @understand-anything/core build`.
- [ ] Rodar `/understand --full` neste repo (o próprio Understand-Anything):
  - `batches.json` gerado; sanity-check da distribuição de tamanho de comunidades (mix de batches pequenos e médios).
  - Pelo menos um batch produz saída multi-part.
  - Contagens de nó/aresta de `assembled-graph.json` dentro do range esperado vs main atual.
  - Dashboard renderiza normalmente.
  - Relatório final da Fase 7 inclui qualquer `$PHASE_WARNINGS` do compute-batches (verificar visualmente que os warnings chegam à saída voltada ao usuário, não só ao stderr).
- [ ] Rodar em um repo de ~100 arquivos batendo com o cenário de ayushghosh; confirmar que não há erros de "output limit".
- [ ] Rodar em um repo pequeno de 5-10 arquivos: caminho de fallback (tudo um batch só) funciona corretamente.

### Não testado

- Corretude do algoritmo Louvain (confiar nos testes próprios do `graphology-communities-louvain`).
- Benchmarks de performance (sub-segundo em 100-500 arquivos é empírico; não gated).
- Variações de teto de saída em múltiplos providers de LLM (thresholds são conservadores para Bedrock OPUS; Anthropic first-party é mais permissivo).

---

## Fora de escopo (rastreado para follow-up)

### Deduplicação do tree-sitter

Atualmente Fase 1 (project-scanner), Fase 1.5 (compute-batches) e Fase 2 (file-analyzer por batch) cada uma roda tree-sitter independentemente. Consolidar em uma única extração de estrutura na Fase 1.5 simplificaria o file-analyzer e pouparia tempo em projetos grandes. Adiar porque requer reorganizar significativamente o protocolo do file-analyzer.

### Resumos LLM no neighborMap

Adicionar resumos de uma frase por arquivo ao neighborMap permitiria que o file-analyzer emitisse arestas `related` entre batches com justificativa semântica. Requer um novo agente leve de summary-pass; adiar até a dedup do tree-sitter aterrissar (Fase 1.5 já terá a estrutura completa → mais barato adicionar).

### Thresholds adaptativos

`60 nós / 120 arestas` são conservadores para Bedrock OPUS. Anthropic first-party suporta tetos de saída bem maiores. Adicionar um CLI `--output-cap=<N>` ao compute-batches e propagar ao file-analyzer destravaria parts maiores em backends permissivos. Rastrear contagens reais de part antes de implementar.

### Auditoria de arestas cross-batch

Uma auditoria pós-merge comparando arestas sugeridas pelo neighborMap vs arestas efetivamente emitidas mostraria gaps. Espelhar o padrão existente `recover_imports_from_scan`. Requer preservar `batches.json` para consumo no merge-time.

### Tratamento de monorepo multi-linguagem

Repos multi-linguagem (TS + Python) tendem a se dividir naturalmente via Louvain (sem imports cross-linguagem). Arquivos-ponte (OpenAPI, protobuf) podem criar comunidades estranhas. Endereçar apenas se reports reais aparecerem.

---

## Ordem de implementação

1. **Protótipo:** esqueleto mínimo de `compute-batches.mjs` — carregar scan-result.json, rodar Louvain, imprimir tamanhos de comunidade. Rodar contra o `scan-result.json` deste repo (gerar se faltar via `/understand --full`). Decidir se o branch de enforcement de tamanho é necessário; se sim, escolher entre split por edge-betweenness e weakly-connected-component.
2. Adicionar extração de exports (reusar TreeSitterPlugin).
3. Adicionar construção do neighborMap + passthrough de batchImportData.
4. Adicionar heurísticas de agrupamento não-código (Grupos A-E).
5. Adicionar caminho de fallback + emissões de warning para cada modo de falha listado na tabela de Modos de falha.
6. Escrever testes unitários para compute-batches (conforme seção Testes), incluindo assertions de texto de warning.
7. Modificar `agents/file-analyzer.md` — adicionar seção de contexto cross-batch, substituir Writing Results.
8. Modificar `skills/understand/SKILL.md` — adicionar Fase 1.5, substituir prosa de batching da Fase 2 ANALYZE, substituir caminho incremental.
9. Adicionar relatório multi-part no stderr + warning de part faltando em `merge-batch-graphs.py`.
10. Escrever testes unitários para tratamento multi-part de `merge-batch-graphs.py`.
11. Adicionar `graphology` + `graphology-communities-louvain` a `understand-anything-plugin/package.json`.
12. Rodar gate de aceite de integração.
13. Bumpar versão em todos os cinco arquivos `package.json` / `plugin.json` conforme a regra de versionamento do CLAUDE.md do projeto.
