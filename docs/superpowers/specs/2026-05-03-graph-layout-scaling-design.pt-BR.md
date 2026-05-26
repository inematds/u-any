# Escalabilidade do Layout de Grafo do Dashboard — Design

> 🇬🇧 Versão original em inglês: [2026-05-03-graph-layout-scaling-design.md](./2026-05-03-graph-layout-scaling-design.md)

## Problema

Quando uma camada do grafo estrutural contém muitos nós, o atual `applyDagreLayout` (direção TB) coloca nós de mesmo rank em uma única linha horizontal. Com 50+ nós por rank, a linha se estende por milhares de pixels e a visão se torna ilegível: nós encolhem, labels desaparecem, arestas se embaraçam, e não há âncoras visuais para orientar o leitor.

Este design substitui o dagre pelo ELK em todas as visualizações estilo estrutural, introduz **containers** baseados em pasta/comunidade para a visualização de detalhe de camada, e computa o layout em **dois estágios preguiçosos** — uma passada única sobre containers, depois layout por container dos filhos sob demanda.

O esquema do grafo e a saída do pipeline (`graph.json`) ficam inalterados. Todas as melhorias derivam dos dados existentes.

## Objetivos

- Eliminar sprawl horizontal nas visualizações de detalhe de camada em ≤100 nós por camada (target atual), e manter usabilidade até 1000+ nós (escalonamento futuro).
- Dar a cada visualização de detalhe de camada âncoras visuais explícitas para que a estrutura seja legível à primeira vista.
- Agregar arestas cross-cluster por padrão; mostrar arestas individuais sob demanda.
- Manter o estilo visual contínuo com a apresentação existente de layer-cluster (nível overview).
- Tratar falhas de layout com o mesmo modelo `GraphIssue` já usado para validação de esquema.

## Não-Objetivos

- Sem regeneração de `graph.json`. Todo agrupamento é derivado no cliente.
- Sem mudança no KnowledgeGraphView (já é force-directed; fora do escopo).
- Sem aninhamento multi-nível de containers (apenas profundidade única no v1).
- Sem reporte remoto de erros (estilo Sentry) — plugin open-source, sem telemetria por padrão.
- Sem comportamento de agrupamento específico por persona além do filtro de tipo de nó existente.

## Escopo

Três visualizações afetadas:

| Visualização | Mudança |
|---|---|
| Overview (layer clusters) | Substituir dagre → ELK. Sem novo agrupamento (camadas já são grupos). |
| DomainGraphView | Substituir dagre → ELK com domain-as-parent de flow/step. |
| Layer-detail | Substituir dagre → ELK + novos containers folder/community + agregação de arestas + layout preguiçoso de dois estágios. |

KnowledgeGraphView permanece em `applyForceLayout` e não é tocado.

---

## §1. Arquitetura

```
existing graph (immutable)
    │
    ▼
deriveContainers(nodes, edges)           // §2 — folder strategy with community fallback
    │
    ▼
buildCompoundGraph()                     // §4 — aggregate inter-container edges, keep intra-container
    │
    ▼
runStage1Layout(containers, aggEdges)    // §6 — ELK on containers only; uses size memory
    │
    ▼   ┌──────────────────────────────┐
    │   │   render: containers laid    │
    │   │   out, children unrendered   │
    │   └──────────────────────────────┘
    │
    │   triggered by: click | zoom > 1.0 | search/focus/tour hit child
    ▼
runStage2Layout(container)               // §6 — ELK on one container's children; cached
    │
    ▼
React Flow render (parentId for parent-child) + visual overlay (selection/diff/search/tour)
```

Dois invariantes preservados do código atual:

1. **Computação de layout é pura e memoizada.** Só re-roda quando topologia do grafo / persona / diff / focus / nodeTypeFilters mudam.
2. **Estado visual é uma passagem de overlay O(n) separada.** Seleção, destaque de busca, destaque de tour, hover não disparam relayout.

Isso casa com o split existente `useLayerDetailTopology` / `useLayerDetailGraph` em `GraphView.tsx`.

---

## §2. Derivação de Container (Apenas Layer-Detail)

### 2.1 Estratégia de pasta (default)

1. Coletar o `filePath` de cada nó na camada.
2. Computar prefixo comum mais longo (LCP) sobre todos os paths e remover.
3. Agrupar pelo **primeiro segmento de path após o LCP**.
   - `auth/login.go` → container `auth`
   - `auth/handlers/oauth.go` → container `auth`
   - `cart/cart.go` → container `cart`
4. Agrupamento só de profundidade única; sem aninhamento recursivo no v1.
5. Nós sem `filePath` (ex.: tipo `concept`) → container `~` (renderizado como `(root)`, atenuado).

### 2.2 Fallback de comunidade (Louvain)

Disparado quando **qualquer** de:

- Todos os nós compartilham a mesma única pasta após o strip do LCP.
- Contagem de buckets (pastas + rooted) `< 2`.
- Qualquer bucket único (pasta ou rooted) detém `> 70%` dos nós.

Roda detecção de comunidades baseada em modularidade Louvain sobre as arestas internas da camada. Cada comunidade vira um container. Nomes são placeholders (`Cluster A`, `Cluster B`, ...) já que nenhum nome semântico está disponível.

Implementação: usar `graphology` + `graphology-communities-louvain` (~30KB no total). JS puro, sem deps nativas, roda na main thread sincronamente para arestas layer-internas.

### 2.3 Casos de borda

| Caso | Comportamento |
|---|---|
| Container tem 1 filho (apenas quando total da camada ≥ 3) | Caixa de container não é renderizada; filho vira nó top-level no layout do Stage 1 |
| Container tem 2 filhos | Container renderizado; label atenuado |
| Todos os nós não têm `filePath` | Todos vão para o container `~`; se virasse single-child, fallback para flat |

### 2.4 Assinatura de função

```ts
function deriveContainers(
  nodes: GraphNode[],
  edges: GraphEdge[],
): {
  containers: Array<{
    id: string;                        // e.g. "container:auth" or "container:cluster-0"
    name: string;                       // "auth" or "Cluster A"
    nodeIds: string[];
    strategy: "folder" | "community";
  }>;
  ungrouped: string[];                  // nodes that bypass containerization
};
```

O campo `strategy` é exposto na UI ("Grouped by folder" vs "Grouped by edge density") para que o usuário saiba como uma camada particular foi organizada.

---

## §3. Integração com ELK

### 3.1 Pacote

- `elkjs` ^0.9 (~250KB gzipped). Usar `elk.bundled.js`, não a variante worker.
- API baseada em Promise. Roda na main thread para grafos ≤500 nós; <100ms tipicamente.

### 3.2 Configuração

```ts
{
  algorithm: "layered",
  "elk.direction": "DOWN",                                 // matches dagre TB
  "elk.layered.spacing.nodeNodeBetweenLayers": 80,
  "elk.spacing.nodeNode": 60,
  "elk.layered.crossingMinimization.strategy": "LAYER_SWEEP",
  "elk.edgeRouting": "ORTHOGONAL",
  "elk.layered.compaction.postCompaction.strategy": "LEFT",
  "elk.padding": "[top=40,left=20,right=20,bottom=20]",   // container internal padding
}
```

`hierarchyHandling: INCLUDE_CHILDREN` **não** é usado — a abordagem de dois estágios (§6) emite chamadas ELK separadas para containers top-level e filhos por container, então um único grafo composto nunca é montado.

### 3.3 Modelagem de entrada por visualização

| Visualização | Input do ELK |
|---|---|
| Overview | Flat. Filhos = nós de layer-cluster. |
| DomainGraphView | Flat no v1 (domain permanece como o único agrupamento; nós flow/step posicionados dentro). |
| Layer-detail Stage 1 | Flat. Filhos = containers (tratados como átomos opacos). |
| Layer-detail Stage 2 | Flat por container. Filhos = arquivos dentro. |

Uma única função `runElk(input): Promise<positioned>` atende os quatro casos.

### 3.4 Fronteiras com o `utils/layout.ts` existente

| Função | Status |
|---|---|
| `applyDagreLayout` | Mantida temporariamente; removida na versão após a migração de layout ser verificada estável |
| `applyForceLayout` | Intocada (somente KnowledgeGraphView) |
| `applyElkLayout` (nova) | Wrapper que lida com repair → ELK → coerção de resultado |

### 3.5 Async + estado de loading

Stage 1 roda em um `useEffect` com cancelamento na mudança de dependência:

```ts
useEffect(() => {
  let cancelled = false;
  setLayoutStatus("computing");
  applyElkLayout(input).then(result => {
    if (!cancelled) {
      setLayout(result);
      setLayoutStatus("ready");
    }
  });
  return () => { cancelled = true };
}, [graph, activeLayerId, persona, diffMode, nodeTypeFilters]);
```

Enquanto `layoutStatus === "computing"`, renderiza um overlay `"Computing layout…"` (semi-transparente, centralizado). Layout stale do estado anterior é mantido por baixo para que o viewport não pisque.

### 3.6 Tratamento de falha — reusa o modelo GraphIssue existente

Antes de invocar o ELK, rodar `repairElkInput()` sobre o input montado. Cada reparo emite um `GraphIssue` consumido pelo `WarningBanner` existente.

| Função de reparo | Disparada por | Nível de issue |
|---|---|---|
| `ensureNodeDimensions` | Nó sem width/height | `auto-corrected` |
| `dedupeNodeIds` | Id de filho duplicado sob o mesmo pai | `auto-corrected` |
| `dropOrphanEdges` | Source/target de aresta não está no conjunto de nós | `dropped` |
| `dropOrphanChildren` | Filho referencia um pai inexistente | `dropped` |
| `dropCircularContainment` | Ciclo de containment de container | `dropped` |

Se o ELK ainda rejeitar após o reparo → emite um `GraphIssue` `fatal`, renderiza um grafo vazio + o banner fatal existente. O texto do fatal é aumentado com "this looks like a dashboard rendering bug — please file an issue with the copied error" para que o usuário saiba endereçar o report ao dashboard, não aos dados do grafo.

### 3.7 Falhas estritas em modo dev

Tanto `repairElkInput` quanto `runElk` aceitam um `strict: boolean`. Em `import.meta.env.DEV`, strict está ligado — reparos e erros de ELK lançam imediatamente em vez de produzir issues graciosas. Isso captura bugs de construção de input durante o desenvolvimento antes que entrem como fallbacks silenciosos.

---

## §4. Agregação de Arestas

### 4.1 Algoritmo

Executado dentro de `buildCompoundGraph()`, antes de qualquer estágio ELK.

```ts
function aggregateContainerEdges(
  nodes: GraphNode[],
  edges: GraphEdge[],
  nodeToContainer: Map<string, string>,
): {
  intraContainer: Edge[];                       // preserved as-is
  interContainerAggregated: AggregatedEdge[];   // one per (sourceContainer, targetContainer)
};
```

Regras:

- Para cada aresta, busque os containers de source/target.
- Mesmo container → intra (inalterado).
- Containers diferentes → bucket por `(sourceContainer, targetContainer)`. Direção importa: A→B e B→A são independentes.
- Cada aresta agregada carrega `count` e `types` (conjunto de tipos de aresta que aparecem no bucket).

### 4.2 Visual

Reusar o padrão de estilização já presente na agregação de aresta no nível overview (`GraphView.tsx` linha ~186):

- `strokeWidth: Math.min(1 + Math.log2(count + 1), 5)`
- Label: número de contagem
- Cor: `rgba(212,165,116,0.4)` existente

### 4.3 Expandir / colapsar

Estado (zustand store):

```ts
expandedContainers: Set<string>;   // currently expanded container ids
```

Triggers:

- **Clicar container** → toggle de membership.
- **Clicar canvas vazio** ou `Esc` → limpar todos.
- **Expansão multi-container é permitida** (usuário comparando relações de duas pastas).

Quando um container é expandido:

- Suas arestas agregadas inter-container (ambas direções) são substituídas pelas arestas individuais file→file subjacentes.
- Arestas agregadas de outros containers permanecem agregadas.
- Re-layout de posição **não** é disparado. Apenas o array de arestas do React Flow muda.

### 4.4 Interações com persona / diff

- **Filtro de persona** altera `count` (apenas arestas pós-filtro). Aresta agregada re-derivada no pipeline memoizado.
- **Modo diff**: aresta agregada contendo qualquer nó alterado → stroke vermelho + animado; ao expandir, arestas individuais seguem estilização de diff normal.

---

## §5. Visual do Container

### 5.1 Novo componente: `ContainerNode`

Um novo tipo de nó React Flow `"container"` registrado junto aos existentes `custom` / `layer-cluster` / `portal`.

Ele **não** reusa `LayerClusterNode` porque:

- Semântica de clique difere (`LayerClusterNode` faz drill em uma camada; `ContainerNode` faz toggle de expansão de aresta).
- Metadados diferem (`ContainerNode` não carrega `aggregateComplexity`).

Linguagem visual é compartilhada: caixa translúcida arredondada, borda dourada, título DM Serif.

### 5.2 Spec

| Elemento | Estilo |
|---|---|
| Borda (default) | `1px solid rgba(212,165,116,0.25)` |
| Borda (hover / expanded) | `1.5px rgba(212,165,116,0.6)`, expanded adiciona chevron `▾` |
| Background | `rgba(255,255,255,0.02)` |
| Raio de canto | `12px` |
| Título | DM Serif, 14px, `#d4a574`, padding top-left `12px 16px` |
| Badge de contagem de filhos | chip top-right, `#a39787`, 11px |
| Padding interno (em volta dos filhos) | `40px top / 20px L,R,B` |

### 5.3 Codificação por cor

Índice de container módulo uma paleta de 12 cores (mesma paleta usada para `layerColorIndex` em `LayerClusterNode`). Matiz é aplicado em saturação baixa apenas em borda + título — nunca no preenchimento do corpo — para que a paleta não sobrepuje os nós individuais dentro.

### 5.4 Estilos de estado

| Estado | Visual |
|---|---|
| `default` | Spec base |
| `hover` | Borda mais brilhante, título sublinhado |
| `expanded` | Borda dourada 1.5px + chevron `▾` |
| `search-hit-inside` | Badge de busca na linha de título mostrando contagem de match |
| `diff-affected` | Borda troca para `rgba(224,82,82,0.5)` |
| `focused-via-child` | Igual ao expanded mais boost de brilho |

### 5.5 Fonte do label

| Estratégia | Label |
|---|---|
| `folder` | Primeiro segmento de path após LCP (ex.: `auth`) |
| `community` | `Cluster A`, `Cluster B`, ... ordenado por id da comunidade |
| `~` (root) | `(root)` em estilo atenuado |

---

## §6. Layout Preguiçoso de Dois Estágios

### 6.1 Máquina de estado

```
[layer entered]
    │
    │ Stage 1: ELK on containers (always runs)
    ▼
[containers laid out, children unrendered]
    │
    ├── click container ─────┐
    ├── zoom > 1.0 in viewport (200ms debounce, hysteresis) ─┤
    └── search / focus / tour hit a child ─┘
                                            ▼
                            Stage 2 (per container)
                                            │
                                            ▼
                       [container expanded, children laid out + rendered]
```

### 6.2 Extensões da store

```ts
expandedContainers: Set<string>;
containerLayoutCache: Map<string, {
  childPositions: Map<string, { x: number; y: number }>;
  actualSize: { width: number; height: number };
}>;
containerSizeMemory: Map<string, { width: number; height: number }>;
```

- `containerLayoutCache` invalidado por `(graphHash, containerId)`.
- `containerSizeMemory` persiste entre colapsos de container para prevenir jitter no próximo expand.

### 6.3 Stage 1

```ts
async function runStage1Layout(containers, aggregatedInterEdges, sizeMemory) {
  const elkInput = {
    id: "root",
    children: containers.map(c => ({
      id: c.id,
      width: sizeMemory.get(c.id)?.width
        ?? Math.sqrt(c.nodeIds.length) * NODE_WIDTH * 1.2,
      height: sizeMemory.get(c.id)?.height
        ?? Math.sqrt(c.nodeIds.length) * NODE_HEIGHT * 1.2,
    })),
    edges: aggregatedInterEdges.map(toElkEdge),
  };
  return runElk(elkInput);
}
```

Tamanho do container é estimado por `sqrt(childCount)` para que cresça sub-linearmente com o conteúdo. Se a memória tem o tamanho real de um run anterior, ele vence.

### 6.4 Stage 2

```ts
async function runStage2Layout(container, intraEdges) {
  if (containerLayoutCache.has(container.id)) {
    return containerLayoutCache.get(container.id)!;
  }
  const elkInput = {
    id: container.id,
    children: container.nodeIds.map(toElkChild),
    edges: intraEdges.filter(e => isWithin(container, e)).map(toElkEdge),
  };
  const result = await runElk(elkInput);
  containerLayoutCache.set(container.id, result);
  containerSizeMemory.set(container.id, result.actualSize);
  return result;
}
```

Se `result.actualSize` diferir da estimativa do Stage 1 em **> 20%** em qualquer dimensão, dispara um re-layout do Stage 1 (re-run completo; <100ms nessa escala, então o usuário percebe um pequeno reflow em vez de dois layouts distintos).

### 6.5 Triggers de auto-expand

| Trigger | Implementação |
|---|---|
| Clique | `onClick` faz toggle de `expandedContainers` |
| Zoom | Listener do React Flow `onMove` (debounce 200ms). Quando o zoom do viewport > 1.0, todos os containers no viewport são adicionados a `expandedContainers`. Histerese: containers não auto-colapsam até zoom < 0.6, prevenindo flapping. |
| Search / focus / tour | `useEffect` observa `searchResults` / `focusNodeId` / `tourHighlightedNodeIds`; encontra o container pai de qualquer nó folha casado e adiciona a `expandedContainers` |

### 6.6 Orçamento de performance

| Operação | Target |
|---|---|
| Stage 1 (qualquer camada) | < 100ms |
| Stage 2 (primeiro expand de um container) | < 100ms |
| Stage 2 (cache hit) | < 5ms |
| Auto-expand dirigido por zoom | debounce 200ms |
| Re-layout do Stage 1 após desvio >20% | < 100ms (reusa o caminho do Stage 1) |

---

## §7. Matriz de Interação

| Feature existente | Comportamento com novo layout |
|---|---|
| Filtro de persona | Dirige a dependência `nodeTypeFilters` no memo do Stage 1. Nós filtrados não entram na derivação de container; containers com todos os filhos filtrados desaparecem. |
| Modo diff | Container com filho alterado ganha borda vermelha (§5.4); arestas agregadas contendo um nó alterado animam vermelho; ao expandir, estilização de diff individual se aplica. |
| Modo focus (1-hop) | Container do nó focado auto-expande. Containers não-vizinhos esmaecem para opacidade 0.2; seus filhos permanecem não-renderizados. |
| Busca | Container com hit ganha badge de busca no título; container **não** auto-expande para evitar expandir muitos de uma vez. Clicar no badge expande e faz `fitView`. |
| Tour | Filho destacado pelo tour auto-expande seu container. `TourFitView` ajusta para as posições do leaf destacado (cacheadas após expand). |
| Drill-in (`overview → layer-detail`) | Inalterado. Após drill-in, Stage 1 roda nos containers da nova camada. |
| Breadcrumb | Containers não entram no breadcrumb. Path permanece `Project > LAYER`. |
| Code viewer | Inalterado. Clicar em um nó de arquivo dentro de um container → visualizador slide-up existente. |
| WarningBanner | Issues de reparo de layout alimentam o mesmo banner. Texto do fatal aumentado para diferenciar bugs de render de bugs de dados. |
| Export (PNG/SVG) | Captura o estado atual incluindo containers expandidos. Filename inclui nome da camada. |

---

## §8. Arquivos e Plano de Testes

### 8.1 Arquivos

```
packages/dashboard/src/
├── utils/
│   ├── layout.ts              [modify] add applyElkLayout export
│   ├── elk-layout.ts          [new]    runElk + repairElkInput + GraphIssue mapping
│   ├── containers.ts          [new]    deriveContainers (folder + community fallback)
│   ├── louvain.ts             [new]    thin wrapper around graphology-communities-louvain
│   └── edgeAggregation.ts     [modify] add aggregateContainerEdges
├── components/
│   ├── ContainerNode.tsx      [new]    container box visual
│   ├── GraphView.tsx          [modify] Stage 1 / Stage 2 wiring, expand state, auto-expand triggers
│   └── DomainGraphView.tsx    [modify] dagre → ELK
├── store.ts                   [modify] expandedContainers, containerLayoutCache, containerSizeMemory
└── package.json               [modify] add elkjs ^0.9, graphology, graphology-communities-louvain
```

### 8.2 Matriz de testes

| Tipo | Alvo | Casos |
|---|---|---|
| Unit | `deriveContainers` | happy path de agrupamento por pasta; fallback all-in-root; fallback <2 buckets; fallback de concentração >70%; nós sem `filePath`; supressão de container single-child (gated por camada ≥ 3) |
| Unit | `aggregateContainerEdges` | arestas vazias; múltiplas arestas same-direction mesclam; arestas bidirecionais split; mix intra + inter; types dedup-ed |
| Unit | `repairElkInput` | cada função de reparo isoladamente; valida nível de `GraphIssue` correto emitido |
| Unit | `runElk` | input mínimo válido; throw estrito em dev; fatal gracioso em produção; cancelamento em mudança de dependência |
| Integração | Fluxo Stage 1 + Stage 2 | fixture de 50 nós; clique → cache miss; segundo clique → cache hit; desvio de tamanho >20% → re-layout |
| Integração | Interações persona / focus / search | trocar persona re-roda Stage 1; focar um filho auto-expande seu container; hit de busca adiciona badge sem auto-expandir |
| Regressão visual (opcional) | Playwright + fixture microservices-demo | screenshots baseline para overview, layer-detail, visualizações de domain |

### 8.3 Benchmarks de performance

Gerar fixtures com `scripts/generate-large-graph.mjs` a 500 / 1000 / 3000 nós. Verificar:

- Stage 1 < 200ms a 500 nós; < 500ms a 3000 nós.
- Stage 2 qualquer container < 100ms.

Se Stage 1 de 3000 nós perder o orçamento, revisitar estimação de tamanho de container ou config ELK — não baixar o orçamento.

---

## Questões em Aberto

Nenhuma neste ponto. Todas as decisões tomadas durante brainstorming estão capturadas acima.

## Notas de Migração

- `applyDagreLayout` é mantido na base de código por um release após este aterrissar, depois removido no próximo. Isso dá um caminho de fallback durante o rollout e uma desinstalação limpa quando estiver estável.
- Sem migração de dados de grafo necessária.
- Novas dependências (elkjs, graphology, graphology-communities-louvain) são JS puro, sem bindings nativos — seguros em toda a matriz de plataformas suportadas.
