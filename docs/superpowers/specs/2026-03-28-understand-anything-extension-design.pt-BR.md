# Understand Anything: Suporte Universal a Tipos de Arquivo

> 🇬🇧 Versão original em inglês: [2026-03-28-understand-anything-extension-design.md](./2026-03-28-understand-anything-extension-design.md)

**Data**: 2026-03-28
**Status**: Aprovado
**Abordagem**: Big Bang — todos os tipos de arquivo em um único release

## Objetivos

1. Estender o Understand Anything para analisar **qualquer** tipo de arquivo, não apenas código
2. Suportar tanto enriquecimento holístico do projeto (arquivos não-código enriquecem grafos de código) quanto análise standalone (repos só de docs, coleções de schema SQL, projetos IaC)
3. Manter compatibilidade retroativa com análise existente só de código

## Tipos de Arquivo Suportados (26 novos)

### Documentação (3)

| Tipo | Extensões | Parser | Tipos de Nó |
|------|-----------|--------|------------|
| Markdown | `.md`, `.mdx` | LLM + regex de extração de heading | `document` |
| reStructuredText | `.rst` | LLM | `document` |
| Texto puro | `.txt` | LLM | `document` |

### Configuração (5)

| Tipo | Extensões | Parser | Tipos de Nó |
|------|-----------|--------|------------|
| YAML | `.yaml`, `.yml` | pacote npm `yaml` | `config` |
| JSON | `.json`, `.jsonc` | `JSON.parse` / `jsonc-parser` | `config`, `schema` |
| TOML | `.toml` | `@iarna/toml` ou similar | `config` |
| .env | `.env`, `.env.*` | Parser de linha por regex | `config` |
| XML | `.xml` | LLM (opcionalmente `fast-xml-parser`) | `config` |

### Infraestrutura e DevOps (7)

| Tipo | Extensões | Parser | Tipos de Nó |
|------|-----------|--------|------------|
| Dockerfile | `Dockerfile`, `Dockerfile.*`, `.dockerfile` | Parser de instrução customizado | `service`, `pipeline` |
| Docker Compose | `docker-compose.yml`, `compose.yml` | Parser YAML + extração de serviço | `service` |
| Terraform | `.tf`, `.tfvars` | Parser de bloco por regex | `resource` |
| Kubernetes | YAML K8s (detectado por campo `apiVersion`) | YAML + detecção de kind | `service`, `resource` |
| GitHub Actions | `.github/workflows/*.yml` | YAML + extração de job/step | `pipeline` |
| Jenkinsfile | `Jenkinsfile` | LLM (DSL Groovy) | `pipeline` |
| Makefile | `Makefile`, `*.mk` | Parser de target por regex | `pipeline` |

### Dados e Schema (6)

| Tipo | Extensões | Parser | Tipos de Nó |
|------|-----------|--------|------------|
| SQL | `.sql` | Parser DDL simples | `table`, `endpoint` |
| GraphQL | `.graphql`, `.gql` | Parser de type/query por regex | `schema`, `endpoint` |
| OpenAPI/Swagger | `openapi.yaml`, `swagger.json` | YAML/JSON + extração de path | `endpoint`, `schema` |
| Protocol Buffers | `.proto` | Parser de message/service por regex | `schema` |
| JSON Schema | `*.schema.json` | JSON + extração de `$ref`/`$defs` | `schema` |
| CSV/TSV | `.csv`, `.tsv` | Extração de linha de header | `table` |

### Shell e Scripts (3)

| Tipo | Extensões | Parser | Tipos de Nó |
|------|-----------|--------|------------|
| Shell | `.sh`, `.bash`, `.zsh` | Parser de função por regex | `file`, `function` |
| PowerShell | `.ps1`, `.psm1` | LLM | `file`, `function` |
| Batch | `.bat`, `.cmd` | LLM | `file` |

### Markup (2)

| Tipo | Extensões | Parser | Tipos de Nó |
|------|-----------|--------|------------|
| HTML | `.html`, `.htm` | LLM (estrutura de tags) | `document` |
| CSS/SCSS/Less | `.css`, `.scss`, `.less` | LLM | `file` |

## Extensões de Esquema

### Novos Tipos de Nó (8)

Adicionados ao existente `file | function | class | module | concept`:

| Tipo de Nó | Propósito | Exemplo |
|-----------|---------|---------|
| `config` | Arquivos de configuração e settings-chave | `package.json`, `tsconfig.json`, env vars |
| `document` | Documentação, prosa, guias | `README.md`, docs de API |
| `service` | Serviços/containers deployáveis | Containers Docker, Deployments K8s |
| `table` | Tabelas de dados, objetos de banco | Tabelas SQL, datasets CSV |
| `endpoint` | Rotas de API, queries, mutations | Paths REST, queries GraphQL |
| `pipeline` | Workflows CI/CD, passos de build | Jobs do GitHub Actions, targets de Makefile |
| `schema` | Definições de tipo para troca de dados | Mensagens Protobuf, JSON Schema |
| `resource` | Recursos de infraestrutura | Resources Terraform, ConfigMaps K8s |

### Novos Tipos de Aresta (8)

Adicionados aos 18 tipos de aresta existentes:

| Tipo de Aresta | Categoria | Significado | Exemplo |
|-----------|----------|---------|---------|
| `deploys` | Infraestrutura | Serviço deploya código | Dockerfile -> código-fonte do app |
| `serves` | Infraestrutura | Serviço expõe endpoint | Service K8s -> endpoint de API |
| `migrates` | Fluxo de dados | Migration modifica tabela | Migration SQL -> tabela |
| `documents` | Semântica | Doc descreve código | README -> módulo |
| `provisions` | Infraestrutura | IaC cria recurso | Terraform -> recurso AWS |
| `routes` | Comportamental | Roteia tráfego para serviço | config nginx -> serviço |
| `defines_schema` | Fluxo de dados | Define forma de dado | Protobuf -> endpoint |
| `triggers` | Comportamental | Dispara pipeline/ação | Git push -> GitHub Actions |

### Aliases de Auto-Fix da Validação de Esquema

Novos aliases de tipo de nó:
- `container` -> `service`, `migration` -> `table`, `workflow` -> `pipeline`
- `route` -> `endpoint`, `doc` -> `document`, `setting` -> `config`, `infra` -> `resource`

Novos aliases de tipo de aresta:
- `describes` -> `documents`, `creates` -> `provisions`, `exposes` -> `serves`

## Mudanças na Arquitetura de Plugins

### Interface AnalyzerPlugin Generalizada

```typescript
interface AnalyzerPlugin {
  name: string;
  languages: string[];
  analyzeFile(filePath: string, content: string): StructuralAnalysis;
  resolveImports?(filePath: string, content: string): ImportResolution[];  // Now optional
  extractCallGraph?(filePath: string, content: string): CallGraphEntry[];
  extractReferences?(filePath: string, content: string): ReferenceResolution[];  // NEW
}

interface ReferenceResolution {
  source: string;      // File making the reference
  target: string;      // Referenced file or identifier
  type: string;        // Reference type: "file", "image", "schema", "service"
  line?: number;
}
```

### StructuralAnalysis Estendida

```typescript
interface StructuralAnalysis {
  // Existing (unchanged)
  functions: FunctionInfo[];
  classes: ClassInfo[];
  imports: ImportInfo[];
  exports: ExportInfo[];
  // New (all optional for backward compat)
  sections?: SectionInfo[];      // Documents: headings, chapters
  definitions?: DefinitionInfo[]; // Schemas: types, messages, tables
  services?: ServiceInfo[];      // Infra: containers, deployments
  endpoints?: EndpointInfo[];    // APIs: routes, queries
  steps?: StepInfo[];            // Pipelines: jobs, stages, targets
  resources?: ResourceInfo[];    // IaC: terraform resources, K8s objects
}
```

### Parsers Customizados (12)

Todos leves — em sua maioria baseados em regex, dependências mínimas:

| Parser | Implementação | Extrai |
|--------|---------------|----------|
| `MarkdownParser` | Regex | Headings, links, blocos de código, front matter |
| `YAMLParser` | npm `yaml` | Hierarquia de chaves, âncoras, multi-doc |
| `JSONParser` | `JSON.parse` built-in | Estrutura de chaves, `$ref`/`$defs` |
| `TOMLParser` | `@iarna/toml` | Estrutura de seção |
| `EnvParser` | Regex | Nomes e referências de variáveis |
| `DockerfileParser` | Regex | Estágios FROM, portas EXPOSE, fontes COPY |
| `SQLParser` | Regex | CREATE TABLE/VIEW/INDEX, colunas, foreign keys |
| `GraphQLParser` | Regex | Types, queries, mutations, subscriptions |
| `ProtobufParser` | Regex | Messages, services, enums, RPCs |
| `TerraformParser` | Regex | Resources, modules, variables, outputs |
| `MakefileParser` | Regex | Targets, dependências, variáveis |
| `ShellParser` | Regex | Funções, arquivos sourced |

## Mudanças no Pipeline de Agentes

### Project Scanner

1. Escanear TODOS os tipos de arquivo (remover filtro de só código)
2. Etiquetar cada arquivo com categoria: `code`, `config`, `docs`, `infra`, `data`, `script`, `markup`
3. Agrupamento de batch inteligente: manter arquivos relacionados juntos (ex.: Dockerfile + docker-compose.yml)

### File Analyzer

Templates de prompt cientes do tipo por categoria:

- **Code**: Comportamento atual (funções, classes, imports, call graph)
- **Config**: Extrair settings-chave, o que configuram, quais arquivos de código afetam
- **Documentation**: Extrair seções, conceitos-chave, quais componentes de código são documentados
- **Infrastructure**: Extrair serviços, portas, volumes, dependências, qual código eles deployam
- **Data/Schema**: Extrair tabelas, colunas, tipos, relações, qual código consome esses dados
- **Pipelines**: Extrair jobs, steps, triggers, qual código/infra eles fazem build/deploy

### Resolução de Referência Cross-Type

Passo pós-análise conectando:
- `COPY` de Dockerfile -> diretórios de código-fonte
- `run: npm test` em config CI -> arquivos de teste
- `image:` em manifest K8s -> Dockerfile
- Foreign keys SQL -> outras tabelas
- `$ref` OpenAPI -> definições de schema
- Links Markdown -> arquivos referenciados

### Architecture Analyzer

Nova detecção de padrões:
- Topologia de deployment: cadeia Dockerfile -> compose -> K8s
- Fluxo de dados: Schema -> migration -> endpoint de API -> código cliente
- Cobertura de documentação: quais módulos têm docs vs. não
- Dependência de configuração: quais arquivos de config afetam quais caminhos de código

### Tour Builder

Incluir paradas de tour não-código:
- Visão geral do README do projeto
- Containerização via Dockerfile
- Schema de banco via migration SQL
- Explicação do pipeline CI/CD

## Visualização no Dashboard

### Novos Estilos Visuais de Nó

| Tipo de Nó | Forma | Cor | Ícone |
|-----------|-------|-------|------|
| `config` | Rounded rect | Teal (#5eead4) | Engrenagem |
| `document` | Rounded rect | Azul céu (#7dd3fc) | Documento |
| `service` | Hexágono | Violeta (#a78bfa) | Container/Caixa |
| `table` | Retângulo | Esmeralda (#6ee7b7) | Grade |
| `endpoint` | Pílula/Estádio | Laranja (#fdba74) | Seta-direita |
| `pipeline` | Rounded rect | Rosa (#fda4af) | Play/Workflow |
| `schema` | Diamante | Âmbar (#fcd34d) | Blueprint |
| `resource` | Forma de nuvem | Índigo (#a5b4fc) | Nuvem |

### Layout do Grafo

1. Agrupamento por camada conforme categoria — nós não-código se agrupam separados dos nós de código
2. Atualização da legenda com 8 novos tipos de nó
3. Controles de filtro — checkboxes para mostrar/esconder cada categoria de arquivo

### Melhorias na Sidebar

Atualizações do painel NodeInfo por tipo de nó:
- **Config**: pares chave-valor, referenciando arquivos de código
- **Document**: outline de headings, componentes de código linkados
- **Service**: portas, volumes, dependências, código deployado
- **Table**: colunas, tipos, relações de foreign key
- **Endpoint**: método HTTP, path, schema de request/response
- **Pipeline**: jobs, triggers, targets deployados
- **Schema**: campos, tipos aninhados, consumidores
- **Resource**: provider, tipo, dependências

Painel ProjectOverview: adicionar quebra "File Types" (distribuição código vs. não-código).

## Novas Dependências

- `yaml` — parseamento YAML (já comum, ~50KB)
- `@iarna/toml` — parseamento TOML (~30KB)
- `jsonc-parser` — JSON com comentários (~20KB)

Sem adições de tree-sitter WASM. Todos os outros parsers são baseados em regex com zero dependências.

## Compatibilidade Retroativa

- Todos os novos campos de `StructuralAnalysis` são opcionais
- `resolveImports` se torna opcional em `AnalyzerPlugin`
- Entradas existentes de `LanguageConfig` inalteradas
- Validação de esquema auto-corrige os novos aliases de tipo
- Grafos de conhecimento existentes permanecem válidos (novos tipos são aditivos)
