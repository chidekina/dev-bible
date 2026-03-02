# Desafios de Frontend

> Desafios práticos de frontend cobrindo design de componentes React, layout CSS, JavaScript puro, otimização de performance e acessibilidade. Cada desafio reflete um cenário real que você encontrará no trabalho. Nível: iniciante ao avançado. Stack: React 18+, TypeScript, CSS moderno (nenhum framework assumido, salvo quando indicado).

---

## Exercício 1 — Feed com Scroll Infinito (Médio)

**Objetivo:** Construir um feed de posts que carrega mais itens conforme o usuário rola até o final da página — sem botão "Carregar mais".

**Requisitos:**
- Buscar a primeira página de posts ao montar o componente, a partir de uma API mock (você pode usar JSONPlaceholder ou um mock local).
- Quando o usuário rolar a menos de 200px do final, buscar a próxima página.
- Mostrar um spinner de carregamento durante a busca.
- Parar de buscar quando a API retornar uma página vazia.
- Cada card de post exibe: título, corpo (truncado em 2 linhas) e nome do autor.

**Critérios de Aceite:**
- [ ] Nenhum post duplicado aparece em re-renders.
- [ ] As buscas não se sobrepõem (sem buscas concorrentes para a mesma página).
- [ ] O spinner aparece apenas durante buscas ativas.
- [ ] Sem memory leaks — o listener de scroll é removido ao desmontar.
- [ ] Funciona corretamente quando o usuário rola muito rápido.

**Dicas:**
1. Use `IntersectionObserver` em um `<div>` sentinela no final da lista — muito mais confiável do que listeners de evento `scroll`.
2. Rastreie um ref `isFetching` (não state) para evitar requisições duplicadas em andamento.
3. Armazene posts em um `useReducer` ou state de adição exclusiva com atualizações funcionais: `setPosts(prev => [...prev, ...newItems])`.
4. Separe a lógica de busca em um hook customizado `usePaginatedFeed`.

---

## Exercício 2 — Busca com Debounce e Autocomplete (Médio)

**Objetivo:** Construir um campo de busca que consulta uma API enquanto o usuário digita, exibe sugestões em um dropdown e suporta navegação pelo teclado.

**Requisitos:**
- Aplicar debounce de 300ms na chamada à API após a última tecla pressionada.
- Exibir até 8 sugestões em uma lista dropdown abaixo do campo.
- Destacar a parte correspondente em cada sugestão (ex.: deixar em negrito a substring digitada).
- Permitir navegação pelo teclado: `ArrowDown`/`ArrowUp` para mover entre itens, `Enter` para selecionar, `Escape` para fechar.
- Clicar em uma sugestão preenche o campo e fecha o dropdown.

**Critérios de Aceite:**
- [ ] Nenhuma chamada à API dispara para teclas parciais dentro da janela de 300ms.
- [ ] Requisições anteriores são canceladas (AbortController) quando uma nova começa.
- [ ] O dropdown fecha quando o foco sai do componente.
- [ ] Atributos ARIA: `role="combobox"`, `aria-expanded`, `aria-activedescendant`, `role="listbox"`.
- [ ] Funciona inteiramente com teclado — o mouse é opcional.

**Dicas:**
1. Use `useRef` para armazenar o `AbortController` e cancelar a cada nova busca.
2. Use `useEffect` com o debounce — limpe o timer no cleanup.
3. Envolva o componente inteiro em um `<div>` com `onBlur` que verifica se `e.relatedTarget` ainda está dentro.
4. O destaque do match: divida a sugestão pela string da query, envolva o match em `<strong>`.

---

## Exercício 3 — Quadro Kanban com Drag-and-Drop (Difícil)

**Objetivo:** Implementar um quadro Kanban com três colunas (A Fazer, Em Progresso, Concluído). Os cards podem ser arrastados entre colunas e reordenados dentro de uma coluna.

**Requisitos:**
- Pelo menos 3 colunas e 5 cards iniciais.
- Arrastar um card para uma coluna diferente para movê-lo.
- Arrastar um card dentro de uma coluna para reordenar.
- Mostrar um placeholder onde o card será solto.
- Persistir o estado do quadro no `localStorage` para sobreviver a uma atualização de página.

**Critérios de Aceite:**
- [ ] Cards renderizam na ordem correta após reordenação.
- [ ] Sem inconsistência de state entre colunas após um drag entre colunas.
- [ ] O elemento placeholder tem a mesma altura do card arrastado.
- [ ] Funciona em telas touch (pointer events, não apenas mouse events).
- [ ] O estado do quadro é carregado do `localStorage` ao montar.

**Dicas:**
1. Use a API de Drag and Drop do HTML5 (`draggable`, `onDragStart`, `onDragOver`, `onDrop`) para uma solução sem biblioteca.
2. Alternativamente, a biblioteca `@dnd-kit/core` lida com pointer events, acessibilidade e touch automaticamente.
3. Armazene o state como `{ columns: { [id]: { title, cardIds[] } }, cards: { [id]: { title, body } } }` — formato normalizado.
4. Use `useReducer` para transições complexas de state do quadro (actions MOVE_CARD, REORDER_CARD).

---

## Exercício 4 — Layout Holy Grail com CSS (Fácil)

**Objetivo:** Implementar o clássico layout "holy grail" usando apenas CSS — sem JavaScript, sem frameworks, sem biblioteca de grid.

**Requisitos:**
- Layout de página inteira: header (altura fixa), seção central com três colunas (sidebar esquerda, conteúdo principal, sidebar direita), footer.
- A área de conteúdo principal ocupa todo o espaço horizontal restante.
- Ambas as sidebars têm larguras fixas (200px cada).
- A seção central preenche todo o espaço vertical entre o header e o footer.
- Responsivo: em telas abaixo de 768px, empilhar as três colunas verticalmente (sidebar → conteúdo → sidebar).

**Critérios de Aceite:**
- [ ] Layout obtido com CSS Grid ou Flexbox (sua escolha — resolva com ambos, separadamente).
- [ ] Nenhum JavaScript utilizado.
- [ ] O footer fica colado ao fundo mesmo quando o conteúdo é curto.
- [ ] O breakpoint responsivo colapsa as colunas corretamente.
- [ ] Funciona no Chrome, Firefox e Safari (sem prefixos de fornecedor para CSS moderno).

**Dicas:**
1. Solução Grid: `grid-template-areas` com `"header header header"`, `"sidebar main aside"`, `"footer footer footer"`.
2. Solução Flexbox: envolva as três colunas em uma flex row; dê ao main `flex: 1`; faça o container externo uma coluna flex com `min-height: 100vh`.
3. Para footer fixo sem JS: `body { display: flex; flex-direction: column; min-height: 100vh }` e `footer { margin-top: auto }`.

---

## Exercício 5 — Modal Dialog Acessível (Médio)

**Objetivo:** Construir um componente de modal dialog reutilizável que seja totalmente acessível e siga o padrão ARIA dialog.

**Requisitos:**
- Ativado por um botão "Abrir Modal".
- Contém um título, conteúdo no corpo e botão de fechar.
- Pressionar `Escape` fecha o modal.
- Quando o modal abre, o foco move para o primeiro elemento focável dentro dele.
- Quando o modal fecha, o foco retorna ao elemento que o abriu.
- O foco fica preso dentro do modal enquanto está aberto — Tab não pode sair.
- O conteúdo de fundo fica inerte enquanto o modal está aberto.

**Critérios de Aceite:**
- [ ] `role="dialog"`, `aria-modal="true"`, `aria-labelledby` apontando para o título.
- [ ] O focus trap funciona para todos os elementos focáveis: buttons, inputs, links, select, textarea.
- [ ] O leitor de tela anuncia o título do modal quando ele abre.
- [ ] `Shift+Tab` move o foco em sentido inverso de forma circular.
- [ ] `document.body` recebe `aria-hidden="true"` (ou atributo `inert`) enquanto o modal está aberto.

**Dicas:**
1. Colete elementos focáveis com `querySelectorAll('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])')`.
2. Adicione um listener `keydown` no modal: intercepte `Tab` e `Shift+Tab`, percorra a lista de focáveis manualmente.
3. Use `useRef` para guardar o elemento ativo (`document.activeElement`) antes de abrir, para restaurar o foco ao fechar.
4. A abordagem moderna: o elemento HTML `<dialog>` com `showModal()` lida com a maior parte disso nativamente — implemente com `<dialog>` e depois sem ele para maior aprendizado.

---

## Exercício 6 — Lista Longa Virtualizada (Difícil)

**Objetivo:** Renderizar uma lista de 100.000 itens sem travar o navegador, usando renderização janelada/virtual.

**Requisitos:**
- A lista tem 100.000 linhas, cada uma com 50px de altura.
- Apenas as linhas visíveis na viewport (mais um pequeno buffer de overscan) estão no DOM.
- A rolagem deve ser suave — sem travamentos.
- Cada linha exibe: índice, uma amostra de cor aleatória e um rótulo.
- O número total de nós DOM deve permanecer abaixo de 100 o tempo todo.

**Critérios de Aceite:**
- [ ] A renderização inicial leva menos de 100ms (meça com Lighthouse ou React DevTools).
- [ ] Performance de scroll: sem frames perdidos visíveis em um dispositivo de médio desempenho.
- [ ] O thumb do scrollbar reflete a altura total de 100.000 itens.
- [ ] As linhas são recicladas (não desmontadas/remontadas a cada evento de scroll).

**Dicas:**
1. Use `react-virtual` ou `@tanstack/react-virtual` para a lógica de windowing — é o padrão da indústria para listas customizadas.
2. Abordagem manual: coloque um container externo alto (`height = count * rowHeight`). Um container interno é posicionado `absolute` em `top = startIndex * rowHeight`. Renderize apenas as linhas de `startIndex` a `endIndex`.
3. Escute `onScroll` no container externo para atualizar `scrollTop`, derive `startIndex = Math.floor(scrollTop / rowHeight)`.
4. Adicione um overscan de 3-5 linhas acima e abaixo da janela visível para evitar flashes em branco.

---

## Exercício 7 — Formulário com Validação Complexa (Médio)

**Objetivo:** Construir um formulário de cadastro em múltiplas etapas (3 etapas) com validação em tempo real e mensagens de erro.

**Requisitos:**
- Etapa 1: Nome (obrigatório, mín. 2 chars), Email (formato válido), Senha (mín. 8 chars, deve conter número e caractere especial).
- Etapa 2: Data de Nascimento (deve ter 18+), País (selecionar da lista), Telefone (opcional, mas se fornecido deve corresponder ao formato `+[código do país] [número]`).
- Etapa 3: Tela de revisão mostrando todos os valores inseridos antes do envio final.
- Erros aparecem inline, abaixo de cada campo, no blur.
- O botão "Próximo" fica desabilitado até que todos os campos obrigatórios da etapa atual sejam válidos.

**Critérios de Aceite:**
- [ ] A validação dispara no blur, não a cada tecla.
- [ ] O indicador de força de senha (fraca/média/forte) atualiza a cada tecla.
- [ ] A navegação de volta preserva os valores inseridos nas etapas anteriores.
- [ ] `aria-describedby` vincula cada input à sua mensagem de erro.
- [ ] O envio mostra um estado de sucesso (sem backend necessário — simule com um delay de 1 segundo).

**Dicas:**
1. Use `react-hook-form` para gerenciamento do state do formulário — minimiza re-renders e integra com Zod ou Yup para validação de schema.
2. Armazene todos os valores das etapas em um contexto compartilhado ou state levantado para que a navegação de volta os restaure.
3. Força de senha: pontuação baseada em comprimento, maiúscula, número, caractere especial — some tudo e mapeie para fraca/média/forte.
4. Mapa de prefixo de telefone por país: mantenha um pequeno objeto JSON `{ US: "+1", BR: "+55", ... }` para validar o prefixo.

---

## Exercício 8 — Toggle de Dark Mode com Preferência do Sistema (Fácil)

**Objetivo:** Implementar um toggle de modo escuro/claro que respeita a preferência do sistema operacional e persiste a escolha entre sessões.

**Requisitos:**
- Na primeira visita, ler a media query `prefers-color-scheme` para definir o modo inicial.
- Um botão toggle alterna entre claro e escuro.
- O modo escolhido é salvo no `localStorage` e restaurado ao recarregar.
- O modo é aplicado via classe CSS no `<body>` (ex.: `.dark`). Todas as cores são custom properties CSS.
- O ícone do botão toggle muda (sol/lua) e tem um label acessível adequado.

**Critérios de Aceite:**
- [ ] Sem flash do tema errado no carregamento da página (dica: defina a classe antes do React hidratar).
- [ ] O valor do `localStorage` tem precedência sobre a media query nas visitas subsequentes.
- [ ] `aria-label` no botão toggle reflete a ação atual ("Mudar para modo escuro" ou "Mudar para modo claro").
- [ ] Pelo menos 5 custom properties (`--bg`, `--text`, `--accent`, `--border`, `--card-bg`) definidas para ambos os temas.

**Dicas:**
1. Adicione um `<script>` inline no `<head>` (antes do bundle React) para ler o `localStorage` e definir a classe de forma síncrona — isso evita o flash de tema.
2. No React: use `useEffect` para sincronizar a classe e o `localStorage` sempre que o state do tema mudar.
3. `window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', ...)` permite reagir a mudanças do sistema operacional em tempo real.

---

## Exercício 9 — Otimizando um Dashboard React Lento (Médio)

**Objetivo:** Você herda um componente de dashboard que é extremamente lento para interagir. Faça o profiling e aplique as otimizações corretas.

**Código inicial (intencionalmente problemático):**
```tsx
// Dashboard.tsx — NÃO mantenha este código como está
export function Dashboard({ users }: { users: User[] }) {
  const [filter, setFilter] = useState('');
  const [sortKey, setSortKey] = useState<keyof User>('name');

  const filtered = users
    .filter(u => u.name.toLowerCase().includes(filter.toLowerCase()))
    .sort((a, b) => a[sortKey] > b[sortKey] ? 1 : -1);

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <select value={sortKey} onChange={e => setSortKey(e.target.value as keyof User)}>
        <option value="name">Name</option>
        <option value="email">Email</option>
      </select>
      {filtered.map(u => <UserCard key={u.id} user={u} onDelete={() => deleteUser(u.id)} />)}
    </>
  );
}
```

**Requisitos:**
- O array `users` contém 10.000 itens.
- `UserCard` é um componente complexo (assuma que faz uma renderização pesada).
- Digitar no campo de filtro faz com que cada `UserCard` seja re-renderizado a cada tecla.
- Corrija a performance sem alterar o comportamento observável.

**Critérios de Aceite:**
- [ ] `UserCard` só re-renderiza quando sua própria prop `user` muda.
- [ ] O cálculo de filtro + ordenação não roda a cada render não relacionado a mudanças de filtro/ordenação.
- [ ] O callback `onDelete` não faz todos os cards re-renderizarem quando o state muda.
- [ ] Faça o profiling com React DevTools antes e depois — mostre pelo menos 5x de melhoria no tempo de renderização.

**Dicas:**
1. Aplique `useMemo` no array `filtered` — recompute apenas quando `users`, `filter` ou `sortKey` mudarem.
2. Envolva `UserCard` com `React.memo` para pular re-renders quando as props não mudaram.
3. Aplique `useCallback` no handler `onDelete` — funções arrow inline criam uma nova referência a cada render, anulando o `React.memo`.
4. Considere se a ordenação deve acontecer dentro ou fora do `useMemo` e qual deve ser seu array de dependências.

---

## Exercício 10 — Animação de Partículas com Canvas (Difícil)

**Objetivo:** Renderizar 500 partículas animadas em um elemento `<canvas>` que se movem e ricocheteiam nas bordas, usando `requestAnimationFrame`.

**Requisitos:**
- Cada partícula tem posição, velocidade, tamanho (2–8px) e cor aleatórios.
- As partículas ricocheteiam em todas as quatro bordas do canvas.
- O canvas redimensiona para preencher a janela; as partículas reescalam de acordo.
- Um botão "Pausar/Retomar" para e inicia a animação sem resetar as posições das partículas.
- Mostrar o FPS atual no canto superior esquerdo.

**Critérios de Aceite:**
- [ ] 60 FPS consistentes em um laptop moderno (meça com o contador de FPS ou Chrome DevTools).
- [ ] Nenhuma partícula escapa dos limites do canvas.
- [ ] O handler de resize não cria loops de animação adicionais.
- [ ] Todas as operações da Canvas API usam o contexto 2D corretamente (sem WebGL necessário).

**Dicas:**
1. Armazene as partículas em um array simples de objetos `{ x, y, vx, vy, r, color }` — mantenha-os fora do state do React.
2. Use um `useRef` para o canvas e o `animationFrameId` para evitar disparar re-renders.
3. Cálculo de FPS: `fps = 1000 / (currentTimestamp - lastTimestamp)`. Suavize com uma média móvel dos últimos 60 frames.
4. No resize: use um `ResizeObserver` no container do canvas, atualize os atributos `width`/`height` do canvas (não as dimensões CSS), e limite as posições das partículas aos novos limites.

---

## Exercício 11 — Hook useLocalStorage Customizado (Fácil)

**Objetivo:** Escrever um hook `useLocalStorage<T>` que funciona como `useState`, mas persiste o valor no `localStorage`.

**Requisitos:**
- Assinatura: `function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void]`
- Ao montar, lê o valor armazenado. Usa `initialValue` como fallback se nada estiver armazenado ou se a análise do JSON falhar.
- Atualiza o `localStorage` sempre que o valor é definido.
- Múltiplas abas devem permanecer sincronizadas via evento `storage`.

**Critérios de Aceite:**
- [ ] Funciona com qualquer tipo `T` serializável em JSON (string, number, object, array).
- [ ] Lida graciosamente quando `localStorage.getItem` retorna `null`.
- [ ] Lida com JSON corrompido (captura erros de análise, usa `initialValue` como fallback).
- [ ] A sincronização entre abas atualiza o state sem recarregar a página.
- [ ] O hook é totalmente tipado — sem `any` na implementação.

**Dicas:**
1. Inicialize o state com uma função inicializadora lazy no `useState` para evitar rodar `localStorage.getItem` a cada render.
2. Em `setValue`, chame tanto `setStoredValue` quanto `localStorage.setItem`.
3. `window.addEventListener('storage', handler)` dispara quando outra aba define a mesma chave. Use `useEffect` para adicionar e remover este listener.
4. O evento `storage` NÃO dispara na mesma aba que fez a mudança.

---

## Exercício 12 — Galeria de Imagens Responsiva com CSS Grid (Fácil)

**Objetivo:** Construir uma galeria de imagens usando CSS Grid que cria um layout estilo masonry sem JavaScript.

**Requisitos:**
- Exibir 20 imagens de alturas variadas.
- Layout: 4 colunas no desktop, 2 em tablet (< 768px), 1 em mobile (< 480px).
- As imagens preenchem a largura da coluna, mantêm a proporção e têm espaçamento consistente.
- Ao passar o mouse, cada imagem mostra um overlay com o título e um botão "Ver".
- O overlay de hover faz a transição suavemente (sem pulo).

**Critérios de Aceite:**
- [ ] Layout usa `display: grid` — sem flexbox para o layout principal.
- [ ] Masonry verdadeiro: itens preenchem lacunas verticais (CSS `grid-template-rows: masonry` ou fallback com `column-count`).
- [ ] Overlay de hover: transição de opacity ou transform, sem layout shift.
- [ ] Imagens são carregadas de forma lazy (`loading="lazy"` no `<img>`).
- [ ] A galeria é navegável pelo teclado — Tab percorre as imagens, Enter ativa o overlay.

**Dicas:**
1. CSS masonry (`grid-template-rows: masonry`) é suportado no Firefox por trás de uma flag; use `columns: 4` como fallback com maior suporte.
2. O overlay de hover: posicione o wrapper da imagem como `relative`, o overlay como `absolute inset-0`. Use `opacity: 0` → `opacity: 1` no `:hover` com `transition: opacity 200ms`.
3. Para teclado: adicione `tabindex="0"` a cada item da galeria e trate `:focus-within` da mesma forma que `:hover` no CSS.
