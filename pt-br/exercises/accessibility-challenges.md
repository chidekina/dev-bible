# Desafios de Acessibilidade

> Exercícios práticos de acessibilidade cobrindo ARIA, navegação por teclado, leitores de tela, contraste de cores e testes automatizados. Cada exercício aborda uma barreira real de acessibilidade. Stack assumida: HTML5, React, TypeScript, jest-axe, Playwright. Nível alvo: iniciante a intermediário.

---

## Exercício 1 — Auditar uma Página com axe-core (Fácil)

**Cenário:** Execute uma auditoria automatizada de acessibilidade em uma página existente e produza um relatório de remediação priorizado.

**Requisitos:**
- Instale `axe-core` e execute-o contra uma página HTML alvo (o `/dashboard` do seu app ou uma página estática fornecida).
- Capture todas as violações com seus critérios WCAG, nível de impacto e elementos afetados.
- Produza um relatório Markdown listando violações agrupadas por impacto: `critical`, `serious`, `moderate`, `minor`.
- Corrija todas as violações `critical` antes de avançar.
- Execute novamente a auditoria após as correções para verificar que zero violações críticas permanecem.

**Critérios de Aceite:**
- [ ] O script de auditoria roda com `node scripts/audit-a11y.js <url>` e gera um arquivo Markdown.
- [ ] O relatório inclui: id da violação, critério WCAG, impacto, número de nós afetados e sugestão de correção.
- [ ] Todas as violações `critical` (ex: alt text faltando, labels de formulário, falhas de contraste de cor) são resolvidas.
- [ ] Executar novamente após as correções produz zero violações `critical` ou `serious`.
- [ ] O script de auditoria é adicionado ao `package.json` como script `"a11y:audit"`.

**Dicas:**
1. axe-core com Playwright: `const { AxeBuilder } = require('@axe-core/playwright'); const results = await new AxeBuilder({ page }).analyze();`.
2. Filtre por impacto: `results.violations.filter(v => v.impact === 'critical')`.
3. As violações críticas mais comuns: `alt` faltando em `<img>`, inputs de formulário sem `<label>`, botões sem nome acessível, contraste de cor insuficiente.
4. `axe-core` captura apenas ~30–40% dos problemas de acessibilidade — auditorias automatizadas são um ponto de partida, não uma auditoria completa.

---

## Exercício 2 — Corrigir Hierarquia de Headings (Fácil)

**Cenário:** Uma página HTML tem uma hierarquia de headings quebrada — headings pulam níveis e são usados para estilização visual em vez de estrutura do documento. Corrija para produzir um outline lógico.

**HTML fornecido (hierarquia quebrada):**
```html
<h1>Dashboard</h1>
<h3>Pedidos Recentes</h3>    <!-- pula h2 -->
<h5>Pedido #1042</h5>        <!-- pula h4 -->
<h2>Configurações da Conta</h2>
<h4>Preferências de Email</h4> <!-- pula h3 -->
<h2>Ajuda</h2>
<h4>Contato com Suporte</h4>   <!-- pula h3 -->
```

**Requisitos:**
- Corrija a hierarquia de headings para que siga um outline lógico (sem níveis pulados).
- Se um heading foi escolhido por estilização visual (tamanho de fonte), substitua a tag de heading por um elemento semântico + classe CSS.
- Verifique a estrutura corrigida com a regra `heading-order` do axe-core.
- Escreva um breve outline do documento (lista com marcadores) representando a estrutura pretendida da página.

**Critérios de Aceite:**
- [ ] Sem violações do `axe` para a regra `heading-order` após as correções.
- [ ] Headings não pulam níveis (ex: `h1 → h2 → h3` é válido; `h1 → h3` não é).
- [ ] Se um `<h5>` foi usado apenas para tamanho visual, é substituído por `<p class="order-label">` ou similar.
- [ ] O outline do documento da página (visível nas DevTools do browser → Accessibility → Headings) é lógico.
- [ ] Corrigir headings não muda a aparência visual (CSS cuida dos tamanhos de fonte independentemente).

**Dicas:**
1. O nível de um heading deve refletir sua posição na hierarquia do documento, não seu tamanho visual.
2. Para verificar o outline: abra Chrome DevTools → Elements → painel Accessibility. Ou use a extensão headingsMap do browser.
3. Se um `<h3>` é usado apenas para um estilo de subtítulo em negrito: mude para `<p class="subheading">` e adicione `font-weight: 600; font-size: 1.1rem` no CSS.
4. A regra: cada nível deve aparecer apenas sob seu nível pai. `h1 → h2 → h3` é correto. `h1 → h3` não é.

---

## Exercício 3 — Adicionar Roles ARIA a um Dropdown Customizado (Médio)

**Cenário:** Um menu dropdown construído customizadamente usa elementos `<div>` e `<span>` sem atributos ARIA. Leitores de tela não conseguem identificá-lo como um listbox. Adicione os roles e atributos ARIA corretos.

**HTML fornecido (inacessível):**
```html
<div class="dropdown" onclick="toggleDropdown()">
  <span class="dropdown-trigger">Selecione um país</span>
  <div class="dropdown-menu" style="display: none">
    <div class="dropdown-item" onclick="select('br')">Brasil</div>
    <div class="dropdown-item" onclick="select('pt')">Portugal</div>
    <div class="dropdown-item" onclick="select('us')">Estados Unidos</div>
  </div>
</div>
```

**Requisitos:**
- Adicione roles ARIA: `combobox`, `listbox`, `option`.
- O elemento trigger deve ter `aria-haspopup="listbox"`, `aria-expanded` (alternado ao abrir/fechar) e `aria-controls` apontando para o id do listbox.
- Cada opção deve ter `role="option"` e `aria-selected`.
- A opção selecionada deve atualizar `aria-selected="true"`; todas as outras `false`.
- Teclado: `Enter`/`Space` abre o dropdown; `ArrowDown`/`ArrowUp` move o foco; `Enter` seleciona; `Escape` fecha.

**Critérios de Aceite:**
- [ ] `axe-core` não reporta violações no dropdown após as correções.
- [ ] Um leitor de tela anuncia o componente como "Selecione um país, combobox, recolhido" ao receber foco.
- [ ] Navegação por teclado funciona conforme especificado sem mouse.
- [ ] `aria-expanded` é `"true"` quando aberto, `"false"` quando fechado.
- [ ] A opção com foco atual tem `aria-selected="true"` e é anunciada pelos leitores de tela.

**Dicas:**
1. Padrão ARIA: siga o Guia de Práticas de Autoria WAI-ARIA (APG) para o padrão combobox — `role="combobox"` no trigger, `role="listbox"` no menu, `role="option"` nos itens.
2. `aria-controls="meu-listbox"` no trigger + `id="meu-listbox"` no menu dropdown.
3. Gerenciamento de foco: use `tabindex="0"` no trigger e `tabindex="-1"` nas opções. Mova o foco programaticamente com `element.focus()` em eventos de tecla de seta.
4. `aria-activedescendant` no combobox pode apontar para o id da opção com foco atual — evita mover o foco DOM para o listbox.

---

## Exercício 4 — Construir uma Lista de Tabs Navegável por Teclado (Médio)

**Cenário:** Construa uma interface de tabs completamente navegável por teclado de acordo com o padrão Tabs do WAI-ARIA.

**Requisitos:**
- Três tabs: "Perfil", "Configurações", "Notificações".
- Comportamento de teclado: `Tab` move o foco para a lista de tabs; `ArrowRight`/`ArrowLeft` circula entre tabs e as ativa; `Home` vai para a primeira tab; `End` vai para a última.
- Tab ativa tem `aria-selected="true"` e `tabindex="0"`; tabs inativas têm `aria-selected="false"` e `tabindex="-1"`.
- Cada painel de tab é associado à sua tab via `aria-controls` / `aria-labelledby`.
- Painéis de tab inativos têm o atributo `hidden` (não apenas `display: none`).

**Critérios de Aceite:**
- [ ] `axe-core` não reporta violações.
- [ ] Pressionar `ArrowRight` na tab "Perfil" move para "Configurações" e mostra seu painel.
- [ ] Pressionar `End` move para "Notificações" independente da posição atual.
- [ ] Cada `tabpanel` tem `role="tabpanel"`, `id` e `aria-labelledby` apontando para o id da sua tab.
- [ ] Leitor de tela anuncia: "Perfil, tab, 1 de 3, selecionada" ao receber foco.

**Dicas:**
1. Atributos de role para tab: `role="tablist"` no container, `role="tab"` em cada botão de tab, `role="tabpanel"` em cada painel.
2. Use elementos `<button>` reais para as tabs — eles têm suporte nativo a teclado e semântica.
3. `tabindex` rotativo: apenas a tab ativa tem `tabindex="0"`; todas as outras têm `tabindex="-1"`. Atualize ao pressionar teclas de seta.
4. `aria-labelledby` no painel: `<div role="tabpanel" id="painel-perfil" aria-labelledby="tab-perfil">`.

---

## Exercício 5 — Implementar Armadilha de Foco em um Modal (Médio)

**Cenário:** Seu diálogo modal permite que o foco do teclado escape para fora do modal. Implemente uma armadilha de foco para que Tab e Shift+Tab circulem apenas dentro do modal enquanto ele estiver aberto.

**Requisitos:**
- Quando o modal abre, o foco move para o primeiro elemento focalizável dentro dele.
- `Tab` circula pelos elementos focalizáveis dentro do modal e vai do último para o primeiro.
- `Shift+Tab` circula em sentido inverso e vai do primeiro para o último.
- Quando o modal fecha, o foco retorna ao elemento que disparou o modal.
- `Escape` fecha o modal.

**Critérios de Aceite:**
- [ ] Tab após o último elemento focalizável dentro do modal move o foco para o primeiro (não para o body do documento).
- [ ] O botão que disparou o modal recebe foco quando o modal é dispensado.
- [ ] Elementos focalizáveis são descobertos dinamicamente: `a[href], button:not(:disabled), input, select, textarea, [tabindex]:not([tabindex="-1"])`.
- [ ] A armadilha de foco ativa ao abrir e desativa ao fechar — não deve afetar outras partes da página.
- [ ] Regra `dialog-name` do `axe-core` passa (modal tem `aria-labelledby` apontando para seu título).

**Dicas:**
1. Colete elementos focalizáveis: `const focusable = modal.querySelectorAll('a[href], button:not(:disabled), input, ...')`.
2. Escute `keydown` no modal: se `Tab` pressionado e `document.activeElement === lastFocusable`, chame `event.preventDefault(); firstFocusable.focus()`.
3. Armazene o trigger: `const trigger = document.activeElement as HTMLElement` antes de abrir. Restaure: `trigger.focus()` ao fechar.
4. Use `aria-modal="true"` no diálogo — isso sinaliza para alguns leitores de tela que o conteúdo fora do modal está inativo.

---

## Exercício 6 — Adicionar Links de Skip Navigation (Fácil)

**Cenário:** Uma página com uma barra de navegação longa exige que usuários de teclado pressionem Tab em todos os links de navegação antes de chegar ao conteúdo principal. Adicione links de skip navigation.

**Requisitos:**
- Adicione um link "Pular para o conteúdo principal" como o primeiro elemento focalizável da página.
- O link é visualmente oculto até receber foco (mostrado como overlay visível quando focado).
- Clicar/ativar o link move o foco para `<main id="main-content">`.
- Adicione links de skip adicionais para: "Pular para a navegação", "Pular para o rodapé".
- Todos os links de skip aparecem na ordem de tab correta e são anunciados por leitores de tela.

**Critérios de Aceite:**
- [ ] "Pular para o conteúdo principal" é o primeiro ponto de Tab em cada página.
- [ ] O link é visualmente invisível quando não focado e fica visível ao receber foco (sem `display:none` — isso o remove da ordem de tab).
- [ ] Ativar o link move o foco do teclado para `<main>` (não apenas rola até ele).
- [ ] `<main>` tem `tabindex="-1"` para que possa receber foco programático.
- [ ] Leitor de tela anuncia "Pular para o conteúdo principal, link" no primeiro pressionamento de Tab.

**Dicas:**
1. CSS para visualmente oculto mas focalizável:
   ```css
   .skip-link { position: absolute; top: -40px; left: 0; }
   .skip-link:focus { top: 0; }
   ```
2. `<main id="main-content" tabindex="-1">` — `tabindex="-1"` permite foco programático mas exclui da sequência de tab.
3. O link: `<a href="#main-content" class="skip-link">Pular para o conteúdo principal</a>`.
4. Para SPAs: re-implemente links de skip como componentes React que gerenciam foco nas mudanças de rota.

---

## Exercício 7 — Corrigir Problemas de Contraste de Cor (Fácil)

**Cenário:** Um arquivo CSS fornecido contém várias combinações de cores que reprovam as proporções de contraste WCAG 2.1 AA. Encontre e corrija todas as combinações com falha.

**CSS fornecido (contém falhas de contraste):**
```css
.primary-button { background: #4A90D9; color: #FFFFFF; } /* verificar */
.secondary-button { background: #F5F5F5; color: #AAAAAA; } /* verificar */
.error-text { color: #FF6B6B; background: #FFFFFF; } /* verificar */
.badge { background: #FFD700; color: #FFFFFF; } /* verificar */
.muted-label { color: #CCCCCC; background: #FFFFFF; } /* verificar */
```

**Requisitos:**
- Calcule a proporção de contraste para cada par de cores (use a fórmula WCAG ou uma ferramenta online).
- WCAG 2.1 AA exige: ≥ 4,5:1 para texto normal, ≥ 3:1 para texto grande (18px+ ou 14px negrito+).
- Corrija todos os pares com falha ajustando a cor de frente ou de fundo.
- Garanta que as cores corrigidas ainda correspondam à intenção do design (não mude um botão azul para vermelho).
- Documente as proporções de contraste antes e depois em um comentário acima de cada regra.

**Critérios de Aceite:**
- [ ] Todos os pares de cores atendem aos requisitos de proporção de contraste WCAG 2.1 AA após as correções.
- [ ] Comentários mostram o formato: `/* contraste: 3,2:1 → FALHA | corrigido: 7,1:1 → OK */`.
- [ ] Cores corrigidas são perceptualmente similares às originais (matiz preservado, apenas mais escuro/claro).
- [ ] A correção não introduz novas falhas de contraste em componentes relacionados.
- [ ] Use o WebAIM Contrast Checker ou pacote npm `color-contrast` para verificar as proporções.

**Dicas:**
1. Pares com falha: `.secondary-button` (#AAAAAA em #F5F5F5 ≈ 2,3:1 — FALHA), `.badge` (branco no dourado ≈ 1,9:1 — FALHA), `.muted-label` (#CCC no branco ≈ 1,6:1 — FALHA).
2. Corrija `.secondary-button`: escureça o texto para `#767676` (mínimo para 4,5:1 no branco). Em #F5F5F5, você precisa de mais escuro — tente `#595959`.
3. Corrija `.badge`: mude o texto de branco para escuro — `#5C4A00` (marrom escuro) no dourado alcança ~7:1.
4. npm `color-contrast`: `const { hex } = require('color-contrast'); hex('#AAAAAA', '#F5F5F5')` retorna a proporção.

---

## Exercício 8 — Escrever Testes jest-axe para um Componente React (Médio)

**Cenário:** Escreva testes de acessibilidade automatizados para um componente React `LoginForm` usando `jest-axe` (ou `vitest-axe`).

**Requisitos:**
- O `LoginForm` renderiza: input de email, input de senha, checkbox "Lembrar de mim" e um botão de envio.
- Escreva testes jest-axe para: estado padrão, estado com erro de validação (campo email mostra erro), estado de carregamento (botão desabilitado).
- Cada teste renderiza o componente e chama `axe(container)` — verifique com `toHaveNoViolations()`.
- Além do axe: escreva um teste separado que verifica manualmente associações de label (cada input tem um `<label>` corretamente associado).

**Critérios de Aceite:**
- [ ] `expect(await axe(container)).toHaveNoViolations()` passa para todos os três estados.
- [ ] O teste para o estado de erro renderiza uma mensagem de erro e verifica que `aria-describedby` no input de email aponta para o id do elemento de erro.
- [ ] O teste de associação de `<label>` usa `getByLabelText('Endereço de email')` — passa apenas se o label está corretamente associado.
- [ ] Testes usam `@testing-library/react` para renderização — sem manipulação direta do DOM.
- [ ] Todos os três casos de teste rodam em menos de 1 segundo combinados.

**Dicas:**
1. Setup: `import { axe, toHaveNoViolations } from 'jest-axe'; expect.extend(toHaveNoViolations);`.
2. Renderize e teste: `const { container } = render(<LoginForm />); expect(await axe(container)).toHaveNoViolations();`.
3. Estado de erro: renderize com `<LoginForm errors={{ email: 'Por favor, insira um email válido' }} />`. O input deve ter `aria-describedby="email-error"` e o span de erro `id="email-error"`.
4. Associação de label: `getByLabelText('Endereço de email')` usa a relação `for`/`id` ou `aria-label`. Se lançar, o label não está corretamente associado.

---

## Exercício 9 — Tornar um Formulário Completamente Acessível (Médio)

**Cenário:** Um formulário de cadastro existente tem vários problemas de acessibilidade: inputs sem label, sem mensagens de erro, sem indicadores de campo obrigatório e sem região live para feedback de validação assíncrona.

**Requisitos:**
- Todo input deve ter um `<label>` visível (não apenas texto placeholder).
- Campos obrigatórios devem ser indicados com `aria-required="true"` e um `*` visível com uma legenda explicando `* = obrigatório`.
- Erros de validação devem ser associados ao seu input via `aria-describedby`.
- Validação assíncrona (ex: "email já cadastrado") deve usar `aria-live="polite"` para anunciar o resultado aos leitores de tela.
- Ao enviar o formulário com erros, o foco deve mover para o primeiro campo com erro.

**Critérios de Aceite:**
- [ ] Todo input tem um `<label>` associado — `getByLabelText('Nome')` funciona nos testes.
- [ ] `aria-required="true"` está em todos os inputs obrigatórios; `aria-describedby` vincula inputs às suas mensagens de erro.
- [ ] Enviar com erros anuncia "Por favor, corrija 2 erros" via região `aria-live`.
- [ ] Após validação assíncrona de email, "Endereço de email já cadastrado" é anunciado sem recarregar a página.
- [ ] O formulário passa em todas as regras do axe-core no estado de erro (o estado mais difícil de acertar).

**Dicas:**
1. Label visível: `<label for="email">Endereço de email <span aria-hidden="true">*</span></label><input id="email" aria-required="true" />`.
2. Associação de erro: `<input id="email" aria-describedby="email-error" aria-invalid="true" />` + `<span id="email-error" role="alert">Email inválido</span>`.
3. Região live para assíncrono: `<div aria-live="polite" aria-atomic="true" id="async-feedback"></div>`. Atualize seu `textContent` via JS — leitores de tela anunciam a mudança.
4. Foco no erro: `document.querySelector('[aria-invalid="true"]')?.focus()` após envio com falha.

---

## Exercício 10 — Checklist de Walkthrough com Leitor de Tela (Fácil)

**Cenário:** Teste manualmente uma página com um leitor de tela para descobrir problemas que ferramentas automatizadas não conseguem detectar. Documente descobertas e correções.

**Requisitos:**
- Escolha um leitor de tela: NVDA (Windows, gratuito), VoiceOver (macOS/iOS, nativo), ou TalkBack (Android).
- Percorra a página alvo usando apenas teclado e leitor de tela — sem mouse.
- Complete o checklist abaixo, marcando Passou/Falhou para cada item com observações.
- Corrija pelo menos 3 problemas encontrados durante o walkthrough.
- Teste novamente após as correções para confirmar a resolução.

**Checklist:**
```
[ ] Título da página é significativo e único (anunciado ao carregar)
[ ] Regiões landmark: header, nav, main, footer todas presentes
[ ] Todas as imagens têm alt text significativo (imagens decorativas têm alt="")
[ ] Links têm texto descritivo (sem "clique aqui" ou "leia mais" sem contexto)
[ ] Botões são distinguíveis de links (botões realizam ações, links navegam)
[ ] Inputs de formulário todos têm labels associados
[ ] Mensagens de erro são anunciadas automaticamente (não apenas mostradas visualmente)
[ ] Widgets customizados (dropdowns, modais, tabs) são operáveis por teclado
[ ] Nenhum conteúdo é transmitido apenas pela cor
[ ] Ordem de leitura no leitor de tela corresponde à ordem visual
```

**Critérios de Aceite:**
- [ ] Todos os 10 itens do checklist são avaliados (Passou, Falhou ou N/A com justificativa).
- [ ] Pelo menos 3 itens com Falha têm uma correção documentada com trechos de código antes/depois.
- [ ] Após as correções, esses 3 itens passam ao ser re-testados com o leitor de tela.
- [ ] Um breve resumo (5–10 frases) descreve a experiência geral da perspectiva de um usuário de leitor de tela.
- [ ] O walkthrough é feito em uma página que você construiu — não em um site de terceiros.

**Dicas:**
1. Atalhos do NVDA: `H` para pular entre headings, `F` para campos de formulário, `B` para botões, `L` para listas, `D` para landmarks, `Insert+F7` para lista completa de links.
2. VoiceOver no macOS: `Ctrl+Option+U` abre o Web Rotor — navegue por headings, links, controles de formulário.
3. "Nenhum conteúdo transmitido apenas pela cor": verifique que estados de erro usam um ícone ou texto, não apenas uma borda vermelha.
4. Ordem de leitura: leitores de tela seguem a ordem do DOM, não a ordem visual. `order` do CSS e `position: absolute` podem causar discrepâncias entre ordem do DOM e visual.
