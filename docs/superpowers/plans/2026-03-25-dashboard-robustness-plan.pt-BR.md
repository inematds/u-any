# Design: Robustez do Dashboard — Carregamento Permissivo do Grafo

> 🇬🇧 Versão original em inglês: [2026-03-25-dashboard-robustness-plan.md](./2026-03-25-dashboard-robustness-plan.md)

## Problema

Quando o agente LLM produz um `knowledge-graph.json` que desvia do schema Zod estrito, o dashboard exibe uma tela em branco com paths de erro Zod crípticos. Os usuários não sabem se é bug de sistema ou problema de geração do agente, e o único recurso é rodar `/understand` do zero.

## Objetivos

1. **Maximizar o que o usuário pode ver** — carregar nodes/edges válidos mesmo que alguns estejam quebrados
2. **Comunicar claramente problemas de geração** — avisos âmbar (não erros vermelhos) com mensagens prontas para copiar e colar
3. **Capacitar correções pontuais** — usuários podem copiar o relatório de issues e pedir ao agente para corrigir problemas específicos em vez de rodar tudo de novo

## Design

### Pipeline de Robustez em Três Camadas

```
Raw JSON → Sanitize (Tier 1) → Normalize + Auto-fix (Tier 2) → Validate per-item (Tier 3) → Fatal check (Tier 4) → Dashboard
```

### Tier 1: Sanitizar Silenciosamente

Manias comuns de LLM que são puro ruído — corrigir sem reportar.

| Issue | Correção |
|-------|----------|
| `null` em campos opcionais (`filePath`, `lineRange`, `description`, `languageNotes`) | Converter para `undefined` |
| Strings de enum com case misto (`"Forward"`, `"SIMPLE"`) | Lowercase antes de casar |

### Tier 2: Auto-correção Com Aviso Informativo

Problemas recuperáveis — aplicar defaults sensatos, rastrear como issues `auto-corrected`.

| Issue | Default | Notas |
|-------|---------|-------|
| `complexity` ausente | `"moderate"` | Omissão mais comum do LLM |
| `tags` ausente | `[]` | Vazio é válido |
| `weight` ausente | `0.5` | Meio do range 0–1 |
| `weight` como string | Coerção para number | ex.: `"0.8"` → `0.8` |
| `direction` ausente | `"forward"` | Default seguro |
| `summary` ausente | Usar `name` do node | Melhor que vazio |
| `tour: null` / `layers: null` | `[]` | Null vs array vazio |
| Aliases de complexity | `low/easy→simple`, `medium/intermediate→moderate`, `high/hard→complex` | |
| Aliases de direction | `to/outbound→forward`, `from/inbound→backward`, `both→bidirectional` | |
| Aliases de tipo de node/edge existentes | Já tratados por `normalizeGraph` | Sem mudança |
| `type` ausente no node | `"file"` | Fallback seguro |
| `type` ausente na edge | `"depends_on"` | Fallback genérico |

### Tier 3: Descartar Com Aviso

Não dá pra adivinhar com segurança — remover o item, rastrear como issue `dropped`.

| Issue | Ação |
|-------|------|
| Edge referencia ID de node inexistente | Descartar edge |
| Node sem `id` | Descartar node |
| Node sem `name` | Descartar node |
| Edge sem `source` ou `target` | Descartar edge |
| Valor de `type` irreconhecível (fora da lista canônica ou de aliases) | Descartar item |
| `weight` não coercível para number | Descartar edge |

### Tier 4: Fatal

Grafo irrecuperável — mostrar banner vermelho de erro.

| Condição | Mensagem |
|----------|----------|
| 0 nodes válidos após filtragem | "No valid nodes found in knowledge graph" |
| Metadados de `project` totalmente ausentes | "Missing project metadata" |
| Input não é objeto / não é JSON válido | "Invalid input format" |

### Tipo de Retorno

```typescript
interface GraphIssue {
  level: 'auto-corrected' | 'dropped' | 'fatal';
  category: string;      // e.g., "missing-field", "invalid-reference", "type-coercion"
  message: string;       // human-readable, copy-paste friendly
  path?: string;         // e.g., "nodes[3].complexity"
}

interface ValidationResult {
  success: boolean;
  data?: KnowledgeGraph;
  issues: GraphIssue[];
  fatal?: string;
}
```

### UI do Dashboard: Componente `WarningBanner`

**Novo componente** em `packages/dashboard/src/components/WarningBanner.tsx`.

**Design visual:**
- **Tema âmbar/dourado** — `bg-amber-900/20`, `border-amber-700`, `text-amber-200`
- Combina com a estética de accent dourado do dashboard; sinaliza "problema de qualidade na geração", não "crash de sistema"
- **Colapsado por padrão** — linha de resumo: "Knowledge graph loaded with 5 auto-corrections and 2 dropped items"
- **Expansível** — clicar revela a lista de issues categorizada
- **Botão de copiar** — um clique copia o relatório completo de issues pré-formatado
- **Rodapé acionável** — orienta o usuário a copiar as issues e pedir ao agente para corrigir

**Formato de saída para copiar/colar:**
```
The following issues were found in your knowledge-graph.json.
These are LLM generation errors — not a system bug.
You can ask your agent to fix these specific issues in the knowledge-graph.json file:

[Auto-corrected] nodes[3] ("AuthService"): missing "complexity" — defaulted to "moderate"
[Auto-corrected] nodes[7] ("utils.ts"): missing "tags" — defaulted to []
[Auto-corrected] edges[12]: weight was string "0.8" — coerced to number
[Dropped] edges[5]: target "file:src/nonexistent.ts" does not exist in nodes
[Dropped] nodes[14]: missing required "id" field — cannot recover
```

**Erros fatais** continuam vermelhos (`bg-red-900/30`) com mensagem: "Knowledge graph is unsalvageable: [reason]. Please re-run `/understand` to generate a new one."

**O banner vermelho existente** para erros de rede/parse JSON permanece como está (esses SÃO problemas de sistema/infra).

### Mudanças em `App.tsx`

- Em `result.success === true` com `result.issues.length > 0`: mostrar `WarningBanner` com as issues, carregar o grafo normalmente
- Em `result.fatal`: mostrar o banner vermelho existente com a mensagem fatal
- `console.warn` para itens auto-corrigidos, `console.error` para itens descartados

### Cobertura de Testes

Tudo em `packages/core/src/__tests__/schema.test.ts`:

- **Tier 1:** campos opcionais `null` viram `undefined` silenciosamente
- **Tier 2:** `complexity`/`tags`/`weight`/`direction`/`summary` ausentes recebem defaults; issues rastreadas
- **Tier 2:** `weight` string é coercido; aliases de complexity/direction são mapeados
- **Tier 3:** referências de edge pendentes são descartadas; nodes sem `id` descartados; issues registradas
- **Tier 4:** grafo vazio após filtragem → fatal; `project` ausente → fatal
- **Integração:** grafo com mix de nodes bons/ruins → carrega com count correto e lista correta de issues

### Arquivos Modificados

| Arquivo | Mudança |
|---------|---------|
| `packages/core/src/schema.ts` | Sanitize, normalize expandido, validate permissivo, novos tipos |
| `packages/dashboard/src/components/WarningBanner.tsx` | Novo componente |
| `packages/dashboard/src/App.tsx` | Conectar issues ao `WarningBanner` |
| `packages/core/src/__tests__/schema.test.ts` | Testes para todas as tiers |

### Arquivos NÃO Modificados

- Prompts dos agentes (podem ser apertados depois como esforço separado)
- Lógica do `GraphView` / store (já tratam objetos `KnowledgeGraph` válidos)
- Mapas de alias de tipo de node/edge existentes (preservados, estendidos ao redor)
