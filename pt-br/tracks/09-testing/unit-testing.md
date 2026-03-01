# Testes Unitários

## Visão Geral

Testes unitários verificam que um pedaço pequeno e isolado de código — uma função, um método de classe, um utilitário — se comporta corretamente para entradas específicas. São os testes mais rápidos de escrever e executar, e fornecem o feedback mais granular quando algo quebra. Uma suíte de testes unitários bem escrita é a espinha dorsal de uma engenharia confiante e ágil. Este capítulo cobre testes unitários com vitest em um contexto Node.js/TypeScript: estrutura, padrões, testes assíncronos e testes de erro.

---

## Pré-requisitos

- Track 09: Fundamentos de Testes (pirâmide de testes, test doubles)
- Conceitos básicos de TypeScript
- Configuração do Vitest do capítulo de fundamentos

---

## Conceitos Fundamentais

### O que faz um bom teste unitário

1. **Rápido** — executa em milissegundos
2. **Isolado** — sem I/O, sem rede real, sem banco de dados real
3. **Determinístico** — mesmo resultado a cada execução, independente do ambiente
4. **Focado** — testa exatamente um comportamento
5. **Auto-descritivo** — o nome do teste diz o que ele verifica

### A função sendo testada

Uma função pura (mesma entrada → mesma saída, sem efeitos colaterais) é a coisa mais fácil de testar unitariamente. A disciplina de escrever código testável frequentemente conduz a um design melhor.

---

## Exemplos Práticos

### Testando funções puras

```typescript
// src/lib/validation.ts
export function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount);
}

export function clamp(value: number, min: number, max: number): number {
  if (min > max) throw new Error('min cannot be greater than max');
  return Math.min(Math.max(value, min), max);
}
```

```typescript
// src/lib/validation.test.ts
import { describe, it, expect } from 'vitest';
import { isValidEmail, formatCurrency, clamp } from './validation.js';

describe('isValidEmail', () => {
  it('returns true for valid email addresses', () => {
    expect(isValidEmail('alice@example.com')).toBe(true);
    expect(isValidEmail('user+tag@domain.co.uk')).toBe(true);
  });

  it('returns false for invalid email addresses', () => {
    expect(isValidEmail('')).toBe(false);
    expect(isValidEmail('not-an-email')).toBe(false);
    expect(isValidEmail('@example.com')).toBe(false);
    expect(isValidEmail('user@')).toBe(false);
  });
});

describe('formatCurrency', () => {
  it('formats USD amounts', () => {
    expect(formatCurrency(1000, 'USD')).toBe('$1,000.00');
    expect(formatCurrency(0.5, 'USD')).toBe('$0.50');
  });

  it('formats EUR amounts', () => {
    expect(formatCurrency(42, 'EUR')).toBe('€42.00');
  });
});

describe('clamp', () => {
  it('returns the value when within range', () => {
    expect(clamp(5, 0, 10)).toBe(5);
    expect(clamp(0, 0, 10)).toBe(0);
    expect(clamp(10, 0, 10)).toBe(10);
  });

  it('clamps to min when value is too low', () => {
    expect(clamp(-5, 0, 10)).toBe(0);
  });

  it('clamps to max when value is too high', () => {
    expect(clamp(15, 0, 10)).toBe(10);
  });

  it('throws when min is greater than max', () => {
    expect(() => clamp(5, 10, 0)).toThrow('min cannot be greater than max');
  });
});
```

### Testando funções assíncronas

```typescript
// src/services/pricing.service.ts
export class PricingService {
  constructor(private exchangeRateApi: ExchangeRateApi) {}

  async convertPrice(amount: number, from: string, to: string): Promise<number> {
    if (from === to) return amount;

    const rate = await this.exchangeRateApi.getRate(from, to);
    if (!rate) throw new Error(`Exchange rate not available: ${from}/${to}`);

    return Math.round(amount * rate * 100) / 100;
  }
}
```

```typescript
// src/services/pricing.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { PricingService } from './pricing.service.js';

const mockExchangeRateApi = {
  getRate: vi.fn(),
};

let service: PricingService;

beforeEach(() => {
  vi.clearAllMocks();
  service = new PricingService(mockExchangeRateApi as any);
});

describe('PricingService.convertPrice', () => {
  it('returns the same amount when currencies match', async () => {
    const result = await service.convertPrice(100, 'USD', 'USD');
    expect(result).toBe(100);
    expect(mockExchangeRateApi.getRate).not.toHaveBeenCalled();
  });

  it('converts using the exchange rate', async () => {
    mockExchangeRateApi.getRate.mockResolvedValue(5.0); // 1 USD = 5 BRL

    const result = await service.convertPrice(100, 'USD', 'BRL');
    expect(result).toBe(500);
  });

  it('throws when exchange rate is unavailable', async () => {
    mockExchangeRateApi.getRate.mockResolvedValue(null);

    await expect(service.convertPrice(100, 'USD', 'XYZ')).rejects.toThrow(
      'Exchange rate not available: USD/XYZ'
    );
  });

  it('rounds to 2 decimal places', async () => {
    mockExchangeRateApi.getRate.mockResolvedValue(1.333);

    const result = await service.convertPrice(10, 'USD', 'EUR');
    expect(result).toBe(13.33); // não 13.330...
  });
});
```

### Testando com spies (rastreando chamadas sem alterar o comportamento)

```typescript
// src/lib/audit.ts
import { logger } from './logger.js';

export function auditLog(action: string, userId: string, metadata?: Record<string, unknown>) {
  logger.info({ action, userId, ...metadata }, 'Audit event');
}
```

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import { logger } from './logger.js';
import { auditLog } from './audit.js';

vi.mock('./logger.js', () => ({
  logger: { info: vi.fn() },
}));

describe('auditLog', () => {
  afterEach(() => vi.clearAllMocks());

  it('logs the action and userId', () => {
    auditLog('login', 'user-123');

    expect(logger.info).toHaveBeenCalledOnce();
    expect(logger.info).toHaveBeenCalledWith(
      { action: 'login', userId: 'user-123' },
      'Audit event'
    );
  });

  it('includes metadata in the log', () => {
    auditLog('update', 'user-456', { field: 'email', oldValue: 'a@b.com' });

    expect(logger.info).toHaveBeenCalledWith(
      { action: 'update', userId: 'user-456', field: 'email', oldValue: 'a@b.com' },
      'Audit event'
    );
  });
});
```

### Testando com fake timers

```typescript
// src/lib/debounce.ts
export function debounce<T extends (...args: any[]) => any>(fn: T, delayMs: number): T {
  let timer: ReturnType<typeof setTimeout>;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delayMs);
  } as T;
}
```

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { debounce } from './debounce.js';

describe('debounce', () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it('delays calling the function', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 300);

    debounced('a');
    expect(fn).not.toHaveBeenCalled(); // ainda não foi chamada

    vi.advanceTimersByTime(300);
    expect(fn).toHaveBeenCalledOnce();
    expect(fn).toHaveBeenCalledWith('a');
  });

  it('only calls once if invoked multiple times within the delay', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 300);

    debounced('a');
    debounced('b');
    debounced('c');

    vi.advanceTimersByTime(300);
    expect(fn).toHaveBeenCalledOnce();
    expect(fn).toHaveBeenCalledWith('c'); // apenas a última chamada
  });
});
```

### Testando limites de erro

```typescript
// src/lib/result.ts — tipo Result no estilo neverthrow
export type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

export function parseJson<T>(json: string): Result<T, SyntaxError> {
  try {
    return { ok: true, value: JSON.parse(json) as T };
  } catch (e) {
    return { ok: false, error: e as SyntaxError };
  }
}
```

```typescript
describe('parseJson', () => {
  it('returns ok:true for valid JSON', () => {
    const result = parseJson<{ name: string }>('{"name":"Alice"}');
    expect(result.ok).toBe(true);
    if (result.ok) {
      expect(result.value.name).toBe('Alice');
    }
  });

  it('returns ok:false for invalid JSON', () => {
    const result = parseJson('not valid json');
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error).toBeInstanceOf(SyntaxError);
    }
  });
});
```

---

## Padrões Comuns e Boas Práticas

- **`beforeEach` para setup compartilhado** — resete mocks, crie instâncias novas antes de cada teste
- **`afterEach` para limpeza** — restaure timers, limpe mocks, feche conexões
- **`vi.clearAllMocks()` no beforeEach** — evita que contagens de chamadas de mock vazem entre testes
- **Testes parametrizados com `it.each`** — evite casos de teste repetitivos

```typescript
describe('clamp', () => {
  it.each([
    [5, 0, 10, 5],   // [value, min, max, expected]
    [-5, 0, 10, 0],
    [15, 0, 10, 10],
    [0, 0, 10, 0],
    [10, 0, 10, 10],
  ])('clamp(%i, %i, %i) = %i', (value, min, max, expected) => {
    expect(clamp(value, min, max)).toBe(expected);
  });
});
```

---

## Anti-Padrões a Evitar

- Fazer mock de tudo — testes que mockam cada dependência não testam nada de real
- Testes longos que verificam muitas coisas não relacionadas — divida em testes focados
- Estado mutável compartilhado entre testes — use `beforeEach` para resetar
- Testar métodos privados — exponha-os pela API pública ou extraia-os para um módulo separado
- Não testar o caminho infeliz — testar apenas casos de sucesso deixa passar a maioria dos bugs reais

---

## Depuração e Resolução de Problemas

**"Meu mock não está sendo chamado"**
Verifique o caminho de importação no `vi.mock()` — ele deve ser exatamente igual ao caminho usado no arquivo sendo testado. Use caminhos relativos de forma consistente.

**"O teste vaza estado para o próximo teste"**
Adicione `vi.clearAllMocks()` ou `vi.resetAllMocks()` no `beforeEach`. Para estado global, resete-o explicitamente.

**"Teste assíncrono dá timeout"**
Certifique-se de que você está fazendo `await` em toda operação assíncrona. Se o teste travar, você pode ter uma promise não resolvida ou uma chamada de I/O real que nunca resolve. Adicione um timeout: `it('...', async () => { ... }, 5000)`.

---

## Leitura Complementar

- [Vitest: guia de mocking](https://vitest.dev/guide/mocking.html)
- [Vitest: fake timers](https://vitest.dev/guide/mocking.html#timers)
- [Kent C. Dodds: Testing implementation details](https://kentcdodds.com/blog/testing-implementation-details)
- Track 09: [Fundamentos de Testes](testing-fundamentals.md)
- Track 09: [Desenvolvimento Guiado por Testes](test-driven-development.md)

---

## Resumo

Testes unitários são a camada mais barata, mais rápida e mais numerosa da pirâmide de testes. Escreva-os para todas as funções com lógica não trivial. Teste funções puras diretamente (sem necessidade de mock). Use `vi.fn()` para dependências que fazem chamadas de I/O ou têm efeitos colaterais. Teste tanto os caminhos de sucesso quanto os de erro. Use `it.each` para casos parametrizados. Mantenha cada teste focado em um único comportamento, com um nome que leia como uma especificação. A suíte de testes resultante não é apenas uma rede de segurança — é documentação viva do que o código deve fazer.
