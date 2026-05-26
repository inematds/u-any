# Understand Anything — Plano de Design e Implementação

> 🇬🇧 Versão original em inglês: [2026-03-14-understand-anything-design.md](./2026-03-14-understand-anything-design.md)

## Contexto

Ferramentas de codificação com IA tornaram escrever código fácil, mas entender código continua difícil. Desenvolvedores juniores, não-programadores (PMs, designers) e até devs experientes trabalhando em linguagens desconhecidas têm dificuldade para compreender bases de código que não escreveram — ou que a IA escreveu por eles. A única entidade que "entende" o código é a própria IA.

**Understand Anything** preenche essa lacuna: uma ferramenta open-source que combina inteligência de LLM com análise estática para produzir um dashboard interativo e multi-persona para entender qualquer base de código. Ela roda como uma skill do Claude Code (aproveitando a sessão ativa) e serve um dashboard web rico.

---

## Arquitetura: Monorepo com Core Compartilhado

```
understand-anything/
├── packages/
│   ├── core/              # Shared analysis engine
│   │   ├── analyzer/      # LLM + tree-sitter analysis
│   │   ├── graph/         # Knowledge graph builder & schema
│   │   ├── plugins/       # Plugin system for language analyzers
│   │   └── persistence/   # JSON read/write, staleness detection
│   ├── skill/             # Claude Code skill (5 commands)
│   └── dashboard/         # React + TypeScript multi-panel workspace
├── plugins/               # Built-in language analyzer plugins
│   └── tree-sitter/       # Tree-sitter based multi-language analyzer
├── docs/
│   └── plans/
├── package.json           # Monorepo root (pnpm workspaces)
├── tsconfig.json
└── .gitignore
```

**Decisões-chave:**
- **Monorepo** (pnpm workspaces) — a skill e o dashboard compartilham o motor de análise central
- **Interchange via JSON** — o grafo de conhecimento é um arquivo JSON, legível tanto pela skill quanto pelo dashboard
- **Commitável + auto-sync** — o grafo persiste em `.understand-anything/`, pode ser commitado no git, detecta automaticamente desatualização via git diff

---

## Esquema do Grafo de Conhecimento

```typescript
interface KnowledgeGraph {
  version: string;
  project: ProjectMeta;
  nodes: GraphNode[];
  edges: GraphEdge[];
  layers: Layer[];
  tour: TourStep[];
}

interface ProjectMeta {
  name: string;
  languages: string[];
  frameworks: string[];
  description: string;           // LLM-generated project summary
  analyzedAt: string;            // ISO timestamp
  gitCommitHash: string;         // For staleness detection
}

interface GraphNode {
  id: string;
  type: "file" | "function" | "class" | "module" | "concept";
  name: string;
  filePath?: string;
  lineRange?: [number, number];
  summary: string;               // Plain-English description
  tags: string[];                // Searchable tags
  complexity: "simple" | "moderate" | "complex";
  languageNotes?: string;        // Language-specific explanations
}

interface GraphEdge {
  source: string;
  target: string;
  type: EdgeType;
  direction: "forward" | "backward" | "bidirectional";
  description?: string;
  weight: number;                // 0-1 importance
}

type EdgeType =
  // Structural
  | "imports" | "exports" | "contains" | "inherits" | "implements"
  // Behavioral
  | "calls" | "subscribes" | "publishes" | "middleware"
  // Data flow
  | "reads_from" | "writes_to" | "transforms" | "validates"
  // Dependencies
  | "depends_on" | "tested_by" | "configures"
  // Semantic
  | "related" | "similar_to";

interface Layer {
  id: string;
  name: string;                  // e.g., "API Layer", "Data Layer"
  description: string;
  nodeIds: string[];
}

interface TourStep {
  order: number;
  title: string;
  description: string;           // Markdown explanation
  nodeIds: string[];             // Nodes to highlight
  languageLesson?: string;       // Optional language concept explanation
}
```

---

## Dashboard: Workspace Multi-Painel (React + TypeScript)

```
┌─────────────────────────────────────────────────────────┐
│  🔍 Natural Language Search: "communication layer"      │
├──────────────────────┬──────────────────────────────────┤
│                      │                                  │
│   GRAPH VIEW         │   CODE VIEWER                    │
│   (React Flow)       │   (Monaco Editor, read-only)     │
│                      │                                  │
│   Interactive node   │   Source code + syntax highlight  │
│   graph. Click to    │   LLM annotations inline.        │
│   select. Search     │                                  │
│   highlights.        │                                  │
├──────────────────────┼──────────────────────────────────┤
│                      │                                  │
│   CHAT PANEL         │   LEARN PANEL                    │
│                      │                                  │
│   Context-aware Q&A  │   Tour mode + Contextual mode    │
│   about selected     │   Language lessons in context     │
│   nodes / project.   │   of YOUR code.                  │
│                      │                                  │
└──────────────────────┴──────────────────────────────────┘
```

**Stack tecnológica:**
- React 18 + TypeScript + Vite
- React Flow — visualização de grafo (feito para grafos de nós, melhor que D3 puro para isso)
- Monaco Editor — visualizador de código com syntax highlighting (o mesmo do VS Code)
- TailwindCSS — estilização
- Zustand — gerenciamento de estado (leve, sem boilerplate)

**Modos de persona:**
- Não-técnico: Nós conceituais de alto nível, visualizador de código oculto, painel de aprendizado expandido
- Dev júnior: Todos os painéis, painel de aprendizado em destaque, indicadores de complexidade
- Dev experiente: Visualizador de código em destaque, painel de chat para mergulhos profundos

**Busca em linguagem natural:**
- Pesquisa contra os campos `tags`, `summary` e `name` dos nós
- Usa similaridade por embeddings se disponível, recorre a casamento por palavras-chave
- Destaca nós correspondentes no grafo, filtra a lista

---

## Comandos da Skill do Claude Code

| Comando | Descrição |
|---------|-------------|
| `/understand` | Análise completa (ou atualização incremental se o grafo existir) + abre o dashboard |
| `/understand-chat "<query>"` | Q&A no terminal usando o grafo de conhecimento |
| `/understand-diff` | Analisa o PR/diff atual — explica mudanças, áreas afetadas, riscos |
| `/understand-explain <path>` | Explicação aprofundada de um arquivo ou função específica |
| `/understand-onboard` | Gera guia de onboarding estruturado para novos membros do time |

**Estratégia de LLM:**
- Dentro do Claude Code → usa a sessão ativa do Claude (custo extra zero)
- Dashboard standalone → usuários fornecem uma API key do Claude para recursos de chat
- Navegação no grafo, busca e modo de aprendizado funcionam offline (dados pré-gerados)

---

## Persistência e Detecção de Desatualização

```
.understand-anything/
├── knowledge-graph.json       # The full graph (committable)
├── meta.json                  # Analysis metadata
│   {
│     "lastAnalyzedAt": "2026-03-14T...",
│     "gitCommitHash": "abc123",
│     "version": "1.0.0",
│     "analyzedFiles": 47
│   }
├── cache/                     # Per-file analysis cache
│   ├── src__index.ts.json
│   └── src__auth__login.ts.json
└── tours/
    └── default-tour.json
```

**Fluxo de auto-sync:**
1. Skill inicia → lê `meta.json` → obtém o último hash de commit analisado
2. Roda `git diff <last-hash>..HEAD --name-only` → obtém arquivos alterados
3. Se não há mudanças → serve o grafo existente
4. Se há mudanças → re-analisa apenas os arquivos alterados → mescla no grafo existente → atualiza meta

---

## Sistema de Plugins

```typescript
interface AnalyzerPlugin {
  name: string;
  languages: string[];
  analyzeFile(filePath: string, content: string): StructuralAnalysis;
  resolveImports(filePath: string, content: string): ImportResolution[];
  extractCallGraph?(filePath: string, content: string): CallGraphEntry[];
}
```

**Dia 1: plugin tree-sitter** — usa `node-tree-sitter` com gramáticas de linguagem para:
- TypeScript/JavaScript, Python, Go, Java, Rust, C/C++
- Extrai: limites de função/classe, declarações de import/export, sites de chamada
- Combinado com análise de LLM para entendimento semântico

**Futuro: plugins da comunidade** para análise profunda específica de cada linguagem.

---

## Fases de Implementação

### Fase 1: Fundação (MVP)
1. Scaffolding do projeto — monorepo, config TypeScript, setup de build
2. Core: Esquema do grafo de conhecimento + persistência JSON
3. Core: Motor de análise por LLM (análise arquivo por arquivo via prompts)
4. Core: Integração tree-sitter para análise estrutural
5. Skill: comando `/understand` — analisa + persiste grafo
6. Dashboard: App React básico que lê e renderiza o grafo
7. Dashboard: Visualização de grafo com React Flow
8. Dashboard: Visualizador de código com Monaco Editor

### Fase 2: Inteligência
9. Busca em linguagem natural sobre os nós do grafo
10. Skill: `/understand-chat` — Q&A no terminal
11. Dashboard: Painel de chat com Q&A contextual
12. Detecção de desatualização + atualizações incrementais
13. Auto-detecção de camadas (agrupar nós em camadas lógicas)

### Fase 3: Modo Aprendizado
14. Geração de tour — passeio guiado pelo projeto
15. Explicações contextuais — clique para explicar
16. Lições específicas de linguagem no contexto do código do usuário
17. Modos de persona (não-técnico / júnior / experiente)

### Fase 4: Avançado
18. Skill: `/understand-diff` — análise de PR/diff
19. Skill: `/understand-explain` — mergulho profundo em arquivos específicos
20. Skill: `/understand-onboard` — geração de guia de onboarding
21. Sistema de plugins da comunidade
22. Busca semântica baseada em embeddings (melhoria opcional)

---

## Verificação

### Como testar de ponta a ponta:
1. **Análise da skill**: Rode `/understand` em um projeto de exemplo → verifique se `.understand-anything/knowledge-graph.json` foi gerado com o esquema correto
2. **Atualização incremental**: Modifique um arquivo → rode `/understand` de novo → verifique que apenas o arquivo alterado é re-analisado
3. **Dashboard**: Abra `http://localhost:5173` → verifique que o grafo renderiza, os nós são clicáveis, a busca funciona
4. **Chat**: Faça uma pergunta no painel de chat → verifique que retorna uma resposta relevante usando o grafo de conhecimento
5. **Modo aprendizado**: Inicie o tour → verifique que ele percorre o projeto passo a passo
6. **Tree-sitter**: Analise um arquivo TypeScript → verifique que os limites de função e as relações de import batem com o código real

### Projetos de teste para validar:
- Um projeto TypeScript pequeno (a própria ferramenta)
- Uma API Python Flask/Django
- Um microsserviço Go
- Um monorepo multi-linguagem
