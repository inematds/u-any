# Plano de Implementação da Arquitetura de Extratores Específicos por Linguagem

> 🇬🇧 Versão original em inglês: [2026-04-15-language-extractors-impl.md](./2026-04-15-language-extractors-impl.md)

> **Para workers agênticos:** SUB-SKILL OBRIGATÓRIA: Use superpowers:subagent-driven-development (recomendado) ou superpowers:executing-plans para implementar este plano tarefa a tarefa. Os passos usam sintaxe de checkbox (`- [ ]`) para acompanhamento.

**Objetivo:** (1) Desacoplar a lógica de extração de AST dos tipos de nó específicos de TS/JS para que 8 linguagens de código adicionais (Python, Go, Rust, Java, Ruby, PHP, C/C++, C#) ganhem análise estrutural com tree-sitter. Swift e Kotlin estão excluídos — não há pacotes de gramática WASM disponíveis. (2) Substituir a geração ad-hoc de scripts regex feita pelo agente file-analyzer por um script de extração tree-sitter pré-construído e determinístico.

**Arquitetura:** Introduzir uma interface `LanguageExtractor` que cada linguagem implementa. O `TreeSitterPlugin` delega a extração ao extractor registrado para a linguagem do arquivo. Um script `extract-structure.mjs` empacotado em `skills/understand/` usa o `PluginRegistry` (que inclui tanto o `TreeSitterPlugin` quanto os parsers não-código) para fornecer extração estrutural determinística para o agente file-analyzer — substituindo a abordagem atual em que o LLM escreve scripts regex descartáveis a cada execução.

**Stack:** web-tree-sitter (WASM), TypeScript, Vitest

---

## Estrutura de Arquivos

```
packages/core/src/plugins/
├── extractors/
│   ├── types.ts              # LanguageExtractor interface + TreeSitterNode re-export
│   ├── base-extractor.ts     # Shared utilities (traverse, getStringValue)
│   ├── typescript-extractor.ts  # TS/JS (moved from tree-sitter-plugin.ts)
│   ├── python-extractor.ts
│   ├── go-extractor.ts
│   ├── rust-extractor.ts
│   ├── java-extractor.ts
│   ├── ruby-extractor.ts
│   ├── php-extractor.ts
│   ├── cpp-extractor.ts
│   ├── csharp-extractor.ts
│   └── index.ts              # builtinExtractors array + re-exports
├── tree-sitter-plugin.ts     # Refactored to use extractors
└── tree-sitter-plugin.test.ts  # Existing tests (should still pass)

packages/core/src/plugins/__tests__/
└── extractors.test.ts        # Tests for all new extractors

skills/understand/
├── extract-structure.mjs     # Pre-built tree-sitter extraction script (NEW)
└── SKILL.md                  # Updated to reference extract-structure.mjs

agents/
└── file-analyzer.md          # Phase 1 rewritten to execute pre-built script
```

---

### Tarefa 1: Criar interface LanguageExtractor e utilitários compartilhados

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/types.ts`
- Criar: `packages/core/src/plugins/extractors/base-extractor.ts`

- [ ] **Passo 1: Criar a interface do extractor**

```typescript
// packages/core/src/plugins/extractors/types.ts
import type { StructuralAnalysis, CallGraphEntry } from "../../types.js";

// Re-export the tree-sitter Node type for use by extractors
export type TreeSitterNode = import("web-tree-sitter").Node;

/**
 * Language-specific extractor that maps a tree-sitter AST
 * to the common StructuralAnalysis / CallGraphEntry types.
 */
export interface LanguageExtractor {
  /** Language IDs this extractor handles (must match LanguageConfig.id) */
  languageIds: string[];

  /** Extract functions, classes, imports, exports from the root AST node */
  extractStructure(rootNode: TreeSitterNode): StructuralAnalysis;

  /** Extract caller→callee relationships from the root AST node */
  extractCallGraph(rootNode: TreeSitterNode): CallGraphEntry[];
}
```

- [ ] **Passo 2: Criar base-extractor com utilitários compartilhados**

Mova `traverse()` e `getStringValue()` de `tree-sitter-plugin.ts` para um módulo compartilhado:

```typescript
// packages/core/src/plugins/extractors/base-extractor.ts
import type { TreeSitterNode } from "./types.js";

/** Recursively traverse an AST tree, calling the visitor for each node. */
export function traverse(
  node: TreeSitterNode,
  visitor: (node: TreeSitterNode) => void,
): void {
  visitor(node);
  for (let i = 0; i < node.childCount; i++) {
    const child = node.child(i);
    if (child) traverse(child, visitor);
  }
}

/** Extract the unquoted string value from a string-like node. */
export function getStringValue(node: TreeSitterNode): string {
  for (let i = 0; i < node.childCount; i++) {
    const child = node.child(i);
    if (child && child.type === "string_fragment") {
      return child.text;
    }
  }
  return node.text.replace(/^['"`]|['"`]$/g, "");
}

/** Find the first child matching a type. */
export function findChild(node: TreeSitterNode, type: string): TreeSitterNode | null {
  for (let i = 0; i < node.childCount; i++) {
    const child = node.child(i);
    if (child && child.type === type) return child;
  }
  return null;
}

/** Find all children matching a type. */
export function findChildren(node: TreeSitterNode, type: string): TreeSitterNode[] {
  const result: TreeSitterNode[] = [];
  for (let i = 0; i < node.childCount; i++) {
    const child = node.child(i);
    if (child && child.type === type) result.push(child);
  }
  return result;
}

/** Check if a node has a child of the given type (used for export/visibility checks). */
export function hasChildOfType(node: TreeSitterNode, type: string): boolean {
  for (let i = 0; i < node.childCount; i++) {
    const child = node.child(i);
    if (child && child.type === type) return true;
  }
  return false;
}
```

- [ ] **Passo 3: Commit**

```bash
git add packages/core/src/plugins/extractors/types.ts packages/core/src/plugins/extractors/base-extractor.ts
git commit -m "feat: add LanguageExtractor interface and shared base utilities"
```

---

### Tarefa 2: Mover lógica de extração TS/JS para TypeScriptExtractor

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/typescript-extractor.ts`
- Modificar: `packages/core/src/plugins/tree-sitter-plugin.ts`

Esta é uma refatoração pura. Todos os testes existentes devem continuar passando com zero alterações.

- [ ] **Passo 1: Criar TypeScriptExtractor**

Mova todos os métodos de extração específicos de TS/JS (`extractFunction`, `extractClass`, `extractVariableDeclarations`, `extractImport`, `processExportStatement`, `extractParams`, `extractReturnType`, `extractImportSpecifiers`, e o walker do grafo de chamadas) de `tree-sitter-plugin.ts` para `typescript-extractor.ts`, implementando a interface `LanguageExtractor`.

Os `languageIds` devem ser `["typescript", "javascript"]`. NÃO inclua `"tsx"` — é uma chave sintética interna ao `TreeSitterPlugin` para seleção de gramática, não um `LanguageConfig.id`. O mapeamento tsx→typescript é tratado em `getExtractor()` abaixo.

- [ ] **Passo 2: Refatorar TreeSitterPlugin para usar extractors**

Substitua a lógica de extração hardcoded em `TreeSitterPlugin` por despacho via extractor:

```typescript
// In TreeSitterPlugin
private extractors = new Map<string, LanguageExtractor>();

registerExtractor(extractor: LanguageExtractor): void {
  for (const id of extractor.languageIds) {
    this.extractors.set(id, extractor);
  }
}

private getExtractor(langKey: string): LanguageExtractor | null {
  // tsx is a synthetic grammar key — extraction logic is identical to typescript
  const key = langKey === "tsx" ? "typescript" : langKey;
  return this.extractors.get(key) ?? null;
}
```

O método `analyzeFile()` torna-se:

```typescript
analyzeFile(filePath: string, content: string): StructuralAnalysis {
  const parser = this.getParser(filePath);
  if (!parser) return { functions: [], classes: [], imports: [], exports: [] };

  const tree = parser.parse(content);
  if (!tree) { parser.delete(); return { functions: [], classes: [], imports: [], exports: [] }; }

  const langKey = this.languageKeyFromPath(filePath);
  const extractor = langKey ? this.getExtractor(langKey) : null;

  let result: StructuralAnalysis;
  if (extractor) {
    result = extractor.extractStructure(tree.rootNode);
  } else {
    result = { functions: [], classes: [], imports: [], exports: [] };
  }

  tree.delete();
  parser.delete();
  return result;
}
```

O método `extractCallGraph()` segue o mesmo padrão — o ciclo de vida do parser deve ser gerenciado identicamente:

```typescript
extractCallGraph(filePath: string, content: string): CallGraphEntry[] {
  const parser = this.getParser(filePath);
  if (!parser) return [];

  const tree = parser.parse(content);
  if (!tree) { parser.delete(); return []; }

  const langKey = this.languageKeyFromPath(filePath);
  const extractor = langKey ? this.getExtractor(langKey) : null;
  const result = extractor ? extractor.extractCallGraph(tree.rootNode) : [];

  tree.delete();
  parser.delete();
  return result;
}
```

O construtor deve aceitar um array opcional `extractors` e registrá-los. Se nenhum for fornecido, registre o `TypeScriptExtractor` embutido para compatibilidade retroativa.

- [ ] **Passo 3: Executar testes existentes para verificar zero mudança de comportamento**

Execute: `pnpm --filter @understand-anything/core test`
Esperado: Todos os 426 testes passam (idênticos aos anteriores)

- [ ] **Passo 4: Commit**

```bash
git add packages/core/src/plugins/extractors/typescript-extractor.ts packages/core/src/plugins/tree-sitter-plugin.ts
git commit -m "refactor: move TS/JS extraction logic to TypeScriptExtractor, dispatch via LanguageExtractor interface"
```

---

### Tarefa 2.5: Adicionar extractCallGraph ao PluginRegistry e atualizar DEFAULT_PLUGIN_CONFIG

**Arquivos:**
- Modificar: `packages/core/src/plugins/registry.ts`
- Modificar: `packages/core/src/plugins/discovery.ts`

**Contexto:** O `PluginRegistry` atualmente expõe apenas `analyzeFile` e `resolveImports` — não tem `extractCallGraph`. O script `extract-structure.mjs` (Tarefa 13) precisa de dados do grafo de chamadas através do registry. Além disso, o `DEFAULT_PLUGIN_CONFIG` hardcoda `["typescript", "javascript"]`, o que precisa refletir todas as linguagens suportadas.

- [ ] **Passo 1: Adicionar extractCallGraph ao PluginRegistry**

```typescript
// In PluginRegistry (registry.ts)
extractCallGraph(filePath: string, content: string): CallGraphEntry[] | null {
  const plugin = this.getPluginForFile(filePath);
  if (!plugin?.extractCallGraph) return null;
  return plugin.extractCallGraph(filePath, content);
}
```

- [ ] **Passo 2: Atualizar DEFAULT_PLUGIN_CONFIG para derivar linguagens dinamicamente**

Em `discovery.ts`, substitua o hardcoded `["typescript", "javascript"]` por uma derivação dinâmica de `builtinLanguageConfigs`:

```typescript
import { builtinLanguageConfigs } from "../languages/configs/index.js";

export const DEFAULT_PLUGIN_CONFIG: PluginConfig = {
  plugins: [
    {
      name: "tree-sitter",
      enabled: true,
      languages: builtinLanguageConfigs
        .filter((c) => c.treeSitter)
        .map((c) => c.id),
    },
  ],
};
```

- [ ] **Passo 3: Executar testes, commit**

```bash
pnpm --filter @understand-anything/core test
git add packages/core/src/plugins/registry.ts packages/core/src/plugins/discovery.ts
git commit -m "feat: add extractCallGraph to PluginRegistry, derive DEFAULT_PLUGIN_CONFIG from configs"
```

---

### Tarefa 3: Adicionar dependências npm e configs treeSitter para as 10 linguagens

**Arquivos:**
- Modificar: `packages/core/package.json` (adicionar 8 deps: python, go, rust, java, ruby, php, cpp, c-sharp)
- Modificar: 10 arquivos de config em `packages/core/src/languages/configs/`

- [ ] **Passo 1: Adicionar dependências de gramáticas tree-sitter ao package.json**

Adicione em `dependencies`:

```json
"tree-sitter-c-sharp": "^0.23.1",
"tree-sitter-cpp": "^0.23.4",
"tree-sitter-go": "^0.25.0",
"tree-sitter-java": "^0.23.5",
"tree-sitter-php": "^0.23.11",
"tree-sitter-python": "^0.25.0",
"tree-sitter-ruby": "^0.23.1",
"tree-sitter-rust": "^0.24.0"
```

Então execute `pnpm install`.

- [ ] **Passo 2: Adicionar campo treeSitter a todas as 10 configs de linguagem**

Cada config recebe um bloco `treeSitter`. Exemplos:

```typescript
// python.ts
treeSitter: { wasmPackage: "tree-sitter-python", wasmFile: "tree-sitter-python.wasm" },

// go.ts
treeSitter: { wasmPackage: "tree-sitter-go", wasmFile: "tree-sitter-go.wasm" },

// rust.ts
treeSitter: { wasmPackage: "tree-sitter-rust", wasmFile: "tree-sitter-rust.wasm" },

// java.ts
treeSitter: { wasmPackage: "tree-sitter-java", wasmFile: "tree-sitter-java.wasm" },

// ruby.ts
treeSitter: { wasmPackage: "tree-sitter-ruby", wasmFile: "tree-sitter-ruby.wasm" },

// php.ts
treeSitter: { wasmPackage: "tree-sitter-php", wasmFile: "tree-sitter-php.wasm" },

// cpp.ts
treeSitter: { wasmPackage: "tree-sitter-cpp", wasmFile: "tree-sitter-cpp.wasm" },

// csharp.ts
treeSitter: { wasmPackage: "tree-sitter-c-sharp", wasmFile: "tree-sitter-c_sharp.wasm" },
```

Nota: as configs Swift e Kotlin NÃO são alteradas (sem pacotes WASM disponíveis).

- [ ] **Passo 3: Executar pnpm install e verificar que os arquivos WASM resolvem**

```bash
pnpm install
node -e "const r=require('module').createRequire(import.meta.url??__filename); console.log(r.resolve('tree-sitter-python/tree-sitter-python.wasm'))"
```

- [ ] **Passo 4: Commit**

```bash
git add packages/core/package.json pnpm-lock.yaml packages/core/src/languages/configs/
git commit -m "feat: add tree-sitter grammar deps and treeSitter configs for 10 languages"
```

---

### Tarefa 4: Criar extractor de Python

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/python-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de Python**

Tipos-chave de nós tree-sitter para Python:
- Functions: `function_definition` (name, parameters, return_type)
- Classes: `class_definition` (name, body → methods + assignments as properties)
- Imports: `import_statement`, `import_from_statement`
- Decorated: `decorated_definition` wrapping function_definition or class_definition
- Calls: `call` (function field)
- No formal exports (all top-level names are "exported")

```typescript
languageIds: ["python"]
```

- [ ] **Passo 2: Escrever testes para o extractor de Python**

Teste com código Python representativo:

```python
import os
from pathlib import Path
from typing import Optional

class DataProcessor:
    name: str
    
    def __init__(self, name: str):
        self.name = name
    
    def process(self, data: list) -> dict:
        return transform(data)

def helper(x: int) -> str:
    return str(x)

@decorator
def decorated_func():
    pass
```

Verificar: 2 funções (helper, decorated_func), 1 classe (DataProcessor com métodos __init__/process e propriedade name), 3 imports, grafo de chamadas (process→transform).

- [ ] **Passo 3: Executar testes**

Execute: `pnpm --filter @understand-anything/core test`

- [ ] **Passo 4: Commit**

---

### Tarefa 5: Criar extractor de Go

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/go-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de Go**

Tipos-chave de nós tree-sitter para Go:
- Functions: `function_declaration` (name, parameter_list, result)
- Methods: `method_declaration` (receiver, name, parameter_list, result)
- Structs: `type_declaration` → `type_spec` → `struct_type`
- Interfaces: `type_declaration` → `type_spec` → `interface_type`
- Imports: `import_declaration` → `import_spec_list` → `import_spec`
- Exports: capitalized first letter of name
- Calls: `call_expression` (function field)

```typescript
languageIds: ["go"]
```

- [ ] **Passo 2: Escrever testes**

Teste com:
```go
package main

import (
    "fmt"
    "os"
)

type Server struct {
    Host string
    Port int
}

func (s *Server) Start() error {
    fmt.Println("starting")
    return nil
}

func NewServer(host string, port int) *Server {
    return &Server{Host: host, Port: port}
}
```

Verificar: 2 funções (Start, NewServer), 1 classe/struct (Server com método Start, propriedades Host/Port), 2 imports, exports (Server, Start, NewServer — todos capitalizados), grafo de chamadas (Start→fmt.Println).

- [ ] **Passo 3: Executar testes e fazer commit**

---

### Tarefa 6: Criar extractor de Rust

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/rust-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de Rust**

Tipos-chave de nós tree-sitter para Rust:
- Functions: `function_item` (name, parameters, return_type via `->`)
- Structs: `struct_item` (name, field_declaration_list)
- Enums: `enum_item`
- Impl blocks: `impl_item` (type, body containing function_items)
- Traits: `trait_item`
- Imports: `use_declaration` (scoped_identifier, use_list, use_wildcard)
- Exports: `visibility_modifier` containing `pub`
- Calls: `call_expression` (function field)

```typescript
languageIds: ["rust"]
```

- [ ] **Passo 2: Escrever testes**

Teste com:
```rust
use std::collections::HashMap;
use std::io::{self, Read};

pub struct Config {
    name: String,
    port: u16,
}

impl Config {
    pub fn new(name: String, port: u16) -> Self {
        Config { name, port }
    }

    fn validate(&self) -> bool {
        check_port(self.port)
    }
}

pub fn check_port(port: u16) -> bool {
    port > 0
}
```

Verificar: 3 funções (new, validate, check_port), 1 classe/struct (Config com métodos new/validate, propriedades name/port), 2 imports, exports (Config, new, check_port — aqueles com `pub`), grafo de chamadas (validate→check_port).

- [ ] **Passo 3: Executar testes e fazer commit**

---

### Tarefa 7: Criar extractor de Java

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/java-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de Java**

Tipos-chave de nós tree-sitter para Java:
- Methods: `method_declaration` (name, formal_parameters, type/dimensions)
- Constructors: `constructor_declaration` (name, formal_parameters)
- Classes: `class_declaration` (name, class_body)
- Interfaces: `interface_declaration`
- Fields: `field_declaration` (declarator → variable_declarator → identifier)
- Imports: `import_declaration` (scoped_identifier)
- Exports: `public` modifier (modifiers node)
- Calls: `method_invocation` (name, object, arguments)

```typescript
languageIds: ["java"]
```

- [ ] **Passo 2: Escrever testes com código Java representativo, executar, fazer commit**

---

### Tarefa 8: Criar extractor de Ruby

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/ruby-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de Ruby**

Tipos-chave de nós tree-sitter para Ruby:
- Methods: `method` (name, parameters)
- Classes: `class` (name, body containing methods)
- Modules: `module` (name)
- Imports: `call` where method is `require` or `require_relative` (Ruby uses method calls for imports)
- Calls: `call` (method, receiver, arguments)
- No formal export syntax

```typescript
languageIds: ["ruby"]
```

- [ ] **Passo 2: Escrever testes, executar, fazer commit**

---

### Tarefa 9: Criar extractor de PHP

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/php-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de PHP**

Tipos-chave de nós tree-sitter para PHP:
- Functions: `function_definition` (name, formal_parameters, return_type)
- Methods: `method_declaration` (name, formal_parameters, return_type)
- Classes: `class_declaration` (name, declaration_list)
- Imports: `namespace_use_declaration` (namespace_use_clause)
- Calls: `function_call_expression` / `member_call_expression`
- Note: PHP tree wraps everything in a `program` → `php_tag` + statements

```typescript
languageIds: ["php"]
```

- [ ] **Passo 2: Escrever testes, executar, fazer commit**

---

### Tarefa 10: Criar extractor de C/C++

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/cpp-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de C/C++**

Tipos-chave de nós tree-sitter para C/C++:
- Functions: `function_definition` (declarator → function_declarator → identifier + parameter_list)
- Classes: `class_specifier` (name, body → field_declaration_list)
- Structs: `struct_specifier` (name, body)
- Includes: `preproc_include` (path → string_literal or system_lib_string)
- Namespaces: `namespace_definition`
- Calls: `call_expression` (function, arguments)

Nota: assinaturas de funções em C/C++ são aninhadas (o nome fica dentro de um `function_declarator` dentro do campo `declarator`).

A `cppConfig` tem `id: "cpp"` e `extensions: [".cpp", ".cc", ".cxx", ".c", ".h", ".hpp", ".hxx"]`. Arquivos C puros (`.c`, `.h`) são parseados com a gramática C++, o que funciona mas não produz tipos de nós específicos de C++ como `class_specifier`. O extractor deve tratar essa ausência graciosamente (retornar arrays vazios para classes ao parsear C puro).

```typescript
languageIds: ["cpp"]
```

- [ ] **Passo 2: Escrever testes para C++ e código C puro, executar, fazer commit**

---

### Tarefa 11: Criar extractor de C#

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/csharp-extractor.ts`

- [ ] **Passo 1: Escrever o extractor de C#**

Tipos-chave de nós tree-sitter para C#:
- Methods: `method_declaration` (name, parameter_list, return type)
- Constructors: `constructor_declaration`
- Classes: `class_declaration` (name, declaration_list)
- Interfaces: `interface_declaration`
- Properties: `property_declaration` (name, type)
- Imports: `using_directive` (qualified_name)
- Calls: `invocation_expression` (identifier/member_access, argument_list)

```typescript
languageIds: ["csharp"]
```

- [ ] **Passo 2: Escrever testes, executar, fazer commit**

---

### Tarefa 12: Criar índice de extractors e conectar ao TreeSitterPlugin

**Arquivos:**
- Criar: `packages/core/src/plugins/extractors/index.ts`
- Modificar: `packages/core/src/plugins/tree-sitter-plugin.ts` (importar builtinExtractors)

- [ ] **Passo 1: Criar index.ts exportando todos os extractors**

```typescript
// packages/core/src/plugins/extractors/index.ts
export type { LanguageExtractor, TreeSitterNode } from "./types.js";
export { traverse, getStringValue, findChild, findChildren, hasChildOfType } from "./base-extractor.js";
export { TypeScriptExtractor } from "./typescript-extractor.js";
export { PythonExtractor } from "./python-extractor.js";
export { GoExtractor } from "./go-extractor.js";
export { RustExtractor } from "./rust-extractor.js";
export { JavaExtractor } from "./java-extractor.js";
export { RubyExtractor } from "./ruby-extractor.js";
export { PhpExtractor } from "./php-extractor.js";
export { CppExtractor } from "./cpp-extractor.js";
export { CSharpExtractor } from "./csharp-extractor.js";

import type { LanguageExtractor } from "./types.js";
import { TypeScriptExtractor } from "./typescript-extractor.js";
import { PythonExtractor } from "./python-extractor.js";
import { GoExtractor } from "./go-extractor.js";
import { RustExtractor } from "./rust-extractor.js";
import { JavaExtractor } from "./java-extractor.js";
import { RubyExtractor } from "./ruby-extractor.js";
import { PhpExtractor } from "./php-extractor.js";
import { CppExtractor } from "./cpp-extractor.js";
import { CSharpExtractor } from "./csharp-extractor.js";

export const builtinExtractors: LanguageExtractor[] = [
  new TypeScriptExtractor(),
  new PythonExtractor(),
  new GoExtractor(),
  new RustExtractor(),
  new JavaExtractor(),
  new RubyExtractor(),
  new PhpExtractor(),
  new CppExtractor(),
  new CSharpExtractor(),
];
```

- [ ] **Passo 2: Conectar builtinExtractors ao construtor do TreeSitterPlugin**

Quando nenhum extractor for fornecido, use `builtinExtractors` por padrão.

- [ ] **Passo 3: Executar suíte completa de testes**

Execute: `pnpm --filter @understand-anything/core test`
Esperado: Todos os testes passam (existentes + novos testes de extractors)

- [ ] **Passo 4: Commit**

---

### Tarefa 13: Criar script empacotado extract-structure.mjs

**Arquivos:**
- Criar: `skills/understand/extract-structure.mjs`

**Contexto:** Atualmente o agente file-analyzer (Fase 1) instrui o LLM a escrever um script Node.js/Python descartável baseado em regex a cada execução. Isso é lento, não-determinístico e ignora a infraestrutura tree-sitter que acabamos de construir. Esta tarefa substitui isso por um script pré-construído que usa `PluginRegistry` (que roteia para `TreeSitterPlugin` para arquivos de código e para os parsers regex para arquivos não-código).

- [ ] **Passo 1: Criar extract-structure.mjs**

O script:
1. Aceita path de JSON de entrada (arg 1) e path de JSON de saída (arg 2)
2. Formato de entrada corresponde ao que file-analyzer.md já especifica: `{ projectRoot, batchFiles: [{path, language, sizeLines, fileCategory}], batchImportData }`
3. Resolve `@understand-anything/core` do próprio `node_modules` do plugin usando `createRequire` relativo à localização do próprio script (dois diretórios acima até a raiz do plugin)
4. Cria um `PluginRegistry` com `TreeSitterPlugin` (todas as configs de linguagem embutidas) + todos os parsers não-código registrados
5. Para cada arquivo: lê o conteúdo, chama `registry.analyzeFile()`, formata a saída para corresponder ao schema de saída do script existente (functions, classes, exports, sections, definitions, services, etc.)
6. Para arquivos de código com suporte tree-sitter: também extrai o grafo de chamadas via `plugin.extractCallGraph()`
7. Para arquivos onde não existe plugin (Swift, Kotlin, linguagens desconhecidas): produz `{ path, language, fileCategory, totalLines, nonEmptyLines, metrics }` com dados estruturais vazios — o agente LLM trata desses na Fase 2
8. Escreve JSON de saída correspondendo ao schema existente `scriptCompleted/filesAnalyzed/filesSkipped/results`

Lógica-chave de resolução (com fallback para diferentes layouts de instalação):
```javascript
import { createRequire } from 'node:module';
import { dirname, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = dirname(fileURLToPath(import.meta.url));
const pluginRoot = resolve(__dirname, '../..');
const require = createRequire(resolve(pluginRoot, 'package.json'));

let core;
try {
  core = await import(require.resolve('@understand-anything/core'));
} catch {
  // Fallback: direct path for installed plugin cache where pnpm symlinks may differ
  core = await import(resolve(pluginRoot, 'packages/core/dist/index.js'));
}
```

- [ ] **Passo 2: Testar o script localmente**

Crie um pequeno JSON de teste com um arquivo TS, um Python e um YAML. Execute:
```bash
node skills/understand/extract-structure.mjs test-input.json test-output.json
```
Verifique que a saída contém dados estruturais para os três.

- [ ] **Passo 3: Commit**

```bash
git add skills/understand/extract-structure.mjs
git commit -m "feat: add bundled tree-sitter extraction script for file-analyzer agent"
```

---

### Tarefa 14: Reescrever Fase 1 do file-analyzer.md para usar o script empacotado

**Arquivos:**
- Modificar: `agents/file-analyzer.md`

**Contexto:** A Fase 1 atualmente tem ~150 linhas instruindo o agente a escrever um script de extração customizado do zero. Substitua por uma seção curta que diga ao agente para executar o script pré-construído `extract-structure.mjs`.

- [ ] **Passo 1: Substituir a Fase 1 em file-analyzer.md**

Apague toda a Fase 1 atual (~150 linhas de instruções de geração de script regex). Substitua por:

1. Diga ao agente para preparar o JSON de entrada (mesmo formato de antes):
   ```bash
   cat > $PROJECT_ROOT/.understand-anything/tmp/ua-file-analyzer-input-<batchIndex>.json << 'ENDJSON'
   {
     "projectRoot": "<project-root>",
     "batchFiles": [<this batch's files including fileCategory>],
     "batchImportData": <batchImportData JSON>
   }
   ENDJSON
   ```

2. Executar o script empacotado:
   ```bash
   node <SKILL_DIR>/extract-structure.mjs \
     $PROJECT_ROOT/.understand-anything/tmp/ua-file-analyzer-input-<batchIndex>.json \
     $PROJECT_ROOT/.understand-anything/tmp/ua-file-extract-results-<batchIndex>.json
   ```

3. Se o script sair com código diferente de zero, leia o stderr, diagnostique e reporte o erro. NÃO faça fallback para escrever um script manual — o script empacotado é o único caminho de extração.

4. Manter o formato de saída existente — a Fase 2 (análise semântica) não muda.

- [ ] **Passo 2: Atualizar SKILL.md para passar SKILL_DIR ao despacho de file-analyzer**

Na Fase 2 do SKILL.md, o prompt de despacho do file-analyzer deve incluir o path do diretório da skill para que o agente possa localizar `extract-structure.mjs`.

Adicione aos parâmetros de despacho:
```
> Skill directory (for bundled scripts): `<SKILL_DIR>`
```

Isso segue o padrão estabelecido — SKILL.md já passa `<SKILL_DIR>` para `merge-batch-graphs.py` (linha 213) e `merge-subdomain-graphs.py` (linha 44) usando o mesmo mecanismo.

- [ ] **Passo 3: Verificar que o formato de saída do file-analyzer não mudou**

A Fase 2 do file-analyzer.md NÃO deve precisar de mudanças — ela lê a mesma estrutura JSON dos resultados do script. Verifique que o schema de saída de `extract-structure.mjs` corresponde ao que a Fase 2 espera.

- [ ] **Passo 4: Commit**

```bash
git add agents/file-analyzer.md skills/understand/SKILL.md
git commit -m "feat: file-analyzer uses bundled tree-sitter script instead of LLM-generated regex"
```

---

### Tarefa 15: Verificação final de integração e limpeza

- [ ] **Passo 1: Adicionar exports a packages/core/src/index.ts**

Isso é obrigatório — `extract-structure.mjs` e consumidores externos precisam destes exports:

```typescript
export type { LanguageExtractor } from "./plugins/extractors/types.js";
export { builtinExtractors } from "./plugins/extractors/index.js";
```

- [ ] **Passo 2: Build do pacote completo**

```bash
pnpm --filter @understand-anything/core build
```

- [ ] **Passo 3: Executar suíte completa de testes uma última vez**

```bash
pnpm --filter @understand-anything/core test
```

- [ ] **Passo 4: Commit final**

```bash
git commit -m "feat: complete language extractor architecture — 10 languages with tree-sitter support"
```

---

## Notas de Implementação

**Convenção de arquivos de teste:** Cada extractor de linguagem ganha seu próprio arquivo de teste em `packages/core/src/plugins/extractors/__tests__/<language>-extractor.test.ts`. Isso segue o padrão existente em que `tree-sitter-plugin.test.ts` é co-localizado.

**Carregamento lazy de gramáticas (otimização futura):** O `TreeSitterPlugin.init()` atual carrega todos os WASMs de gramática de uma vez via `Promise.all`. Com 10 gramáticas (~12MB de WASM total), isso pode causar atraso notável de inicialização. Uma melhoria futura: carregar TS/JS eagerly (mais comum), adiar outras para primeiro uso. Não é obrigatório para este PR — meça primeiro.

**Efeito colateral de fingerprint:** `buildFingerprintStore` em `fingerprint.ts` usa `PluginRegistry.analyzeFile` internamente. Uma vez que os novos extractors estejam conectados, o fingerprinting para Python/Go/Rust/etc. automaticamente produzirá fingerprints estruturais em vez de apenas hash-de-conteúdo. Sem mudanças de código necessárias — acontece de graça.

**Nota sobre gramática PHP:** `tree-sitter-php` envia tanto `tree-sitter-php.wasm` (PHP completo + HTML/CSS/JS embutidos) quanto `tree-sitter-php_only.wasm` (apenas PHP). Usamos `tree-sitter-php.wasm`. O extractor PHP deve ser robusto a nós de AST não-PHP que aparecem ao parsear arquivos com templates HTML embutidos.
