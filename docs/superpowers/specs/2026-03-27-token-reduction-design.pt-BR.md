# Design de Redução de Tokens

> 🇬🇧 Versão original em inglês: [2026-03-27-token-reduction-design.md](./2026-03-27-token-reduction-design.md)

**Data:** 2026-03-27
**Status:** Rascunho
**Objetivo:** Reduzir o custo total de tokens do `/understand` em ~85-90% em bases de código grandes (200+ arquivos)

---

## Problema

Para bases de código grandes, o pipeline `/understand` gasta a grande maioria dos seus tokens em **injeção repetida de contexto**. Os mesmos dados são enviados independentemente para cada subagente, mesmo quando esses dados poderiam ser computados uma vez e compartilhados.

### Quebra do custo de tokens (projeto TypeScript+React de 500 arquivos, baseline)

| Fonte | Fase | Tokens (input) | % do total |
|---|---|---|---|
| lista `allProjectFiles` × 67 batches | Fase 2 | ~167,000 | ~50% |
| `file-analyzer-prompt.md` × 67 batches | Fase 2 | ~134,000 | ~40% |
| Addendums de linguagem/framework × 67 batches | Fase 2 | ~68,000 | ~20% |
| Payload do tour builder (todos os nós + arestas) | Fase 5 | ~80,000 | ~24% |
| Graph reviewer (grafo montado + inventário) | Fase 6 | ~58,000 | ~17% |
| Payload do architecture analyzer | Fase 4 | ~22,000 | ~7% |
| **Total** | | **~529,000** | |

A causa raiz: **A Fase 2 roda 67 batches (com 5-10 arquivos cada), e cada batch recebe a lista completa de 500 arquivos para resolução de imports.** Só a lista de arquivos custa ~2,500 tokens × 67 repetições = 167,000 tokens no input, fazendo trabalho totalmente redundante entre batches.

---

## Objetivos

- Reduzir tokens de input totais em 85%+ em um projeto de 500 arquivos
- Sem degradação na qualidade do grafo para projetos padrão
- Preservar as flags `--full` / incremental / de escopo
- Manter compatibilidade retroativa com o schema de output `knowledge-graph.json` existente

---

## Mudanças

Cinco mudanças compõem a abordagem completa (C1–C5). Cada uma é independente e pode ser entregue separadamente, mas as cinco são necessárias para a redução total.

---

### C1 — Pré-resolver imports no project scanner

**Causa raiz endereçada:** `allProjectFiles` (a lista de arquivos inteira) é injetada em cada batch do file-analyzer somente para que o script de extração de cada batch consiga resolver imports relativos. Isso é redundante: a lista completa de arquivos está disponível na Fase 1, e a resolução de imports é determinística. Deveria acontecer uma vez, não 67 vezes.

**Mudança:** Estender o script scanner da Fase 1 para também parsear instruções de import de cada arquivo-fonte e resolver imports relativos contra a lista de arquivos descoberta. Os resultados resolvidos são gravados em `scan-result.json` como um novo campo `importMap`. Batches do file-analyzer então recebem apenas os imports pré-resolvidos do seu próprio batch — não a lista de arquivos completa.

#### Adição na saída do scanner

`scan-result.json` ganha:

```json
{
  "importMap": {
    "src/index.ts": ["src/utils.ts", "src/config.ts"],
    "src/utils.ts": [],
    "src/components/App.tsx": ["src/hooks/useAuth.ts", "src/store/index.ts"]
  }
}
```

- Chaves são caminhos relativos ao projeto (batendo com `files[*].path`)
- Valores são apenas caminhos resolvidos relativos ao projeto (imports externos/não-resolvíveis são omitidos)
- Imports externos (`node_modules`, caminhos não-resolvíveis) são totalmente excluídos do mapa

#### Adições ao script do scanner (Passo 8 da Fase 1)

Após os 7 passos existentes, o script do scanner adiciona um novo passo:

```
Step 8 — Import Resolution

For each file in the discovered source list:
  1. Read the file content
  2. Extract import statements (language-specific patterns per Step 3's language detection):
     - TypeScript/JavaScript: `import ... from '...'`, `require('...')`
     - Python: `import ...`, `from ... import ...`
     - Go: `import "..."` blocks
     - Rust: `use ...` statements
     - Java/Kotlin: `import ...` statements
     - Ruby: `require`, `require_relative`
  3. For each relative import (starts with `./` or `../`):
     a. Compute the resolved path from the current file's directory
     b. Normalize to project-relative format
     c. Try common extension variants if the import has no extension:
        `.ts`, `.tsx`, `.js`, `.jsx`, `/index.ts`, `/index.js`, `/index.tsx`
     d. If any variant exists in the discovered file list, record it; otherwise skip
  4. For absolute imports (no `.` prefix): skip (external package)

Output the full importMap in the JSON result.
```

#### Mudança no schema de input do file-analyzer

**Antes:**
```json
{
  "projectRoot": "/path/to/project",
  "allProjectFiles": ["src/index.ts", "src/utils.ts", "...500 paths..."],
  "batchFiles": [
    {"path": "src/index.ts", "language": "typescript", "sizeLines": 150}
  ]
}
```

**Depois:**
```json
{
  "projectRoot": "/path/to/project",
  "batchFiles": [
    {"path": "src/index.ts", "language": "typescript", "sizeLines": 150}
  ],
  "batchImportData": {
    "src/index.ts": ["src/utils.ts", "src/config.ts"],
    "src/components/App.tsx": ["src/hooks/useAuth.ts"]
  }
}
```

`allProjectFiles` é removido por completo. `batchImportData` contém apenas os imports pré-resolvidos para os arquivos deste batch (fatiados de `importMap` pelo orquestrador).

#### Mudança no script de extração do file-analyzer

O script de extração não realiza mais resolução de imports. Ele:
- Ainda extrai: funções, classes, exports, métricas (inalterado)
- Para imports: lê `batchImportData[file.path]` do JSON de input — sem cross-reference necessário
- O array `imports` em cada resultado de arquivo vira: `batchImportData[file.path]` mapeado para objetos de aresta de import com `resolvedPath` já populado, `isExternal: false`

#### Mudança no SKILL.md Fase 2

Remover a injeção de `allProjectFiles` do prompt de dispatch do batch. Substituir por um slice `batchImportData` por batch:

```
For each batch, slice importData from the importMap read in Phase 1:
batchImportData = { [file.path]: importMap[file.path] ?? [] }
  for each file in this batch
```

#### Estimativa de economia de tokens

| | Batches | Tokens/batch | Total |
|---|---|---|---|
| Antes | 67 | ~2,500 (lista de arquivos) | ~167,500 |
| Depois (C1 sozinho) | 67 | ~200 (batch importData) | ~13,400 |
| **Economia** | | | **~154,100** |

---

### C2 — Aumentar tamanho de batch de 5-10 para 20-30 arquivos

**Causa raiz endereçada:** Cada batch incorre o custo completo de `file-analyzer-prompt.md` (~2,000 tokens) mais o overhead do dispatch de batch. Com 67 batches, isso soma mesmo sem `allProjectFiles`. Batches menos numerosos e maiores reduzem diretamente essa repetição.

**Mudança:** No SKILL.md Fase 2, mudar a orientação de tamanho de batch:

- **Antes:** "Batch the file list from Phase 1 into groups of **5-10 files each**"
- **Depois:** "Batch the file list from Phase 1 into groups of **20-30 files each** (aim for ~25 per batch)"

Também atualizar o limite de concorrência de 3 para **5** batches concorrentes. Menos batches totais significa que podemos pagar por mais paralelismo sem sobrecarregar o sistema.

#### Trade-offs

| | Batches menores (atual) | Batches maiores (novo) |
|---|---|---|
| Arquivos por batch | 5-10 | 20-30 |
| Total de batches (500 arquivos) | ~67 | ~20 |
| Repetição de prompt | 67× | 20× |
| Risco de qualidade | Menor (focado) | Levemente maior (mais arquivos por subagente) |
| Concorrência | 3 | 5 |

Risco de qualidade é baixo: cada subagente ainda opera em grupos de arquivo distintos e não-sobrepostos. O script de extração é determinístico independentemente do tamanho de batch. Análise semântica (resumos, tags) pode ficar marginalmente menos focada, mas a diferença de qualidade é desprezível na prática para arquivos bem estruturados.

#### Estimativa de economia de tokens (combinada com C1)

| | Batches | Tokens/batch (prompt) | Total |
|---|---|---|---|
| Antes (C1 só) | 67 | ~2,000 | ~134,000 |
| Depois (C1+C2) | 20 | ~2,000 | ~40,000 |
| **Economia de C2** | | | **~94,000** |

C1+C2 combinados eliminam ~248,000 tokens da Fase 2 (de ~301,500 para ~53,500, uma redução de ~82% da Fase 2).

---

### C3 — Remover addendums de linguagem/framework dos batches do file-analyzer

**Causa raiz endereçada:** `languages/typescript.md` (~600 tokens) e `frameworks/react.md` (~700 tokens) são lidos e injetados em cada prompt de batch do file-analyzer. Para um projeto TypeScript+React com 20 batches (após C2), isso custa 20 × 1,300 = 26,000 tokens adicionais — e o modelo já tem conhecimento profundo dessas linguagens vindo do treinamento.

**Mudança:** Parar de injetar arquivos de addendum nos prompts de batch da Fase 2 por completo. Os addendums permanecem injetados na Fase 4 (architecture analyzer), onde há apenas **uma** chamada de subagente, tornando o custo aceitável.

Em vez disso, adicionar uma seção compacta de "Language and Framework Hints" diretamente em `file-analyzer-prompt.md`. Essa seção é uma adição única e destilada (~150 tokens no total) que captura os padrões mais úteis de todos os addendums em uma tabela de lookup concisa.

#### Nova seção em `file-analyzer-prompt.md` (substitui injeção de addendum)

```markdown
## Language and Framework Quick Reference

Use these hints to improve tag and edge accuracy. These supplement your training knowledge.

| Signal | Tag(s) | Note |
|---|---|---|
| File in `hooks/`, exports function starting with `use` | `hook`, `service` | React custom hook |
| File in `contexts/`, exports a Provider | `service`, `state` | React context |
| File in `pages/` or `views/` | `ui`, `routing` | Page-level component |
| File in `store/`, `slices/`, `reducers/` | `state` | State management |
| File in `services/`, `api/` | `service` | Data-fetching / API client |
| `__init__.py` with re-exports | `entry-point`, `barrel` | Python package root |
| `manage.py` at project root | `entry-point` | Django management entry |
| File named `mod.rs` | `barrel` | Rust module barrel |
| File named `main.go` in `cmd/` | `entry-point` | Go binary entry |

For React: create `depends_on` edges from components to hooks they call. Create `publishes`/`subscribes` edges for Context provider/consumer patterns.
```

#### Mudança no SKILL.md Fase 2

Remover os passos 2 e 3 do bloco "Build the combined prompt template":
- **Remover:** Passo 2 (Injeção de contexto de linguagem — ler `./languages/<language-id>.md` por linguagem detectada)
- **Remover:** Passo 3 (Injeção de addendum de framework — ler `./frameworks/<framework-id>.md` por framework detectado)
- **Manter:** Passo 1 (Ler o template base em `./file-analyzer-prompt.md`)

Os passos de injeção de addendum **permanecem inalterados** na Fase 4 (architecture analyzer), já que rodam uma vez.

#### Estimativa de economia de tokens

| | Batches | Tokens de addendum/batch | Total |
|---|---|---|---|
| Antes (após C2) | 20 | ~1,300 (TS+React) | ~26,000 |
| Depois | 20 | ~150 (hints inline) | ~3,000 |
| **Economia** | | | **~23,000** |

---

### C4 — Enxugar payloads das Fases 4 e 5

**Causa raiz endereçada:** A Fase 5 (tour builder) recebe todos os nós (file + function + class) e todas as arestas (imports + contains + calls + exports + ...). Para um projeto de 500 arquivos, isso pode incluir 1,500+ nós e 3,000+ arestas. A maioria desses dados não é necessária para design de tour.

#### Fase 4 (Architecture Analyzer) — corte menor

A Fase 4 já só envia nós do tipo file, o que está correto. Mudança menor: explicitamente remover `languageNotes` de cada objeto de nó no payload (não é útil para atribuição de camada e pode ser verboso). Também remover `name` — é sempre derivável como o basename de `filePath`.

**Antes por nó:** `{id, name, filePath, summary, tags, complexity, languageNotes?}`
**Depois por nó:** `{id, filePath, summary, tags}`

Economia: ~15-20% menos tokens por nó, ~3,000–5,000 tokens no total para a Fase 4.

#### Fase 5 (Tour Builder) — corte grande

Três mudanças no que o orquestrador injeta no subagente do tour-builder:

**1. Apenas nós de arquivo (remover nós de função/classe)**

O tour referencia IDs de nó para wayfinding. Na prática o tour sempre referencia nós `file:` — nós de função e classe são visíveis na sidebar NodeInfo do dashboard quando um arquivo é selecionado, mas o próprio tour navega no nível de arquivo.

- **Antes:** todos os nós (file + function + class) — para 500 arquivos, talvez 1,500+ nós
- **Depois:** apenas nós do tipo file — 500 nós

**2. Formato de nó enxuto**

O script do tour builder só usa IDs, nomes e tipos de nó para computação de grafo. Resumos e tags são usados na Fase 2 (escrita de narrativa pedagógica). Remover campos opcionais pesados do payload injetado:

- **Antes por nó:** `{id, name, filePath, summary, type, tags, complexity, languageNotes?}`
- **Depois por nó:** `{id, name, filePath, summary, type}` (remove tags, complexity, languageNotes)

**3. Arestas enxutas (apenas imports + calls) e camadas enxutas**

A travessia BFS do tour só percorre arestas `imports` e `calls`. `contains`, `exports`, `tested_by`, `depends_on` e outros tipos de aresta não adicionam valor à travessia e inflam o payload.

- **Antes (arestas):** todos os tipos de aresta (~3,000+ arestas incluindo todas as arestas `contains` para nós de função/classe)
- **Depois (arestas):** apenas tipos de aresta `imports` e `calls` (~400–800 arestas para projetos típicos)

Para camadas, o tour builder usa dados de camada apenas para informar o arco narrativo do tour (qual camada introduzir primeiro, segundo, etc.). Não precisa dos arrays `nodeIds` completos — esses podem ser bem grandes.

- **Antes por camada:** `{id, name, description, nodeIds: [...centenas de IDs]}`
- **Depois por camada:** `{id, name, description}` (remove nodeIds)

#### Estimativa de economia de tokens (Fase 5)

| Dado | Antes | Depois |
|---|---|---|
| Contagem de nós | ~1,500 × ~180 chars | ~500 × ~120 chars |
| Tokens de nó | ~67,500 | ~15,000 |
| Contagem de arestas | ~3,000 × ~80 chars | ~600 × ~80 chars |
| Tokens de aresta | ~60,000 | ~12,000 |
| Tokens de camada | ~5,000 | ~500 |
| **Total Fase 5** | **~132,500** | **~27,500** |
| **Economia** | | **~105,000** |

#### Mudanças no SKILL.md

No template de prompt de dispatch da **Fase 4**, atualizar o formato do nó de arquivo:
```
File nodes:
[list of {id, filePath, summary, tags} for all file-type nodes]
```

No template de prompt de dispatch da **Fase 5**, atualizar todas as três specs de payload:
```
Nodes (file nodes only):
[list of {id, name, filePath, summary, type} for all file-type nodes only — do NOT include function or class nodes]

Key edges (imports and calls only):
[list of edges where type is "imports" or "calls" only]

Layers:
[list of {id, name, description} — omit nodeIds]
```

---

### C5 — Colocar o subagente graph-reviewer atrás de `--review`

**Causa raiz endereçada:** O subagente graph-reviewer (Fase 6) lê o grafo montado inteiro (~500 nós, todas as arestas, camadas, tour) e roda uma validação powered by LLM. Porém, sua Fase 1 é inteiramente um script determinístico, e sua Fase 2 é uma simples decisão de threshold: se `issues.length === 0`, aprovar. Não há julgamento de LLM necessário para o caminho feliz.

**Mudança:** Por padrão, pular o subagente graph-reviewer. O orquestrador realiza validação determinística inline usando um script pré-escrito. Apenas quando `--review` é explicitamente passado em `$ARGUMENTS` é que o subagente reviewer LLM completo roda.

#### Caminho padrão (sem `--review`)

Na Fase 6, em vez de despachar o subagente graph-reviewer, o orquestrador:

1. Escreve um script de validação compacto inline (embutido no SKILL.md, ~50 linhas de Node.js):
   - Checa: cada source/target de aresta referencia um ID de nó real
   - Checa: cada nó de arquivo aparece em exatamente uma camada
   - Checa: cada nodeId de step do tour existe
   - Checa: sem IDs de nó duplicados
   - Checa: campos obrigatórios presentes em nós e arestas
2. Roda o script contra `assembled-graph.json`
3. Se `issues.length === 0`: prossegue para a Fase 7 (save)
4. Se `issues.length > 0`: aplica as mesmas correções automatizadas de antes (remover arestas pendentes, preencher defaults), depois salva

Isso é suficiente para runs padrão. O reviewer LLM agrega valor para pegar problemas sutis de qualidade (resumos genéricos, nós órfãos, coerência de steps de tour) — mas esses são nice-to-have, não bloqueantes.

#### Caminho `--review`

Quando `--review` está em `$ARGUMENTS`, o subagente graph-reviewer completo roda como hoje. Sem mudança nesse caminho de código.

#### Estimativa de economia de tokens

| Caminho | Tokens |
|---|---|
| Atual (sempre roda reviewer LLM) | ~58,000 input + ~500 output |
| Padrão (script inline, sem LLM) | ~0 |
| `--review` (inalterado) | ~58,000 (igual ao atual) |
| **Economia para runs padrão** | **~58,500** |

---

## Resumo combinado de economia

| Mudança | Tokens antes | Tokens depois | Economia |
|---|---|---|---|
| C1+C2: import map + consolidação de batch | ~301,500 | ~53,500 | ~248,000 |
| C3: remover addendums dos batches | ~26,000 | ~3,000 | ~23,000 |
| C4: enxugar payloads Fases 4+5 | ~154,500 | ~33,000 | ~121,500 |
| C5: colocar reviewer atrás de flag (caminho padrão) | ~58,500 | ~0 | ~58,500 |
| **Total** | **~540,500** | **~89,500** | **~451,000 (~83%)** |

Estimativas são para um projeto TypeScript+React de 500 arquivos. A economia real escala com o tamanho do projeto — um projeto de 1,000 arquivos veria economia proporcionalmente maior de C1+C2 (mais batches = mais repetição eliminada).

---

## Mudanças de arquivo

| Arquivo | Mudança |
|---|---|
| `skills/understand/project-scanner-prompt.md` | Adiciona Step 8 (resolução de imports); adiciona `importMap` ao schema de saída |
| `skills/understand/file-analyzer-prompt.md` | Substitui `allProjectFiles` por `batchImportData` no schema de input; atualiza o script de extração para usar imports pré-resolvidos; adiciona seção compacta Language/Framework Quick Reference; remove passos de injeção de addendum |
| `skills/understand/SKILL.md` | Fase 1: nota importMap no resultado do scan; Fase 2: remove injeção de addendum (passos 2+3), aumenta tamanho de batch 5-10→20-30, aumenta concorrência 3→5, substitui injeção de `allProjectFiles` por slice de `batchImportData`; Fase 4: formato de nó enxuto no dispatch; Fase 5: apenas nós de arquivo + arestas enxutas + camadas enxutas no dispatch; Fase 6: reviewer condicional — script inline por padrão, flag `--review` para reviewer LLM |
| `skills/understand/architecture-analyzer-prompt.md` | Sem mudança (addendums ainda injetados aqui) |
| `skills/understand/tour-builder-prompt.md` | Atualizar schema de input para refletir apenas nós de arquivo, apenas arestas imports+calls, formato de camada enxuto |
| `skills/understand/graph-reviewer-prompt.md` | Sem mudança (só usado quando flag `--review` é passada) |

---

## Riscos e mitigações

| Risco | Probabilidade | Mitigação |
|---|---|---|
| Resolução de import do scanner perde casos de borda (re-exports complexos, imports dinâmicos) | Média | Logar imports não-resolvidos; file-analyzer ainda usa dados resolvidos e cria arestas só para casamentos confirmados. Imports perdidos = arestas faltando, que é o mesmo comportamento de antes para imports não-resolvíveis |
| Batches maiores (C2) reduzem qualidade de resumo | Baixa | Qualidade do resumo é dirigida pela análise do modelo de arquivos individuais. Tamanho de batch afeta principalmente quantos arquivos compartilham a janela de contexto de um subagente, não qualidade por arquivo. 20-30 arquivos permanece bem dentro dos limites de contexto |
| Remover nós de função/classe do tour (C4) quebra steps de tour existentes | Nenhum | Steps de tour referenciam IDs de nó `file:`. Nenhum dado de tour existente referencia nós de função/classe no nível de step |
| Remover reviewer por padrão (C5) perde erros de grafo | Baixa | O script determinístico inline pega todos os problemas estruturais críticos (refs pendentes, camadas faltando, IDs duplicados). O valor adicional do reviewer LLM é avisos de qualidade (nós órfãos, resumos genéricos), que são não-bloqueantes |
| Geração do import map deixa a Fase 1 mais lenta | Baixa | O script scanner já lê todos os arquivos para contagem de linhas. Parseamento de import adiciona uma passada de regex por arquivo — overhead desprezível |

---

## Recomendação de rollout faseado

Dado o perfil de risco, implementar nesta ordem:

1. **C5 primeiro** — colocar o reviewer atrás de flag, menor risco, economia imediata de 58K tokens por run
2. **C4** — enxugar payload da Fase 5, sem mudanças no scanner, sem risco de qualidade
3. **C3** — remover addendums dos batches, adicionar hints inline
4. **C1+C2 juntos** — mudanças no scanner e consolidação de batch, testar exaustivamente em projetos pequenos/médios/grandes antes de liberar
