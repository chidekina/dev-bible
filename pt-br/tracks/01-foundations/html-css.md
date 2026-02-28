# HTML & CSS

> HTML dá estrutura à web; CSS dá apresentação. Juntos formam a base de toda interface. Este arquivo cobre marcação semântica, acessibilidade, o box model, posicionamento, contextos de empilhamento, Flexbox, Grid, design responsivo, propriedades customizadas CSS e animações com performance.

---

## 1. O Que São e Por Que Importam

HTML e CSS são as linguagens de programação mais antigas e mais amplamente implantadas na existência. Ainda assim, também estão entre as mais incompreendidas — frequentemente descartadas como "não são programação de verdade" até que um layout se recuse a funcionar, um leitor de tela interprete mal o conteúdo ou um bug de z-index apareça em algum browser aleatório. Dominar HTML e CSS significa:

- Construir interfaces acessíveis que funcionam para todos, incluindo usuários com deficiência
- Escrever CSS sustentável que não degenera em guerras de especificidade
- Entender sistemas de layout bem o suficiente para implementar qualquer design sem gambiarras
- Saber quais propriedades CSS disparam layout/paint caros vs. mudanças baratas de apenas composite
- Escrever marcação semântica que melhora SEO e é corretamente interpretada por mecanismos de busca e tecnologias assistivas

---

## 2. Conceitos Fundamentais

### HTML Semântico

Elementos semânticos carregam significado sobre o conteúdo que contêm. Esse significado é usado por browsers, leitores de tela, rastreadores de mecanismos de busca e ferramentas de desenvolvimento.

| Elemento | Significado |
|---------|---------|
| `<header>` | Conteúdo introdutório para seu ancestral seccionante mais próximo (nível de página ou dentro de `<article>`) |
| `<nav>` | Um conjunto de links de navegação |
| `<main>` | O conteúdo dominante do `<body>` — apenas um por página |
| `<article>` | Conteúdo autocontido que faz sentido independentemente (post de blog, artigo de notícia, comentário) |
| `<section>` | Um agrupamento temático de conteúdo com um heading — não é um wrapper genérico (use `<div>` para isso) |
| `<aside>` | Conteúdo tangencialmente relacionado ao conteúdo principal (sidebar, links relacionados) |
| `<footer>` | Rodapé para seu ancestral seccionante mais próximo (rodapé da página ou dentro de `<article>`) |
| `<figure>` + `<figcaption>` | Conteúdo visual autocontido com uma legenda |
| `<time datetime="2024-01-01">` | Uma data/hora específica — `datetime` fornece formato legível por máquina |
| `<address>` | Informações de contato para o `<article>` ou `<body>` mais próximo |
| `<mark>` | Texto destacado (resultados de busca) |
| `<abbr title="...">` | Abreviação com expansão completa ao passar o mouse |

**Por que HTML semântico importa:**
- Leitores de tela usam landmarks (`<nav>`, `<main>`, `<aside>`) para deixar usuários pular diretamente para seções
- Mecanismos de busca pesam mais o conteúdo em `<article>` e `<h1>`
- Ferramentas de desenvolvimento (VS Code IntelliSense, auditorias do Lighthouse) podem sinalizar elementos mal usados
- `<button>` pode receber foco via teclado e ser ativado por Enter/Space; um `<div>` estilizado como botão não pode

---

### Acessibilidade (a11y)

**ARIA (Accessible Rich Internet Applications):**

Atributos ARIA adicionam informações semânticas quando a semântica nativa do HTML é insuficiente. A primeira regra do ARIA: use semântica HTML nativa primeiro. Só adicione ARIA se não houver elemento nativo que faça o que você precisa.

```html
<!-- Prefira botão nativo ao invés de role ARIA -->
<button type="button">Salvar</button>  <!-- correto -->
<div role="button" tabindex="0">Salvar</div>  <!-- apenas se necessário -->

<!-- Associar label a input de formulário -->
<label for="email">Endereço de e-mail</label>
<input id="email" type="email" aria-describedby="email-hint" />
<span id="email-hint">Nunca compartilharemos seu e-mail</span>

<!-- Roles de landmark (redundantes em elementos semânticos, necessários em divs) -->
<div role="navigation" aria-label="Navegação principal">...</div>

<!-- Live regions — anuncia conteúdo dinâmico para leitores de tela -->
<div aria-live="polite" aria-atomic="true" id="status-message">
  <!-- Conteúdo injetado aqui é anunciado sem mudança de foco -->
</div>

<!-- Valores de aria-live:
     polite   — anuncia quando o usuário está ocioso (mensagens de validação de formulário)
     assertive — interrompe imediatamente (erros críticos) -->
```

**Navegação por teclado:**
- Todos os elementos interativos devem ser alcançáveis por Tab em ordem lógica
- O foco deve ser visível (não use `outline: none` sem um substituto)
- Modais devem prender o foco enquanto abertos; retornar o foco ao gatilho quando fechados
- Menus: use teclas de seta dentro do menu, Escape para fechar

**Texto alternativo:**
```html
<!-- Imagem informativa: descreva o que ela transmite -->
<img src="grafico.png" alt="Gráfico de barras mostrando receita do Q4 subindo 23% ano a ano" />

<!-- Imagem decorativa: atributo alt vazio (leitor de tela pula) -->
<img src="divisor.png" alt="" role="presentation" />

<!-- Imagens como links: descreva o destino -->
<a href="/inicio"><img src="logo.png" alt="Nome da empresa — página inicial" /></a>
```

---

### O Box Model

Todo elemento em CSS é uma caixa. O box model define como as dimensões dessa caixa são calculadas:

```
┌─────────────────────────────────┐  ← borda de margem
│            margem               │
│  ┌───────────────────────────┐  │  ← borda de border
│  │          border           │  │
│  │  ┌─────────────────────┐  │  │  ← borda de padding
│  │  │       padding       │  │  │
│  │  │  ┌───────────────┐  │  │  │  ← borda de conteúdo
│  │  │  │    conteúdo   │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**`box-sizing: content-box` (padrão, confuso):**
```css
.caixa {
  width: 200px;
  padding: 20px;
  border: 5px solid;
  /* largura renderizada real = 200 + 20*2 + 5*2 = 250px */
}
```

**`box-sizing: border-box` (o que você realmente quer):**
```css
.caixa {
  width: 200px;
  padding: 20px;
  border: 5px solid;
  /* largura renderizada real = 200px (padding e border cabem dentro) */
}
```

> Aplique `border-box` globalmente como reset:
> ```css
> *, *::before, *::after { box-sizing: border-box; }
> ```
> Isso está incluído em todo reset CSS moderno (Normalize, Tailwind base).

**Colapso de margem:** Margens verticais adjacentes colapsam para a maior das duas. Isso só acontece no fluxo normal (não em containers flex ou grid).
```css
/* h2 tem margin-bottom: 16px, p tem margin-top: 8px */
/* O espaço entre eles é 16px (o maior), NÃO 24px */
```

---

### Posicionamento

```
static    → fluxo normal (padrão, top/left/right/bottom não têm efeito)
relative  → fluxo normal + deslocamento da posição natural (cria contexto de empilhamento com z-index)
absolute  → removido do fluxo, posicionado relativo ao ancestral não-static mais próximo
fixed     → removido do fluxo, posicionado relativo ao viewport (permanece ao rolar)
sticky    → fluxo normal até o limite, depois fixed dentro do container de rolagem
```

**Diagramas:**

```
static (padrão):
┌─────────┐
│  bloco  │  ocupa seu espaço natural no fluxo
└─────────┘

relative (deslocado, ainda ocupa espaço original):
┌─────────┐          ┌ ─ ─ ─ ─ ─┐
│         │  ──────> │  elemento │ (deslocado 20px à direita, espaço fantasma permanece)
└─────────┘          └ ─ ─ ─ ─ ─┘

absolute (removido do fluxo, prende ao ancestral):
Pai (position: relative)
┌─────────────────────────┐
│                         │
│              ┌────────┐ │ ← filho absolute, top: 10px, right: 10px
│              │ filho  │ │
│              └────────┘ │
└─────────────────────────┘

fixed (relativo ao viewport, ignora rolagem):
┌─────────────────────────────────────┐ ← viewport
│  ┌──────────────────────────────┐   │
│  │         navbar fixa          │   │ ← fixed, top: 0
│  └──────────────────────────────┘   │
│                                     │
│  ... conteúdo rolável ...           │
└─────────────────────────────────────┘

sticky (relativo até o limite, depois fixed dentro do container de rolagem):
Após ultrapassar o limite:
┌─────────────────────────────────────┐ ← container de rolagem
│  ┌──────────────────────────────┐   │
│  │   cabeçalho de seção sticky  │   │ ← preso em top: 0
│  └──────────────────────────────┘   │
│  ... conteúdo da seção rolando ...  │
└─────────────────────────────────────┘
```

---

### Contexto de Empilhamento

Um contexto de empilhamento é uma camada independente no eixo Z. Elementos dentro de um contexto de empilhamento são empilhados relativamente entre si, não à pilha global.

**O que cria um contexto de empilhamento:**
- `position: relative/absolute/fixed/sticky` + `z-index` diferente de `auto`
- `transform` (qualquer valor diferente de `none`)
- `opacity` menor que 1
- `filter`, `backdrop-filter`
- `will-change: transform/opacity`
- `isolation: isolate` (cria explicitamente um sem efeitos colaterais)
- `clip-path`, `mask`
- Filhos de Flex/Grid com `z-index` diferente de `auto`

> ⚠️ **Por que z-index "não funciona":** Se o pai do seu elemento tem um z-index menor em seu próprio contexto de empilhamento, seu filho nunca pode escapar visualmente dele. Definir `z-index: 9999` em um filho é inútil se o contexto de empilhamento do pai está empilhado abaixo do contexto de empilhamento de outro elemento.

```html
<!-- Exemplo de armadilha de z-index -->
<div style="position: relative; z-index: 1;">  <!-- contexto de empilhamento A -->
  <div style="z-index: 9999;">...</div>         <!-- z-index 9999 dentro de A -->
</div>
<div style="position: relative; z-index: 2;">  <!-- contexto de empilhamento B, acima de A -->
  <div style="z-index: 1;">...</div>            <!-- ainda acima de todo A -->
</div>
```

---

## 3. Como Funciona

### Flexbox — Guia Completo

Flexbox é um sistema de layout unidimensional (linha ou coluna).

```css
/* Propriedades do container */
.container {
  display: flex;

  flex-direction: row;           /* row | row-reverse | column | column-reverse */
  justify-content: flex-start;   /* eixo principal: flex-start | center | flex-end | space-between | space-around | space-evenly */
  align-items: stretch;          /* eixo transversal: stretch | flex-start | center | flex-end | baseline */
  flex-wrap: nowrap;             /* nowrap | wrap | wrap-reverse */
  gap: 16px;                     /* atalho para row-gap + column-gap */
  align-content: normal;         /* eixo transversal multi-linha: mesmos valores que justify-content */
}

/* Propriedades dos itens */
.item {
  flex-grow: 0;      /* quanto espaço extra este item ocupa (0 = não cresce) */
  flex-shrink: 1;    /* quanto este item encolhe quando o espaço é restrito (0 = não encolhe) */
  flex-basis: auto;  /* tamanho inicial antes de crescer/encolher (pode ser px, %, auto) */
  flex: 1;           /* atalho: flex-grow flex-shrink flex-basis = 1 1 0% */

  align-self: auto;  /* sobrescreve align-items para este item */
  order: 0;          /* mudar ordem visual (padrão 0, menor = anterior) */
}
```

**Padrões comuns de flex:**
```css
/* Centralizar qualquer coisa horizontal e verticalmente */
.center { display: flex; justify-content: center; align-items: center; }

/* Padrão de rodapé fixo */
.pagina { display: flex; flex-direction: column; min-height: 100vh; }
.conteudo { flex: 1; }  /* ocupa todo o espaço restante */

/* Colunas de largura igual que encolhem graciosamente */
.cols { display: flex; gap: 16px; }
.col { flex: 1; min-width: 0; }  /* min-width: 0 evita overflow em itens flex */
```

---

### CSS Grid — Guia Completo

Grid é um sistema de layout bidimensional (linhas e colunas simultaneamente).

```css
/* Container */
.grid {
  display: grid;

  /* Define colunas */
  grid-template-columns: 1fr 2fr 1fr;          /* 3 colunas, a do meio é 2x mais larga */
  grid-template-columns: repeat(3, 1fr);        /* igual */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));  /* colunas responsivas */
  grid-template-columns: 250px 1fr;             /* sidebar fixo + conteúdo fluido */

  /* Define linhas (opcional — linhas se auto-criam por padrão) */
  grid-template-rows: auto 1fr auto;            /* cabeçalho, conteúdo, rodapé */

  /* Áreas nomeadas */
  grid-template-areas:
    "header  header  header"
    "sidebar content content"
    "footer  footer  footer";

  gap: 16px;              /* ou row-gap + column-gap separadamente */
  align-items: stretch;   /* alinha itens no eixo de bloco */
  justify-items: stretch; /* alinha itens no eixo inline */
}

/* Itens */
.header { grid-area: header; }        /* usa área nomeada */
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }

/* Ou usa números de linha */
.span-dois {
  grid-column: 1 / 3;    /* linha inicial 1, linha final 3 (abrange 2 colunas) */
  grid-row: 2 / 4;       /* linha de linha 2, linha final 4 */
}
.largura-total {
  grid-column: 1 / -1;   /* -1 = última linha, abrange todas as colunas */
}
```

**`auto-fill` vs `auto-fit`:**
```css
/* auto-fill: cria o máximo de colunas possível, deixa as vazias */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* auto-fit: cria colunas, colapsa as vazias (itens se esticam para preencher) */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));

/* Com poucos itens: auto-fit os estica, auto-fill deixa espaço vazio */
```

> **Flexbox vs Grid:** Use Flexbox ao dispor itens em uma única direção (links de nav, grupos de botões, cards em uma linha) e o layout deve se adaptar ao tamanho do conteúdo. Use Grid quando precisar de controle sobre linhas e colunas simultaneamente (layouts de página, galerias de imagem, formulários complexos).

---

### Design Responsivo

**Abordagem mobile-first** — escreva estilos base para telas pequenas, depois adicione media queries `min-width` para telas maiores:

```css
/* Base mobile — sem media query necessária */
.card { padding: 16px; }
.grid { grid-template-columns: 1fr; }

/* Tablet */
@media (min-width: 768px) {
  .grid { grid-template-columns: repeat(2, 1fr); }
}

/* Desktop */
@media (min-width: 1024px) {
  .card { padding: 24px; }
  .grid { grid-template-columns: repeat(3, 1fr); }
}
```

**Meta tag de viewport** — necessária para design responsivo em mobile:
```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
<!-- Sem isso, browsers mobile renderizam na largura desktop e reduzem o zoom -->
```

**Tipografia fluida com `clamp()`:**
```css
/* Fonte escala de 16px (em viewport 320px) para 24px (em viewport 1200px) */
.heading {
  font-size: clamp(1rem, 0.5rem + 2.5vw, 1.5rem);
  /* mínimo, preferido (relativo ao viewport), máximo */
}
```

**Container queries** (browsers modernos, sem IE):
```css
/* Estiliza baseado na largura do container, não do viewport */
.card-container { container-type: inline-size; }

@container (min-width: 400px) {
  .card { display: flex; gap: 16px; }
}
```

---

### Propriedades Customizadas CSS (Variáveis)

```css
/* Declaração — em :root para escopo global */
:root {
  --color-primary: #3b82f6;
  --color-text: #1f2937;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --radius: 6px;
}

/* Uso */
.botao {
  background: var(--color-primary);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius);
}

/* Valor de fallback */
.caixa { color: var(--color-brand, #000); }

/* Sobrescrita com escopo */
.tema-escuro {
  --color-text: #f9fafb;
  --color-primary: #60a5fa;
}

/* Acesso via JavaScript */
const root = document.documentElement;
root.style.setProperty('--color-primary', '#ef4444');
const value = getComputedStyle(root).getPropertyValue('--color-primary');
```

---

### Animações e Transições

**Transições** — mudança suave quando uma propriedade CSS muda:
```css
.botao {
  background: blue;
  transform: scale(1);
  transition: background 200ms ease, transform 150ms ease;
  /* propriedade duração timing-function */
}
.botao:hover {
  background: darkblue;
  transform: scale(1.05);
}
```

**Animações com @keyframes:**
```css
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-10px); }
  to   { opacity: 1; transform: translateY(0); }
}

.modal {
  animation: fadeIn 300ms ease forwards;
  /* nome duração timing fill-mode */
}

/* Pausar com preferência de movimento reduzido */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**Performance — propriedades apenas de composite:**

Mudar a maioria das propriedades CSS dispara layout → paint → composite (caro). Apenas duas propriedades são compostas na GPU sem disparar layout ou paint:
- `transform` (translate, scale, rotate)
- `opacity`

```css
/* Ruim — dispara reflow de layout em cada frame */
.animacao-ruim { animation: mover-ruim 1s; }
@keyframes mover-ruim { to { left: 100px; } }

/* Bom — GPU-composited, sem layout, 60fps */
.animacao-boa { animation: mover-bom 1s; }
@keyframes mover-bom { to { transform: translateX(100px); } }

/* Informa ao browser para preparar uma camada de composição */
.vai-animar { will-change: transform; }
/* Remova após a animação se houver muitos elementos — will-change usa memória GPU */
```

---

## 4. Exemplos de Código

### Estrutura semântica de página
```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Título do Artigo — Nome do Site</title>
  <meta name="description" content="Uma descrição de 150 caracteres para SEO." />
</head>
<body>
  <header>
    <nav aria-label="Navegação principal">
      <a href="/">Início</a>
      <a href="/sobre" aria-current="page">Sobre</a>
    </nav>
  </header>

  <main>
    <article>
      <header>
        <h1>Título do Artigo</h1>
        <time datetime="2024-01-15">15 de janeiro de 2024</time>
        <address rel="author">Por <a href="/autores/alice">Alice</a></address>
      </header>

      <section aria-labelledby="intro-heading">
        <h2 id="intro-heading">Introdução</h2>
        <p>...</p>
      </section>
    </article>

    <aside aria-label="Artigos relacionados">
      <h2>Relacionados</h2>
      <ul>...</ul>
    </aside>
  </main>

  <footer>
    <p><small>&copy; 2024 Nome do Site</small></p>
  </footer>
</body>
</html>
```

### Especificidade CSS e a cascata
```css
/* Especificidade: (inline, IDs, classes/attrs/pseudos, elementos) */

p { color: black; }                        /* (0,0,0,1) */
.text { color: blue; }                     /* (0,0,1,0) vence sobre p */
#content { color: green; }                 /* (0,1,0,0) vence sobre .text */
style="color: red"                         /* (1,0,0,0) vence sobre tudo */

/* Calculando */
nav ul li a.active:hover { ... }
/* 0 IDs, 2 classes/pseudos (active, hover), 4 elementos (nav, ul, li, a) = (0,0,2,4) */

/* !important sobrescreve tudo (exceto outro !important com especificidade maior) */
/* Evite — torna a depuração um pesadelo */
```

### Padrões de centralização
```css
/* 1. Flexbox (recomendado para a maioria dos casos) */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 2. Grid */
.container {
  display: grid;
  place-items: center;  /* atalho para align-items + justify-items */
}

/* 3. Posicionamento absoluto (quando você sabe as dimensões ou usa transform) */
.filho {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* 4. Margin auto (apenas horizontal, elementos de bloco com largura conhecida) */
.filho {
  width: 300px;
  margin: 0 auto;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Guerras de especificidade**: Quando múltiplos desenvolvedores adicionam `!important` para sobrescrever uns aos outros, a cascata entra em colapso. Solução: use um sistema de nomenclatura consistente (BEM, CSS Modules, classes utilitárias) que evita conflitos de especificidade por design. O seletor universal `*` tem especificidade 0; elementos têm 1; classes têm 10; IDs têm 100.

> ⚠️ **z-index sem contexto de posicionamento**: `z-index` não tem efeito em elementos com `position: static` (o padrão). Você deve definir `position: relative` (ou qualquer valor não-static) para o z-index funcionar. Este é o bug de z-index mais comum.

> ⚠️ **Usar `!important`**: Quebra a cascata e torna futuras sobrescritas praticamente impossíveis. O único caso de uso válido é classes utilitárias que devem sempre vencer (como `display: none` em `.hidden { display: none !important }`).

> ⚠️ **Surpresa com colapso de margem**: Margens verticais entre irmãos de bloco adjacentes colapsam para o maior valor. Margens também colapsam entre um pai e seu primeiro/último filho se não houver border, padding ou contexto de formatação de bloco entre eles. Containers Flexbox e Grid não colapsam margens.

> ⚠️ **Não usar `min-width: 0` em filhos flex**: Itens flex padrão com `min-width: auto`, o que impede que encolham abaixo do tamanho do conteúdo. Texto longo ou imagens podem vazar do container. Defina `min-width: 0` em filhos flex que contenham conteúdo potencialmente transbordante.

> ⚠️ **Animar propriedades que disparam layout**: Animar `width`, `height`, `top`, `left`, `margin` ou `padding` força o browser a recalcular o layout em cada frame, causando queda de desempenho abaixo de 60fps. Use `transform` e `opacity` em vez disso.

> ⚠️ **Falta do atributo `lang` em `<html>`**: Leitores de tela usam isso para determinar quais regras de pronúncia aplicar. Sem ele, conteúdo em português pode ser pronunciado incorretamente.

---

## 6. Quando Usar / Não Usar

**Use Flexbox quando:**
- Dispor itens em uma única direção (nav horizontal, pilha vertical)
- Os tamanhos dos itens devem ser determinados pelo conteúdo
- Você precisa de centralização vertical fácil

**Use Grid quando:**
- Precisar controlar linhas e colunas simultaneamente
- Construir layouts em nível de página (header/sidebar/content/footer)
- Criar galerias de imagem ou grids de cards
- Itens precisam se alinhar entre linhas e colunas

**Use elementos semânticos quando:**
- Sempre — quase sempre há um elemento semântico que corresponde ao conteúdo

**Use ARIA quando:**
- Um widget customizado não tem equivalente HTML nativo (ex: combobox customizado, tabs, tree)
- A semântica nativa precisa ser corrigida (ex: `<div role="main">` como polyfill)
- Conteúdo dinâmico precisa ser anunciado (`aria-live`)

**Use propriedades customizadas CSS quando:**
- Construindo um design system com tokens (cores, espaçamento, tipografia)
- Temas (toggle de modo claro/escuro via troca de classe em `:root`)
- Valores necessários em JavaScript

---

## 7. Cenário do Mundo Real

### Construindo uma barra de navegação responsiva com menu hambúrguer

```html
<header class="header">
  <a href="/" class="logo">Marca</a>

  <button class="nav-toggle" aria-controls="main-nav" aria-expanded="false">
    <span class="sr-only">Alternar navegação</span>
    <span class="hamburger" aria-hidden="true"></span>
  </button>

  <nav id="main-nav" class="nav" aria-label="Principal">
    <ul class="nav__list">
      <li><a href="/" class="nav__link">Início</a></li>
      <li><a href="/sobre" class="nav__link">Sobre</a></li>
      <li><a href="/contato" class="nav__link">Contato</a></li>
    </ul>
  </nav>
</header>
```

```css
/* Base (mobile) */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 16px;
  height: 60px;
  background: var(--color-surface);
}

.nav {
  display: none;  /* oculto no mobile */
  position: absolute;
  top: 60px;
  left: 0;
  right: 0;
  background: var(--color-surface);
  padding: 16px;
}
.nav.is-open { display: block; }

.nav__list { list-style: none; margin: 0; padding: 0; }
.nav__link { display: block; padding: 12px 0; text-decoration: none; }

/* Ícone hambúrguer */
.hamburger,
.hamburger::before,
.hamburger::after {
  display: block;
  width: 24px;
  height: 2px;
  background: currentColor;
  transition: transform 300ms ease;
}
.hamburger { position: relative; }
.hamburger::before { content: ''; position: absolute; top: -7px; }
.hamburger::after  { content: ''; position: absolute; bottom: -7px; }

/* Estado aberto — ícone X */
.nav-toggle[aria-expanded="true"] .hamburger { background: transparent; }
.nav-toggle[aria-expanded="true"] .hamburger::before { transform: rotate(45deg) translate(5px, 5px); }
.nav-toggle[aria-expanded="true"] .hamburger::after  { transform: rotate(-45deg) translate(5px, -5px); }

/* Utilitário visível apenas para leitores de tela */
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
  border: 0;
}

/* Desktop — nav horizontal */
@media (min-width: 768px) {
  .nav-toggle { display: none; }
  .nav { display: block; position: static; padding: 0; }
  .nav__list { display: flex; gap: 24px; }
  .nav__link { padding: 0; }
}
```

```javascript
// Toggle com suporte a teclado
const toggle = document.querySelector('.nav-toggle');
const nav = document.querySelector('#main-nav');

toggle.addEventListener('click', () => {
  const isOpen = toggle.getAttribute('aria-expanded') === 'true';
  toggle.setAttribute('aria-expanded', String(!isOpen));
  nav.classList.toggle('is-open', !isOpen);
});

// Fecha no Escape
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape' && nav.classList.contains('is-open')) {
    nav.classList.remove('is-open');
    toggle.setAttribute('aria-expanded', 'false');
    toggle.focus();
  }
});
```

---

## 8. Perguntas de Entrevista

**P1: Explique o box model CSS com `box-sizing: border-box`.**

R: Todo elemento CSS é composto de camadas de conteúdo, padding, border e margem. Com o `box-sizing: content-box` padrão, `width` se aplica apenas à área de conteúdo — padding e border são adicionados em cima, tornando o elemento renderizado mais largo do que declarado. Com `border-box`, `width` inclui padding e border, então um elemento `width: 200px` tem exatamente 200px de largura independentemente do padding ou border. `border-box` é quase sempre o que você quer e deve ser aplicado globalmente.

---

**P2: Como funciona o z-index e o que é um contexto de empilhamento?**

R: `z-index` controla a ordem de empilhamento dos elementos ao longo do eixo Z. Funciona apenas em elementos posicionados (`position` não é `static`). Um contexto de empilhamento é uma camada 3D isolada. Elementos dentro de um contexto de empilhamento são empilhados relativamente entre si e não podem ser intercalados com elementos em contextos de empilhamento irmãos. Contextos de empilhamento são criados por: elementos posicionados com z-index não-auto, transforms, opacity < 1, filters, will-change, entre outros. Bug comum: um filho com `z-index: 9999` ainda não pode aparecer acima de um contexto de empilhamento irmão se o contexto pai tem um z-index menor que aquele irmão.

---

**P3: Quando você usaria Flexbox vs Grid?**

R: Flexbox é unidimensional — ótimo para layouts de linha única ou coluna única onde o tamanho dos itens é orientado pelo conteúdo (links de nav, grupos de botões, linhas de cards). Grid é bidimensional — ótimo para layouts que exigem controle sobre ambos os eixos simultaneamente (estrutura de página, galerias de imagem, layouts de formulário onde labels e inputs devem se alinhar entre linhas). Na prática, você geralmente usa Grid para o frame da página e Flexbox dentro de cada componente.

---

**P4: O que é especificidade CSS?**

R: Especificidade determina qual regra CSS vence quando múltiplas regras miram no mesmo elemento e propriedade. É calculada como uma tupla (inline, IDs, classes/atributos/pseudos, elementos). Estilos inline vencem sobre tudo exceto `!important`. IDs vencem sobre classes. Classes vencem sobre elementos. Quando as especificidades são iguais, a última regra declarada vence (cascata). Armadilhas comuns: seletores de ID tornam muito difícil sobrescrever estilos; `!important` quebra a cascata e deve ser evitado exceto para classes utilitárias.

---

**P5: Como você centraliza uma div vertical e horizontalmente?**

R: Abordagem mais moderna: no pai, `display: flex; justify-content: center; align-items: center;`. Ou com Grid: `display: grid; place-items: center;`. A abordagem antiga para elementos com posicionamento absoluto: `position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%)` — funciona sem conhecer as dimensões do elemento.

---

**P6: O que é um contexto de empilhamento e por que o z-index às vezes não funciona?**

R: Respondido na P2. O sintoma comum: o desenvolvedor define `z-index: 9999` em um backdrop de modal e ele ainda aparece atrás de um tooltip de outro componente. A causa raiz é que o elemento pai do modal criou um contexto de empilhamento com z-index menor. Solução: mova o modal mais alto no DOM (fora do pai problemático), ou use `isolation: isolate` estrategicamente, ou achate o contexto de empilhamento do pai.

---

**P7: O que é CSS mobile-first e por que é preferido?**

R: Mobile-first significa escrever estilos base para o menor viewport, depois usar media queries `min-width` para aprimorar progressivamente para telas maiores. É preferido porque: os estilos base são mais simples (layouts mobile geralmente são mais simples), a especificidade CSS é mais fácil de gerenciar (você adiciona complexidade, não a sobrescreve) e se alinha com Progressive Enhancement — o site funciona em qualquer dispositivo por padrão e fica melhor com mais capacidade.

---

**P8: Como as animações CSS se comportam em termos de performance? Quais propriedades são "gratuitas"?**

R: O pipeline de renderização do browser é: JavaScript → Style → Layout → Paint → Composite. Mudar propriedades que afetam o layout (width, height, margin, top, left) dispara todo o pipeline — caro. Mudar propriedades apenas de paint (color, background, box-shadow) pula o layout mas ainda dispara o paint — custo moderado. Mudar apenas `transform` e `opacity` pula tanto o layout quanto o paint e vai direto para o composite (acelerado por GPU) — essencialmente gratuito a 60fps. Sempre anime transform/opacity quando possível, e use `will-change: transform` para indicar ao browser para pré-promover um elemento para sua própria camada GPU.

---

## 9. Exercícios

**Exercício 1 — Barra de nav responsiva com menu hambúrguer:**

Construa a nav descrita na seção 7 do zero. Requisitos:
- Mobile: nav oculta, botão hambúrguer visível
- Desktop (min-width: 768px): nav horizontal, sem hambúrguer
- Ícone hambúrguer deve ser desenhado apenas com CSS (sem SVG, sem imagem)
- Deve usar `aria-expanded` no botão de alternância
- Deve fechar com a tecla Escape e retornar o foco ao botão
- Deve ser navegável por teclado (Tab pelos links, sem armadilha no mobile)

---

**Exercício 2 — Layout de revista com CSS Grid:**

Construa um layout de 12 colunas com:
- Um header de largura total
- Uma seção de destaque de 3 colunas onde o primeiro item abrange 2 colunas e 2 linhas
- Um grid de cards de 4 colunas abaixo que envolve responsivamente (mín. 200px por card)
- Um rodapé de largura total

Use áreas de grid nomeadas para o layout em nível de página e `auto-fill` + `minmax` para o grid de cards. Sem Flexbox permitido (exceto dentro de cards individuais).

---

**Exercício 3 — Acordeão CSS suave sem JavaScript:**

Construa um FAQ em acordeão usando apenas HTML e CSS:
- Cada item tem uma pergunta (clicável) e uma resposta (inicialmente oculta)
- Clicar em uma pergunta revela suavemente a resposta
- Apenas uma resposta visível por vez
- Usa elementos `<details>` e `<summary>` (semântico, acessível por teclado)
- Anime a revelação com `transition` em `max-height` ou via a regra `@starting-style`

Dica: `<details>` tem um estado aberto/fechado integrado gerenciado pelo browser. Estilize `details[open] .resposta` para o estado aberto. Para animação suave, `transition` CSS em `max-height` de 0 para um valor grande funciona; alternativamente, browsers modernos suportam `transition: height allow-discrete` com `@starting-style`.

---

## 10. Leituras Complementares

- **MDN — Referência de elementos HTML**: https://developer.mozilla.org/pt-BR/docs/Web/HTML/Element
- **MDN — Referência CSS**: https://developer.mozilla.org/pt-BR/docs/Web/CSS/Reference
- **Um Guia Completo para Flexbox** (CSS-Tricks): https://css-tricks.com/snippets/css/a-guide-to-flexbox/
- **Um Guia Completo para CSS Grid** (CSS-Tricks): https://css-tricks.com/snippets/css/complete-guide-grid/
- **WCAG 2.2** (Diretrizes de Acessibilidade de Conteúdo Web): https://www.w3.org/TR/WCAG22/
- **Inclusive Components** por Heydon Pickering: https://inclusive-components.design/ — padrões de UI acessíveis
- **Every Layout** (Heydon Pickering & Andy Bell): https://every-layout.dev/ — padrões de layout CSS algorítmicos
- **web.dev — Learn CSS**: https://web.dev/learn/css/ — curso interativo cobrindo todos os aspectos do CSS
- **Smashing Magazine — Especificidade CSS**: https://www.smashingmagazine.com/2007/07/css-specificity-things-you-should-know/
- **Chrome DevTools — aba Rendering**: Habilite "Paint flashing" e "Layer borders" para visualizar camadas de composição e disparadores de paint
