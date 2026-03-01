# Fundamentos de Testes

## Visão Geral

Testes são a prática de verificar que o software se comporta como esperado. Sem uma suíte de testes, cada mudança é uma aposta — você faz deploy em produção e torce para que nada quebre. Com uma boa suíte de testes, você pode refatorar sem medo, integrar novos engenheiros com segurança e fazer entregas com confiança. Este capítulo cobre a filosofia e o vocabulário dos testes: a pirâmide de testes, os tipos de test doubles, o que testar e como pensar sobre coverage.

---

## Pré-requisitos

- Fundamentos de JavaScript/TypeScript
- Familiaridade básica com async/await
- Entendimento de funções e módulos

---

## Conceitos Fundamentais

### A pirâmide de testes

```
        /\
       /  \    E2E (poucos, lentos, alta confiança)
      /----\
     /      \  Integração (moderados, pega bugs de integração)
    /--------\
   /          \ Unitários (muitos, rápidos, baratos de escrever)
  /____________\
```

| Camada | Velocidade | Custo | Confiança | Isolamento |
|--------|------------|-------|-----------|------------|
| Unitário | Milissegundos | Baixo | Baixa (testa lógica em isolamento) | Alto |
| Integração | Segundos | Médio | Média (testa conexões reais) | Médio |
| E2E | Minutos | Alto | Alta (testa o sistema completo) | Baixo |

A forma de pirâmide reflete a proporção recomendada: muitos testes unitários rápidos na base, menos testes de integração no meio e um pequeno número de testes E2E no topo.

**Inverter a pirâmide é um erro comum.** Times que dependem principalmente de testes E2E têm CI lento, suítes frágeis e ciclos de feedback ruins.

### Tipos de test doubles

Test doubles substituem dependências reais nos testes unitários:

| Tipo | Descrição | Quando usar |
|------|-----------|-------------|
| **Stub** | Retorna um valor pré-definido | Fornecer respostas prontas a consultas |
| **Mock** | Verifica que um método foi chamado com argumentos específicos | Testar comportamento, não apenas estado |
| **Spy** | Registra chamadas sem alterar o comportamento | Verificar efeitos colaterais |
| **Fake** | Implementação funcional simplificada (ex.: banco de dados em memória) | Testes parecidos com integração sem infraestrutura real |
| **Dummy** | Placeholder sem comportamento | Preencher parâmetros obrigatórios que não importam |

### O que testar

Teste o comportamento, não a implementação. Pergunte: "se eu mudar a implementação sem mudar o comportamento, esse teste deve quebrar?" Se sim, você está testando detalhes de implementação.

**Teste:**
- A API pública de funções e classes
- Casos extremos (arrays vazios, valores nulos, valores máximos)
- Caminhos de tratamento de erros
- Regras de negócio e invariantes

**Não teste:**
- Métodos privados diretamente
- Internals de frameworks
- Getters/setters simples sem lógica
- Comportamento de bibliotecas de terceiros

---

## Exemplos Práticos

### Configurando o vitest

```bash
npm install --save-dev vitest @vitest/ui
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,           // não precisa importar describe/it/expect
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
      },
    },
  },
});
```

```json
// scripts do package.json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

### Anatomia de um teste

```typescript
// src/lib/math.test.ts
import { describe, it, expect } from 'vitest';
import { add, divide } from './math.js';

describe('add', () => {
  it('returns the sum of two positive numbers', () => {
    // Arrange — preparar
    const a = 3;
    const b = 4;

    // Act — executar
    const result = add(a, b);

    // Assert — verificar
    expect(result).toBe(7);
  });

  it('handles negative numbers', () => {
    expect(add(-1, 1)).toBe(0);
    expect(add(-5, -3)).toBe(-8);
  });
});

describe('divide', () => {
  it('divides two numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });

  it('throws when dividing by zero', () => {
    expect(() => divide(10, 0)).toThrow('Cannot divide by zero');
  });
});
```

O padrão AAA (Arrange-Act-Assert) é a estrutura padrão. Cada teste deve verificar exatamente um comportamento.

### Test doubles no vitest

```typescript
// src/services/user.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { UserService } from './user.service.js';
import { EmailService } from './email.service.js';
import { UserRepository } from '../repositories/user.repository.js';

// Faz mock do módulo inteiro
vi.mock('../repositories/user.repository.js');
vi.mock('./email.service.js');

describe('UserService', () => {
  let userService: UserService;
  let mockUserRepo: UserRepository;
  let mockEmailService: EmailService;

  beforeEach(() => {
    mockUserRepo = {
      findById: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
    } as unknown as UserRepository;

    mockEmailService = {
      send: vi.fn(),
    } as unknown as EmailService;

    userService = new UserService(mockUserRepo, mockEmailService);
  });

  describe('createUser', () => {
    it('creates a user and sends a welcome email', async () => {
      // Stub: valor de retorno quando repo.create é chamado
      const fakeUser = { id: '1', email: 'alice@example.com', name: 'Alice' };
      vi.mocked(mockUserRepo.create).mockResolvedValue(fakeUser);

      const result = await userService.createUser({ email: 'alice@example.com', name: 'Alice' });

      // Verificar estado
      expect(result).toEqual(fakeUser);

      // Verificar comportamento: email foi enviado
      expect(mockEmailService.send).toHaveBeenCalledOnce();
      expect(mockEmailService.send).toHaveBeenCalledWith({
        to: 'alice@example.com',
        template: 'welcome',
        variables: { name: 'Alice' },
      });
    });

    it('does not send email if user creation fails', async () => {
      vi.mocked(mockUserRepo.create).mockRejectedValue(new Error('DB error'));

      await expect(
        userService.createUser({ email: 'alice@example.com', name: 'Alice' })
      ).rejects.toThrow('DB error');

      expect(mockEmailService.send).not.toHaveBeenCalled();
    });
  });
});
```

### Coverage — o que ela diz e o que ela não diz

```bash
# Executar testes com coverage
npm run test:coverage

# Exemplo de saída
----------|---------|----------|---------|---------|
File      | % Stmts | % Branch | % Funcs | % Lines |
----------|---------|----------|---------|---------|
All files |   87.5  |   75.0   |   90.0  |   87.5  |
 math.ts  |   100   |   100    |   100   |   100   |
 user.ts  |   80.0  |   60.0   |   85.7  |   80.0  |
----------|---------|----------|---------|---------|
```

Coverage mede quais linhas foram executadas durante os testes, não se essas linhas foram testadas corretamente. 100% de coverage não significa que o código está correto — significa que cada linha executou pelo menos uma vez. Trate coverage como um piso (mínimo de 80%), não como uma meta.

---

## Padrões Comuns e Boas Práticas

- **Uma assertion por teste** — quando um teste falha, você sabe exatamente o que quebrou
- **Nomes de teste descritivos** — "creates a user and sends welcome email", não "test createUser"
- **Estrutura Arrange-Act-Assert** — mantém os testes legíveis e focados
- **Teste a interface, não a implementação** — testes devem sobreviver a refatorações
- **Testes rápidos** — testes unitários devem rodar em milissegundos; testes lentos são pulados
- **Testes determinísticos** — evite dados aleatórios, dependências de tempo e estado compartilhado entre testes

---

## Anti-Padrões a Evitar

- Deletar testes para fazer a suíte passar — você está escondendo bugs
- Testar internals de frameworks (reconciliador do React, internals de roteamento do Express)
- Testes que dependem da ordem de execução — cada teste deve ser independente
- Usar chamadas de rede reais em testes unitários — são lentas, instáveis e testam a coisa errada
- Testar métodos privados — teste apenas a interface pública
- Verificar detalhes de implementação: "esse método foi chamado 3 vezes" quando o que importa é o comportamento

---

## Depuração e Resolução de Problemas

**"Testes passam localmente mas falham no CI"**
Causas mais prováveis: diferenças de ambiente (variáveis de ambiente faltando), dependências na ordem dos testes ou testes sensíveis a tempo. Rode com `--sequence randomize` localmente para detectar problemas de ordenação.

**"O relatório de coverage mostra uma linha como não coberta mas eu sei que os testes a chamam"**
Verifique se a linha está em um branch não percorrido. `if/else` — o branch `else` está sendo testado? `try/catch` — o bloco `catch` está sendo testado?

**"vi.mock não está funcionando — o módulo real ainda está sendo chamado"**
Declarações de mock devem estar no topo do arquivo (são hoistadas). Não coloque `vi.mock` dentro de `beforeEach` ou dentro de `describe`.

---

## Leitura Complementar

- [Documentação do Vitest](https://vitest.dev/)
- [Princípios do Testing Library](https://testing-library.com/docs/guiding-principles)
- [The practical test pyramid — Martin Fowler](https://martinfowler.com/articles/practical-test-pyramid.html)
- Track 09: [Testes Unitários](unit-testing.md)
- Track 09: [Testes de Integração](integration-testing.md)

---

## Resumo

Testes são uma habilidade que se acumula com o tempo. Use a pirâmide de testes como guia: muitos testes unitários rápidos, alguns testes de integração, poucos testes E2E. Use test doubles para isolar unidades. Teste comportamentos e contratos, não detalhes de implementação. Coverage é um piso útil mas um teto perigoso — 80% de cobertura de linhas com testes bem projetados é melhor que 100% com testes que apenas verificam se as funções existem. O hábito mais importante é escrever testes como parte do desenvolvimento, não como etapa posterior — testes escritos depois do código tendem a testar o que o código faz, não o que ele deveria fazer.
