# Testes End-to-End

## Visão Geral

Testes end-to-end (E2E) simulam interações reais de usuário em um navegador real, verificando que toda a pilha da aplicação funciona em conjunto do ponto de vista do usuário. Eles testam o fluxo de login, o processo de checkout, a renderização do dashboard — exatamente como um usuário os vivenciaria. O Playwright é a principal ferramenta de testes E2E em 2025, com excelente suporte a TypeScript, testes em múltiplos navegadores e uma API poderosa. Este capítulo cobre a configuração do Playwright, o padrão Page Object Model e a integração de testes E2E no CI.

---

## Pré-requisitos

- Track 09: Fundamentos de Testes
- Conhecimento básico de React/HTML
- Familiaridade com async/await

---

## Exemplos Práticos

### Configuração do Playwright

```bash
npm install --save-dev @playwright/test
npx playwright install  # instala os binários dos navegadores
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,  // tenta novamente testes instáveis no CI
  workers: process.env.CI ? 2 : undefined,

  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',   // salva trace para depurar testes instáveis
    screenshot: 'only-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    // Mobile
    { name: 'mobile-chrome', use: { ...devices['Pixel 7'] } },
  ],

  // Inicia o servidor de desenvolvimento antes dos testes
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Testes E2E básicos

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email-input"]', 'alice@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.click('[data-testid="login-button"]');

    // Aguarda o redirecionamento para o dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome, Alice')).toBeVisible();
  });

  test('shows error with invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email-input"]', 'alice@example.com');
    await page.fill('[data-testid="password-input"]', 'wrong-password');
    await page.click('[data-testid="login-button"]');

    await expect(page.getByText('Invalid credentials')).toBeVisible();
    await expect(page).toHaveURL('/login');
  });

  test('redirects unauthenticated users to login', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).toHaveURL('/login');
  });
});
```

### Page Object Model (POM)

O POM encapsula interações com a página em classes, evitando duplicação e tornando os testes legíveis:

```typescript
// e2e/pages/login.page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;
  private readonly errorMessage: Locator;

  constructor(private page: Page) {
    this.emailInput = page.getByTestId('email-input');
    this.passwordInput = page.getByTestId('password-input');
    this.submitButton = page.getByTestId('login-button');
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorMessage(): Promise<string | null> {
    return this.errorMessage.textContent();
  }
}
```

```typescript
// e2e/pages/dashboard.page.ts
import { Page, expect } from '@playwright/test';

export class DashboardPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/dashboard');
  }

  async isLoaded() {
    await expect(this.page).toHaveURL('/dashboard');
  }

  async getWelcomeText(): Promise<string | null> {
    return this.page.getByRole('heading', { name: /welcome/i }).textContent();
  }

  async clickNewProject() {
    await this.page.getByTestId('new-project-button').click();
  }
}
```

```typescript
// e2e/auth.spec.ts — usando POM
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page.js';
import { DashboardPage } from './pages/dashboard.page.js';

test('full login flow', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login('alice@example.com', 'SecurePassword123!');

  await dashboardPage.isLoaded();
  const welcome = await dashboardPage.getWelcomeText();
  expect(welcome).toContain('Alice');
});
```

### Estado de autenticação — evite fazer login em cada teste

```typescript
// e2e/auth.setup.ts — executar uma vez para salvar o estado de autenticação
import { test as setup } from '@playwright/test';
import { LoginPage } from './pages/login.page.js';

setup('authenticate', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login(process.env.TEST_USER_EMAIL!, process.env.TEST_USER_PASSWORD!);

  // Salva o estado autenticado (cookies + localStorage) em um arquivo
  await page.context().storageState({ path: 'e2e/.auth/user.json' });
});
```

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: '**/auth.setup.ts',
    },
    {
      name: 'authenticated',
      use: {
        storageState: 'e2e/.auth/user.json', // reutiliza o estado de autenticação
      },
      dependencies: ['setup'],
    },
  ],
});
```

### Testando formulários e interações

```typescript
// e2e/create-project.spec.ts
import { test, expect } from '@playwright/test';

test.use({ storageState: 'e2e/.auth/user.json' });

test('create a new project', async ({ page }) => {
  await page.goto('/dashboard');
  await page.getByTestId('new-project-button').click();

  // Preenche o formulário
  await page.getByLabel('Project Name').fill('My Awesome Project');
  await page.getByLabel('Description').fill('A test project');
  await page.getByRole('combobox', { name: 'Category' }).selectOption('web');

  // Envia
  await page.getByRole('button', { name: 'Create Project' }).click();

  // Verifica o sucesso
  await expect(page.getByRole('alert')).toContainText('Project created');
  await expect(page).toHaveURL(/\/projects\/.+/);
  await expect(page.getByRole('heading')).toContainText('My Awesome Project');
});

test('shows validation errors for empty form', async ({ page }) => {
  await page.goto('/projects/new');
  await page.getByRole('button', { name: 'Create Project' }).click();

  await expect(page.getByText('Project name is required')).toBeVisible();
});
```

### Executando testes E2E no CI

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run E2E tests
        env:
          E2E_BASE_URL: ${{ secrets.STAGING_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
        run: npx playwright test --project=chromium

      - name: Upload test artifacts on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

---

## Padrões Comuns e Boas Práticas

- **Atributos `data-testid`** — seletores estáveis que não mudam quando o estilo muda
- **`getByRole` e `getByLabel`** — seletores semânticos que também validam acessibilidade
- **Salve o estado de autenticação** — faça login uma vez por execução de testes, reutilize o estado
- **Execute em modo headless no CI** — modo com interface visível para depuração local
- **Tente novamente em caso de falha** no CI — testes E2E têm instabilidade inerente; 2 tentativas capturam problemas transitórios
- **Trace em caso de falha** — traces do Playwright são inestimáveis para depurar falhas no CI

---

## Anti-Padrões a Evitar

- Testes E2E para lógica que pertence aos testes unitários — E2E é para fluxos de usuário, não regras de negócio
- Depender de timing (`waitForTimeout(2000)`) em vez de aguardar elementos específicos (`waitForSelector`)
- Seletores de classe CSS ou texto que quebram quando a UI muda — use `data-testid` ou roles ARIA
- Não limpar dados de teste — testes E2E devem criar e limpar seus próprios dados

---

## Depuração e Resolução de Problemas

**"Teste passa localmente mas falha no CI"**
Diferenças de timing de rede ou renderização. Use `await page.waitForLoadState('networkidle')` ou `await expect(element).toBeVisible()` em vez de timeouts fixos. Verifique se o ambiente de CI tem as variáveis de ambiente corretas.

**"Erro de Locator: encontrou 3 elementos correspondentes"**
Seu seletor é muito amplo. Torne-o mais específico: `page.getByTestId('submit-button')` é melhor que `page.getByRole('button')`.

**"Como depuro um teste falhando?"**
Execute com `--headed` e `--slow-mo=500`:
`npx playwright test --headed --slow-mo=500 e2e/auth.spec.ts`
Ou abra a interface do Playwright: `npx playwright test --ui`

---

## Leitura Complementar

- [Documentação do Playwright](https://playwright.dev/docs/intro)
- [Boas práticas do Playwright](https://playwright.dev/docs/best-practices)
- [Padrão Page Object Model](https://playwright.dev/docs/pom)
- Track 09: [Fundamentos de Testes](testing-fundamentals.md)

---

## Resumo

Testes E2E ficam no topo da pirâmide de testes: poucos em número, altos em confiança. O Playwright é a ferramenta de escolha para projetos Node.js — nativo em TypeScript, multi-browser e com excelente tooling para desenvolvedores. O Page Object Model mantém os testes manuteníveis conforme a UI evolui. Salvar o estado de autenticação e reutilizá-lo entre testes reduz drasticamente o tempo de execução da suíte E2E. No CI, combine os testes E2E com gravação de trace e upload de artefatos para tornar as falhas diagnosticáveis sem precisar reproduzi-las localmente. Limite os testes E2E às jornadas críticas do usuário — login, cadastro, checkout, fluxo principal — e deixe os casos extremos e a lógica de negócio para os testes unitários e de integração.
