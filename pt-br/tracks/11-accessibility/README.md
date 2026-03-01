# Track 11 — Acessibilidade

Construa interfaces que funcionam para todos, incluindo usuários que dependem de screen readers, keyboard navigation ou tecnologia assistiva. Acessibilidade é um requisito legal em muitas jurisdições e um indicador de qualidade de engenharia.

**Tempo estimado:** 1–2 semanas

---

## Tópicos

1. [WCAG Standards](wcag-standards.md) — níveis A/AA/AAA, critérios de sucesso, os quatro princípios (POUR)
2. [Semantic HTML](semantic-html.md) — elementos landmark, hierarquia de headings, labels de formulário, controles nativos
3. [ARIA Roles](aria-roles.md) — quando usar ARIA, roles, states, properties, a primeira regra do ARIA
4. [Keyboard Navigation](keyboard-navigation.md) — gerenciamento de focus, tab order, focus traps, skip links
5. [Testing Accessibility](testing-accessibility.md) — axe-core, testes com screen reader, auditoria Lighthouse a11y

---

## Pré-requisitos

- Track 02 — Frontend (HTML, CSS, React)
- Compreensão básica do DOM e da renderização do browser

---

## O que você vai construir

- Um modal dialog totalmente acessível com focus trap, tratamento da tecla Escape e atributos ARIA corretos
- Um componente dropdown customizado que passa no axe-core sem nenhuma violação
- Uma tabela de dados navegável por teclado com colunas ordenáveis
- Uma suite de testes de acessibilidade automatizados usando axe-core integrado a testes E2E com Playwright
