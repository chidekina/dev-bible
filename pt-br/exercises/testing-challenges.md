# Desafios de Testes

> Exercícios práticos de testes cobrindo unitários, integração, E2E, TDD e testes de performance. Cada exercício reflete um cenário real de testes em produção. Stack assumida: Node.js 20+, TypeScript, Vitest, Fastify, Playwright, MSW. Nível alvo: iniciante a avançado.

---

## Exercício 1 — Testes Unitários para uma Função Pura (Fácil)

**Cenário:** A seguinte função utilitária não tem testes. Escreva uma suite de testes unitários abrangente para ela.

**Código fornecido:**
```typescript
export function paginate<T>(items: T[], page: number, limit: number): {
  data: T[];
  meta: { page: number; limit: number; total: number; totalPages: number; hasNext: boolean; hasPrev: boolean };
} {
  if (page < 1) throw new Error('Page must be >= 1');
  if (limit < 1 || limit > 100) throw new Error('Limit must be between 1 and 100');
  const total = items.length;
  const totalPages = Math.ceil(total / limit);
  const start = (page - 1) * limit;
  const data = items.slice(start, start + limit);
  return {
    data,
    meta: { page, limit, total, totalPages, hasNext: page < totalPages, hasPrev: page > 1 },
  };
}
```

**Requisitos:**
- Alcance 100% de cobertura de branches e statements na função `paginate`.
- Use Vitest com blocos `describe`/`it`, não chamadas `test` planas.
- Cada `it` testa um comportamento — sem testes "funciona tudo" monolíticos.
- Use `expect.assertions(N)` em testes que verificam erros lançados.

**Critérios de Aceite:**
- [ ] Caminho feliz: primeira página, página do meio, última página retornam os slices corretos de `data`.
- [ ] Borda: `page = 1, limit = 100, items.length = 0` retorna `{ data: [], meta: { total: 0, totalPages: 0, hasNext: false, hasPrev: false } }`.
- [ ] `page < 1` lança `'Page must be >= 1'`.
- [ ] `limit = 101` lança `'Limit must be between 1 and 100'`.
- [ ] `hasNext` e `hasPrev` são `true`/`false` nas páginas corretas.
- [ ] Relatório de cobertura do Vitest mostra 100% de cobertura.

**Dicas:**
1. Use `it.each` para casos de caminho feliz parametrizados: `[[1, 10, 100], [2, 10, 100], [10, 10, 100]]` (page, limit, total de itens).
2. Para casos de erro: `expect(() => paginate([], 0, 10)).toThrow('Page must be >= 1')`.
3. Gere itens de teste com `Array.from({ length: N }, (_, i) => i + 1)`.
4. Agrupe testes: `describe('cálculos de paginação')`, `describe('casos de erro')`, `describe('flags de meta')`.

---

## Exercício 2 — Testes de Integração para uma Rota Fastify (Médio)

**Cenário:** Escreva testes de integração para uma rota `POST /users` contra um banco de dados SQLite real (em memória). Sem mock da camada de banco de dados.

**Requisitos:**
- Use `better-sqlite3` como banco de dados em memória para testes.
- Cada teste recebe um banco de dados novo (configurado em `beforeEach`, destruído em `afterEach`).
- Testes constroem o app Fastify usando a mesma função factory da produção.
- Teste na camada HTTP (use `fastify.inject`) — não chamando funções de serviço diretamente.
- Cubra: caso de sucesso, falha de validação, conflito de email duplicado.

**Critérios de Aceite:**
- [ ] `POST /users` com dados válidos retorna `201` e o usuário criado (sem o hash da senha).
- [ ] `POST /users` com campo `email` faltando retorna `422` com erro por campo.
- [ ] `POST /users` com email já cadastrado retorna `409 Conflict`.
- [ ] Cada teste é independente — criar um usuário no teste 1 não afeta o teste 2.
- [ ] O banco de dados de teste é criado com migrações aplicadas (mesmos arquivos de migração da produção).

**Dicas:**
1. Factory do app: `export function buildApp(db: Database) { const fastify = Fastify(); /* registre rotas com db injetado */; return fastify; }`.
2. Em `beforeEach`: `const db = new Database(':memory:'); applyMigrations(db); app = buildApp(db); await app.ready();`.
3. Em `afterEach`: `await app.close(); db.close();`.
4. Exemplo com `fastify.inject`: `const res = await app.inject({ method: 'POST', url: '/users', payload: { email: 'a@b.com', password: 'secret' } })`.

---

## Exercício 3 — E2E com Playwright para um Fluxo de Login (Médio)

**Cenário:** Escreva uma suite de testes E2E com Playwright cobrindo o fluxo de login de uma aplicação web: login válido, senha errada e redirecionamento após login.

**Requisitos:**
- Testes rodam contra um servidor em execução local.
- Use o Page Object Model do Playwright — crie uma classe `LoginPage` com helpers de locator.
- Cubra: login bem-sucedido → redirecionamento para `/dashboard`, senha errada → mensagem de erro, formulário vazio → validação HTML5.
- Testes devem ser independentes — cada teste começa de um estado limpo (conta de usuário nova).
- Execute testes em modo headed localmente e headless no CI.

**Critérios de Aceite:**
- [ ] A classe `LoginPage` tem métodos: `goto()`, `fillEmail(email)`, `fillPassword(password)`, `submit()`, `getErrorMessage()`.
- [ ] O teste de login bem-sucedido verifica: URL muda para `/dashboard` e nome do usuário aparece na barra de navegação.
- [ ] O teste de senha errada verifica: mensagem de erro `"Email ou senha inválidos"` está visível.
- [ ] Testes criam um usuário de teste via API (`POST /test-helpers/users`) antes de cada teste — não pela UI.
- [ ] Config de CI (`playwright.config.ts`) usa `headless: true` e roda em `ubuntu-latest`.

**Dicas:**
1. Page Object:
   ```typescript
   class LoginPage {
     constructor(private page: Page) {}
     goto() { return this.page.goto('/login'); }
     fillEmail(email: string) { return this.page.fill('[data-testid="email"]', email); }
   }
   ```
2. Use `test.beforeEach` para criar um usuário de teste novo via `request.post('/test-helpers/users', { data: { email, password } })`.
3. `test.use({ baseURL: 'http://localhost:3000' })` em `playwright.config.ts`.
4. Use `await expect(page).toHaveURL('/dashboard')` — Playwright aguarda automaticamente a navegação.

---

## Exercício 4 — Kata TDD: Carrinho de Compras (Médio)

**Cenário:** Implemente um carrinho de compras usando Test-Driven Development. Escreva os testes primeiro, depois implemente apenas o código suficiente para fazer cada teste passar. Não escreva nenhum código de implementação antes do teste correspondente.

**Requisitos (implemente nesta ordem):**
1. Um carrinho vazio tem 0 itens e total de 0.
2. Adicionar um item aumenta a contagem de itens.
3. Adicionar o mesmo item duas vezes aumenta sua quantidade (não adiciona duplicata).
4. `removeItem` diminui a quantidade em 1; remover a última unidade remove o item.
5. `getTotal` retorna a soma de `price * quantity` para todos os itens.
6. Um cupom de desconto de 10% reduz o total em 10%.
7. Aplicar um cupom inválido lança `InvalidCouponError`.

**Critérios de Aceite:**
- [ ] Histórico do git mostra um commit por ciclo red-green-refactor (pelo menos 7 commits para 7 comportamentos).
- [ ] Nenhum código de implementação existe antes do teste com falha correspondente.
- [ ] A implementação final tem menos de 60 linhas de código.
- [ ] Todos os 7 comportamentos têm testes; cobertura é 100%.
- [ ] A classe `Cart` não tem dependências — é uma estrutura de dados puramente em memória.

**Dicas:**
1. Ciclo TDD: escreva um teste com falha → escreva código mínimo para passar → refatore → commit → repita.
2. Comece com: `it('começa vazio', () => { const cart = new Cart(); expect(cart.itemCount).toBe(0); })`.
3. Estado interno do `Cart`: `items: Map<string, { price: number; quantity: number }>`.
4. Validação de cupom: um conjunto hardcoded de cupons válidos (`{ 'SAVE10': 0.10 }`) é suficiente para o kata.

---

## Exercício 5 — Mock de API HTTP Externa com MSW (Médio)

**Cenário:** Seu serviço chama uma API de clima de terceiros. Os testes atualmente fazem requisições HTTP reais, tornando-os lentos e instáveis. Substitua-os por mocks com MSW (Mock Service Worker).

**Requisitos:**
- Configure handlers MSW no lado do servidor para `GET https://api.clima.com/v1/current?city=<city>`.
- Handler do caminho feliz retorna `{ city, temperature, unit: 'celsius', condition: 'sunny' }`.
- Adicione handlers de erro: 404 para cidades desconhecidas, 500 para `city=error`.
- Testes para sua função de serviço `getWeather(city: string)` devem usar MSW — sem `jest.mock` ou `vi.mock` no cliente HTTP.
- O servidor MSW inicia antes da suite de testes e reinicia handlers entre testes.

**Critérios de Aceite:**
- [ ] `getWeather('São Paulo')` retorna o payload mockado (sem chamada HTTP real).
- [ ] `getWeather('cidade-desconhecida')` lança `WeatherNotFoundError`.
- [ ] `getWeather('error')` lança `WeatherServiceError`.
- [ ] Requisições HTTP reais são bloqueadas durante testes (modo `onUnhandledRequest: 'error'` do MSW).
- [ ] Adicionar um novo teste que precisa de uma resposta diferente usa `server.use(http.get(...))` para sobrescrever o handler padrão apenas para aquele teste.

**Dicas:**
1. Setup MSW para Node.js (não browser): `import { setupServer } from 'msw/node'`.
2. Handler: `http.get('https://api.clima.com/v1/current', ({ request }) => { const url = new URL(request.url); const city = url.searchParams.get('city'); return HttpResponse.json({ city, temperature: 25 }); })`.
3. Em `beforeAll`: `server.listen({ onUnhandledRequest: 'error' })`. Em `afterEach`: `server.resetHandlers()`. Em `afterAll`: `server.close()`.
4. Sobrescrever por teste: `server.use(http.get('https://api.clima.com/v1/current', () => HttpResponse.error()))`.

---

## Exercício 6 — Testando Tratamento de Erros Assíncronos (Médio)

**Cenário:** Uma função de serviço busca dados do usuário, envia um email de boas-vindas e registra o evento. Escreva testes que verificam o tratamento correto de erros quando cada etapa falha independentemente.

**Código fornecido:**
```typescript
async function onboardUser(userId: string): Promise<void> {
  const user = await userRepo.findById(userId);
  if (!user) throw new UserNotFoundError(userId);
  await emailService.sendWelcome(user.email);
  await analytics.track('user_onboarded', { userId });
}
```

**Requisitos:**
- Teste: usuário não encontrado → `UserNotFoundError` é lançado, email não é enviado.
- Teste: serviço de email lança → erro se propaga, analytics não é chamado.
- Teste: analytics lança → erro se propaga (email já foi enviado, sem rollback).
- Teste: caminho feliz → email é enviado e analytics é rastreado.
- Todas as dependências (`userRepo`, `emailService`, `analytics`) são injetadas — sem mock a nível de módulo.

**Critérios de Aceite:**
- [ ] `userRepo.findById` retornando `null` faz `onboardUser` lançar `UserNotFoundError`.
- [ ] `emailService.sendWelcome` nunca é chamado quando o usuário não é encontrado.
- [ ] `analytics.track` nunca é chamado quando `emailService.sendWelcome` lança.
- [ ] Cada teste usa spies do Vitest (`vi.fn()`) injetados via construtor ou argumentos de função.
- [ ] Sem `try/catch` nos corpos dos testes — use `expect(promise).rejects.toThrow(UserNotFoundError)`.

**Dicas:**
1. Injete dependências: `function onboardUser(userId, deps: { userRepo, emailService, analytics })`.
2. Mock `userRepo.findById`: `vi.fn().mockResolvedValue(null)` para o caso de não encontrado.
3. Mock de falha de email: `vi.fn().mockRejectedValue(new Error('SMTP timeout'))`.
4. Verificar que não foi chamado: `expect(emailService.sendWelcome).not.toHaveBeenCalled()`.

---

## Exercício 7 — Alcançar 80% de Cobertura em um Módulo Dado (Médio)

**Cenário:** Um colega escreveu um módulo `discountCalculator.ts` com 0% de cobertura de testes. Sua tarefa é alcançar 80% de cobertura de linhas e branches sem modificar a implementação.

**Código fornecido:**
```typescript
export function calculateDiscount(
  price: number,
  userType: 'regular' | 'vip' | 'employee',
  promoCode?: string
): number {
  if (price <= 0) return 0;
  let discount = 0;
  if (userType === 'vip') discount += 0.15;
  if (userType === 'employee') discount += 0.30;
  if (promoCode === 'SUMMER20') discount += 0.20;
  if (promoCode === 'FLASH50') discount = 0.50; // sobrescreve outros descontos
  if (discount > 0.50) discount = 0.50; // limite de 50%
  return Math.round(price * (1 - discount) * 100) / 100;
}
```

**Requisitos:**
- Escreva testes para alcançar pelo menos 80% de cobertura de linhas e 80% de cobertura de branches.
- Documente em um comentário quais branches você intencionalmente pulou e por quê.
- Use `vitest --coverage` para verificar os thresholds de cobertura.
- Configure `vitest.config.ts` para impor os thresholds de 80% — build falha abaixo do threshold.

**Critérios de Aceite:**
- [ ] `vitest --coverage` reporta >= 80% de linhas e >= 80% de branches para `discountCalculator.ts`.
- [ ] A combinação VIP + SUMMER20 (limitada a 50%) é testada.
- [ ] A sobrescrita do `FLASH50` (ignora outros descontos) é testada.
- [ ] `vitest.config.ts` `coverage.thresholds` é definido para `{ lines: 80, branches: 80 }`.
- [ ] Nenhuma mudança de implementação — apenas novo arquivo de teste.

**Dicas:**
1. Conte branches: cada `if` tem 2 branches (verdadeiro/falso). Mapeie-os antes de escrever testes.
2. VIP (15%) + SUMMER20 (20%) = 35% — abaixo do limite. Funcionário (30%) + SUMMER20 (20%) = 50% — exatamente no limite. Teste ambos.
3. `vitest.config.ts`: `coverage: { provider: 'v8', thresholds: { lines: 80, branches: 80 } }`.
4. Os branches "intencionalmente pulados" (ex: `price === 0` exato) devem ser documentados.

---

## Exercício 8 — Teste de Snapshot de um Componente React (Fácil)

**Cenário:** Escreva testes de snapshot para um componente React `UserCard` para capturar regressões de UI não intencionais.

**Requisitos:**
- O componente `UserCard` renderiza: avatar, nome, email, badge de papel e um rótulo "Admin" opcional.
- Escreva testes de snapshot para: estado padrão, com rótulo admin, sem avatar (iniciais como fallback).
- Use Vitest + `@testing-library/react` com jsdom.
- Quando o componente mudar intencionalmente, atualize snapshots com `--updateSnapshot` — nunca edite manualmente os arquivos de snapshot.
- Adicione um teste comportamental (não de snapshot): clicar no botão "Editar" chama uma prop `onEdit`.

**Critérios de Aceite:**
- [ ] Três testes de snapshot, cada um em seu próprio bloco `it` com nome descritivo.
- [ ] Arquivos de snapshot são commitados no repositório (`./__snapshots__/UserCard.test.tsx.snap`).
- [ ] O teste comportamental usa `fireEvent.click` e `expect(onEdit).toHaveBeenCalledOnce()`.
- [ ] Snapshots usam `toMatchInlineSnapshot` para componentes pequenos ou `toMatchSnapshot` para maiores.
- [ ] Testes de snapshot passam no CI sem regenerar (snapshots são commitados).

**Dicas:**
1. Setup: `import { render } from '@testing-library/react'`. Use `container.firstChild` ou `asFragment()` como sujeito do snapshot.
2. `toMatchInlineSnapshot()` é preferido para componentes com menos de ~20 linhas de HTML renderizado.
3. Mock de imagens: forneça uma string `src` — jsdom não carrega imagens reais.
4. Atualize snapshots: `npx vitest --updateSnapshot`. Revise o diff cuidadosamente antes de commitar.

---

## Exercício 9 — Teste de Carga de um Endpoint com k6 (Médio)

**Cenário:** Defina um teste de carga k6 para o endpoint `GET /products` com SLOs (Service Level Objectives) explícitos. O teste deve falhar se os SLOs não forem atendidos.

**Requisitos:**
- Suba gradualmente para 500 usuários virtuais ao longo de 1 minuto, mantenha por 3 minutos, reduza ao longo de 1 minuto.
- SLOs: tempo de resposta p95 < 200ms, p99 < 500ms, taxa de erro < 0,1%.
- O script k6 deve usar asserções `check` e thresholds customizados.
- Parametrize a URL base para que o mesmo script funcione em staging e produção.
- Produza um relatório de resumo (saída JSON padrão do k6) adequado para upload como artifact de CI.

**Critérios de Aceite:**
- [ ] Script k6 usa `stages` para definir o padrão de rampa/manutenção/redução.
- [ ] `thresholds` são definidos para `http_req_duration` (p95 e p99) e `http_req_failed`.
- [ ] O script sai com código não-zero quando qualquer threshold é violado.
- [ ] URL base é lida de `__ENV.BASE_URL` com fallback para `http://localhost:3000`.
- [ ] Workflow do GitHub Actions roda o teste k6 no push para `main` e faz upload do JSON de resumo.

**Dicas:**
1. Stages do k6:
   ```javascript
   export const options = {
     stages: [
       { duration: '1m', target: 500 },
       { duration: '3m', target: 500 },
       { duration: '1m', target: 0 },
     ],
     thresholds: {
       'http_req_duration': ['p(95)<200', 'p(99)<500'],
       'http_req_failed': ['rate<0.001'],
     },
   };
   ```
2. URL base: `const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';`.
3. CI: `k6 run --out json=report.json script.js` depois `actions/upload-artifact` com `path: report.json`.
4. Adicione `check(res, { 'status 200': r => r.status === 200 })` para rastrear falhas de check separadamente de erros HTTP.

---

## Exercício 10 — Corrigir um Teste Instável (Médio)

**Cenário:** O teste a seguir passa às vezes e falha outras vezes. Diagnostique a instabilidade e corrija-a sem remover nenhuma asserção.

**Teste instável fornecido:**
```typescript
it('processa jobs em ordem', async () => {
  const results: number[] = [];
  const queue = new JobQueue();

  queue.add(() => new Promise(res => setTimeout(() => { results.push(1); res(undefined); }, 100)));
  queue.add(() => new Promise(res => setTimeout(() => { results.push(2); res(undefined); }, 50)));
  queue.add(() => new Promise(res => setTimeout(() => { results.push(3); res(undefined); }, 10)));

  await new Promise(res => setTimeout(res, 200));

  expect(results).toEqual([1, 2, 3]);
});
```

**Requisitos:**
- Diagnostique: explique em um comentário de código por que o teste é instável.
- Corrija o teste para que seja determinístico independente de carga da CPU ou variações de timing.
- A correção não deve usar `setTimeout` para sincronização — use a própria API da fila.
- Se a classe `JobQueue` precisar de um novo método para suportar uma espera limpa, adicione-o.

**Critérios de Aceite:**
- [ ] O teste corrigido passa 100 vezes seguidas (`vitest --repeat 100`).
- [ ] Um comentário explica a causa raiz da instabilidade.
- [ ] A correção não aumenta desnecessariamente a duração do teste.
- [ ] A asserção `expect(results).toEqual([1, 2, 3])` é preservada sem alterações.
- [ ] A classe `JobQueue` processa jobs sequencialmente — testes devem verificar este contrato.

**Dicas:**
1. Causa raiz: o teste usa um timeout mágico (`200ms`) para "esperar a conclusão". Sob alta carga de CPU, os jobs podem não terminar a tempo, causando uma falha falsa.
2. Correção: adicione um método `queue.drain(): Promise<void>` que resolve quando todos os jobs enfileirados foram concluídos. `await queue.drain()` substitui a espera com `setTimeout`.
3. Garantia sequencial: se a fila processa jobs em ordem, o job 2 começa apenas após o job 1 completar — a ordem de conclusão é determinística independente da duração de cada job.
4. Fake timers: alternativamente, use `vi.useFakeTimers()` para controlar `setTimeout` deterministicamente. Mas uma API `drain()` é mais limpa.

---

## Exercício 11 — Testar um Processador de Job BullMQ (Médio)

**Cenário:** Escreva testes para um worker BullMQ que processa jobs `send-email`. Testes não devem requerer uma instância real de Redis.

**Requisitos:**
- A função do worker: `async function processEmailJob(job: Job<EmailPayload>): Promise<void>` chama `emailService.send(job.data)`.
- Testes cobrem: processamento bem-sucedido, `emailService` lança → job deve falhar com o erro, payload malformado → job lança `ValidationError`.
- Use `ioredis-mock` ou teste a função do processador diretamente (sem a classe `Worker` do BullMQ).
- Mock `emailService` usando spies do Vitest.

**Critérios de Aceite:**
- [ ] Caminho feliz: `emailService.send` é chamado com o payload correto.
- [ ] Falha de email: chamar `processEmailJob` com um `emailService` que falha lança o erro de email.
- [ ] Payload inválido (campo `to` faltando): lança `ValidationError` antes de chamar `emailService.send`.
- [ ] Testes não iniciam uma instância real de `Worker` BullMQ ou conectam ao Redis.
- [ ] Arquivo de teste roda em menos de 500ms (sem delays assíncronos reais).

**Dicas:**
1. Teste a função do processador diretamente: `await processEmailJob(mockJob)` — sem necessidade de instanciar um `Worker`.
2. Mock do job: `const mockJob = { data: { to: 'a@b.com', subject: 'Olá', body: 'Bem-vindo' }, id: '1' } as Job<EmailPayload>`.
3. Mock do serviço de email: `const emailService = { send: vi.fn().mockResolvedValue(undefined) }`. Injete via DI.
4. Validação: execute `emailPayloadSchema.parse(job.data)` no topo do processador. O Zod lança `ZodError` — encapsule em `ValidationError` para uma API mais limpa.

---

## Exercício 12 — Testes de Contrato com Pact (Difícil)

**Cenário:** Seu frontend (consumidor) chama um endpoint `GET /users/:id` no backend (provedor). Escreva um teste de contrato Pact para o consumidor e um teste de verificação para o provedor.

**Requisitos:**
- Teste do consumidor: defina a requisição/resposta esperada para `GET /users/1` e gere um arquivo Pact.
- Teste do provedor: carregue o arquivo Pact e verifique se o backend real satisfaz o contrato.
- O contrato cobre: caminho feliz (200 + objeto de usuário), não encontrado (404 + body de erro).
- Arquivos Pact são armazenados em `./pacts/` e commitados no repositório.
- A verificação do provedor roda no CI após o step de build do backend.

**Critérios de Aceite:**
- [ ] O teste do consumidor gera um arquivo Pact `frontend-backend.json` em `./pacts/`.
- [ ] O arquivo Pact define interações para os casos 200 e 404.
- [ ] O teste de verificação do provedor inicia o servidor Fastify e reproduz as interações Pact contra ele.
- [ ] Se o backend mudar o shape da resposta (ex: renomear `email` para `emailAddress`), a verificação do provedor falha.
- [ ] Um job do GitHub Actions executa a verificação do provedor após `npm run build` ter sucesso.

**Dicas:**
1. Setup do teste do consumidor: use `@pact-foundation/pact` `PactV3`. Defina a interação:
   ```typescript
   await provider.addInteraction({ uponReceiving: 'uma requisição para o usuário 1', withRequest: { method: 'GET', path: '/users/1' }, willRespondWith: { status: 200, body: like({ id: 1, name: string('Alice') }) } });
   ```
2. Matcher `like()`: corresponde a qualquer valor do mesmo tipo — evita hardcoding de dados de teste.
3. Verificação do provedor: use `Verifier` de `@pact-foundation/pact`. Aponte para o servidor em execução e o diretório `./pacts/`.
4. CI: execute testes do consumidor no job do frontend (gera o arquivo Pact), depois execute a verificação do provedor no job do backend usando o arquivo Pact commitado.
