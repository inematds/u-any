# Contribuindo para o Understand Anything

> 🇬🇧 Read this in [English](./CONTRIBUTING.md).

Obrigado pelo seu interesse em contribuir com o Understand Anything! Este documento fornece diretrizes e instruções para contribuir com o projeto.

> Nota: nomes de skills, agents e slash commands (ex.: `/understand`, `project-scanner`) permanecem em inglês porque são identificadores usados pelas ferramentas para acionar funcionalidades.

## 🌟 Formas de Contribuir

- **Bug Reports**: Encontrou um bug? Abra uma issue com passos detalhados de reprodução
- **Pedidos de Funcionalidade**: Tem uma ideia? Compartilhe na seção de issues
- **Documentação**: Melhore ou traduza a documentação
- **Código**: Corrija bugs, adicione funcionalidades ou melhore performance
- **Testes**: Escreva testes para melhorar a cobertura

## 🚀 Começando

### Pré-requisitos

- Node.js >= 22 (desenvolvido em v24)
- pnpm >= 10 (fixado via campo `packageManager` no `package.json` raiz)
- Git para controle de versão

### Setup

1. **Fork e Clone**
   ```bash
   git clone https://github.com/YOUR_USERNAME/Understand-Anything.git
   cd Understand-Anything
   ```

2. **Instale as Dependências**
   ```bash
   pnpm install
   ```

3. **Builde o Pacote Core**
   ```bash
   pnpm --filter @understand-anything/core build
   ```

4. **Rode os Testes**
   ```bash
   pnpm --filter @understand-anything/core test
   pnpm --filter @understand-anything/skill test
   ```

5. **Inicie o Dashboard (Opcional)**
   ```bash
   pnpm dev:dashboard
   ```

## 📝 Fluxo de Desenvolvimento

### 1. Crie uma Branch

Crie um nome de branch descritivo:
```bash
git checkout -b feat/my-feature        # Para novas funcionalidades
git checkout -b fix/bug-description    # Para correções de bug
git checkout -b docs/update-readme     # Para documentação
```

### 2. Faça Alterações

- Escreva código limpo e legível
- Siga o estilo e convenções existentes
- Adicione testes para novas funcionalidades
- Atualize a documentação conforme necessário

### 3. Teste Suas Alterações

```bash
# Roda todos os testes
pnpm --filter @understand-anything/core test
pnpm --filter @understand-anything/skill test

# Roda o linter
pnpm lint

# Builda os pacotes
pnpm build
```

### 4. Faça Commit das Alterações

Escreva mensagens de commit claras e descritivas:
```bash
git add .
git commit -m "feat: add keyboard shortcuts to dashboard"
```

**Convenção de Mensagem de Commit:**
- `feat:` - Nova funcionalidade
- `fix:` - Correção de bug
- `docs:` - Alterações de documentação
- `style:` - Alterações de estilo de código (formatação, etc.)
- `refactor:` - Refatoração de código
- `test:` - Adição ou atualização de testes
- `chore:` - Tarefas de manutenção

### 5. Push e Crie um Pull Request

```bash
git push origin your-branch-name
```

Em seguida, abra um Pull Request no GitHub com:
- Título claro descrevendo a alteração
- Descrição detalhada do que mudou e por quê
- Link para issues relacionadas (se houver)
- Screenshots (para mudanças de UI)

## 🧪 Diretrizes de Teste

### Escrevendo Testes

- Use Vitest para testes
- Coloque testes em diretórios `__tests__` ou arquivos `*.test.ts`
- Vise alta cobertura de testes para novas funcionalidades
- Teste casos de borda e condições de erro

Exemplo de estrutura de teste:
```typescript
import { describe, it, expect } from 'vitest';

describe('MyFeature', () => {
  it('should do something', () => {
    // Arrange
    const input = 'test';

    // Act
    const result = myFunction(input);

    // Assert
    expect(result).toBe('expected');
  });
});
```

### Rodando Testes

```bash
# Roda todos os testes
pnpm test

# Roda testes de um pacote específico
pnpm --filter @understand-anything/core test

# Roda testes em watch mode
pnpm --filter @understand-anything/core test --watch
```

## 📚 Diretrizes de Estilo de Código

### TypeScript

- Use TypeScript em strict mode
- Defina tipos explícitos para parâmetros de função e valores de retorno
- Evite o tipo `any` - use `unknown` se o tipo for de fato desconhecido
- Use interfaces para formas de objeto
- Use type aliases para uniões e tipos complexos

### Formatação

- O projeto usa ESLint para qualidade de código
- Indentação consistente (2 espaços)
- Use nomes significativos para variáveis e funções
- Mantenha funções pequenas e focadas

### React/Dashboard

- Use functional components com hooks
- Mantenha componentes focados e com propósito único
- Use Zustand para gerenciamento de estado
- Siga a estrutura de componentes existente

### Stack Tecnológica

TypeScript, pnpm workspaces, React 18, Vite, TailwindCSS v4, React Flow, Zustand, web-tree-sitter, Fuse.js, Zod, Dagre

### Organização de Arquivos

```
understand-anything-plugin/
├── packages/
│   ├── core/              # Motor de análise core
│   │   ├── src/
│   │   └── package.json
│   └── dashboard/         # Dashboard React
│       ├── src/
│       │   ├── components/
│       │   ├── utils/
│       │   └── store.ts
│       └── package.json
├── src/                   # Implementação das skills do plugin
├── agents/                # Prompts dos agentes de IA
└── skills/                # Definições de skills
```

## 🌍 Diretrizes de Tradução

### Adicionando um Novo Idioma

1. Crie `README.{language-code}.md` (ex.: `README.fr-FR.md`)
2. Traduza todas as seções mantendo a formatação
3. Atualize o `README.md` principal incluindo o link do idioma
4. Mantenha termos técnicos em inglês quando apropriado
5. Garanta que todos os links continuem funcionando

Exemplo:
```markdown
<a href="README.md">English</a> | <a href="README.fr-FR.md">Français</a>
```

## 🐛 Bug Reports

Ao reportar bugs, inclua:

- **Descrição**: Descrição clara do problema
- **Passos para Reproduzir**: Passos detalhados para reproduzir o bug
- **Comportamento Esperado**: O que você esperava que acontecesse
- **Comportamento Real**: O que realmente aconteceu
- **Ambiente**: SO, versão do Node, versão do pnpm
- **Screenshots**: Se aplicável
- **Mensagens de Erro**: Saída completa do erro

## 💡 Pedidos de Funcionalidade

Ao pedir funcionalidades:

- **Caso de Uso**: Descreva o problema que está tentando resolver
- **Solução Proposta**: Como você imagina a funcionalidade funcionando
- **Alternativas**: Outras soluções consideradas
- **Contexto Adicional**: Qualquer outra informação relevante

## 📋 Checklist de Pull Request

Antes de submeter um PR, garanta que:

- [ ] O código segue as diretrizes de estilo do projeto
- [ ] Todos os testes passam (`pnpm test`)
- [ ] Código novo tem cobertura de testes
- [ ] A documentação foi atualizada (se necessário)
- [ ] As mensagens de commit seguem a convenção
- [ ] A descrição do PR explica claramente as mudanças
- [ ] Nenhum console.log ou código de debug foi deixado
- [ ] A branch está atualizada com a main

## 🤝 Processo de Revisão de Código

1. **Checagens Automatizadas**: CI roda testes e linting
2. **Revisão de Mantenedor**: Mantenedores do projeto revisam o código
3. **Feedback**: Atenda às alterações solicitadas
4. **Aprovação**: Uma vez aprovado, o PR será mesclado
5. **Limpeza**: Delete sua branch após o merge

## 📞 Conseguindo Ajuda

- **Issues**: Para bugs e pedidos de funcionalidade
- **Discussions**: Para perguntas e discussão geral
- **Documentação**: Verifique os docs existentes primeiro

## 📄 Licença

Ao contribuir, você concorda que suas contribuições serão licenciadas sob a Licença MIT.

## 🙏 Reconhecimento

Contribuidores serão reconhecidos em:
- Lista de contribuidores do GitHub
- Notas de release (para contribuições significativas)
- Menções especiais para contribuições excepcionais

---

**Obrigado por contribuir com o Understand Anything! Suas contribuições ajudam a tornar a compreensão de código acessível a todos.** 🚀
