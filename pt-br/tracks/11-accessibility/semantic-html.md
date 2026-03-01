# Semantic HTML

## Visão Geral

Semantic HTML significa usar elementos pelo seu significado intencional, não apenas pela sua aparência visual. Um `<button>` comunica interatividade. Um `<nav>` comunica navegação. Um `<h2>` comunica um subtítulo de seção. Quando você usa o elemento certo, browsers, mecanismos de busca e tecnologias assistivas recebem informações precisas sobre o seu conteúdo — de graça, sem JavaScript nem ARIA.

O oposto do semantic HTML é o "div soup": envolver tudo em `<div>` e `<span>` com classes que descrevem estilo visual, mas nada sobre o significado. O div soup produz páginas com a mesma aparência, mas invisíveis para screen readers, impossíveis de navegar por teclado e mais difíceis de manter.

Este capítulo cobre os elementos semânticos mais críticos para acessibilidade: landmarks, hierarquia de headings, listas, formulários, tabelas e elementos de mídia.

## Pré-requisitos

- Fundamentos de HTML (block vs inline, atributos)
- Um browser com DevTools de acessibilidade (Chrome ou Firefox)
- Opcional: um screen reader instalado (NVDA para Windows, VoiceOver para macOS/iOS)

## Conceitos Fundamentais

### Elementos Landmark

Os landmarks dão aos usuários de screen reader um mapa da página. Cada landmark é uma região para a qual eles podem saltar diretamente, sem precisar ler tudo o que está entre eles.

| Elemento | Role | Finalidade |
|---------|------|---------|
| `<header>` | `banner` | Cabeçalho do site (apenas uma vez por página, exceto dentro de `<article>`) |
| `<nav>` | `navigation` | Menus de navegação |
| `<main>` | `main` | Conteúdo principal — apenas um por página |
| `<aside>` | `complementary` | Conteúdo complementar (barras laterais, destaques) |
| `<footer>` | `contentinfo` | Rodapé do site |
| `<section>` | `region` | Seção nomeada (precisa de `aria-label` ou `aria-labelledby` para criar um landmark) |
| `<article>` | `article` | Conteúdo autocontido (post de blog, comentário, card) |
| `<form>` | `form` | Região de formulário (precisa de nome acessível para se tornar um landmark) |

Usuários de screen reader podem listar todos os landmarks de uma página e saltar diretamente para eles. Se você usar `<div>` para tudo, essa navegação desaparece.

### Hierarquia de Headings

Os headings (`<h1>` a `<h6>`) definem o esboço do documento. Usuários de screen reader frequentemente navegam por headings — "saltar para o próximo heading" é um dos atalhos mais comuns.

Regras:
- **Um único `<h1>` por página** — deve ser o tópico principal
- **Não pule níveis** — após um `<h2>`, use `<h3>`, não `<h4>`
- **Níveis de heading refletem aninhamento, não tamanho** — use CSS para controlar o tamanho visual
- **Cada seção de conteúdo deve ser alcançável via um heading**

```html
<h1>Developer's Guide to Node.js</h1>

<h2>Getting Started</h2>
  <h3>Installation</h3>
  <h3>Your First Script</h3>

<h2>Core Modules</h2>
  <h3>File System</h3>
    <h4>Reading Files</h4>
    <h4>Writing Files</h4>
  <h3>HTTP</h3>
```

### Listas

Use listas quando o conteúdo for enumerável. Listas comunicam agrupamento e contagem para screen readers ("lista de 5 itens").

```html
<!-- Lista não ordenada — a ordem não importa -->
<ul>
  <li>TypeScript</li>
  <li>Node.js</li>
  <li>PostgreSQL</li>
</ul>

<!-- Lista ordenada — a sequência importa -->
<ol>
  <li>Install dependencies</li>
  <li>Configure environment</li>
  <li>Run migrations</li>
  <li>Start the server</li>
</ol>

<!-- Lista de descrição — pares termo/definição, dados chave/valor -->
<dl>
  <dt>Timeout</dt>
  <dd>Maximum time in milliseconds before the request fails. Default: 5000.</dd>

  <dt>Retries</dt>
  <dd>Number of times to retry a failed request. Default: 3.</dd>
</dl>
```

Menus de navegação também são listas:
```html
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/docs">Documentation</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>
```

### Formulários

Formulários são uma das áreas de maior risco para acessibilidade. Um formulário inacessível pode impedir usuários de se cadastrar, finalizar uma compra ou enviar tickets de suporte.

A **associação de label** é a base:

```html
<!-- Método 1: for/id (mais confiável) -->
<label for="email">Email address</label>
<input id="email" type="email" autocomplete="email">

<!-- Método 2: label envolvendo o input (sem necessidade de id) -->
<label>
  Email address
  <input type="email" autocomplete="email">
</label>

<!-- Ruim: sem label -->
<input type="email" placeholder="Email">

<!-- Ruim: label sem associação programática -->
<span>Email address</span>
<input type="email">
```

**Agrupando inputs relacionados:**
```html
<fieldset>
  <legend>Shipping address</legend>

  <label for="street">Street</label>
  <input id="street" type="text" autocomplete="street-address">

  <label for="city">City</label>
  <input id="city" type="text" autocomplete="address-level2">

  <label for="postcode">Postcode</label>
  <input id="postcode" type="text" autocomplete="postal-code">
</fieldset>

<fieldset>
  <legend>Payment method</legend>
  <label>
    <input type="radio" name="payment" value="card"> Credit card
  </label>
  <label>
    <input type="radio" name="payment" value="pix"> PIX
  </label>
</fieldset>
```

**Campos obrigatórios e estados de erro:**
```html
<label for="username">
  Username
  <span aria-hidden="true"> *</span>
  <span class="sr-only">(obrigatório)</span>
</label>
<input
  id="username"
  type="text"
  required
  aria-required="true"
  aria-invalid="true"
  aria-describedby="username-error"
>
<span id="username-error" role="alert">
  Username is required. Use only letters, numbers, and underscores.
</span>
```

### Tabelas

Tabelas transmitem relações entre dados em linhas e colunas. Devem ser usadas apenas para dados tabulares — nunca para layout.

```html
<table>
  <caption>Monthly server costs by region (USD)</caption>
  <thead>
    <tr>
      <th scope="col">Region</th>
      <th scope="col">Compute</th>
      <th scope="col">Storage</th>
      <th scope="col">Total</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">us-east-1</th>
      <td>$320</td>
      <td>$48</td>
      <td>$368</td>
    </tr>
    <tr>
      <th scope="row">eu-west-1</th>
      <td>$290</td>
      <td>$41</td>
      <td>$331</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <th scope="row">Total</th>
      <td>$610</td>
      <td>$89</td>
      <td>$699</td>
    </tr>
  </tfoot>
</table>
```

Atributos importantes:
- `<caption>` — título da tabela, lido primeiro pelos screen readers
- `scope="col"` — o header se aplica à coluna abaixo
- `scope="row"` — o header se aplica à linha à direita
- `<thead>`, `<tbody>`, `<tfoot>` — agrupamento estrutural

### Figure e Figcaption

Use `<figure>` com `<figcaption>` para imagens, diagramas, amostras de código ou gráficos que tenham uma legenda.

```html
<!-- Imagem com legenda -->
<figure>
  <img
    src="architecture-diagram.png"
    alt="Three-tier architecture showing web layer, application layer, and database layer"
  >
  <figcaption>Figure 1: High-level architecture of the API service</figcaption>
</figure>

<!-- Bloco de código com legenda -->
<figure>
  <figcaption>Listing 1: Basic Fastify server setup</figcaption>
  <pre><code class="language-typescript">
import Fastify from 'fastify';
const app = Fastify({ logger: true });
app.get('/', async () => ({ status: 'ok' }));
await app.listen({ port: 3000 });
  </code></pre>
</figure>

<!-- Gráfico onde o alt é vazio e a legenda carrega o resumo dos dados -->
<figure>
  <img src="revenue-chart.png" alt="">
  <figcaption>
    Revenue grew from $1.2M in Q1 to $1.9M in Q4, a 58% increase.
    <a href="/data/revenue-2024.csv">Download raw data (CSV)</a>
  </figcaption>
</figure>
```

## Exemplos Práticos

### Auditando Estrutura de Headings com Playwright

```typescript
import { chromium } from 'playwright';

async function auditHeadings(url: string) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(url);

  const headings = await page.$$eval(
    'h1, h2, h3, h4, h5, h6',
    (elements) =>
      elements.map((el) => ({
        level: parseInt(el.tagName.slice(1)),
        text: el.textContent?.trim() ?? '',
      }))
  );

  console.log('Heading structure:');
  for (const h of headings) {
    console.log(`${'  '.repeat(h.level - 1)}H${h.level}: ${h.text}`);
  }

  // Detectar níveis pulados
  for (let i = 1; i < headings.length; i++) {
    const prev = headings[i - 1].level;
    const curr = headings[i].level;
    if (curr > prev + 1) {
      console.warn(`Skipped H${prev} → H${curr} at: "${headings[i].text}"`);
    }
  }

  await browser.close();
}

auditHeadings('http://localhost:3000');
```

### Validando Presença de Landmarks

```typescript
async function auditLandmarks(url: string) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(url);

  const landmarks = await page.$$eval(
    'main, nav, header, footer, aside, section[aria-label], section[aria-labelledby]',
    (elements) =>
      elements.map((el) => ({
        tag: el.tagName.toLowerCase(),
        label: el.getAttribute('aria-label') ?? null,
      }))
  );

  console.log('Landmarks found:');
  landmarks.forEach(({ tag, label }) =>
    console.log(`  <${tag}>${label ? ` "${label}"` : ''}`)
  );

  if (!landmarks.some((l) => l.tag === 'main')) {
    console.error('ERROR: no <main> element found');
  }

  const navs = landmarks.filter((l) => l.tag === 'nav');
  if (navs.length > 1 && navs.some((n) => !n.label)) {
    console.warn('WARNING: multiple <nav> elements without aria-label');
  }

  await browser.close();
}
```

### Verificando Associação de Labels em Formulários

```typescript
import { test, expect } from '@playwright/test';

test('all form inputs have associated labels', async ({ page }) => {
  await page.goto('/signup');

  // Encontrar inputs sem associação de label
  const unlabeled = await page.$$eval(
    'input:not([type="hidden"]):not([type="submit"]):not([type="button"]), select, textarea',
    (inputs) =>
      inputs
        .filter((input) => {
          const id = input.getAttribute('id');
          const ariaLabel = input.getAttribute('aria-label');
          const ariaLabelledby = input.getAttribute('aria-labelledby');
          const wrappingLabel = input.closest('label');
          const labelFor = id ? document.querySelector(`label[for="${id}"]`) : null;
          return !ariaLabel && !ariaLabelledby && !wrappingLabel && !labelFor;
        })
        .map((input) => ({
          type: input.getAttribute('type') ?? input.tagName,
          id: input.getAttribute('id') ?? '(no id)',
          name: input.getAttribute('name') ?? '(no name)',
        }))
  );

  expect(unlabeled).toEqual([]);
});
```

## Padrões e Boas Práticas

### Template Completo de Página

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Dashboard — Acme App</title>
</head>
<body>

  <a href="#main-content" class="skip-link">Ir para o conteúdo principal</a>

  <header>
    <a href="/"><img src="/logo.svg" alt="Acme App home"></a>
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/dashboard" aria-current="page">Dashboard</a></li>
        <li><a href="/projects">Projects</a></li>
        <li><a href="/settings">Settings</a></li>
      </ul>
    </nav>
  </header>

  <main id="main-content">
    <h1>Dashboard</h1>

    <section aria-labelledby="recent-heading">
      <h2 id="recent-heading">Recent Projects</h2>
      <!-- cards de projeto como elementos <article> -->
    </section>

    <section aria-labelledby="activity-heading">
      <h2 id="activity-heading">Recent Activity</h2>
      <!-- lista de atividades -->
    </section>
  </main>

  <aside aria-labelledby="tips-heading">
    <h2 id="tips-heading">Quick Tips</h2>
    <!-- dicas da barra lateral -->
  </aside>

  <footer>
    <nav aria-label="Footer navigation">
      <ul>
        <li><a href="/privacy">Privacy Policy</a></li>
        <li><a href="/terms">Terms of Service</a></li>
      </ul>
    </nav>
    <p><small>&copy; 2024 Acme Corp.</small></p>
  </footer>

</body>
</html>
```

### Skip Link (CSS)

```css
.skip-link {
  position: absolute;
  top: -999px;
  left: 0;
  padding: 8px 16px;
  background: #000;
  color: #fff;
  text-decoration: none;
  z-index: 9999;
  border-radius: 0 0 4px 0;
}

.skip-link:focus {
  top: 0;
}
```

### Classe Visualmente Oculta

```css
/* Legível por AT, invisível na tela */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## Anti-Padrões a Evitar

**`<div>` para elementos interativos**
```html
<!-- Ruim -->
<div class="btn" onclick="submit()">Submit</div>

<!-- Bom -->
<button type="submit">Submit</button>
```

**Tabelas para layout**
```html
<!-- Ruim -->
<table><tr><td><nav>...</nav></td><td><main>...</main></td></tr></table>

<!-- Bom: CSS grid ou flexbox -->
<div class="layout-grid">
  <nav>...</nav>
  <main>...</main>
</div>
```

**Níveis de heading escolhidos pelo tamanho visual, não pela hierarquia**
```html
<!-- Ruim -->
<h1>Settings</h1>
<h4>Email Preferences</h4>  <!-- pulou h2, h3 -->

<!-- Bom: use CSS para tamanho de fonte, HTML para estrutura -->
<h1>Settings</h1>
<h2 class="subsection-heading">Email Preferences</h2>
```

**Placeholder como único label**
```html
<!-- Ruim: o label desaparece quando o usuário digita -->
<input type="email" placeholder="Email address">

<!-- Bom: label persistente + placeholder útil -->
<label for="email">Email address</label>
<input id="email" type="email" placeholder="you@example.com">
```

**Múltiplos elementos `<main>`**
```html
<!-- Ruim: apenas um <main> por página -->
<main>Dashboard content</main>
<main>Sidebar content</main>

<!-- Bom: use <aside> para conteúdo complementar -->
<main>Dashboard content</main>
<aside>Sidebar content</aside>
```

## Depuração e Solução de Problemas

### Painel Accessibility do Chrome

1. Abra DevTools → Elements
2. Selecione qualquer elemento
3. Clique na aba "Accessibility" (lado direito)
4. Veja: role, nome acessível, descrição acessível, propriedades e a accessibility tree

### Firefox Accessibility Inspector

1. DevTools → Accessibility
2. Ative "Turn on accessibility features" se necessário
3. Passe o mouse sobre elementos para ver role, nome e estado
4. O botão "Check for issues" executa uma auditoria rápida

### Validando Headers de Tabela

```bash
# Verificar tabelas sem captions e th sem scope
npx axe https://localhost:3000 --rules table-duplicate-name,th-has-data-cells,td-headers-attr
```

### Regras axe Comuns para Semântica

```bash
npx axe https://localhost:3000 \
  --rules landmark-one-main,page-has-heading-one,region,\
          label,list,listitem,definition-list,dlitem
```

## Cenários do Mundo Real

### Cenário 1: Componente Card

Cards são comuns em dashboards. Um card que linka para uma view de detalhe precisa ser um link correto, não um `<div>` com onClick.

```tsx
interface ProjectCard {
  id: string;
  name: string;
  status: 'active' | 'paused' | 'completed';
  lastUpdated: string;
}

function ProjectCard({ id, name, status, lastUpdated }: ProjectCard) {
  return (
    <article>
      <h3>
        {/* O heading contém o link — todo o heading é clicável */}
        <a href={`/projects/${id}`}>{name}</a>
      </h3>
      <dl>
        <dt>Status</dt>
        <dd>
          {/* Combine ícone + texto — não dependa apenas da cor */}
          <span aria-hidden="true">{status === 'active' ? '✓' : '○'}</span>
          {' '}{status}
        </dd>
        <dt>Last updated</dt>
        <dd><time dateTime={lastUpdated}>{lastUpdated}</time></dd>
      </dl>
    </article>
  );
}
```

### Cenário 2: Formulário de Cadastro em Múltiplos Passos

```tsx
function RegistrationForm() {
  const [step, setStep] = useState(1);
  const totalSteps = 3;

  return (
    <form aria-label="Account registration" noValidate>
      <nav aria-label="Registration progress">
        <ol>
          {['Account', 'Profile', 'Review'].map((label, i) => (
            <li
              key={label}
              aria-current={step === i + 1 ? 'step' : undefined}
            >
              {label}
            </li>
          ))}
        </ol>
      </nav>

      <p>
        <span className="sr-only">Passo {step} de {totalSteps}: </span>
        {step === 1 && 'Crie sua conta'}
        {step === 2 && 'Configure seu perfil'}
        {step === 3 && 'Revise e confirme'}
      </p>

      {step === 1 && (
        <fieldset>
          <legend>Account credentials</legend>
          <label htmlFor="reg-email">Email</label>
          <input id="reg-email" type="email" autoComplete="email" required />
          <label htmlFor="reg-password">Password</label>
          <input id="reg-password" type="password" autoComplete="new-password" required />
        </fieldset>
      )}

      {/* ... outros passos ... */}

      <div>
        {step > 1 && (
          <button type="button" onClick={() => setStep((s) => s - 1)}>
            Voltar
          </button>
        )}
        {step < totalSteps ? (
          <button type="button" onClick={() => setStep((s) => s + 1)}>
            Próximo
          </button>
        ) : (
          <button type="submit">Criar conta</button>
        )}
      </div>
    </form>
  );
}
```

## Leitura Complementar

- [HTML Living Standard — Semantics](https://html.spec.whatwg.org/multipage/semantics.html)
- [WAI-ARIA Authoring Practices — Landmark Regions](https://www.w3.org/WAI/ARIA/apg/practices/landmark-regions/)
- [WebAIM: Semantic Structure](https://webaim.org/techniques/semanticstructure/)
- [W3C Accessible Forms Tutorial](https://www.w3.org/WAI/tutorials/forms/)
- [W3C Accessible Tables Tutorial](https://www.w3.org/WAI/tutorials/tables/)

## Resumo

Semantic HTML oferece acessibilidade de graça. Os elementos-chave:

- **Landmarks** (`<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`) — mapa de navegação da página para usuários de screen reader; múltiplos elementos `<nav>` precisam de `aria-label` para serem diferenciados
- **Headings** — esboço do documento; um único `<h1>`, nunca pule níveis, use CSS para tamanho visual
- **Listas** — `<ul>`, `<ol>`, `<dl>` comunicam agrupamento e contagem; menus de navegação são listas
- **Formulários** — todo input precisa de um label via `for`/`id` ou `<label>` envolvente; agrupe inputs relacionados com `<fieldset>`/`<legend>`; marque erros com `aria-invalid` e `aria-describedby`
- **Tabelas** — apenas para dados tabulares; sempre use `<caption>`, `<thead>` e `scope` nos elementos `<th>`
- **Figure/Figcaption** — legendas para imagens, diagramas e exemplos de código

A mudança de maior impacto que você pode fazer: substituir wrappers `<div>` por elementos landmark com significado e garantir que todo input de formulário tenha um `<label>` associado programaticamente.
