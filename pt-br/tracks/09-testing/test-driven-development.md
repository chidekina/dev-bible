# Desenvolvimento Guiado por Testes

## Visão Geral

O Desenvolvimento Guiado por Testes (TDD, do inglês Test-Driven Development) é uma prática de desenvolvimento onde você escreve o teste antes da implementação. O ciclo é curto: escreva um teste que falha (Vermelho), faça-o passar com o código mínimo necessário (Verde), depois limpe o código sem quebrar o teste (Refatorar). Essa disciplina molda um design melhor — código difícil de testar geralmente está mal projetado. O TDD não é sempre adequado, mas saber quando e como aplicá-lo é uma marca de um engenheiro experiente.

---

## Pré-requisitos

- Track 09: Testes Unitários
- Configuração do Vitest e escrita básica de testes

---

## Conceitos Fundamentais

### O ciclo red-green-refactor

```
RED → Escreva um teste que falha (a especificação)
  ↓
GREEN → Escreva o código mínimo para fazê-lo passar
  ↓
REFACTOR → Limpe sem quebrar o teste
  ↓
(repetir)
```

**Red:** O teste falha porque a funcionalidade ainda não existe. Isso confirma que seu teste realmente testa algo.

**Green:** Escreva o código mais simples que faz o teste passar — mesmo que seja um hack. O teste agora está verde; o comportamento está correto.

**Refactor:** Agora limpe a implementação — extraia funções, remova duplicação, melhore os nomes. O teste garante que você não quebrou nada.

### Quando o TDD vale a pena

- **Regras de negócio** com muitos casos extremos — os testes se tornam especificações executáveis
- **APIs e camadas de serviço** — os testes definem o contrato antes da implementação
- **Correção de bugs** — escreva um teste falhando que reproduz o bug, depois corrija-o (proteção contra regressão)
- **Transformações de dados** — funções puras com entradas/saídas bem definidas

### Quando o TDD é menos útil

- **Prototipagem exploratória** — você ainda não conhece o design; escreva os testes depois
- **Componentes de UI com renderização complexa** — teste o comportamento, não os detalhes de renderização
- **Boilerplate** — endpoints CRUD sem lógica; escreva o código, teste a integração

---

## Exemplos Práticos

### Kata de TDD: carrinho de compras

Construa um módulo de carrinho de compras usando TDD. Definimos o comportamento por meio de testes antes de escrever qualquer implementação.

**Iteração 1 — adicionar itens**

```typescript
// src/lib/cart.test.ts
import { describe, it, expect } from 'vitest';
import { Cart } from './cart.js';

describe('Cart', () => {
  it('starts empty', () => {
    const cart = new Cart();
    expect(cart.items).toHaveLength(0);
    expect(cart.total).toBe(0);
  });
});
```

Execute: RED — `Cart` não existe.

```typescript
// src/lib/cart.ts — mínimo para ficar verde
export class Cart {
  items: CartItem[] = [];
  get total() { return 0; }
}
```

GREEN. Agora expanda:

```typescript
// Próximo teste
it('adds an item to the cart', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 9.99, quantity: 1 });

  expect(cart.items).toHaveLength(1);
  expect(cart.items[0].name).toBe('Widget');
});
```

RED. Implemente `add`:

```typescript
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

export class Cart {
  items: CartItem[] = [];

  get total(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  add(item: CartItem): void {
    this.items.push(item);
  }
}
```

GREEN.

**Iteração 2 — quantidade e totais**

```typescript
it('calculates total for multiple items', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 2 });
  cart.add({ id: '2', name: 'Gadget', price: 5, quantity: 3 });

  expect(cart.total).toBe(35); // 20 + 15
});

it('increments quantity when same item is added again', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 1 });
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 1 });

  expect(cart.items).toHaveLength(1);
  expect(cart.items[0].quantity).toBe(2);
});
```

RED para o segundo teste. Atualize `add`:

```typescript
add(item: CartItem): void {
  const existing = this.items.find((i) => i.id === item.id);
  if (existing) {
    existing.quantity += item.quantity;
  } else {
    this.items.push({ ...item });
  }
}
```

GREEN.

**Iteração 3 — remover e descontos**

```typescript
it('removes an item from the cart', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 1 });
  cart.remove('1');

  expect(cart.items).toHaveLength(0);
});

it('applies a percentage discount', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 100, quantity: 1 });
  cart.applyDiscount(10); // 10%

  expect(cart.total).toBe(90);
});

it('throws if discount is out of range', () => {
  const cart = new Cart();
  expect(() => cart.applyDiscount(-1)).toThrow('Discount must be between 0 and 100');
  expect(() => cart.applyDiscount(101)).toThrow('Discount must be between 0 and 100');
});
```

Implemente `remove` e `applyDiscount`:

```typescript
export class Cart {
  items: CartItem[] = [];
  private discount = 0;

  get total(): number {
    const subtotal = this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    return Math.round(subtotal * (1 - this.discount / 100) * 100) / 100;
  }

  add(item: CartItem): void {
    const existing = this.items.find((i) => i.id === item.id);
    if (existing) {
      existing.quantity += item.quantity;
    } else {
      this.items.push({ ...item });
    }
  }

  remove(id: string): void {
    this.items = this.items.filter((i) => i.id !== id);
  }

  applyDiscount(percent: number): void {
    if (percent < 0 || percent > 100) {
      throw new Error('Discount must be between 0 and 100');
    }
    this.discount = percent;
  }
}
```

GREEN. Os testes servem como especificação viva — lê-los diz exatamente o que o carrinho deve fazer.

### TDD para correção de bugs

Um usuário relata: "Aplicar um desconto duas vezes acumula — 10% + 10% = 20%."

1. Escreva um teste falhando que reproduz o bug:

```typescript
it('replaces existing discount when applied twice', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 100, quantity: 1 });
  cart.applyDiscount(10);
  cart.applyDiscount(10); // aplicar duas vezes não deve acumular

  expect(cart.total).toBe(90); // não 81
});
```

2. Execute: RED — confirma que o bug existe (total é 81).
3. A correção já está na implementação acima (`this.discount = percent` substitui, não soma).
4. GREEN — bug corrigido, e o teste previne regressão.

---

## Padrões Comuns e Boas Práticas

- **Um teste falhando por vez** — não escreva múltiplos testes antes de fazer o primeiro ficar verde
- **Nomes de testes são especificações** — "retorna 90 quando um desconto de 10% é aplicado a um carrinho de R$100"
- **Commit no verde** — commits pequenos e frequentes com a suíte de testes passando
- **A etapa de refactor não é opcional** — código verde com duplicação é dívida técnica; limpe imediatamente
- **Os testes guiam o design** — se um teste é difícil de escrever, o código provavelmente está mal projetado

---

## Anti-Padrões a Evitar

- **Desenvolvimento test-after se autodenominando TDD** — "vou escrever os testes quando terminar" perde o ciclo de feedback de design
- **Pular a etapa de refactor** — acumular código verde mas bagunçado
- **Casos de teste grandes** — cada teste deve testar exatamente um cenário; quebre testes grandes em menores
- **Abandonar o TDD porque o primeiro teste é difícil de escrever** — essa dificuldade é um feedback; redesenhe a interface

---

## Depuração e Resolução de Problemas

**"Meu ciclo TDD está muito lento"**
Cada iteração deve durar menos de 5 minutos. Se um ciclo está levando 20 minutos, você escreveu demais de uma vez. Reduza o passo: escreva o teste mais trivial possível, faça-o ficar verde, depois avance para o próximo.

**"Não sei por onde começar"**
Comece com o comportamento mais simples possível: "um carrinho vazio tem zero itens". O TDD faz as funcionalidades crescerem a partir do caso mais simples para fora.

---

## Leitura Complementar

- [Kent Beck: Test-Driven Development by Example](https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530)
- [Martin Fowler: Is TDD Dead?](https://martinfowler.com/articles/is-tdd-dead/) (série de vídeos)
- [Growing Object-Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com/)
- Track 09: [Testes Unitários](unit-testing.md)

---

## Resumo

O TDD inverte a sequência de desenvolvimento: teste, depois código, depois limpeza. A disciplina tem dois ganhos — design melhor (código fácil de testar tende a ser modular e desacoplado) e proteção imediata contra regressão (você sempre tem uma suíte de testes passando). O ciclo red-green-refactor mantém os passos pequenos e o feedback rápido. O TDD é mais valioso para lógica de negócio complexa, contratos de API e correção de bugs. Aplicado de forma consistente, ele produz uma suíte de testes que é também uma especificação precisa do comportamento do sistema.
