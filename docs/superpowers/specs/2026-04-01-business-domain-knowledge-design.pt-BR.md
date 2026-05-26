# Extração de Conhecimento de Domínio de Negócio — Spec de Design

> 🇬🇧 Versão original em inglês: [2026-04-01-business-domain-knowledge-design.md](./2026-04-01-business-domain-knowledge-design.md)

**Issue:** [#61](https://github.com/Lum1104/Understand-Anything/issues/61)
**Data:** 2026-04-01

## Problema

O grafo de conhecimento atual mostra relações de dependência em nível de arquivo, mas isso tem valor limitado — você já consegue ver imports em uma IDE. Quando há muitos arquivos, listar arestas de dependência não reduz a carga cognitiva; você ainda reconstrói mentalmente o que o código *faz*. O que é necessário é conhecimento de domínio de negócio: a lógica e os conceitos de domínio embutidos no código, não a fiação estrutural.

## Visão Geral da Solução

Uma nova skill `/understand-domain` que extrai conhecimento de domínio de negócio e o renderiza como um grafo de fluxo horizontal no dashboard. Dois modos de visualização: uma **visualização de Domínio** de alto nível (default quando disponível) e a existente **visualização Estrutural**, com um toggle para alternar entre elas.

## Arquitetura: Arquivo Separado, Esquema Compartilhado (Abordagem C)

Os dados de domínio vivem em um **arquivo separado** (`domain-graph.json`) usando o **mesmo sistema de tipos `KnowledgeGraph`** — estendido com novos tipos de nó/aresta. O dashboard detecta ambos os arquivos e oferece um toggle de visualização. Nós de domínio podem referenciar nós estruturais por ID para drill-down.

**Por que arquivos separados:**
- `/understand-domain` funciona standalone (leve) ou junto com o grafo completo
- Esquema compartilhado significa que busca, validação e filtragem funcionam para ambos
- Sem risco de poluir o grafo estrutural
- Cada arquivo é independentemente válido

## Seção 1: Esquema do Grafo de Domínio

### Hierarquia de Três Níveis

1. **Domínio de Negócio** (topo) — ex.: "Purchasing", "Logistics", "Warehouse Management"
2. **Fluxo de Negócio** (meio) — ex.: "Create Order", "Process Refund"
3. **Passo de Negócio** (folha) — ex.: "Validate input", "Check inventory", "Persist order"

### Novos Tipos de Nó (3)

| Tipo | Propósito | Exemplo |
|------|---------|---------|
| `domain` | Cluster de domínio de negócio | "Order Management", "Logistics" |
| `flow` | Um processo de negócio dentro de um domínio | "Create Order", "Process Refund" |
| `step` | Um único passo em um fluxo | "Validate order input" |

### Novos Tipos de Aresta (4)

| Tipo | Propósito |
|------|---------|
| `contains_flow` | domain → flow |
| `flow_step` | flow → step (ordenado via campo `weight`, ex.: 0.1, 0.2, ...) |
| `cross_domain` | domain → domain (interação entre domínios) |
| `implements` | step → ID de nó file/function (referência para o grafo estrutural) |

### Estrutura do Nó de Domínio

```typescript
// domain node
{
  id: "domain:order-management",
  type: "domain",
  name: "Order Management",
  summary: "Handles the complete order lifecycle...",
  tags: ["e-commerce", "core-business"],
  complexity: "complex",
  domainMeta?: {
    entities: ["Order", "LineItem", "OrderStatus"],
    businessRules: ["Orders require inventory check before confirmation"],
    crossDomainInteractions: ["Triggers Logistics on order confirmed", "Reads from Customer Service for buyer info"]
  }
}
```

### Estrutura do Nó de Fluxo

```typescript
{
  id: "flow:create-order",
  type: "flow",
  name: "Create Order",
  summary: "Customer submits a new order through the API",
  tags: ["write-path", "api"],
  complexity: "moderate",
  domainMeta?: {
    entryPoint: "POST /api/orders",
    entryType: "http" | "cli" | "event" | "cron" | "manual"
  }
}
```

### Estrutura do Nó de Passo

```typescript
{
  id: "step:create-order:validate-input",
  type: "step",
  name: "Validate order input",
  summary: "Checks request body against order schema, rejects invalid payloads",
  tags: ["validation"],
  complexity: "simple",
  filePath: "src/validators/order-validator.ts",
  lineRange: [12, 45]
}
```

### Saída em Arquivo

Salvo em `.understand-anything/domain-graph.json` — mesmo formato `KnowledgeGraph`, válido por si só.

## Seção 2: Pipeline de Análise

### Dois Caminhos, Mesma Saída

**Caminho 1: Scan leve (sem grafo existente)**

```
File tree scan
    → Static entry point detection (tree-sitter)
        → Route definitions, exported handlers, main(), event listeners, cron decorators
    → Feed to LLM: file tree + detected entry points + sampled file contents
        → LLM outputs: domains, flows, steps, cross-domain interactions
    → Build domain-graph.json
```

Custo de tokens: ~10-20% de um scan completo de `/understand`.

**Caminho 2: Derivar do grafo existente**

```
Load knowledge-graph.json
    → Extract: all nodes, edges, layers, summaries, tour
    → Feed to LLM: graph data as structured context
        → LLM outputs: domains, flows, steps, cross-domain interactions
    → Build domain-graph.json
```

Muito barato — não precisa ler arquivos, o LLM raciocina sobre resumos existentes e arestas de chamada.

**Seleção de Caminho:** `/understand-domain` verifica se `.understand-anything/knowledge-graph.json` existe. Se sim → Caminho 2. Se não → Caminho 1.

### Estrutura de Agente

Um novo agente: **`domain-analyzer`** (modelo opus). Lida com ambos os caminhos. Para bases de código grandes, pode batch-ar por grupos de entry points detectados.

## Seção 3: Script de Pré-processamento e Integração com a Skill

### Script: `understand-anything-plugin/skills/understand-domain/extract-domain-context.py`

Empacotado com a skill (não em `scripts/`, que é para tooling de desenvolvimento). Roda antes do agente LLM. Saída em `.understand-anything/intermediate/domain-context.json`:

```json
{
  "fileTree": ["src/api/orders.ts", "src/services/...", "..."],
  "entryPoints": [
    {
      "file": "src/api/orders.ts",
      "type": "http",
      "method": "POST",
      "path": "/api/orders",
      "handler": "createOrder",
      "lineRange": [15, 45],
      "snippet": "async function createOrder(req, res) { ... }"
    }
  ],
  "fileSignatures": {
    "src/services/order-service.ts": {
      "exports": ["createOrder", "cancelOrder", "getOrderById"],
      "imports": ["inventory-service", "pricing-service", "order-repo"],
      "summary": null
    }
  }
}
```

Script Python (sem dependências pesadas — usa `ast` para Python, regex para outras linguagens). Usa:
- Caminhar pela árvore de arquivos (respeitando `.gitignore`)
- Detectar entry points por padrão: decoradores de rota, `app.get/post`, `export default handler`, `main()`, event listeners
- Extrair assinaturas de função e listas de import/export por arquivo
- Manter snippets de código curtos (assinatura + primeiras linhas, não corpos completos)

### Integração com a Skill

O markdown da skill `/understand-domain`:

1. Roda `understand-anything-plugin/skills/understand-domain/extract-domain-context.py`
2. Verifica `knowledge-graph.json` existente
3. Se existe → passa tanto `domain-context.json` + dados do grafo para o agente domain-analyzer
4. Se não → passa apenas `domain-context.json`
5. Agente gera `domain-graph.json`
6. Limpa arquivos intermediários
7. Dispara automaticamente `/understand-dashboard`

## Seção 4: Dashboard — Visualização de Domínio

### Toggle de Visualização

- Canto superior esquerdo: toggle em pílula — **"Domain" / "Structural"**
- A visualização de Domínio é default quando `domain-graph.json` existe
- Se só um arquivo de grafo existe, nenhum toggle é mostrado
- Trocar de visualização preserva o estado da sidebar

### Layout de Fluxo Horizontal

- **Engine de layout:** Dagre com `rankdir: "LR"` (left-to-right)
- **Níveis de zoom:**
  - **Zoom out:** Clusters de domínio como retângulos arredondados grandes, arestas `cross_domain` entre eles
  - **Clique no domínio:** Expande para mostrar fluxos como pistas horizontais
  - **Clique no fluxo:** Mostra trace passo a passo da esquerda para a direita

### Renderização do Cluster de Domínio

```
┌─────────────────────────────────────┐
│  Order Management                   │
│  "Handles the complete order..."    │
│                                     │
│  Entities: Order, LineItem, Status  │
│  Flows: Create Order, Cancel Order  │
│  Rules: "Requires inventory check"  │
└─────────────────────────────────────┘
          ──cross_domain──→  [Logistics]
```

- Borda dourada/âmbar para clusters de domínio (combina com o tema existente)
- Mostra summary, lista de entidades, contagem de fluxos na face do cluster
- Arestas cross-domain: linhas tracejadas grossas com labels

### Renderização de Trace de Fluxo

```
POST /api/orders
  ┌──────────┐    ┌──────────────┐    ┌───────────┐    ┌──────────┐    ┌────────────┐
  │ Validate  │───→│ Check        │───→│ Calculate  │───→│ Persist   │───→│ Send       │
  │ Input     │    │ Inventory    │    │ Pricing    │    │ Order     │    │ Confirm    │
  └──────────┘    └──────────────┘    └───────────┘    └──────────┘    └────────────┘
```

- Steps conectados da esquerda para a direita por arestas `flow_step` (ordenados por `weight`)
- Label de entry point à esquerda como trigger do fluxo
- Clicar em um step → sidebar mostra detalhe + link para a visualização estrutural

### Adaptações da Sidebar

**Nó de domínio selecionado:** Summary, regras de negócio, entidades, interações cross-domain, lista de fluxos (clicáveis)

**Nó de fluxo selecionado:** Info do entry point, lista de steps em ordem, complexidade

**Nó de step selecionado:** Descrição, link "View in code" (troca para a visualização estrutural + navega para o arquivo/função), links anterior/próximo

### Drill-Down: Domínio → Estrutural

Quando um step tem uma aresta `implements` referenciando um ID de nó estrutural:
- Botão "View implementation" na sidebar
- Troca para a visualização estrutural e navega para aquele nó
- Breadcrumb: `Domain: Order Management > Flow: Create Order > Step: Validate Input → [structural view]`

## Seção 5: Definição da Skill

### Skill `/understand-domain`

- **Arquivo:** `skills/understand-domain.md`
- **Argumentos:** Flag opcional `--full` para forçar o Caminho 1 (rescan mesmo se o grafo existir)

### Fluxo de Execução

```
1. Run scripts/extract-domain-context.mjs
2. Check for .understand-anything/knowledge-graph.json
   ├── Exists → Path 2: load graph + domain-context.json
   └── Missing → Path 1: domain-context.json only
3. Invoke domain-analyzer agent (opus)
4. Validate output against schema
5. Save .understand-anything/domain-graph.json
6. Clean up intermediate/domain-context.json
7. Auto-trigger /understand-dashboard
```

### Agente Domain Analyzer

- **Arquivo:** `agents/domain-analyzer.md`
- **Modelo:** opus
- **Input:** Ou (file tree + entry points) ou (knowledge graph existente)
- **Output:** JSON completo de grafo de domínio

### Mapa de Mudanças

| Área | Mudanças |
|------|---------|
| `packages/core/src/types.ts` | Adiciona 3 tipos de nó, 4 tipos de aresta, campo opcional `domainMeta` |
| `packages/core/src/schema.ts` | Estende schemas Zod + aliases para novos tipos |
| `packages/core/src/persistence/` | Adiciona `loadDomainGraph()` / `saveDomainGraph()` |
| `understand-anything-plugin/skills/understand-domain/extract-domain-context.py` | Novo script de pré-processamento (empacotado com a skill) |
| `agents/domain-analyzer.md` | Nova definição de agente |
| `skills/understand-domain.md` | Nova definição de skill |
| `packages/dashboard/src/store.ts` | Adiciona estado `domainGraph`, `viewMode` |
| `packages/dashboard/src/components/` | Novos: `DomainGraphView.tsx`, `DomainClusterNode.tsx`, `FlowTraceNode.tsx`, `StepNode.tsx` |
| `packages/dashboard/src/components/` | Modificar: `App.tsx` (toggle de visualização), `NodeInfo.tsx` (sidebar de domínio), `FilterPanel.tsx` (filtros de domínio) |
| `packages/dashboard/src/utils/` | Novo: `domain-layout.ts` (config Dagre horizontal) |

## Seção 6: Tolerância a Erros

### Tolerância em Nível de Pipeline

| Estágio | Tratamento de Erro |
|-------|---------------|
| Script de pré-processamento | Se o tree-sitter falhar em um arquivo, pular e continuar. Logar arquivos pulados. Detecção de entry point é best-effort. |
| Parseamento da saída do LLM | Mesma estratégia do `parseTourGenerationResponse()` existente — extrair JSON de markdown, lidar com respostas parciais. |
| Validação de esquema | Pipeline de auto-fix existente: sanitize → normalize (aliases) → apply defaults → validate. Descartar nós/arestas quebrados, não falhar o grafo inteiro. |
| Referências cross-graph | Arestas `implements` apontando para IDs de nó estrutural inexistentes → manter aresta mas marcar como `unresolved`. Dashboard mostra step sem link de drill-down. |

### Regras de Validação Específicas de Domínio

- **Domínio sem fluxos:** Avisar, manter (summary/entidades ainda úteis)
- **Fluxo sem steps:** Avisar, manter (info de entry point ainda valiosa)
- **Steps com ordenação quebrada:** Renumerar sequencialmente por posição no array se valores `weight` faltarem/duplicarem
- **Steps órfãos:** Steps não conectados a nenhum fluxo → anexar a um fluxo sintético "Uncategorized"
- **Domínios duplicados:** Mesclar por similaridade de nome (fuzzy match), combinar fluxos
- **Grafo de domínio vazio:** Banner de erro no dashboard: "Domain extraction failed — try running `/understand` first for richer context, then `/understand-domain`"

### Resiliência do Dashboard

- Se `domainMeta` estiver faltando em um nó de domínio, sidebar mostra apenas summary/tags
- Se `domain-graph.json` falhar a validação totalmente, fazer fallback para a visualização estrutural com banner de aviso
- Grafos parciais renderizam o que for válido

### Aliases de Normalização para Tipos de Domínio

```typescript
// Node type aliases
"business_domain" → "domain"
"process" → "flow"
"workflow" → "flow"
"action" → "step"
"task" → "step"

// Edge type aliases
"has_flow" → "contains_flow"
"next_step" → "flow_step"
"interacts_with" → "cross_domain"
"implemented_by" → "implements"
```
