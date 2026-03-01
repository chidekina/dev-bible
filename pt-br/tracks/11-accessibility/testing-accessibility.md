# Testing Accessibility

## Visão Geral

Testar acessibilidade combina varredura automatizada, testes manuais de teclado e testes com screen reader. Ferramentas automatizadas detectam cerca de 30–40% dos problemas da WCAG — são rápidas, baratas e rodam no CI. Os 60–70% restantes exigem julgamento humano: a focus order faz sentido? O anúncio desta live region faz sentido no contexto? Este label é específico o suficiente?

Este capítulo cobre a stack completa de testes: axe-core com Playwright para testes de integração, jest-axe para testes de componentes, Lighthouse CI para monitoramento e um fluxo prático de testes com screen reader.

## Pré-requisitos

- Node.js 18+, TypeScript
- Playwright instalado (`npm install -D @playwright/test`)
- Uma aplicação funcionando com servidor de desenvolvimento ou testes

## Conceitos Fundamentais

### Testes Automatizados vs Manuais

| Tipo | Cobertura | Velocidade | Quando |
|------|----------|-------|------|
| axe-core (Playwright) | ~30–40% da WCAG | Rápido (CI) | A cada build |
| jest-axe | Nível de componente | Rápido (unitário) | A cada mudança de componente |
| Lighthouse CI | Regressão de score | Médio | A cada PR |
| Testes de teclado | Focus order, traps, controles | Manual | Após conclusão de feature |
| Testes com screen reader | Compreensão, anúncios | Manual | Antes do release |
| Testes com usuários de AT | Compreensão real | Lento | Releases principais |

### Como o axe-core Funciona

axe-core é um motor de acessibilidade JavaScript da Deque. Ele inspeciona o DOM ao vivo — incluindo renderização CSS, accessibility tree computada e states ARIA — contra um conjunto de regras mapeadas para critérios de sucesso da WCAG.

As regras são categorizadas como:
- **Violations** — falhas definitivas (bloqueiam o release)
- **Incomplete** (precisa de revisão) — axe não conseguiu determinar pass/fail (revisar manualmente)
- **Passes** — verificações com resultado confirmado de sucesso
- **Inapplicable** — a regra não se aplica a esta página

Cada violation inclui:
- `id` — nome da regra
- `impact` — `critical`, `serious`, `moderate`, `minor`
- `description` — o que está errado
- `help` — como corrigir
- `helpUrl` — link para dequeuniversity.com
- `nodes` — array de elementos afetados com trecho HTML e sugestões de correção

### Conjuntos de Tags axe

Filtre quais critérios WCAG testar passando tags:

| Tag | Cobre |
|-----|--------|
| `wcag2a` | WCAG 2.0 Nível A |
| `wcag2aa` | WCAG 2.0 Nível AA |
| `wcag21a` | Adições WCAG 2.1 Nível A |
| `wcag21aa` | Adições WCAG 2.1 Nível AA |
| `wcag22aa` | Adições WCAG 2.2 Nível AA |
| `best-practice` | Não é WCAG, mas recomendado |
| `TTv5` | Trusted Tester v5 (governo dos EUA) |

Padrão recomendado: `['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa']`

## Exemplos Práticos

### Configurando axe-core com Playwright

```bash
npm install -D @axe-core/playwright @playwright/test
npx playwright install chromium
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/accessibility',
  use: {
    baseURL: 'http://localhost:3000',
    // Testes de acessibilidade funcionam melhor com um browser real
    channel: 'chrome',
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

```typescript
// tests/accessibility/pages.test.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

// Helper para formatar violations com saída legível
function formatViolations(violations: AxeResults['violations']): string {
  return violations
    .map(
      (v) =>
        `[${v.impact?.toUpperCase()}] ${v.id}: ${v.description}\n` +
        v.nodes
          .slice(0, 2)
          .map((n) => `  → ${n.html.slice(0, 100)}`)
          .join('\n')
    )
    .join('\n\n');
}

const pages = [
  { name: 'Home', path: '/' },
  { name: 'Login', path: '/login' },
  { name: 'Dashboard', path: '/dashboard' },
  { name: 'Settings', path: '/settings' },
];

for (const { name, path } of pages) {
  test(`${name} page: no WCAG 2.1 AA violations`, async ({ page }) => {
    await page.goto(path);

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      // Excluir widgets de terceiros que você não controla
      .exclude('#intercom-iframe')
      .analyze();

    expect(results.violations, formatViolations(results.violations)).toEqual([]);
  });
}
```

### Testando Componentes Específicos

```typescript
// tests/accessibility/components.test.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('modal dialog meets WCAG', async ({ page }) => {
  await page.goto('/');
  await page.click('button:has-text("Open dialog")');

  // Aguarda o dialog aparecer
  await page.waitForSelector('[role="dialog"]');

  // Escaneia apenas o componente dialog, não a página inteira
  const results = await new AxeBuilder({ page })
    .include('[role="dialog"]')
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});

test('form has no label violations', async ({ page }) => {
  await page.goto('/signup');

  const results = await new AxeBuilder({ page })
    .include('form')
    .withRules(['label', 'label-content-name-mismatch', 'form-field-multiple-labels'])
    .analyze();

  expect(results.violations).toEqual([]);
});

test('data table is accessible', async ({ page }) => {
  await page.goto('/reports');

  const results = await new AxeBuilder({ page })
    .include('table')
    .withRules([
      'table-duplicate-name',
      'td-headers-attr',
      'th-has-data-cells',
      'scope-attr-valid',
    ])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### jest-axe para Testes Unitários de Componentes

```bash
npm install -D jest-axe @testing-library/react @testing-library/jest-dom
```

```typescript
// src/components/Button.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { Button } from './Button';

expect.extend(toHaveNoViolations);

describe('Button accessibility', () => {
  it('has no violations when rendered with text', async () => {
    const { container } = render(<Button onClick={() => {}}>Save changes</Button>);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('has no violations when rendered as icon button', async () => {
    const { container } = render(
      <Button onClick={() => {}} aria-label="Delete item">
        <TrashIcon aria-hidden />
      </Button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('has no violations when disabled', async () => {
    const { container } = render(
      <Button disabled>Cannot submit</Button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

```typescript
// src/components/FormField.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { FormField } from './FormField';

expect.extend(toHaveNoViolations);

describe('FormField accessibility', () => {
  it('passes with label and no error', async () => {
    const { container } = render(
      <FormField id="email" label="Email address" type="email" />
    );
    expect(await axe(container)).toHaveNoViolations();
  });

  it('passes with error state', async () => {
    const { container } = render(
      <FormField
        id="email"
        label="Email address"
        type="email"
        error="Please enter a valid email address"
      />
    );
    expect(await axe(container)).toHaveNoViolations();
  });

  it('passes with required indicator', async () => {
    const { container } = render(
      <FormField id="name" label="Full name" required />
    );
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

### Reporter axe Customizado para CI

```typescript
// scripts/a11y-report.ts
import { chromium } from 'playwright';
import AxeBuilder from '@axe-core/playwright';
import { writeFileSync } from 'fs';

interface Report {
  url: string;
  violations: number;
  serious: number;
  critical: number;
  details: Array<{
    id: string;
    impact: string;
    description: string;
    affectedCount: number;
  }>;
}

async function auditPages(urls: string[]): Promise<Report[]> {
  const browser = await chromium.launch();
  const reports: Report[] = [];

  for (const url of urls) {
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: 'networkidle' });

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    reports.push({
      url,
      violations: results.violations.length,
      serious: results.violations.filter((v) => v.impact === 'serious').length,
      critical: results.violations.filter((v) => v.impact === 'critical').length,
      details: results.violations.map((v) => ({
        id: v.id,
        impact: v.impact ?? 'unknown',
        description: v.description,
        affectedCount: v.nodes.length,
      })),
    });

    await page.close();
  }

  await browser.close();
  return reports;
}

async function main() {
  const urls = [
    'http://localhost:3000/',
    'http://localhost:3000/login',
    'http://localhost:3000/dashboard',
  ];

  const reports = await auditPages(urls);
  writeFileSync('a11y-report.json', JSON.stringify(reports, null, 2));

  let hasErrors = false;
  for (const report of reports) {
    console.log(`\n${report.url}`);
    console.log(`  Violations: ${report.violations} (${report.critical} critical, ${report.serious} serious)`);
    for (const v of report.details) {
      const marker = v.impact === 'critical' || v.impact === 'serious' ? '[FAIL]' : '[WARN]';
      console.log(`  ${marker} ${v.id} — ${v.affectedCount} element(s): ${v.description}`);
      if (v.impact === 'critical' || v.impact === 'serious') hasErrors = true;
    }
  }

  if (hasErrors) {
    console.error('\nViolações críticas ou sérias encontradas. Reprovando no CI.');
    process.exit(1);
  }
}

main();
```

### Integração com Lighthouse CI

```bash
npm install -D @lhci/cli
```

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/login'],
      numberOfRuns: 1,
      settings: {
        onlyCategories: ['accessibility'],
      },
    },
    assert: {
      assertions: {
        'categories:accessibility': ['error', { minScore: 0.9 }],
        // Reprovar em regras específicas
        'audits[color-contrast].score': ['error', { minScore: 1 }],
        'audits[document-title].score': ['error', { minScore: 1 }],
        'audits[html-has-lang].score': ['error', { minScore: 1 }],
        'audits[image-alt].score': ['error', { minScore: 1 }],
        'audits[label].score': ['error', { minScore: 1 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

```yaml
# .github/workflows/a11y.yml
name: Accessibility

on: [push, pull_request]

jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run build
      - name: Start server
        run: npm start &
      - name: Wait for server
        run: npx wait-on http://localhost:3000
      - name: Run axe tests
        run: npx playwright test tests/accessibility/
      - name: Lighthouse CI
        run: npx lhci autorun
```

## Padrões e Boas Práticas

### Testando Conteúdo Dinâmico

```typescript
test('error summary announces to screen readers', async ({ page }) => {
  await page.goto('/signup');

  // Enviar com campos obrigatórios vazios
  await page.click('button[type="submit"]');

  // Aguarda o estado de erro
  await page.waitForSelector('[role="alert"]');

  // Verifica se a live region tem conteúdo
  const alertText = await page.textContent('[role="alert"]');
  expect(alertText).toBeTruthy();
  expect(alertText).toContain('error');

  // Roda o axe novamente após mudança de estado dinâmico
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Testando em Diferentes Tamanhos de Viewport

```typescript
test('navigation is accessible on mobile viewport', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('/');

  // Navegação mobile frequentemente tem menus hambúrguer — teste o toggle
  const menuBtn = page.getByRole('button', { name: /menu/i });
  await menuBtn.click();
  await page.waitForSelector('[role="navigation"][aria-expanded="true"]');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Snapshot Testing para a Accessibility Tree

```typescript
test('accessibility tree snapshot for card component', async ({ page }) => {
  await page.goto('/');
  const card = page.locator('.project-card').first();
  const snapshot = await card.evaluateHandle((el) =>
    (window as any).axe.utils.getNodeFromTree(el)
  );
  // Usa o snapshot de acessibilidade do Playwright
  const a11ySnapshot = await page.accessibility.snapshot({
    root: await card.elementHandle() ?? undefined,
  });
  expect(a11ySnapshot).toMatchSnapshot();
});
```

## Anti-Padrões a Evitar

**Rodar axe apenas na homepage**
Problemas de acessibilidade frequentemente aparecem em páginas específicas (formulários, dialogs, tabelas de dados). Teste cada tipo de página único e cada estado dinâmico (erro, carregando, vazio, preenchido).

**Ignorar resultados `incomplete`**
O axe marca algumas verificações como `incomplete` quando não consegue determinar pass/fail automaticamente (contraste de cor sobre imagens, conteúdo dinâmico). Revise-as manualmente — muitas vezes contêm problemas reais.

**Rodar apenas testes automatizados**
```typescript
// Não é suficiente — axe não detecta:
// - problemas lógicos na focus order
// - conteúdo de anúncios de live region
// - ordem de leitura do screen reader
// - acessibilidade cognitiva

// Complemente com checklist manual a cada feature
```

**Desabilitar regras que incomodam**
```typescript
// Ruim: silenciando um problema real
const results = await new AxeBuilder({ page })
  .disableRules(['color-contrast'])  // "vamos corrigir depois"
  .analyze();

// Bom: corrija o problema, ou documente a exclusão com um plano com prazo definido
```

**Testar apenas em um browser**
O comportamento da accessibility tree varia entre browsers. Teste no Chromium (padrão do axe) E no Firefox:

```typescript
// playwright.config.ts
projects: [
  { name: 'chromium', use: { browserName: 'chromium' } },
  { name: 'firefox',  use: { browserName: 'firefox' } },
],
```

## Depuração e Solução de Problemas

### Interpretando Violations do axe

```typescript
// Registrar detalhes completos das violations para depuração
const results = await new AxeBuilder({ page }).analyze();

for (const violation of results.violations) {
  console.log(`\n--- ${violation.id} (${violation.impact}) ---`);
  console.log('Issue:', violation.description);
  console.log('Fix:', violation.help);
  console.log('WCAG:', violation.tags.filter((t) => t.startsWith('wcag')).join(', '));
  console.log('More info:', violation.helpUrl);
  console.log('Affected elements:');
  for (const node of violation.nodes) {
    console.log('  HTML:', node.html);
    if (node.failureSummary) console.log('  Failure:', node.failureSummary);
    if (node.any.length) {
      console.log('  To fix, satisfy one of:');
      node.any.forEach((c) => console.log(`    - ${c.message}`));
    }
    if (node.all.length) {
      console.log('  To fix, satisfy all of:');
      node.all.forEach((c) => console.log(`    - ${c.message}`));
    }
  }
}
```

### Falsos Positivos e Exclusões

```typescript
// Exclusão documentada com motivo e issue de rastreamento
const results = await new AxeBuilder({ page })
  .exclude('#stripe-card-element')  // iframe de terceiro, sem controle — rastreado em JIRA-123
  .analyze();
```

### Quando o axe Passa mas o Screen Reader Falha

Cenários que o axe não consegue detectar:
1. **Ordem de leitura** — elementos na ordem correta do DOM, mas reordenados visualmente com CSS
2. **Qualidade do conteúdo da live region** — região existe, mas anuncia texto confuso
3. **Gerenciamento de focus** — modal abre, mas o focus não se move (focus não é um atributo do DOM)
4. **Carga cognitiva** — muitas opções, instruções pouco claras, linguagem complexa

Para esses casos, faça testes manuais.

## Cenários do Mundo Real

### Cenário 1: Fluxo de Testes com Screen Reader (NVDA + Chrome no Windows)

**Configuração:**
1. Instale o NVDA (gratuito, nvaccess.org)
2. Instale o Chrome
3. Configure o NVDA para usar o "Browse mode" para conteúdo web (padrão)

**Atalhos de navegação:**
| Tecla | Ação |
|-----|--------|
| `H` | Próximo heading |
| `Shift+H` | Heading anterior |
| `B` | Próximo button |
| `F` | Próximo campo de formulário |
| `T` | Próxima tabela |
| `L` | Próxima lista |
| `D` | Próximo landmark |
| `Insert+F7` | Listar todos os links/headings/landmarks |
| `Insert+F5` | Listar todos os campos de formulário |
| `Tab` | Próximo elemento interativo |
| `Enter` | Ativar link/button |
| `Space` | Ativar button/checkbox |
| `Teclas de seta` | Navegar dentro de widget ou ler linha por linha |

**Checklist de testes:**
- [ ] O título da página é lido ao carregar
- [ ] A navegação por landmarks faz sentido (liste os landmarks com `Insert+F7`)
- [ ] O esboço de headings é lógico (liste os headings)
- [ ] Todas as imagens têm alternativas em texto significativas
- [ ] Os labels de formulário são lidos quando o input recebe focus
- [ ] As mensagens de erro são anunciadas (via live region ou movimentação de focus)
- [ ] O dialog captura o focus e pode ser dispensado
- [ ] As mudanças de conteúdo dinâmico são anunciadas

**VoiceOver no macOS (Safari):**
- `Cmd+F5` — ativar/desativar VoiceOver
- `VO+U` — abrir rotor (escolher headings, links, landmarks, controles de formulário)
- `VO+Space` — ativar elemento
- `VO+Shift+M` — menu de atalho do elemento

### Cenário 2: Integrando ao Processo de Code Review

```markdown
## Checklist de acessibilidade para revisores de PR

### Automatizado (bloqueado pelo CI)
- [ ] Testes Playwright com axe-core passam
- [ ] Testes de componente com jest-axe passam
- [ ] Score de acessibilidade do Lighthouse ≥ 90

### Manual (responsabilidade do revisor)
- [ ] Todos os novos elementos interativos são acessíveis por teclado
- [ ] Novos formulários têm labels associados
- [ ] Mudanças de conteúdo dinâmico usam live regions ou gerenciamento de focus
- [ ] Componentes modal/dialog capturam o focus e o restauram ao fechar
- [ ] Nenhuma nova informação transmitida apenas por cor
- [ ] Novas imagens têm texto alternativo apropriado
```

### Cenário 3: Arquivo de Teste Completo para uma Página de Feature

```typescript
// tests/accessibility/signup.test.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Signup page accessibility', () => {
  test('initial page state passes WCAG 2.1 AA', async ({ page }) => {
    await page.goto('/signup');
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test('form error state passes WCAG 2.1 AA', async ({ page }) => {
    await page.goto('/signup');
    await page.click('button[type="submit"]');
    await page.waitForSelector('[aria-invalid="true"]');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test('all form inputs have labels', async ({ page }) => {
    await page.goto('/signup');
    const results = await new AxeBuilder({ page })
      .include('form')
      .withRules(['label', 'label-content-name-mismatch'])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test('submit button is keyboard accessible', async ({ page }) => {
    await page.goto('/signup');
    await page.keyboard.press('Tab');
    // Tabula até o botão de envio
    let focused = '';
    for (let i = 0; i < 20; i++) {
      focused = await page.evaluate(() => document.activeElement?.textContent?.trim() ?? '');
      if (focused === 'Create account') break;
      await page.keyboard.press('Tab');
    }
    expect(focused).toBe('Create account');
    // Ativar com teclado
    await page.keyboard.press('Enter');
    // Verificar se o envio do formulário foi tentado
    await page.waitForSelector('[aria-invalid="true"]'); // validação executou
  });

  test('error messages are associated with inputs', async ({ page }) => {
    await page.goto('/signup');
    await page.click('button[type="submit"]');
    await page.waitForSelector('[aria-invalid="true"]');

    // Cada input inválido deve ter aria-describedby apontando para o erro
    const invalids = await page.$$('[aria-invalid="true"]');
    for (const input of invalids) {
      const describedby = await input.getAttribute('aria-describedby');
      expect(describedby).toBeTruthy();
      const errorEl = page.locator(`#${describedby}`);
      await expect(errorEl).toBeVisible();
      const text = await errorEl.textContent();
      expect(text?.length).toBeGreaterThan(0);
    }
  });
});
```

## Leitura Complementar

- [axe-core GitHub](https://github.com/dequelabs/axe-core) — documentação de regras e contribuição
- [@axe-core/playwright](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/playwright) — integração com Playwright
- [jest-axe](https://github.com/nickcolley/jest-axe) — integração com Jest/Vitest
- [Lighthouse Accessibility Audits](https://developer.chrome.com/docs/lighthouse/accessibility/) — o que cada verificação do Lighthouse cobre
- [Deque University](https://dequeuniversity.com/) — treinamento com screen reader
- [NVDA User Guide](https://www.nvaccess.org/files/nvda/documentation/userGuide.html)

## Resumo

Uma estratégia de testes de acessibilidade precisa das três camadas:

1. **Automatizado (gate no CI)** — axe-core com Playwright em cada página e estado dinâmico; jest-axe em componentes; Lighthouse CI para rastreamento de regressão. Detecta ~35% dos problemas da WCAG, mas roda a cada PR.

2. **Testes de teclado** — navegue por todos os fluxos de usuário sem o mouse. Verifique o gerenciamento de focus para modais, SPAs e conteúdo dinâmico. Faça isso em cada nova feature.

3. **Testes com screen reader** — NVDA+Chrome no Windows ou VoiceOver+Safari no macOS. Teste os cenários que importam: envio de formulário com erros, modal dialogs, resultados de busca dinâmicos, anúncios de live region. Faça isso antes do release.

Lembretes de configuração:
- Instale `@axe-core/playwright` e rode com `withTags(['wcag2a','wcag2aa','wcag21a','wcag21aa'])`
- No CI: repove em violações `critical` e `serious`; avise em `moderate`
- Nunca desabilite regras para fazer os testes passarem — corrija o problema subjacente
- Documente exclusões (iframes de terceiros) com o motivo e uma issue de rastreamento
