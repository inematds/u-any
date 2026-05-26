# /understand-knowledge — Design do Plugin de Base de Conhecimento Pessoal

> 🇬🇧 Versão original em inglês: [2026-04-09-understand-knowledge-design.md](./2026-04-09-understand-knowledge-design.md)

## Visão Geral

Uma nova skill `/understand-knowledge` dentro do plugin Understand Anything existente que recebe qualquer pasta de notas em markdown e produz um grafo de conhecimento interativo visualizado no dashboard existente.

Inspirada no padrão LLM Wiki de Andrej Karpathy — onde um LLM compila e mantém uma wiki estruturada a partir de fontes brutas — esse plugin vai além adicionando descoberta de relações tipadas e visualização interativa de grafo que ferramentas como Obsidian e Logseq não fornecem.

### Objetivos

- Aceitar qualquer base de conhecimento baseada em markdown (Obsidian vault, Logseq graph, Dendron workspace, Foam, LLM wiki estilo Karpathy, Zettelkasten ou markdown puro)
- Auto-detectar o formato e adaptar o parsing de acordo
- Usar análise por LLM para descobrir relações implícitas além dos links explícitos
- Produzir um grafo de conhecimento com nós e arestas tipados
- Visualizar no dashboard existente com layout, sidebar e reading mode específicos para conhecimento

### Não-Objetivos

- Sincronização em tempo real com a ferramenta de base de conhecimento (Obsidian, Logseq, etc.)
- Substituir a ferramenta PKM existente do usuário — isso é uma camada de visualização/análise sobre ela
- Suportar formatos não-markdown (PDFs, bookmarks) no v1

---

## Extensões de Esquema

### Novos Tipos de Nó (5)

Adicionados à união `NodeType` existente (atualmente 16 tipos):

```typescript
export type NodeType =
  // existing (16)
  | "file" | "function" | "class" | "module" | "concept"
  | "config" | "document" | "service" | "table" | "endpoint"
  | "pipeline" | "schema" | "resource"
  | "domain" | "flow" | "step"
  // knowledge (5 new → 21 total)
  | "article" | "entity" | "topic" | "claim" | "source";
```

| Tipo | O que representa | Exemplo |
|------|-------------------|---------|
| `article` | Uma página de wiki/nota — a unidade primária de conteúdo | "LLM Knowledge Bases.md" |
| `entity` | Uma coisa nomeada: pessoa, ferramenta, paper, org, projeto | "Andrej Karpathy", "Obsidian" |
| `topic` | Um cluster temático agrupando artigos relacionados | "Personal Knowledge Management" |
| `claim` | Uma asserção específica, insight ou conclusão | "RAG loses context at chunk boundaries" |
| `source` | Material raw/de referência do qual artigos são compilados | URL de um paper, referência de PDF raw |

### Novos Tipos de Aresta (6)

Adicionados à união `EdgeType` existente (atualmente 29 tipos):

```typescript
export type EdgeType =
  // existing (29)
  | ...
  // knowledge (6 new → 35 total)
  | "cites" | "contradicts" | "builds_on"
  | "exemplifies" | "categorized_under" | "authored_by";
```

| Tipo | Direção | Significado |
|------|-----------|---------|
| `cites` | article → source | Referencia ou se baseia em |
| `contradicts` | claim → claim | Entra em conflito ou discorda |
| `builds_on` | article → article | Estende, refina ou aprofunda |
| `exemplifies` | entity → concept/topic | É um exemplo concreto de |
| `categorized_under` | article/entity → topic | Pertence a este tema |
| `authored_by` | article → entity | Escrito ou criado por |

### Nova Interface de Metadados

```typescript
export interface KnowledgeMeta {
  format?: "obsidian" | "logseq" | "dendron" | "foam" | "karpathy" | "zettelkasten" | "plain";
  wikilinks?: string[];
  backlinks?: string[];
  frontmatter?: Record<string, unknown>;
  sourceUrl?: string;
  confidence?: number; // 0-1, for LLM-inferred relationships
}
```

Adicionada como um campo opcional em `GraphNode`:

```typescript
export interface GraphNode {
  // ...existing fields
  knowledgeMeta?: KnowledgeMeta;
}
```

### Flag de Kind em Nível de Grafo

```typescript
export interface KnowledgeGraph {
  version: string;
  kind: "codebase" | "knowledge"; // NEW
  project: ProjectMeta;
  nodes: GraphNode[];
  edges: GraphEdge[];
  layers: Layer[];
  tour: TourStep[];
}
```

O campo `kind` diz ao dashboard qual layout, sidebar e estilização visual usar. Para compatibilidade retroativa, grafos sem campo `kind` defaultam para `"codebase"`.

---

## Detecção de Formato e Guias de Formato

### Lógica de Auto-Detecção

Escaneia o diretório alvo por arquivos/padrões de assinatura. Ordem de prioridade (primeiro match vence):

| Prioridade | Sinal | Formato Detectado |
|----------|--------|----------------|
| 1 | diretório `.obsidian/` | Obsidian |
| 2 | diretórios `logseq/` + `pages/` | Logseq |
| 3 | `.dendron.yml` ou `*.schema.yml` | Dendron |
| 4 | `.foam/` ou `.vscode/foam.json` | Foam |
| 5 | `raw/` + `wiki/` + `index.md` | Karpathy |
| 6 | `[[wikilinks]]` + prefixos de ID únicos em nomes de arquivo | Zettelkasten |
| 7 | Fallback | Markdown puro |

### Guias de Formato

Localizados em `skills/understand-knowledge/formats/`. Cada guia diz aos agentes LLM como parsear aquele formato:

```
skills/understand-knowledge/
  SKILL.md
  formats/
    obsidian.md        — [[wikilinks]], [[note|alias]], [[note#heading]],
                         #tags, YAML frontmatter, .obsidian/ config,
                         dataview annotations, canvas files
    logseq.md          — block-based outliner, ((block-refs)),
                         journals/YYYY_MM_DD.md, pages/,
                         property:: value syntax, TODO/DONE states
    dendron.md         — dot-delimited hierarchy (a.b.c.md),
                         .schema.yml for structure validation,
                         cross-vault links, refactoring rules
    foam.md            — [[wikilinks]] + link reference definitions
                         at file bottom, .foam/config, placeholder links
    karpathy.md        — raw/ → wiki/ pipeline, index.md master map,
                         log.md append-only record, _meta/ state,
                         LLM-maintained cross-references
    zettelkasten.md    — atomic notes, unique ID prefixes (timestamps),
                         typed semantic links, one idea per note
    plain.md           — standard [markdown](links), folder hierarchy,
                         heading structure, no special conventions
```

Cada guia de formato cobre:
- Como parsear links (wikilinks vs standard vs block refs)
- Onde vivem os metadados (frontmatter vs propriedades inline vs propriedades de bloco)
- O que a estrutura de pastas significa (journals/ = notas diárias, pages/ = notas permanentes)
- Que convenções respeitar vs o que inferir

### Processo de Autoria dos Guias de Formato

Guias de formato devem ser pesquisa-embasados. Durante a implementação, o agente que constrói cada guia deve:
1. Ler a documentação oficial daquele formato (Obsidian Help, Logseq docs, Dendron wiki, Foam docs, etc.)
2. Estudar exemplos reais da estrutura daquele formato
3. Escrever o guia com base em comportamento verificado, não em suposições

---

## Pipeline de Agentes

```
knowledge-scanner → format-detector → article-analyzer → relationship-builder → graph-reviewer
```

### Definições de Agentes

| Agente | Input | Output | Modelo |
|-------|-------|--------|-------|
| `knowledge-scanner` | Path do diretório alvo | Manifest de arquivos: todos `.md` com paths, tamanhos, preview das primeiras 20 linhas | `inherit` |
| `format-detector` | Manifest de arquivos + estrutura de diretório | Formato detectado + hints de parsing específicos do formato | `inherit` |
| `article-analyzer` | Arquivo `.md` individual + guia de formato | Nós por arquivo (article, entities, claims) + arestas explícitas (wikilinks, tags) | `inherit` |
| `relationship-builder` | Todos os resultados por arquivo | Arestas implícitas cross-file (builds_on, contradicts, categorized_under) + agrupamento de tópicos + camadas | `inherit` |
| `graph-reviewer` | Grafo montado | Grafo validado — entidades dedup-ed, pesos de aresta consistentes, detecção de órfãos | `inherit` |

### Diferenças-Chave do Pipeline de Codebase

- **Sem tree-sitter** — parsing de markdown é mais simples, principalmente regex + interpretação por LLM
- **format-detector** substitui detecção de framework — escolhe o guia de formato correto
- **article-analyzer** substitui file-analyzer — extrai conceitos de conhecimento em vez de estrutura de código
- **relationship-builder** é o passo LLM pesado — descobre conexões implícitas cross-file que links explícitos perdem
- **graph-reviewer** permanece similar — valida o grafo montado por consistência

### Arquivos Intermediários

Mesmo padrão da análise de codebase:

```
.understand-anything/intermediate/
  knowledge-manifest.json      — scanner output
  format-detection.json        — detected format + hints
  article-*.json               — per-file analysis
  relationships.json           — cross-file edges
  knowledge-graph.json         — final assembled graph
```

Arquivos intermediários são limpos após a montagem do grafo (igual ao fluxo de codebase).

### Modo Incremental (`--ingest`)

Quando o usuário roda `/understand-knowledge --ingest path/to/new-source.md`:

1. **knowledge-scanner** — roda só nos arquivos novos
2. **format-detector** — pulado (formato já conhecido do scan inicial)
3. **article-analyzer** — processa apenas arquivos novos/alterados
4. **relationship-builder** — roda em novos nós contra o grafo existente, encontra conexões com o que já está lá
5. **graph-reviewer** — valida o resultado mesclado

Nós existentes são preservados; apenas novos nós/arestas são adicionados ou atualizados.

---

## Mudanças no Dashboard

Todas as mudanças são escopadas a grafos com `"kind": "knowledge"`.

### Layout de Fluxo Vertical

- Default para layout vertical top-down (como a visualização atual de fluxo domain/business)
- Tópicos no topo → artigos no meio → entities/claims/sources embaixo
- Lê-se como uma hierarquia de conhecimento: temas amplos fluem para baixo até as especificidades
- Usuário ainda pode trocar para layout horizontal ou force-directed via controles

### Sidebar de Conhecimento

Substitui o NodeInfo quando um grafo de conhecimento é carregado:

| Seleção | Sidebar Mostra |
|-----------|---------------|
| Nada selecionado | ProjectOverview: formato detectado, total de articles/entities/topics/claims/sources |
| Nó article | Título, summary, tags, metadados de frontmatter, lista de backlinks (clicáveis), links de saída, tópicos relacionados |
| Nó entity | Nome, tipo (person/tool/paper/org), artigos que mencionam, relações com outras entities |
| Nó topic | Descrição, artigos filhos, entities filhas, conexões cross-topic |
| Nó claim | Texto da asserção, artigos que sustentam, claims contradizentes (se houver), score de confiança |
| Nó source | URL/path original, artigos que citam, data de ingestão |

### Reading Mode

- Clicar em um nó article dispara um painel de leitura que desliza de baixo (mesmo padrão do overlay atual de visualizador de código)
- Mostra o markdown completo compilado renderizado como HTML
- Inclui uma mini sidebar de backlinks dentro do painel
- Clicar em um `[[wikilink]]` ou referência de entity no painel de leitura navega o grafo até aquele nó

### Estilização Visual de Nó

| Tipo de Nó | Forma | Acento de Cor |
|-----------|-------|-------------|
| `article` | Retângulo arredondado | Âmbar quente |
| `entity` | Círculo | Azul suave |
| `topic` | Retângulo arredondado grande | Dourado mudo |
| `claim` | Diamante | Verde/vermelho dependendo de contradições |
| `source` | Quadrado pequeno | Cinza |

### Estilização Visual de Aresta

| Tipo de Aresta | Estilo |
|-----------|-------|
| `cites` | Linha tracejada |
| `contradicts` | Linha vermelha |
| `builds_on` | Sólida com seta |
| `categorized_under` | Fina cinza |
| `authored_by` | Pontilhada azul |
| `exemplifies` | Pontilhada verde |

---

## Interface da Skill

### Uso

```bash
# Full scan — first time or rescan
/understand-knowledge

# Point at a specific directory
/understand-knowledge path/to/my-notes

# Incremental ingest — add new sources to existing graph
/understand-knowledge --ingest path/to/new-note.md
/understand-knowledge --ingest path/to/new-folder/
```

### Comportamento

1. Auto-detecta o formato (Obsidian, Logseq, Karpathy, etc.)
2. Anuncia: "Detected Obsidian vault with 342 notes. Scanning..."
3. Roda o pipeline de agentes (scanner → detector → analyzer → relationship-builder → reviewer)
4. Escreve `knowledge-graph.json` em `.understand-anything/` com `"kind": "knowledge"`
5. Dispara automaticamente `/understand-dashboard` após o término

### Estrutura de Arquivos

```
skills/understand-knowledge/
  SKILL.md                     — skill entry point, orchestration logic
  formats/
    obsidian.md
    logseq.md
    dendron.md
    foam.md
    karpathy.md
    zettelkasten.md
    plain.md
```

### Coexistência com `/understand`

- `/understand` produz grafos `"kind": "codebase"`
- `/understand-knowledge` produz grafos `"kind": "knowledge"`
- Ambos escrevem em `.understand-anything/knowledge-graph.json`
- Rodar um substitui o outro
- Para escopar a análise de conhecimento a um subdiretório (ex.: `docs/` dentro de um repo de código), use `/understand-knowledge path/to/docs`

---

## O Que Isso Habilita Que Nada Mais Faz

| Ferramentas Existentes | Limitação | Nossa Vantagem |
|---------------|-----------|---------------|
| Graph view do Obsidian | Arestas sem tipo — todos os links parecem iguais | Arestas tipadas: cites, contradicts, builds_on |
| Graph do Logseq | Só mostra links explícitos | LLM descobre relações implícitas |
| Todas as ferramentas PKM | Suportam só um formato | Suporte cross-format com auto-detecção |
| LLM Wiki do Karpathy | Wiki plana de texto, sem visualização | Dashboard de grafo interativo com tours guiados |
| Nenhuma | Sem tours de grafo de conhecimento | Modo tour percorre uma base de conhecimento passo a passo |
