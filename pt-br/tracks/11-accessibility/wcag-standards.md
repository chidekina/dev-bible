# WCAG Standards

## Visão Geral

As Web Content Accessibility Guidelines (WCAG) são o padrão internacional para tornar o conteúdo web acessível a pessoas com deficiências. Publicadas pela Web Accessibility Initiative (WAI) do W3C, as WCAG definem critérios de sucesso mensuráveis organizados em torno de quatro princípios: Perceivable, Operable, Understandable e Robust — frequentemente lembrados pela sigla **POUR**.

A WCAG 2.1 (publicada em 2018) adicionou 17 novos critérios com foco em dispositivos móveis, baixa visão e deficiências cognitivas. A WCAG 2.2 (publicada em 2023) acrescentou mais 9, com novos critérios de focus visible e autenticação acessível. Compreender esses padrões é essencial para entregar software que funcione para todos e para atender a requisitos legais (ADA nos EUA, EN 301 549 na Europa, requisitos relacionados à LGPD no Brasil).

## Pré-requisitos

- Conhecimento básico de HTML (formulários, headings, links, imagens)
- Familiaridade com DevTools do browser
- Entendimento conceitual de tecnologias assistivas (screen readers, navegação só por teclado)

## Conceitos Fundamentais

### Níveis de Conformidade

A WCAG define três níveis de conformidade:

| Nível | Descrição | Requisito típico |
|-------|-----------|-----------------|
| **A** | Acessibilidade mínima — deve ser atendida | Base legal na maioria das jurisdições |
| **AA** | Acessibilidade padrão — deve ser atendida | Exigida pela maioria das leis (ADA, EN 301 549) |
| **AAA** | Acessibilidade aprimorada — aspiracional | Recomendada para públicos especializados |

A maioria das organizações mira a **conformidade AA**. AAA não é exigida como meta geral, mas critérios AAA individuais devem ser atendidos quando viável.

### Os Princípios POUR

#### 1. Perceivable (Perceptível)

Informações e componentes de interface devem ser apresentáveis de formas que o usuário consiga perceber.

Critérios de sucesso principais:
- **1.1.1 Non-text Content (A)** — Todas as imagens, ícones e elementos não-textuais têm uma alternativa em texto
- **1.3.1 Info and Relationships (A)** — Estrutura transmitida visualmente também é transmitida programaticamente (headings, listas, labels)
- **1.3.3 Sensory Characteristics (A)** — Instruções não dependem exclusivamente de forma, cor, tamanho ou posição ("clique no botão vermelho" não passa)
- **1.4.1 Use of Color (A)** — A cor não é o único meio visual de transmitir informação
- **1.4.3 Contrast (Minimum) (AA)** — Razão de contraste de texto ≥ 4,5:1 (normal), ≥ 3:1 (texto grande, 18pt+ ou 14pt+ negrito)
- **1.4.4 Resize Text (AA)** — O texto pode ser redimensionado para 200% sem perda de conteúdo ou funcionalidade
- **1.4.10 Reflow (AA, 2.1)** — O conteúdo se adapta a 320px de largura sem scroll horizontal
- **1.4.11 Non-text Contrast (AA, 2.1)** — Componentes de interface e objetos gráficos têm contraste ≥ 3:1
- **1.4.12 Text Spacing (AA, 2.1)** — Nenhum conteúdo ou funcionalidade é perdido ao aumentar espaçamento entre letras/palavras/linhas

#### 2. Operable (Operável)

Componentes de interface e navegação devem ser operáveis.

Critérios de sucesso principais:
- **2.1.1 Keyboard (A)** — Toda funcionalidade está disponível pelo teclado
- **2.1.2 No Keyboard Trap (A)** — O usuário consegue sair de qualquer componente usando apenas o teclado
- **2.4.1 Bypass Blocks (A)** — Mecanismo para pular blocos de navegação repetidos (skip links)
- **2.4.3 Focus Order (A)** — A ordem do focus preserva significado e operabilidade
- **2.4.4 Link Purpose (A)** — O propósito do link é identificável a partir do próprio texto do link (ou do contexto)
- **2.4.7 Focus Visible (AA)** — O indicador de keyboard focus é visível
- **2.4.11 Focus Appearance (AA, 2.2)** — O indicador de focus tem tamanho e contraste suficientes
- **2.5.3 Label in Name (A, 2.1)** — O texto do label visível está contido no nome acessível

#### 3. Understandable (Compreensível)

Informações e operação da interface devem ser compreensíveis.

Critérios de sucesso principais:
- **3.1.1 Language of Page (A)** — O idioma padrão da página é determinado programaticamente (atributo `lang`)
- **3.2.1 On Focus (A)** — Nenhuma mudança de contexto ocorre apenas pelo focus
- **3.2.2 On Input (A)** — Nenhuma mudança de contexto ocorre ao inserir dados, a menos que o usuário seja avisado
- **3.3.1 Error Identification (A)** — Erros de entrada são descritos ao usuário em texto
- **3.3.2 Labels or Instructions (A)** — Labels ou instruções são fornecidos para entradas do usuário
- **3.3.7 Redundant Entry (A, 2.2)** — Informações já inseridas são preenchidas automaticamente ou ficam disponíveis
- **3.3.8 Accessible Authentication (AA, 2.2)** — Nenhum teste de função cognitiva é exigido para autenticação (sem "digite os caracteres que você vê")

#### 4. Robust (Robusto)

O conteúdo deve ser robusto o suficiente para ser interpretado de forma confiável por tecnologias assistivas.

Critérios de sucesso principais:
- **4.1.1 Parsing (A)** — Sem IDs duplicados, elementos aninhados corretamente (descontinuado na WCAG 2.2 — os browsers já tratam isso)
- **4.1.2 Name, Role, Value (A)** — Todos os componentes de interface têm nome, role e estado determinável pela AT
- **4.1.3 Status Messages (AA, 2.1)** — Mensagens de status podem ser determinadas programaticamente sem receber focus (live regions ARIA)

### Entendendo a Estrutura dos Critérios de Sucesso

Cada critério possui:
- **Intent** — qual problema ele resolve
- **Benefits** — quem se beneficia
- **Examples** — casos de sucesso e falha
- **Sufficient techniques** — como atendê-lo (não prescritivo, independente de técnica)
- **Advisory techniques** — melhorias opcionais
- **Failures** — formas documentadas de falhar

## Exemplos Práticos

### Verificando Razões de Contraste

```typescript
// Utilitário para calcular a razão de contraste WCAG entre duas cores hex
function hexToRgb(hex: string): [number, number, number] {
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  if (!result) throw new Error(`Invalid hex color: ${hex}`);
  return [
    parseInt(result[1], 16),
    parseInt(result[2], 16),
    parseInt(result[3], 16),
  ];
}

function relativeLuminance(r: number, g: number, b: number): number {
  const [rs, gs, bs] = [r, g, b].map((c) => {
    const s = c / 255;
    return s <= 0.03928 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4);
  });
  return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
}

function contrastRatio(hex1: string, hex2: string): number {
  const [r1, g1, b1] = hexToRgb(hex1);
  const [r2, g2, b2] = hexToRgb(hex2);
  const l1 = relativeLuminance(r1, g1, b1);
  const l2 = relativeLuminance(r2, g2, b2);
  const lighter = Math.max(l1, l2);
  const darker = Math.min(l1, l2);
  return (lighter + 0.05) / (darker + 0.05);
}

function meetsWCAG(
  foreground: string,
  background: string,
  level: 'AA' | 'AAA' = 'AA',
  large = false
): boolean {
  const ratio = contrastRatio(foreground, background);
  if (level === 'AAA') return large ? ratio >= 4.5 : ratio >= 7;
  return large ? ratio >= 3 : ratio >= 4.5;
}

// Uso
console.log(contrastRatio('#ffffff', '#767676').toFixed(2)); // ~4.54 — passa AA texto normal
console.log(meetsWCAG('#ffffff', '#767676')); // true
console.log(meetsWCAG('#ffffff', '#aaaaaa')); // false — apenas 2.32:1
```

### Automatizando Verificações de Contraste no CI

```typescript
// scripts/check-contrast.ts
import { contrastRatio, meetsWCAG } from './color-utils';

const brandColors = {
  primary: '#1a56db',
  primaryText: '#ffffff',
  secondary: '#6b7280',
  secondaryText: '#ffffff',
  danger: '#dc2626',
  dangerText: '#ffffff',
  warning: '#d97706',
  warningText: '#000000',
};

const pairs: Array<{ name: string; fg: string; bg: string; large?: boolean }> = [
  { name: 'Primary button', fg: brandColors.primaryText, bg: brandColors.primary },
  { name: 'Secondary button', fg: brandColors.secondaryText, bg: brandColors.secondary },
  { name: 'Danger badge', fg: brandColors.dangerText, bg: brandColors.danger },
  { name: 'Warning badge (large)', fg: brandColors.warningText, bg: brandColors.warning, large: true },
];

let failed = false;
for (const { name, fg, bg, large } of pairs) {
  const ratio = contrastRatio(fg, bg).toFixed(2);
  const passes = meetsWCAG(fg, bg, 'AA', large);
  const status = passes ? 'PASS' : 'FAIL';
  console.log(`[${status}] ${name}: ${ratio}:1`);
  if (!passes) failed = true;
}

if (failed) process.exit(1);
```

### Declaração de Idioma

```html
<!-- Idioma raiz — sempre defina isso -->
<html lang="pt-BR">

<!-- Troca de idioma inline para conteúdo multilíngue -->
<p>
  A frase em inglês
  <span lang="en">hello world</span>
  significa "olá mundo".
</p>
```

### Auditando uma Página com Axe

```typescript
// playwright test — veja testing-accessibility.md para o setup completo
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('homepage meets WCAG 2.1 AA', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

## Padrões e Boas Práticas

### Construindo um Checklist de Auditoria de Acessibilidade

Percorra essas categorias em cada página nova:

**Perceivable**
- [ ] Todas as imagens têm texto `alt` significativo (imagens decorativas usam `alt=""`)
- [ ] Contraste de cor ≥ 4,5:1 para texto normal, ≥ 3:1 para texto grande e elementos de interface
- [ ] Nenhuma informação transmitida apenas por cor (adicione ícones, padrões ou rótulos de texto)
- [ ] A página funciona com o tamanho de fonte ampliado para 200%
- [ ] A página se adapta corretamente a 320px de largura (sem scroll horizontal)

**Operable**
- [ ] Todos os elementos interativos são alcançáveis e operáveis com teclado
- [ ] Indicador de focus visível em todos os elementos focáveis
- [ ] Skip navigation link presente
- [ ] Sem keyboard traps
- [ ] Links com texto descritivo (não "clique aqui" ou "leia mais")

**Understandable**
- [ ] `<html lang="...">` definido corretamente
- [ ] Campos de formulário têm labels visíveis (não apenas placeholder)
- [ ] Mensagens de erro identificam o campo e descrevem o problema
- [ ] Instruções não dependem apenas de pistas sensoriais

**Robust**
- [ ] Sem IDs duplicados na página
- [ ] Todos os controles de formulário têm elementos `<label>` associados
- [ ] Elementos interativos têm roles ARIA corretos se não usarem HTML nativo
- [ ] Mudanças dinâmicas anunciadas via live regions quando necessário

### Priorizando Problemas

Use níveis de severidade ao triageitar:

| Severidade | Exemplo | Ação |
|------------|---------|------|
| Crítico | Sem acesso por teclado ao fluxo principal | Bloquear release |
| Sério | Labels de formulário ausentes | Corrigir na sprint atual |
| Moderado | Contraste insuficiente em texto secundário | Corrigir na próxima sprint |
| Menor | `lang` ausente em frase inline | Registrar no backlog |

## Anti-Padrões a Evitar

**Usar cor sozinha para transmitir estado**
```html
<!-- Ruim: apenas a cor diferencia obrigatório de opcional -->
<label style="color: red;">Email</label>

<!-- Bom: indicador de texto + cor -->
<label>Email <span aria-hidden="true">*</span><span class="sr-only">(obrigatório)</span></label>
```

**Suprimir o outline de focus sem substituição**
```css
/* Ruim: remove completamente o indicador de focus */
* { outline: none; }

/* Bom: substitua por um indicador personalizado visível */
:focus-visible {
  outline: 3px solid #1a56db;
  outline-offset: 2px;
}
```

**Texto de link vago**
```html
<!-- Ruim -->
<a href="/report.pdf">Clique aqui</a>

<!-- Bom -->
<a href="/report.pdf">Baixar relatório anual (PDF, 2,4 MB)</a>
```

**Placeholder como único label**
```html
<!-- Ruim: quando o usuário começa a digitar, o label desaparece -->
<input type="email" placeholder="Email address">

<!-- Bom -->
<label for="email">Email address</label>
<input id="email" type="email" placeholder="you@example.com">
```

**Imagens com alt redundante ou ausente**
```html
<!-- Ruim: redundante -->
<img src="logo.png" alt="Image of our company logo">

<!-- Ruim: ausente para imagem significativa -->
<img src="chart.png" alt="">

<!-- Bom: alt significativo -->
<img src="chart.png" alt="Bar chart showing 40% increase in revenue from Q1 to Q4 2024">

<!-- Bom: imagem decorativa -->
<img src="divider.png" alt="" role="presentation">
```

## Depuração e Solução de Problemas

### Executando Auditorias Automatizadas

```bash
# Instalar o CLI do axe
npm install -g @axe-core/cli

# Auditar uma URL
axe https://example.com --tags wcag2a,wcag2aa

# Auditar com regras específicas
axe https://example.com --rules color-contrast,label,image-alt
```

### Usando as DevTools do Browser

**Painel Accessibility do Chrome:**
1. Abra DevTools → Elements
2. Selecione um elemento
3. Clique na aba "Accessibility" no painel
4. Inspecione a accessibility tree, o nome computado, o role e as propriedades

**Auditoria Lighthouse de acessibilidade:**
1. DevTools → Lighthouse
2. Marque "Accessibility"
3. Gere o relatório — pontuação de cada critério e links para os elementos com falha

### Testando Apenas com Teclado

Desconecte o mouse e navegue por todo o fluxo:
1. `Tab` — mover para o próximo elemento focável
2. `Shift+Tab` — mover para o anterior
3. `Enter` / `Space` — ativar botões e links
4. Teclas de seta — navegar dentro de componentes (menus, tabs, sliders)
5. `Escape` — fechar dialogs, dispensar overlays

Se não conseguir completar o fluxo, há uma violação da WCAG 2.1.1.

### Verificando a Accessibility Tree

```typescript
// Playwright: inspecionar nome acessível computado e role
test('button has accessible name', async ({ page }) => {
  await page.goto('/dashboard');
  const btn = page.getByRole('button', { name: 'Save changes' });
  await expect(btn).toBeVisible();
  // Playwright usa a accessibility tree, então isso já valida o nome da AT
});
```

## Cenários do Mundo Real

### Cenário 1: Erros de Validação de Formulário

Quando o envio de um formulário falha, usuários de screen reader precisam saber o que deu errado. Um erro comum é exibir os erros visualmente, mas não programaticamente.

```tsx
// Componente React com tratamento de erro acessível
interface FieldProps {
  id: string;
  label: string;
  error?: string;
  type?: string;
}

function FormField({ id, label, error, type = 'text' }: FieldProps) {
  const errorId = `${id}-error`;
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        type={type}
        aria-describedby={error ? errorId : undefined}
        aria-invalid={error ? 'true' : undefined}
      />
      {error && (
        <span id={errorId} role="alert" className="error-message">
          {error}
        </span>
      )}
    </div>
  );
}
```

### Cenário 2: Atualizações de Conteúdo Dinâmico

Um dashboard que exibe uma notificação quando novos dados são carregados:

```tsx
function Dashboard() {
  const [status, setStatus] = useState('');

  async function refreshData() {
    setStatus('Carregando dados...');
    await fetchData();
    setStatus('Dados atualizados com sucesso');
  }

  return (
    <div>
      {/* Live region anuncia mudanças dinâmicas para screen readers */}
      <div role="status" aria-live="polite" className="sr-only">
        {status}
      </div>
      <button onClick={refreshData}>Atualizar</button>
      {/* ... tabela de dados ... */}
    </div>
  );
}
```

### Cenário 3: Layout Responsivo para Reflow (1.4.10)

```css
/* Ruim: largura fixa causa scroll horizontal a 320px */
.card {
  width: 600px;
  padding: 24px;
}

/* Bom: layout fluido sem scroll horizontal */
.card {
  width: 100%;
  max-width: 600px;
  padding: clamp(12px, 4vw, 24px);
  box-sizing: border-box;
}
```

## Leitura Complementar

- [WCAG 2.2 Quick Reference](https://www.w3.org/WAI/WCAG22/quickref/) — lista filtrável de todos os critérios de sucesso
- [Understanding WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/) — intenção e técnicas para cada critério
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) — ferramenta online rápida
- [Deque University](https://dequeuniversity.com/) — treinamento estruturado em acessibilidade
- [A11y Project Checklist](https://www.a11yproject.com/checklist/) — checklist de implementação prático
- [axe-core rules](https://github.com/dequelabs/axe-core/blob/develop/doc/rule-descriptions.md) — lista completa de verificações automatizadas

## Resumo

A WCAG organiza os requisitos de acessibilidade em torno de quatro princípios — Perceivable, Operable, Understandable, Robust — em três níveis de conformidade (A, AA, AAA). A maioria das organizações e legislações exige conformidade AA. A WCAG 2.1 adicionou critérios para mobile e cognição; a WCAG 2.2 adicionou critérios melhores de focus e autenticação.

Pontos essenciais para internalizar:
- Razão de contraste AA é 4,5:1 para texto normal, 3:1 para texto grande e componentes de interface
- Todo elemento interativo deve ser acessível por teclado
- Cor sozinha não pode transmitir informação
- Todos os campos de formulário precisam de labels programáticos
- Mudanças de conteúdo dinâmico precisam de live regions para que screen readers as anunciem
- Automatize o que puder (axe-core), mas também teste manualmente com teclado e screen readers
