# Security Headers

## Visão Geral

Security headers HTTP são a forma mais rápida de fortalecer uma aplicação web. Eles instruem o navegador sobre como se comportar ao renderizar seu conteúdo — prevenindo clickjacking, desabilitando funcionalidades perigosas, impondo HTTPS e restringindo quais recursos podem ser carregados. Um único `npm install @fastify/helmet` e cinco minutos de configuração eliminam toda uma classe de vulnerabilidades. Este capítulo cobre os headers que importam, o que fazem e como configurá-los corretamente.

---

## Pré-requisitos

- Conhecimento básico de headers HTTP
- Fastify e Node.js
- Familiaridade com conceitos de Content Security Policy

---

## Conceitos Fundamentais

Security headers se dividem em três categorias:

1. **Segurança de transporte** — impõe HTTPS (`Strict-Transport-Security`)
2. **Segurança de conteúdo** — controla quais recursos podem ser carregados e executados (`Content-Security-Policy`, `X-Content-Type-Options`)
3. **Controle de frames e acesso** — previne clickjacking e controla conteúdo incorporado (`X-Frame-Options`, `Permissions-Policy`)

---

## Exemplos Práticos

### Helmet.js — a linha base em uma linha

```typescript
import Fastify from 'fastify';
import helmet from '@fastify/helmet';

const fastify = Fastify();

await fastify.register(helmet);
// Adiciona 11 security headers com padrões sensatos
```

O Helmet configura esses headers por padrão:
- `Content-Security-Policy`
- `Cross-Origin-Embedder-Policy`
- `Cross-Origin-Opener-Policy`
- `Cross-Origin-Resource-Policy`
- `Origin-Agent-Cluster`
- `Referrer-Policy`
- `Strict-Transport-Security`
- `X-Content-Type-Options`
- `X-DNS-Prefetch-Control`
- `X-Download-Options`
- `X-Frame-Options`
- `X-Permitted-Cross-Domain-Policies`
- `X-XSS-Protection` (desabilitado por padrão — navegadores modernos não precisam dele)

---

### Strict-Transport-Security (HSTS)

Instrui os navegadores a acessar este site apenas por HTTPS, mesmo que o usuário digite `http://`.

```typescript
await fastify.register(helmet, {
  hsts: {
    maxAge: 63072000,        // 2 anos — exigido para lista de preload HSTS
    includeSubDomains: true, // aplica a todos os subdomínios
    preload: true,           // submete às listas de preload dos navegadores
  },
});
```

Após configurar HSTS com um `maxAge` longo, é difícil removê-lo. Não defina `preload: true` até ter certeza de que todo o domínio (e todos os subdomínios) serve HTTPS.

---

### Content-Security-Policy (CSP)

O header mais poderoso e mais complexo. Restringe quais recursos o navegador carrega e executa.

```typescript
await fastify.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      // Padrão para todos os tipos de recurso não listados explicitamente
      defaultSrc: ["'self'"],

      // Scripts: apenas da própria origem e um CDN confiável
      scriptSrc: ["'self'", 'https://cdn.jsdelivr.net'],

      // Estilos: própria origem e Google Fonts
      styleSrc: ["'self'", 'https://fonts.googleapis.com', "'unsafe-inline'"],
      // 'unsafe-inline' para estilos é menos arriscado que para scripts; idealmente substitua por nonces

      // Fontes
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],

      // Imagens: própria origem, data URIs e CDN
      imgSrc: ["'self'", 'data:', 'https://cdn.example.com'],

      // XHR e fetch
      connectSrc: ["'self'", 'https://api.example.com'],

      // Sem plugins, sem tags object/embed
      objectSrc: ["'none'"],

      // Sem iframes de outras origens
      frameSrc: ["'none'"],

      // Atualiza referências de recursos HTTP para HTTPS
      upgradeInsecureRequests: [],
    },
  },
});
```

**Usando nonces para scripts inline (necessário para Next.js, muitos SPAs):**

```typescript
import crypto from 'crypto';

// Gera nonce por requisição
fastify.addHook('onRequest', async (request) => {
  request.cspNonce = crypto.randomBytes(16).toString('base64');
});

await fastify.register(helmet, {
  contentSecurityPolicy: {
    useDefaults: true,
    directives: {
      scriptSrc: [
        "'self'",
        (req) => `'nonce-${(req as any).cspNonce}'`,
      ],
    },
  },
});

// No seu template engine:
// <script nonce="{{cspNonce}}">...</script>
```

**Modo report-only (para implantação gradual):**

```typescript
// Não bloqueia — apenas envia relatórios de violação
reply.header(
  'Content-Security-Policy-Report-Only',
  "default-src 'self'; report-uri /api/csp-report"
);

// Endpoint para receber relatórios de violação
fastify.post('/api/csp-report', async (request) => {
  fastify.log.warn({ cspReport: request.body }, 'Violação de CSP');
  return { ok: true };
});
```

---

### X-Frame-Options

Impede que seu site seja incorporado em um iframe — a raiz dos ataques de clickjacking.

```typescript
await fastify.register(helmet, {
  frameguard: {
    action: 'deny', // nunca permita framing — use 'sameorigin' se você incorpora suas próprias páginas
  },
});
```

A diretiva `frame-ancestors` do CSP substitui `X-Frame-Options` nos navegadores modernos. Use ambos para compatibilidade:

```typescript
contentSecurityPolicy: {
  directives: {
    frameAncestors: ["'none'"], // navegadores modernos
    // X-Frame-Options: DENY configurado pelo frameguard para navegadores mais antigos
  },
},
```

---

### X-Content-Type-Options

Previne MIME type sniffing — o navegador deve usar o `Content-Type` declarado, e não tentar adivinhar pelo conteúdo:

```typescript
// Helmet configura isso por padrão: X-Content-Type-Options: nosniff
await fastify.register(helmet, {
  noSniff: true,
});
```

Sem isso, um arquivo `.jpg` contendo JavaScript poderia ser executado como script.

---

### Permissions-Policy

Controla o acesso a funcionalidades do navegador (câmera, microfone, geolocalização, etc.):

```typescript
// Helmet inclui um Permissions-Policy padrão
// Para personalizar:
fastify.addHook('onSend', async (request, reply) => {
  reply.header(
    'Permissions-Policy',
    [
      'camera=()',           // desabilita acesso à câmera
      'microphone=()',       // desabilita acesso ao microfone
      'geolocation=(self)',  // permite geolocalização apenas da própria origem
      'payment=()',          // desabilita a API de pagamento
      'usb=()',              // desabilita acesso USB
    ].join(', ')
  );
});
```

---

### Referrer-Policy

Controla quanta informação de referrer é incluída quando o usuário navega para fora:

```typescript
await fastify.register(helmet, {
  referrerPolicy: {
    policy: 'strict-origin-when-cross-origin',
    // URL completa para same-origin, apenas origin para cross-origin HTTPS, nada para HTTP
  },
});
```

| Policy | O que é enviado |
|--------|----------------|
| `no-referrer` | Nada — máxima privacidade |
| `strict-origin-when-cross-origin` | URL completa same-origin, apenas origin cross-origin — bom padrão |
| `unsafe-url` | URL completa sempre — vaza query params sensíveis para terceiros |

---

### Cross-Origin-Resource-Policy (CORP)

Controla quais origins podem carregar seus recursos:

```typescript
await fastify.register(helmet, {
  crossOriginResourcePolicy: {
    policy: 'same-origin', // apenas sua própria origin pode carregar esses recursos
    // 'cross-origin' para arquivos de CDN destinados a serem carregados em qualquer lugar
    // 'same-site' para recursos compartilhados entre subdomínios
  },
});
```

---

### Configuração completa de produção

```typescript
await fastify.register(helmet, {
  // HSTS
  hsts: {
    maxAge: 63072000,
    includeSubDomains: true,
    preload: true,
  },

  // Sem iframing
  frameguard: { action: 'deny' },

  // Proteção contra MIME sniffing
  noSniff: true,

  // Referrer policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },

  // Cross-origin resource policy
  crossOriginResourcePolicy: { policy: 'same-origin' },

  // CSP
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'"],
      imgSrc: ["'self'", 'data:'],
      connectSrc: ["'self'"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
});
```

---

## Padrões Comuns e Boas Práticas

- **Registre o Helmet antes das rotas** — os hooks são executados na ordem de registro
- **Comece com CSP em modo report-only** — audite as violações antes de aplicar
- **Teste headers com securityheaders.com** — dá uma nota ao seu site e sinaliza headers ausentes
- **CSP diferente para grupos de rotas diferentes** — mais restrito para o painel admin, mais flexível para o site público
- **Use nonces, não `unsafe-inline`** — `unsafe-inline` anula o CSP para scripts
- **Configure `Cache-Control` em endpoints sensíveis** — `no-store` para respostas de autenticação e dados de API

---

## Anti-Padrões a Evitar

- Configurar `X-XSS-Protection: 1; mode=block` — depreciado, pode introduzir XSS em alguns navegadores
- Usar `'unsafe-eval'` no CSP — permite `eval()` e `Function()` nos quais XSS se apoia
- Configurar HSTS com `preload: true` em um subdomínio que pode não servir sempre HTTPS
- Pular security headers em desenvolvimento — bugs frequentemente surgem de diferenças entre ambientes
- CSP totalmente aberto (`default-src *`) — nega o propósito do header

---

## Debugging e Resolução de Problemas

**"Minha aplicação React quebrou depois de adicionar o Helmet"**
O CSP está bloqueando scripts inline. Next.js e create-react-app injetam scripts inline. Use nonces (apps renderizados no servidor) ou valores CSP baseados em hash, ou temporariamente permita `'unsafe-inline'` enquanto você audita.

**"Como verifico quais headers estou configurando?"**
`curl -I https://seu-site.com` ou abra a aba Network do DevTools, clique na requisição principal e veja os Response Headers.

**"CSP está bloqueando Google Analytics / Intercom / outro script de terceiros"**
Adicione o domínio do terceiro ao `scriptSrc` e seus endpoints de API ao `connectSrc`. Consulte a documentação do terceiro para a configuração CSP necessária.

---

## Cenários do Mundo Real

**Cenário: Backend somente de API (sem renderização de HTML)**

Para uma REST API pura, apenas um subconjunto de headers se aplica:

```typescript
await fastify.register(helmet, {
  // API não serve HTML — CSP não é necessário para respostas JSON
  contentSecurityPolicy: false,
  // Estes ainda se aplicam:
  hsts: { maxAge: 63072000, includeSubDomains: true },
  noSniff: true,
  referrerPolicy: { policy: 'no-referrer' },
  frameguard: { action: 'deny' },
});
```

---

## Leitura Complementar

- [MDN: HTTP Security Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#security)
- [securityheaders.com](https://securityheaders.com/) — escaneie seu site
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/) — ferramenta do Google para verificar a força do CSP
- [Documentação do Helmet.js](https://helmetjs.github.io/)
- Track 07: [CORS & CSRF](cors-csrf.md)
- Track 07: [HTTPS & TLS](https-tls.md)

---

## Resumo

Security headers são a melhoria de segurança com melhor custo-benefício que você pode implementar em uma tarde. O Helmet.js define uma linha base forte com uma linha de código. Os headers que merecem mais atenção são: HSTS (impõe HTTPS), Content-Security-Policy (restringe execução de scripts), X-Frame-Options/frame-ancestors (previne clickjacking), X-Content-Type-Options (previne MIME sniffing) e Referrer-Policy (protege privacidade de URLs). Implante o CSP de forma incremental usando o modo report-only para capturar violações antes de aplicar. Verifique sua configuração final com securityheaders.com.
