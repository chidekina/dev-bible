# Mocking & Stubs

## Visão Geral

Mocking e stubbing são as técnicas que tornam os testes unitários possíveis. Sem elas, cada teste precisaria de um banco de dados real, um serviço de e-mail ativo e uma API de terceiros em funcionamento — tornando os testes lentos, frágeis e dependentes de estado externo. Este capítulo cobre a mecânica do mocking no vitest, quando usar mocks versus alternativas, e a disciplina crítica de não fazer over-mocking. Também cobre o MSW (Mock Service Worker) para mockar HTTP em testes de frontend.

---

## Pré-requisitos

- Track 09: Fundamentos de Testes (vocabulário de test doubles)
- Track 09: Testes Unitários
- Entendimento básico de módulos ES

---

## Conceitos Fundamentais

### O espectro do mocking

```
Sem mocking (dependências reais)
  → Território de testes de integração
  → Captura bugs reais de integração
  → Lento, requer infraestrutura

Fake (implementação em memória)
  → Rápido, comportamento realista
  → Bom para repositórios, armazenamento

Stub (retorna valores pré-definidos)
  → Rápido, setup mínimo
  → Bom para APIs externas, configuração

Mock (verifica comportamento de chamadas)
  → Verifica COMO o código interage com dependências
  → Use com moderação — acopla o teste à implementação

Mocking total (mocka tudo)
  → Testes não testam nada de real
  → Anti-padrão
```

### A regra "Não mocke o que você não controla"

Evite mockar tipos que você não controla (ex.: `fetch`, `axios`, o `PrismaClient` do Prisma). Mocke na fronteira que você controla — mocke sua interface de repositório, não o Prisma diretamente. Isso desacopla os testes dos internals das bibliotecas.

---

## Exemplos Práticos

### Mocking de módulo com vi.mock

```typescript
// src/services/notification.service.ts
import { sendEmail } from '../lib/mailer.js'; // dependência externa

export async function notifyUser(userId: string, message: string) {
  const user = await getUserById(userId);
  await sendEmail({ to: user.email, subject: 'Notification', body: message });
  return { sent: true };
}
```

```typescript
// src/services/notification.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

// vi.mock é hoistado — executa antes dos imports
vi.mock('../lib/mailer.js', () => ({
  sendEmail: vi.fn().mockResolvedValue({ id: 'email-123' }),
}));

vi.mock('../lib/user.js', () => ({
  getUserById: vi.fn().mockResolvedValue({ id: 'u1', email: 'alice@example.com' }),
}));

import { notifyUser } from './notification.service.js';
import { sendEmail } from '../lib/mailer.js';
import { getUserById } from '../lib/user.js';

beforeEach(() => vi.clearAllMocks());

describe('notifyUser', () => {
  it('sends an email to the user', async () => {
    await notifyUser('u1', 'Hello!');

    expect(sendEmail).toHaveBeenCalledWith({
      to: 'alice@example.com',
      subject: 'Notification',
      body: 'Hello!',
    });
  });

  it('returns sent:true on success', async () => {
    const result = await notifyUser('u1', 'Hello!');
    expect(result.sent).toBe(true);
  });
});
```

### Espiando métodos sem substituí-los

```typescript
import { vi, describe, it, expect, afterEach } from 'vitest';
import { logger } from '../lib/logger.js';

describe('logger spy', () => {
  afterEach(() => vi.restoreAllMocks());

  it('logs the correct message', () => {
    const spy = vi.spyOn(logger, 'info');

    doSomethingThatLogs();

    expect(spy).toHaveBeenCalledWith(expect.objectContaining({ action: 'user.login' }), 'Audit');
  });
});
```

`vi.spyOn` envolve o método — a implementação real ainda executa, mas as chamadas são registradas.

### Construtores de mock no estilo factory

```typescript
// Cria factories de mock reutilizáveis
function createMockUserRepo() {
  return {
    findById: vi.fn<[string], Promise<User | null>>().mockResolvedValue(null),
    findByEmail: vi.fn<[string], Promise<User | null>>().mockResolvedValue(null),
    create: vi.fn<[CreateUserDto], Promise<User>>(),
    update: vi.fn<[string, Partial<User>], Promise<User>>(),
    delete: vi.fn<[string], Promise<void>>().mockResolvedValue(undefined),
  };
}

describe('UserService', () => {
  it('throws when user not found', async () => {
    const mockRepo = createMockUserRepo();
    mockRepo.findById.mockResolvedValue(null); // explícito para clareza

    const service = new UserService(mockRepo);
    await expect(service.getUser('non-existent')).rejects.toThrow('User not found');
  });

  it('updates and returns the user', async () => {
    const mockRepo = createMockUserRepo();
    const existingUser = { id: '1', email: 'a@b.com', name: 'Alice', role: 'user' } as User;
    const updatedUser = { ...existingUser, name: 'Alicia' };

    mockRepo.findById.mockResolvedValue(existingUser);
    mockRepo.update.mockResolvedValue(updatedUser);

    const service = new UserService(mockRepo);
    const result = await service.updateUser('1', { name: 'Alicia' });

    expect(result.name).toBe('Alicia');
    expect(mockRepo.update).toHaveBeenCalledWith('1', { name: 'Alicia' });
  });
});
```

### MSW — mockando requisições HTTP

O MSW intercepta chamadas `fetch`/`axios` no nível da rede, sem precisar substituir o fetch global:

```bash
npm install --save-dev msw
```

```typescript
// src/test/msw-handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('https://api.github.com/users/:username', ({ params }) => {
    return HttpResponse.json({
      login: params.username,
      name: 'Test User',
      public_repos: 42,
    });
  }),

  http.post('https://api.stripe.com/v1/charges', () => {
    return HttpResponse.json({ id: 'ch_test_123', status: 'succeeded' });
  }),

  // Simula respostas de erro
  http.get('https://api.example.com/flaky', () => {
    return new HttpResponse(null, { status: 503 });
  }),
];
```

```typescript
// src/test/msw-setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './msw-handlers.js';

export const server = setupServer(...handlers);

// No arquivo de setup do vitest:
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers()); // restaura padrões entre testes
afterAll(() => server.close());
```

```typescript
// src/services/github.service.test.ts
import { describe, it, expect } from 'vitest';
import { server } from '../test/msw-setup.js';
import { http, HttpResponse } from 'msw';
import { getGithubUser } from './github.service.js';

describe('getGithubUser', () => {
  it('returns user data from GitHub API', async () => {
    const user = await getGithubUser('octocat');
    expect(user.login).toBe('octocat');
    expect(user.publicRepos).toBe(42);
  });

  it('throws on rate limit', async () => {
    // Sobrescreve o handler apenas para este teste específico
    server.use(
      http.get('https://api.github.com/users/:username', () => {
        return new HttpResponse(null, { status: 429 });
      })
    );

    await expect(getGithubUser('anyone')).rejects.toThrow('Rate limited');
  });
});
```

### Mocking parcial — mocke apenas o que você precisa

```typescript
// Mocka apenas o export problemático, não o módulo inteiro
vi.mock('../lib/db.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('../lib/db.js')>();
  return {
    ...actual, // mantém todos os exports reais
    db: {      // sobrescreve apenas o cliente db
      user: {
        findUnique: vi.fn(),
        create: vi.fn(),
      },
    },
  };
});
```

---

## Padrões Comuns e Boas Práticas

- **Mocke na fronteira que você controla** — interface de repositório, não o ORM
- **Resete mocks no `beforeEach`** — evita poluição entre testes
- **Use `mockResolvedValue` para async** e `mockReturnValue` para sync
- **Verifique o comportamento, não a contagem de chamadas do mock** quando possível
- **Use MSW para HTTP** — mais realista que substituir o `fetch`; detecta incompatibilidades de URL
- **Tipar seus mocks** — `vi.fn<[argType], ReturnType>()` detecta erros de tipo no setup do mock

---

## Anti-Padrões a Evitar

- Mockar cada dependência em cada teste — over-mocking cria testes que testam os próprios mocks
- Verificar a contagem de chamadas do mock quando é um detalhe de implementação — prefira verificar o resultado
- Não resetar mocks entre testes — o valor de retorno do mock de um teste vaza para o próximo
- Mockar módulos que são funções puras sem efeitos colaterais — basta chamar a função real

---

## Depuração e Resolução de Problemas

**"vi.mock está sendo ignorado — o módulo real ainda está sendo chamado"**
`vi.mock` deve estar no nível mais alto do arquivo (ele é hoistado pelo Vitest). Se estiver dentro de um `describe` ou `it`, não vai funcionar. Verifique também se o caminho do módulo é igual ao import no arquivo sendo testado.

**"MSW está capturando requisições que não deveria"**
Configure `onUnhandledRequest: 'error'` no `server.listen()` — isso faz com que requisições não tratadas lancem um erro, ajudando a descobrir onde requisições reais estão escapando.

**"Mock retorna undefined"**
Você esqueceu o `.mockResolvedValue(...)` ou `.mockReturnValue(...)`. Um `vi.fn()` sem valor de retorno configurado retorna `undefined` por padrão.

---

## Leitura Complementar

- [Guia de mocking do Vitest](https://vitest.dev/guide/mocking.html)
- [Documentação do MSW](https://mswjs.io/docs/)
- [Kent C. Dodds: Stop mocking fetch](https://kentcdodds.com/blog/stop-mocking-fetch)
- Track 09: [Testes Unitários](unit-testing.md)
- Track 09: [Testes de Integração](integration-testing.md)

---

## Resumo

Mocking é uma ferramenta para isolamento, não uma filosofia de testes. Use stubs para fornecer respostas prontas, spies para verificar efeitos colaterais, e MSW para interceptar HTTP sem substituir globals. Sempre resete os mocks entre os testes. Mocke na fronteira da interface que você controla — não vá fundo nos internals das bibliotecas. A contenção mais importante é saber quando não mockar: se uma função é pura ou um teste de integração daria mais confiança, prefira a coisa real. Testes com over-mocking são fardos de manutenção que quebram quando a implementação muda, não quando o comportamento muda.
