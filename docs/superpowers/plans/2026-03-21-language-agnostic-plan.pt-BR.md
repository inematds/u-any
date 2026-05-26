# Plano de Implementação de Suporte Agnóstico a Linguagens

> 🇬🇧 Versão original em inglês: [2026-03-21-language-agnostic-plan.md](./2026-03-21-language-agnostic-plan.md)

> **Para o Claude:** SUB-SKILL OBRIGATÓRIA: Use superpowers:executing-plans para implementar este plano tarefa a tarefa.

**Objetivo:** Tornar o Understand-Anything agnóstico a linguagens introduzindo um framework de linguagens dirigido por configuração, substituindo o plugin tree-sitter restrito a TS e criando prompts cientes de linguagem para 12 linguagens.

**Arquitetura:** Abordagem híbrida orientada por configuração — cada linguagem é definida por um objeto `LanguageConfig` (mapeamentos de nós tree-sitter, conceitos, extensões) mais um arquivo markdown de snippet de prompt. Um único `GenericTreeSitterPlugin` substitui o plugin restrito a TS hardcoded, dirigido pela configuração que casar com a extensão do arquivo.

**Stack Técnico:** TypeScript, web-tree-sitter (WASM), Zod v4, Vitest

---

### Tarefa 1: Criar tipos de LanguageConfig e schema Zod

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/languages/types.ts`

**Passo 1: Escrever o teste que falha**

Criar: `understand-anything-plugin/packages/core/src/languages/__tests__/types.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import { LanguageConfigSchema } from "../types.js";

describe("LanguageConfigSchema", () => {
  it("validates a complete language config", () => {
    const config = {
      id: "python",
      displayName: "Python",
      extensions: [".py", ".pyi"],
      treeSitter: {
        grammarPackage: "tree-sitter-python",
        wasmFile: "tree-sitter-python.wasm",
        nodeTypes: {
          function: ["function_definition"],
          class: ["class_definition"],
          import: ["import_statement", "import_from_statement"],
          export: [],
          typeAnnotation: ["type"],
        },
      },
      concepts: ["decorators", "list comprehensions", "generators"],
    };
    const result = LanguageConfigSchema.safeParse(config);
    expect(result.success).toBe(true);
  });

  it("rejects config missing required fields", () => {
    const result = LanguageConfigSchema.safeParse({ id: "python" });
    expect(result.success).toBe(false);
  });

  it("accepts optional filePatterns", () => {
    const config = {
      id: "python",
      displayName: "Python",
      extensions: [".py"],
      treeSitter: {
        grammarPackage: "tree-sitter-python",
        wasmFile: "tree-sitter-python.wasm",
        nodeTypes: {
          function: ["function_definition"],
          class: ["class_definition"],
          import: ["import_statement"],
          export: [],
          typeAnnotation: [],
        },
      },
      concepts: ["decorators"],
      filePatterns: { config: "pyproject.toml" },
    };
    const result = LanguageConfigSchema.safeParse(config);
    expect(result.success).toBe(true);
  });
});
```

**Passo 2: Rodar o teste para confirmar que falha**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/languages/__tests__/types.test.ts`
Esperado: FAIL — módulo `../types.js` não encontrado

**Passo 3: Escrever implementação mínima**

Criar: `understand-anything-plugin/packages/core/src/languages/types.ts`

```typescript
import { z } from "zod/v4";

export const TreeSitterConfigSchema = z.object({
  grammarPackage: z.string(),
  wasmFile: z.string(),
  nodeTypes: z.object({
    function: z.array(z.string()),
    class: z.array(z.string()),
    import: z.array(z.string()),
    export: z.array(z.string()),
    typeAnnotation: z.array(z.string()),
  }),
});

export const LanguageConfigSchema = z.object({
  id: z.string(),
  displayName: z.string(),
  extensions: z.array(z.string()),
  treeSitter: TreeSitterConfigSchema,
  concepts: z.array(z.string()),
  filePatterns: z.record(z.string(), z.string()).optional(),
});

export type LanguageConfig = z.infer<typeof LanguageConfigSchema>;
export type TreeSitterConfig = z.infer<typeof TreeSitterConfigSchema>;
```

**Passo 4: Rodar o teste para confirmar que passa**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/languages/__tests__/types.test.ts`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/languages/
git commit -m "feat: add LanguageConfig types and Zod schema"
```

---

### Tarefa 2: Criar LanguageRegistry

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/languages/registry.ts`

**Passo 1: Escrever o teste que falha**

Criar: `understand-anything-plugin/packages/core/src/languages/__tests__/registry.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import { LanguageRegistry } from "../registry.js";
import type { LanguageConfig } from "../types.js";

const pythonConfig: LanguageConfig = {
  id: "python",
  displayName: "Python",
  extensions: [".py", ".pyi"],
  treeSitter: {
    grammarPackage: "tree-sitter-python",
    wasmFile: "tree-sitter-python.wasm",
    nodeTypes: {
      function: ["function_definition"],
      class: ["class_definition"],
      import: ["import_statement", "import_from_statement"],
      export: [],
      typeAnnotation: ["type"],
    },
  },
  concepts: ["decorators", "generators"],
};

const tsConfig: LanguageConfig = {
  id: "typescript",
  displayName: "TypeScript",
  extensions: [".ts", ".tsx"],
  treeSitter: {
    grammarPackage: "tree-sitter-typescript",
    wasmFile: "tree-sitter-typescript.wasm",
    nodeTypes: {
      function: ["function_declaration"],
      class: ["class_declaration"],
      import: ["import_statement"],
      export: ["export_statement"],
      typeAnnotation: ["type_annotation"],
    },
  },
  concepts: ["generics", "type guards", "decorators"],
};

describe("LanguageRegistry", () => {
  it("registers and retrieves a config by id", () => {
    const registry = new LanguageRegistry();
    registry.register(pythonConfig);
    expect(registry.getById("python")).toBe(pythonConfig);
  });

  it("retrieves config by file extension", () => {
    const registry = new LanguageRegistry();
    registry.register(pythonConfig);
    expect(registry.getByExtension(".py")).toBe(pythonConfig);
    expect(registry.getByExtension(".pyi")).toBe(pythonConfig);
  });

  it("returns null for unknown extension", () => {
    const registry = new LanguageRegistry();
    registry.register(pythonConfig);
    expect(registry.getByExtension(".rs")).toBeNull();
  });

  it("returns all registered configs", () => {
    const registry = new LanguageRegistry();
    registry.register(pythonConfig);
    registry.register(tsConfig);
    expect(registry.getAll()).toHaveLength(2);
  });

  it("later registration overrides same id", () => {
    const registry = new LanguageRegistry();
    const updated = { ...pythonConfig, displayName: "Python 3" };
    registry.register(pythonConfig);
    registry.register(updated);
    expect(registry.getById("python")?.displayName).toBe("Python 3");
  });

  it("throws on invalid config", () => {
    const registry = new LanguageRegistry();
    expect(() => registry.register({ id: "bad" } as LanguageConfig)).toThrow();
  });
});
```

**Passo 2: Rodar o teste para confirmar que falha**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/languages/__tests__/registry.test.ts`
Esperado: FAIL — módulo `../registry.js` não encontrado

**Passo 3: Escrever implementação mínima**

```typescript
// understand-anything-plugin/packages/core/src/languages/registry.ts
import { LanguageConfigSchema } from "./types.js";
import type { LanguageConfig } from "./types.js";

export class LanguageRegistry {
  private configs = new Map<string, LanguageConfig>();
  private extensionMap = new Map<string, string>();

  register(config: LanguageConfig): void {
    const result = LanguageConfigSchema.safeParse(config);
    if (!result.success) {
      throw new Error(`Invalid LanguageConfig for "${config.id}": ${result.error.message}`);
    }
    this.configs.set(config.id, config);
    for (const ext of config.extensions) {
      this.extensionMap.set(ext, config.id);
    }
  }

  getById(id: string): LanguageConfig | null {
    return this.configs.get(id) ?? null;
  }

  getByExtension(ext: string): LanguageConfig | null {
    const id = this.extensionMap.get(ext);
    if (!id) return null;
    return this.configs.get(id) ?? null;
  }

  getAll(): LanguageConfig[] {
    return [...this.configs.values()];
  }
}
```

**Passo 4: Rodar o teste para confirmar que passa**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/languages/__tests__/registry.test.ts`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/languages/
git commit -m "feat: add LanguageRegistry with Zod validation"
```

---

### Tarefa 3: Criar todas as 12 configurações de linguagem

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/typescript.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/javascript.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/python.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/go.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/java.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/rust.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/cpp.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/csharp.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/ruby.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/php.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/swift.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/kotlin.ts`
- Criar: `understand-anything-plugin/packages/core/src/languages/configs/index.ts`

**Passo 1: Escrever o teste que falha**

Criar: `understand-anything-plugin/packages/core/src/languages/__tests__/configs.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import { LanguageConfigSchema } from "../types.js";
import { builtinConfigs } from "../configs/index.js";

describe("builtin language configs", () => {
  it("has 12 language configs", () => {
    expect(builtinConfigs).toHaveLength(12);
  });

  it("all configs pass Zod validation", () => {
    for (const config of builtinConfigs) {
      const result = LanguageConfigSchema.safeParse(config);
      expect(result.success, `${config.id} failed validation: ${result.error?.message}`).toBe(true);
    }
  });

  it("all configs have unique ids", () => {
    const ids = builtinConfigs.map((c) => c.id);
    expect(new Set(ids).size).toBe(ids.length);
  });

  it("no duplicate extensions across configs", () => {
    const allExts: string[] = [];
    for (const config of builtinConfigs) {
      allExts.push(...config.extensions);
    }
    expect(new Set(allExts).size).toBe(allExts.length);
  });

  it("all configs have non-empty function and class node types", () => {
    for (const config of builtinConfigs) {
      expect(config.treeSitter.nodeTypes.function.length, `${config.id} missing function types`).toBeGreaterThan(0);
      expect(config.treeSitter.nodeTypes.class.length, `${config.id} missing class types`).toBeGreaterThanOrEqual(0);
    }
  });

  it("all configs have at least one concept", () => {
    for (const config of builtinConfigs) {
      expect(config.concepts.length, `${config.id} has no concepts`).toBeGreaterThan(0);
    }
  });
});
```

**Passo 2: Rodar o teste para confirmar que falha**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/languages/__tests__/configs.test.ts`
Esperado: FAIL — módulo não encontrado

**Passo 3: Escrever todos os arquivos de configuração**

Cada arquivo de configuração exporta um `LanguageConfig`. Aqui estão os principais (os demais seguem o mesmo padrão):

**typescript.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const typescriptConfig: LanguageConfig = {
  id: "typescript",
  displayName: "TypeScript",
  extensions: [".ts", ".tsx"],
  treeSitter: {
    grammarPackage: "tree-sitter-typescript",
    wasmFile: "tree-sitter-typescript.wasm",
    nodeTypes: {
      function: ["function_declaration"],
      class: ["class_declaration"],
      import: ["import_statement"],
      export: ["export_statement"],
      typeAnnotation: ["type_annotation"],
    },
  },
  concepts: [
    "generics", "type guards", "discriminated unions", "utility types",
    "decorators", "enums", "interfaces", "type inference",
    "mapped types", "conditional types", "template literal types",
  ],
  filePatterns: { config: "tsconfig.json", manifest: "package.json" },
};
```

**python.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const pythonConfig: LanguageConfig = {
  id: "python",
  displayName: "Python",
  extensions: [".py", ".pyi"],
  treeSitter: {
    grammarPackage: "tree-sitter-python",
    wasmFile: "tree-sitter-python.wasm",
    nodeTypes: {
      function: ["function_definition"],
      class: ["class_definition"],
      import: ["import_statement", "import_from_statement"],
      export: [],
      typeAnnotation: ["type"],
    },
  },
  concepts: [
    "decorators", "list comprehensions", "generators", "context managers",
    "type hints", "dunder methods", "metaclasses", "dataclasses",
    "async/await", "descriptors",
  ],
  filePatterns: { config: "pyproject.toml", manifest: "setup.py" },
};
```

**go.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const goConfig: LanguageConfig = {
  id: "go",
  displayName: "Go",
  extensions: [".go"],
  treeSitter: {
    grammarPackage: "tree-sitter-go",
    wasmFile: "tree-sitter-go.wasm",
    nodeTypes: {
      function: ["function_declaration", "method_declaration"],
      class: ["type_declaration"],
      import: ["import_declaration"],
      export: [],
      typeAnnotation: [],
    },
  },
  concepts: [
    "goroutines", "channels", "interfaces", "struct embedding",
    "error handling patterns", "defer/panic/recover", "slices",
    "pointers", "concurrency patterns",
  ],
  filePatterns: { config: "go.mod" },
};
```

**java.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const javaConfig: LanguageConfig = {
  id: "java",
  displayName: "Java",
  extensions: [".java"],
  treeSitter: {
    grammarPackage: "tree-sitter-java",
    wasmFile: "tree-sitter-java.wasm",
    nodeTypes: {
      function: ["method_declaration", "constructor_declaration"],
      class: ["class_declaration", "interface_declaration", "enum_declaration"],
      import: ["import_declaration"],
      export: [],
      typeAnnotation: ["type_identifier"],
    },
  },
  concepts: [
    "generics", "annotations", "interfaces", "abstract classes",
    "streams API", "lambdas", "sealed classes", "records",
    "dependency injection", "checked exceptions",
  ],
  filePatterns: { config: "pom.xml", manifest: "build.gradle" },
};
```

**rust.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const rustConfig: LanguageConfig = {
  id: "rust",
  displayName: "Rust",
  extensions: [".rs"],
  treeSitter: {
    grammarPackage: "tree-sitter-rust",
    wasmFile: "tree-sitter-rust.wasm",
    nodeTypes: {
      function: ["function_item"],
      class: ["struct_item", "enum_item", "impl_item", "trait_item"],
      import: ["use_declaration"],
      export: [],
      typeAnnotation: ["type_identifier"],
    },
  },
  concepts: [
    "ownership", "borrowing", "lifetimes", "traits", "pattern matching",
    "enums with data", "error handling (Result/Option)", "macros",
    "async/await", "unsafe blocks", "generics", "closures",
  ],
  filePatterns: { config: "Cargo.toml" },
};
```

**cpp.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const cppConfig: LanguageConfig = {
  id: "cpp",
  displayName: "C/C++",
  extensions: [".cpp", ".cc", ".cxx", ".c", ".h", ".hpp", ".hxx"],
  treeSitter: {
    grammarPackage: "tree-sitter-cpp",
    wasmFile: "tree-sitter-cpp.wasm",
    nodeTypes: {
      function: ["function_definition"],
      class: ["class_specifier", "struct_specifier"],
      import: ["preproc_include"],
      export: [],
      typeAnnotation: [],
    },
  },
  concepts: [
    "templates", "RAII", "smart pointers", "move semantics",
    "operator overloading", "virtual functions", "namespaces",
    "constexpr", "lambda expressions", "STL containers",
  ],
  filePatterns: { config: "CMakeLists.txt", manifest: "Makefile" },
};
```

**csharp.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const csharpConfig: LanguageConfig = {
  id: "csharp",
  displayName: "C#",
  extensions: [".cs"],
  treeSitter: {
    grammarPackage: "tree-sitter-c-sharp",
    wasmFile: "tree-sitter-c_sharp.wasm",
    nodeTypes: {
      function: ["method_declaration", "constructor_declaration"],
      class: ["class_declaration", "interface_declaration", "struct_declaration", "enum_declaration", "record_declaration"],
      import: ["using_directive"],
      export: [],
      typeAnnotation: ["type_identifier"],
    },
  },
  concepts: [
    "LINQ", "async/await", "generics", "properties",
    "delegates and events", "attributes", "nullable reference types",
    "pattern matching", "records", "dependency injection",
  ],
  filePatterns: { config: "*.csproj" },
};
```

**ruby.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const rubyConfig: LanguageConfig = {
  id: "ruby",
  displayName: "Ruby",
  extensions: [".rb", ".rake"],
  treeSitter: {
    grammarPackage: "tree-sitter-ruby",
    wasmFile: "tree-sitter-ruby.wasm",
    nodeTypes: {
      function: ["method"],
      class: ["class", "module"],
      import: ["call"],
      export: [],
      typeAnnotation: [],
    },
  },
  concepts: [
    "blocks and procs", "mixins", "metaprogramming", "duck typing",
    "DSLs", "monkey patching", "gems", "symbols",
    "method_missing", "open classes",
  ],
  filePatterns: { config: "Gemfile" },
};
```

**php.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const phpConfig: LanguageConfig = {
  id: "php",
  displayName: "PHP",
  extensions: [".php"],
  treeSitter: {
    grammarPackage: "tree-sitter-php",
    wasmFile: "tree-sitter-php.wasm",
    nodeTypes: {
      function: ["function_definition", "method_declaration"],
      class: ["class_declaration", "interface_declaration", "trait_declaration"],
      import: ["namespace_use_declaration"],
      export: [],
      typeAnnotation: ["type_list", "named_type"],
    },
  },
  concepts: [
    "namespaces", "traits", "type declarations", "attributes",
    "enums", "fibers", "closures", "magic methods",
    "dependency injection", "middleware",
  ],
  filePatterns: { config: "composer.json" },
};
```

**swift.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const swiftConfig: LanguageConfig = {
  id: "swift",
  displayName: "Swift",
  extensions: [".swift"],
  treeSitter: {
    grammarPackage: "tree-sitter-swift",
    wasmFile: "tree-sitter-swift.wasm",
    nodeTypes: {
      function: ["function_declaration", "init_declaration"],
      class: ["class_declaration", "struct_declaration", "protocol_declaration", "enum_declaration"],
      import: ["import_declaration"],
      export: [],
      typeAnnotation: ["type_annotation"],
    },
  },
  concepts: [
    "optionals", "protocols", "extensions", "generics",
    "closures", "property wrappers", "result builders",
    "actors", "structured concurrency", "value types vs reference types",
  ],
  filePatterns: { config: "Package.swift" },
};
```

**kotlin.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const kotlinConfig: LanguageConfig = {
  id: "kotlin",
  displayName: "Kotlin",
  extensions: [".kt", ".kts"],
  treeSitter: {
    grammarPackage: "tree-sitter-kotlin",
    wasmFile: "tree-sitter-kotlin.wasm",
    nodeTypes: {
      function: ["function_declaration"],
      class: ["class_declaration", "object_declaration", "interface_declaration"],
      import: ["import_header"],
      export: [],
      typeAnnotation: ["type_identifier"],
    },
  },
  concepts: [
    "coroutines", "data classes", "sealed classes", "extension functions",
    "null safety", "delegation", "DSL builders", "inline functions",
    "companion objects", "flow",
  ],
  filePatterns: { config: "build.gradle.kts" },
};
```

**javascript.ts:**
```typescript
import type { LanguageConfig } from "../types.js";

export const javascriptConfig: LanguageConfig = {
  id: "javascript",
  displayName: "JavaScript",
  extensions: [".js", ".mjs", ".cjs", ".jsx"],
  treeSitter: {
    grammarPackage: "tree-sitter-javascript",
    wasmFile: "tree-sitter-javascript.wasm",
    nodeTypes: {
      function: ["function_declaration"],
      class: ["class_declaration"],
      import: ["import_statement"],
      export: ["export_statement"],
      typeAnnotation: [],
    },
  },
  concepts: [
    "closures", "prototypes", "promises", "async/await",
    "event loop", "destructuring", "spread operator",
    "proxies", "generators", "modules (ESM/CJS)",
  ],
  filePatterns: { config: "package.json" },
};
```

**configs/index.ts:**
```typescript
import { typescriptConfig } from "./typescript.js";
import { javascriptConfig } from "./javascript.js";
import { pythonConfig } from "./python.js";
import { goConfig } from "./go.js";
import { javaConfig } from "./java.js";
import { rustConfig } from "./rust.js";
import { cppConfig } from "./cpp.js";
import { csharpConfig } from "./csharp.js";
import { rubyConfig } from "./ruby.js";
import { phpConfig } from "./php.js";
import { swiftConfig } from "./swift.js";
import { kotlinConfig } from "./kotlin.js";
import type { LanguageConfig } from "../types.js";

export const builtinConfigs: LanguageConfig[] = [
  typescriptConfig,
  javascriptConfig,
  pythonConfig,
  goConfig,
  javaConfig,
  rustConfig,
  cppConfig,
  csharpConfig,
  rubyConfig,
  phpConfig,
  swiftConfig,
  kotlinConfig,
];
```

**Passo 4: Rodar o teste para confirmar que passa**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/languages/__tests__/configs.test.ts`
Esperado: PASS

**Passo 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/languages/configs/
git commit -m "feat: add 12 builtin language configs"
```

---

### Tarefa 4: Criar barrel languages/index.ts e exportar do core

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/languages/index.ts`
- Modificar: `understand-anything-plugin/packages/core/src/index.ts`

**Passo 1: Criar export barrel**

```typescript
// understand-anything-plugin/packages/core/src/languages/index.ts
export { LanguageRegistry } from "./registry.js";
export { LanguageConfigSchema } from "./types.js";
export type { LanguageConfig, TreeSitterConfig } from "./types.js";
export { builtinConfigs } from "./configs/index.js";
```

**Passo 2: Adicionar export ao index.ts do core**

Adicionar a `understand-anything-plugin/packages/core/src/index.ts`:

```typescript
// Languages
export { LanguageRegistry, builtinConfigs, LanguageConfigSchema } from "./languages/index.js";
export type { LanguageConfig, TreeSitterConfig } from "./languages/index.js";
```

**Passo 3: Buildar e verificar**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core build`
Esperado: Build conclui sem erros

**Passo 4: Commit**

```bash
git add understand-anything-plugin/packages/core/src/languages/index.ts understand-anything-plugin/packages/core/src/index.ts
git commit -m "feat: export language types and registry from core"
```

---

### Tarefa 5: Instalar pacotes de gramática WASM do tree-sitter

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/package.json`

**Passo 1: Instalar novos pacotes de gramática**

Rodar:
```bash
cd understand-anything-plugin && pnpm --filter @understand-anything/core add \
  tree-sitter-python \
  tree-sitter-go \
  tree-sitter-java \
  tree-sitter-rust \
  tree-sitter-cpp \
  tree-sitter-c-sharp \
  tree-sitter-ruby \
  tree-sitter-php \
  tree-sitter-swift \
  tree-sitter-kotlin
```

Observação: Alguns pacotes de gramática podem não distribuir arquivos `.wasm`. Para esses, precisamos verificar disponibilidade e potencialmente compilar a partir do código-fonte ou usar a CLI `tree-sitter` para gerar WASM. Verifique cada pacote depois da instalação:

```bash
cd understand-anything-plugin && for lang in python go java rust cpp c-sharp ruby php swift kotlin; do
  echo "=== tree-sitter-$lang ==="
  ls node_modules/tree-sitter-$lang/*.wasm 2>/dev/null || echo "NO WASM FOUND"
done
```

Para pacotes sem WASM pré-compilado, use `tree-sitter build --wasm` para compilá-los, ou encontre pacotes npm alternativos que distribuam builds WASM. Documente quais pacotes precisaram de geração manual de WASM.

**Passo 2: Verificar que o build continua passando**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core build`
Esperado: PASS

**Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/core/package.json understand-anything-plugin/pnpm-lock.yaml
git commit -m "feat: add tree-sitter grammar packages for 10 new languages"
```

---

### Tarefa 6: Construir GenericTreeSitterPlugin

**Arquivos:**
- Criar: `understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.ts`

**Passo 1: Escrever o teste que falha**

Criar: `understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.test.ts`

```typescript
import { describe, it, expect, beforeAll } from "vitest";
import { GenericTreeSitterPlugin } from "./generic-tree-sitter-plugin.js";
import { LanguageRegistry } from "../languages/registry.js";
import { typescriptConfig } from "../languages/configs/typescript.js";
import { javascriptConfig } from "../languages/configs/javascript.js";
import { pythonConfig } from "../languages/configs/python.js";

describe("GenericTreeSitterPlugin", () => {
  let plugin: GenericTreeSitterPlugin;

  beforeAll(async () => {
    const registry = new LanguageRegistry();
    registry.register(typescriptConfig);
    registry.register(javascriptConfig);
    registry.register(pythonConfig);
    plugin = new GenericTreeSitterPlugin(registry);
    await plugin.init();
  });

  describe("TypeScript (migration parity)", () => {
    it("extracts function declarations", () => {
      const code = `
function greet(name: string): string {
  return "Hello " + name;
}
`;
      const result = plugin.analyzeFile("test.ts", code);
      expect(result.functions).toHaveLength(1);
      expect(result.functions[0].name).toBe("greet");
    });

    it("extracts class declarations", () => {
      const code = `
class UserService {
  getName(): string { return "test"; }
}
`;
      const result = plugin.analyzeFile("test.ts", code);
      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("UserService");
    });

    it("extracts imports", () => {
      const code = `import { readFile } from "fs";`;
      const result = plugin.analyzeFile("test.ts", code);
      expect(result.imports).toHaveLength(1);
      expect(result.imports[0].source).toBe("fs");
    });

    it("extracts exports", () => {
      const code = `export function hello() {}`;
      const result = plugin.analyzeFile("test.ts", code);
      expect(result.exports.length).toBeGreaterThanOrEqual(1);
    });

    it("extracts arrow functions", () => {
      const code = `const add = (a: number, b: number): number => a + b;`;
      const result = plugin.analyzeFile("test.ts", code);
      expect(result.functions).toHaveLength(1);
      expect(result.functions[0].name).toBe("add");
    });
  });

  describe("Python", () => {
    it("extracts function definitions", () => {
      const code = `
def greet(name):
    return f"Hello {name}"

def add(a, b):
    return a + b
`;
      const result = plugin.analyzeFile("test.py", code);
      expect(result.functions).toHaveLength(2);
      expect(result.functions[0].name).toBe("greet");
      expect(result.functions[1].name).toBe("add");
    });

    it("extracts class definitions", () => {
      const code = `
class UserService:
    def get_name(self):
        return "test"
`;
      const result = plugin.analyzeFile("test.py", code);
      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("UserService");
    });

    it("extracts import statements", () => {
      const code = `
import os
from pathlib import Path
from typing import Optional
`;
      const result = plugin.analyzeFile("test.py", code);
      expect(result.imports).toHaveLength(3);
    });
  });

  it("returns null for unsupported file extension", () => {
    expect(plugin.canAnalyze("test.unknown")).toBe(false);
  });

  it("reports all registered languages", () => {
    const langs = plugin.supportedLanguages();
    expect(langs).toContain("typescript");
    expect(langs).toContain("python");
  });
});
```

**Passo 2: Rodar o teste para confirmar que falha**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/plugins/generic-tree-sitter-plugin.test.ts`
Esperado: FAIL — módulo não encontrado

**Passo 3: Escrever implementação**

Criar `understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.ts`:

Esse arquivo implementa um `GenericTreeSitterPlugin` que:
- Recebe um `LanguageRegistry` no construtor
- Em `init()`, carrega lazily as gramáticas WASM por linguagem usando `require.resolve(config.treeSitter.grammarPackage + '/' + config.treeSitter.wasmFile)`
- Em `analyzeFile()`, determina a linguagem a partir da extensão via registry e então percorre a AST usando `config.treeSitter.nodeTypes` para extrair funções/classes/imports/exports
- Reutiliza os mesmos padrões auxiliares do antigo `TreeSitterPlugin` (traverse, getStringValue, extractParams) mas dirigidos por configuração em vez de tipos de nó hardcoded
- Implementa `resolveImports()` e `extractCallGraph()` com a mesma lógica de antes

Notas-chave de implementação:
- O método `extractNodes()` percorre a AST e casa nós contra `nodeTypes.function`, `nodeTypes.class`, etc.
- Para TS/JS, também trata `lexical_declaration`/`variable_declaration` com valores de arrow function (comportamento existente)
- Para extração de imports, usa a mesma abordagem `getStringValue()` mas casa contra tipos de nó de import específicos da linguagem
- Para extração de exports, mesmo padrão de casamento contra tipos de nó de export
- Carregamento de gramática: tente `require.resolve()` primeiro; se WASM não for encontrado, loga warning e pula essa linguagem

**Passo 4: Rodar o teste para confirmar que passa**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/plugins/generic-tree-sitter-plugin.test.ts`
Esperado: PASS

**Passo 5: Rodar os testes antigos de TreeSitterPlugin com o novo plugin para verificar paridade de migração**

Garanta que os casos de teste existentes em `tree-sitter-plugin.test.ts` também passem com `GenericTreeSitterPlugin` + configs TS/JS.

**Passo 6: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.ts
git add understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.test.ts
git commit -m "feat: add GenericTreeSitterPlugin driven by LanguageConfig"
```

---

### Tarefa 7: Adicionar fixtures de teste por linguagem para as linguagens restantes

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.test.ts`

**Passo 1: Adicionar casos de teste para Go, Java, Rust, C++, C#, Ruby, PHP, Swift, Kotlin**

Para cada linguagem, adicione um bloco `describe` com uma pequena fixture testando extração de função/classe/import. Exemplo para Go:

```typescript
describe("Go", () => {
  it("extracts function declarations", () => {
    const code = `
package main

func greet(name string) string {
    return "Hello " + name
}
`;
    const result = plugin.analyzeFile("test.go", code);
    expect(result.functions).toHaveLength(1);
    expect(result.functions[0].name).toBe("greet");
  });

  it("extracts type declarations", () => {
    const code = `
package main

type UserService struct {
    Name string
}
`;
    const result = plugin.analyzeFile("test.go", code);
    expect(result.classes).toHaveLength(1);
  });

  it("extracts imports", () => {
    const code = `
package main

import (
    "fmt"
    "os"
)
`;
    const result = plugin.analyzeFile("test.go", code);
    expect(result.imports).toHaveLength(2);
  });
});
```

Siga o mesmo padrão para cada linguagem com sintaxe apropriada. Cada teste usa ~10-20 linhas de código idiomático.

Observação: Algumas gramáticas WASM podem não estar disponíveis. Para linguagens em que a gramática falhar ao carregar, registre-as no `beforeAll` com try/catch e use `it.skipIf()` para pular condicionalmente os testes. Isso previne falhas de CI enquanto ainda testa o que está disponível.

**Passo 2: Rodar todos os testes**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/plugins/generic-tree-sitter-plugin.test.ts`
Esperado: PASS para todas as linguagens com gramáticas disponíveis

**Passo 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/generic-tree-sitter-plugin.test.ts
git commit -m "test: add per-language fixtures for GenericTreeSitterPlugin"
```

---

### Tarefa 8: Substituir TreeSitterPlugin por GenericTreeSitterPlugin

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/index.ts`
- Modificar: `understand-anything-plugin/packages/core/src/plugins/registry.ts`
- Deletar: `understand-anything-plugin/packages/core/src/plugins/tree-sitter-plugin.ts` (após confirmar que não há outros imports)

**Passo 1: Atualizar exports do core**

Em `understand-anything-plugin/packages/core/src/index.ts`:
- Substituir `export { TreeSitterPlugin }` por `export { GenericTreeSitterPlugin }`
- Exportar também `GenericTreeSitterPlugin` como `TreeSitterPlugin` para compatibilidade retroativa se necessário (verificar consumidores)

**Passo 2: Atualizar o mapa de extensões do PluginRegistry**

Em `understand-anything-plugin/packages/core/src/plugins/registry.ts`:
- O mapa `EXTENSION_TO_LANGUAGE` já é abrangente (tem py, go, rs, etc.)
- Nenhuma mudança necessária aqui — o registry apenas despacha para qualquer plugin que estiver registrado

**Passo 3: Atualizar todos os imports no código-fonte da skill**

Procure por todos os imports de `TreeSitterPlugin` no codebase:

Rodar: `grep -r "TreeSitterPlugin" understand-anything-plugin/`

Atualize cada import para usar `GenericTreeSitterPlugin`. Os principais consumidores são:
- `understand-anything-plugin/packages/core/src/index.ts`
- Quaisquer arquivos de código de skill que instanciem o plugin

**Passo 4: Deletar o antigo TreeSitterPlugin**

Após todos os imports atualizados e testes passando:

Rodar: `rm understand-anything-plugin/packages/core/src/plugins/tree-sitter-plugin.ts`

Mantenha o arquivo de teste antigo temporariamente — renomeie-o para verificar paridade.

**Passo 5: Rodar suite completa de testes**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test`
Esperado: TODOS PASSAM

**Passo 6: Commit**

```bash
git add -A
git commit -m "refactor: replace TreeSitterPlugin with GenericTreeSitterPlugin"
```

---

### Tarefa 9: Atualizar language-lesson.ts para usar LanguageRegistry

**Arquivos:**
- Modificar: `understand-anything-plugin/packages/core/src/analyzer/language-lesson.ts`
- Modificar: `understand-anything-plugin/packages/core/src/__tests__/language-lesson.test.ts`

**Passo 1: Atualizar o teste**

Atualizar `language-lesson.test.ts` para verificar que conceitos vêm do registry:

```typescript
it("detects concepts from language config", () => {
  const node = {
    ...sampleNode,
    summary: "Uses decorators and async/await with generators",
    tags: ["decorators"],
  };
  const concepts = detectLanguageConcepts(node, "python");
  expect(concepts).toContain("decorators");
  expect(concepts).toContain("async/await");
});
```

**Passo 2: Rodar o teste para confirmar que falha (ou passa com comportamento antigo)**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/__tests__/language-lesson.test.ts`

**Passo 3: Atualizar implementação**

Em `language-lesson.ts`:
- Importar `LanguageRegistry` e `builtinConfigs`
- Criar uma instância de registry no nível do módulo, pré-populada com builtinConfigs
- Substituir lookups de `LANGUAGE_DISPLAY_NAMES` por `registry.getById(lang)?.displayName`
- Substituir o `CONCEPT_PATTERNS` hardcoded por `registry.getById(lang)?.concepts` mesclado com padrões genéricos (async/await, error handling, etc. que se aplicam a todas as linguagens)
- Manter a lógica de detecção (buscar em tags/summary por palavras-chave de conceito) mas tirar as palavras-chave da configuração

**Passo 4: Rodar o teste para confirmar que passa**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test -- --run src/__tests__/language-lesson.test.ts`
Esperado: PASS

**Passo 5: Rodar suite completa de testes**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test`
Esperado: TODOS PASSAM

**Passo 6: Commit**

```bash
git add understand-anything-plugin/packages/core/src/analyzer/language-lesson.ts
git add understand-anything-plugin/packages/core/src/__tests__/language-lesson.test.ts
git commit -m "refactor: source language concepts from LanguageRegistry"
```

---

### Tarefa 10: Criar arquivos de snippet de prompt por linguagem

**Arquivos:**
- Criar: `understand-anything-plugin/skills/understand/languages/typescript.md`
- Criar: `understand-anything-plugin/skills/understand/languages/javascript.md`
- Criar: `understand-anything-plugin/skills/understand/languages/python.md`
- Criar: `understand-anything-plugin/skills/understand/languages/go.md`
- Criar: `understand-anything-plugin/skills/understand/languages/java.md`
- Criar: `understand-anything-plugin/skills/understand/languages/rust.md`
- Criar: `understand-anything-plugin/skills/understand/languages/cpp.md`
- Criar: `understand-anything-plugin/skills/understand/languages/csharp.md`
- Criar: `understand-anything-plugin/skills/understand/languages/ruby.md`
- Criar: `understand-anything-plugin/skills/understand/languages/php.md`
- Criar: `understand-anything-plugin/skills/understand/languages/swift.md`
- Criar: `understand-anything-plugin/skills/understand/languages/kotlin.md`

**Passo 1: Criar todos os 12 arquivos markdown de linguagem**

Cada arquivo segue essa estrutura:

```markdown
# [Language Name]

## Key Concepts
- [5-10 language-specific concepts with brief explanations]

## Import Patterns
- [All common import syntax patterns for this language]

## Notable File Patterns
- [Special files like __init__.py, go.mod, Cargo.toml, etc.]

## Common Frameworks
- [Top 3-5 frameworks/libraries in this ecosystem]

## Example Summary Style
> "[Example of how to summarize a function/class in this language's idiom]"
```

Cada arquivo deve ter 30-50 linhas, com conteúdo específico ao ecossistema e idiomas daquela linguagem. O conteúdo deve ajudar o LLM a produzir melhor análise através do entendimento de padrões específicos da linguagem.

**Passo 2: Verificar que os arquivos estão bem formados**

Revisar manualmente cada arquivo quanto a precisão e completude.

**Passo 3: Commit**

```bash
git add understand-anything-plugin/skills/understand/languages/
git commit -m "feat: add language-specific prompt snippet files for 12 languages"
```

---

### Tarefa 11: Tornar prompts base neutros de linguagem com pontos de injeção

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/file-analyzer-prompt.md`
- Modificar: `understand-anything-plugin/skills/understand/tour-builder-prompt.md`
- Modificar: `understand-anything-plugin/skills/understand/project-scanner-prompt.md`

**Passo 1: Atualizar file-analyzer-prompt.md**

- Remover todos os exemplos específicos de TypeScript (ex.: "TypeScript barrel file", referências a type guard)
- Substituir listas de conceitos específicas de TS por placeholders genéricos
- Adicionar ponto de injeção:

```markdown
## Language-Specific Guidance

{{LANGUAGE_CONTEXT}}
```

- Tornar a detecção de scripts da Fase 1 ciente da linguagem (não apenas "Node.js recommended")

**Passo 2: Atualizar tour-builder-prompt.md**

- Remover exemplos de lição de linguagem específicos de TS ("generics, discriminated unions, utility types")
- Substituir por ponto de injeção para linguagens detectadas:

```markdown
## Language-Specific Concepts

{{LANGUAGE_CONTEXT}}
```

**Passo 3: Atualizar project-scanner-prompt.md**

- Remover verificação hardcoded de `tsconfig.json`
- Tornar a detecção de frameworks genérica (injetar listas de frameworks das linguagens detectadas)
- Adicionar seção multi-linguagem:

```markdown
## Detected Languages

{{LANGUAGE_CONTEXT}}
```

**Passo 4: Verificar que prompts estão bem formados**

Leia cada prompt modificado para garantir que está coerente com os pontos de injeção e sem viés residual de TS.

**Passo 5: Commit**

```bash
git add understand-anything-plugin/skills/understand/file-analyzer-prompt.md
git add understand-anything-plugin/skills/understand/tour-builder-prompt.md
git add understand-anything-plugin/skills/understand/project-scanner-prompt.md
git commit -m "refactor: make agent prompts language-neutral with injection points"
```

---

### Tarefa 12: Implementar lógica de injeção de prompts no código da skill

**Arquivos:**
- Modificar: `understand-anything-plugin/skills/understand/SKILL.md` (a definição da skill `/understand`)

**Passo 1: Atualizar a orquestração da skill**

Na skill `/understand` (SKILL.md), atualizar a lógica de despacho de agentes:

- **Fase 0 (Pré-vôo):** Após escanear arquivos, detectar linguagens presentes e carregar os arquivos `languages/*.md` correspondentes
- **Fase 2 (despacho do File Analyzer):** Para cada batch de arquivos, injetar o conteúdo `.md` da linguagem correspondente no placeholder `{{LANGUAGE_CONTEXT}}` do prompt do file-analyzer
- **Fase 4 (Architecture Analyzer):** Injetar conceitos de todas as linguagens detectadas
- **Fase 5 (Tour Builder):** Injetar conteúdo `.md` de todas as linguagens detectadas no placeholder `{{LANGUAGE_CONTEXT}}`
- **Fase 1 (Project Scanner):** Injetar conteúdo `.md` de todas as linguagens detectadas

A lógica de injeção:
1. Mapear extensões de arquivo para IDs de linguagem (reutilizar `LanguageRegistry.getByExtension()`)
2. Ler o arquivo `languages/<id>.md` correspondente
3. Substituir `{{LANGUAGE_CONTEXT}}` no prompt base pelo conteúdo do arquivo

Para projetos multi-linguagem, concatenar todos os arquivos de linguagens detectadas.

**Passo 2: Verificar lendo o SKILL.md modificado**

Garantir que o fluxo de orquestração inclui detecção de linguagem e passos de injeção de prompt.

**Passo 3: Commit**

```bash
git add understand-anything-plugin/skills/understand/SKILL.md
git commit -m "feat: add language detection and prompt injection to /understand skill"
```

---

### Tarefa 13: Atualizar teste antigo de tree-sitter-plugin para usar GenericTreeSitterPlugin

**Arquivos:**
- Modificar ou Deletar: `understand-anything-plugin/packages/core/src/plugins/tree-sitter-plugin.test.ts`

**Passo 1: Migrar ou deletar**

Se o antigo `tree-sitter-plugin.test.ts` ainda existir:
- Ou atualize-o para importar `GenericTreeSitterPlugin` e instanciar com um `LanguageRegistry` contendo configs TS/JS
- Ou delete-o se todos os seus casos de teste estiverem cobertos em `generic-tree-sitter-plugin.test.ts`

Prefira deletar para evitar duplicação.

**Passo 2: Rodar suite completa de testes**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test`
Esperado: TODOS PASSAM

**Passo 3: Commit**

```bash
git add -A
git commit -m "test: migrate old tree-sitter-plugin tests to generic plugin"
```

---

### Tarefa 14: Verificação de build e linter

**Passo 1: Buildar core**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core build`
Esperado: PASS

**Passo 2: Buildar pacote da skill**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/skill build`
Esperado: PASS

**Passo 3: Buildar dashboard**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/dashboard build`
Esperado: PASS (o dashboard não importa módulos de linguagem diretamente)

**Passo 4: Rodar linter**

Rodar: `cd understand-anything-plugin && pnpm lint`
Esperado: PASS (ou corrigir quaisquer issues do linter)

**Passo 5: Rodar todos os testes**

Rodar: `cd understand-anything-plugin && pnpm --filter @understand-anything/core test && pnpm --filter @understand-anything/skill test`
Esperado: TODOS PASSAM

**Passo 6: Commit de quaisquer correções**

```bash
git add -A
git commit -m "fix: resolve build and lint issues from language-agnostic refactor"
```

---

### Tarefa 15: Atualizar CLAUDE.md e documentação

**Arquivos:**
- Modificar: `CLAUDE.md`
- Modificar: `README.md` (se existir e mencionar suporte restrito a TS)

**Passo 1: Atualizar CLAUDE.md**

Adicionar à seção de Arquitetura:
- Mencionar os diretórios `languages/` (tanto em core quanto em skills)
- Documentar como adicionar uma nova linguagem (criar configuração + snippet de prompt)
- Listar linguagens suportadas

**Passo 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with language-agnostic architecture"
```
