# PWA Fundamentals

## Visão Geral

Progressive Web Apps (PWAs) fecham a lacuna entre sites e apps nativos. São aplicações web comuns aprimoradas com APIs modernas que permitem instalação na tela inicial, funcionalidade offline, sincronização em segundo plano e push notifications — sem precisar de uma loja de apps. Uma PWA não é uma base de código separada; é seu app web existente com um service worker, um web app manifest e um deploy em HTTPS.

PWAs são a escolha certa quando você precisa de amplo alcance sem a fricção de revisão de loja de apps, quando seu time é 100% web, ou quando seu app de conteúdo pode rodar totalmente no browser. Não são a escolha certa quando você precisa de integração profunda com hardware (Bluetooth, NFC, acesso à câmera RAW) ou quando a consistência de UI específica da plataforma é inegociável.

---

## Pré-requisitos

- Sólido entendimento de JavaScript e do event loop do browser
- Familiaridade com a Fetch API e Promises
- Entendimento básico de headers de cache HTTP
- Track 02 — Frontend (React ou framework equivalente)

---

## Conceitos Principais

### O Service Worker

Um service worker é um arquivo JavaScript que o browser executa em uma thread de segundo plano, separada da página principal. Ele age como um proxy de rede programável: toda requisição fetch que a página faz pode ser interceptada, modificada, respondida a partir do cache ou encaminhada para a rede.

Características principais:
- Roda em um contexto de worker separado (sem acesso ao DOM)
- Se comunica com as páginas via `postMessage`
- Persiste entre carregamentos de página e reinicializações do browser
- Tem seu próprio ciclo de vida: install → activate → fetch

**Ciclo de Vida do Service Worker**

```
Browser analisa o script do SW
       ↓
  evento install dispara
  (faça cache dos assets estáticos aqui)
       ↓
  waiting (SW antigo ainda ativo)
       ↓
  evento activate dispara
  (limpe caches antigos aqui)
       ↓
  eventos fetch são interceptados
```

**Registro**

```javascript
// main.js — registrar a partir da página
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/', // controla quais URLs o SW gerencia
      });
      console.log('SW registrado:', registration.scope);
    } catch (error) {
      console.error('Falha no registro do SW:', error);
    }
  });
}
```

**Service worker mínimo**

```javascript
// sw.js
const CACHE_NAME = 'my-app-v1';
const STATIC_ASSETS = ['/', '/index.html', '/app.js', '/styles.css'];

self.addEventListener('install', (event) => {
  // Faz cache dos assets estáticos durante o install
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(STATIC_ASSETS))
  );
  // Pula a espera para que o novo SW seja ativado imediatamente
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  // Remove caches antigos
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      )
    )
  );
  // Assume o controle dos clientes existentes
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => cached ?? fetch(event.request))
  );
});
```

### Estratégias de Cache

A estratégia de cache que você escolhe depende do tipo de conteúdo.

**Cache First (assets estáticos)**
Serve do cache se disponível; cai para a rede como fallback. Melhor para assets versionados (bundles JS com hashes de conteúdo).

```javascript
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  const cache = await caches.open(CACHE_NAME);
  cache.put(request, response.clone()); // clone porque response é um stream
  return response;
}
```

**Network First (respostas de API)**
Tenta a rede; cai para o cache se offline. Melhor para dados atualizados com frequência.

```javascript
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch {
    return caches.match(request);
  }
}
```

**Stale While Revalidate (dados de feed)**
Retorna a versão em cache imediatamente; busca uma versão fresca em segundo plano para a próxima vez.

```javascript
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then((response) => {
    cache.put(request, response.clone());
    return response;
  });

  return cached ?? fetchPromise;
}
```

### Web App Manifest

O manifest é um arquivo JSON que diz ao browser como apresentar seu app quando instalado.

```json
{
  "name": "Meu App Incrível",
  "short_name": "MeuApp",
  "description": "Uma progressive web application",
  "start_url": "/?utm_source=pwa",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide"
    }
  ],
  "categories": ["productivity", "utilities"],
  "shortcuts": [
    {
      "name": "Nova Entrada",
      "short_name": "Nova",
      "description": "Criar uma nova entrada",
      "url": "/new",
      "icons": [{ "src": "/icons/new-96.png", "sizes": "96x96" }]
    }
  ]
}
```

Vincule o manifest no seu HTML:

```html
<link rel="manifest" href="/manifest.json" />
<!-- iOS requer meta tags separadas -->
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="default" />
<meta name="apple-mobile-web-app-title" content="MeuApp" />
<link rel="apple-touch-icon" href="/icons/icon-192.png" />
```

**Valores de `display`:**
- `standalone` — parece um app nativo (sem chrome do browser)
- `minimal-ui` — controles mínimos do browser exibidos
- `fullscreen` — esconde toda a UI do browser (jogos)
- `browser` — abre na aba normal do browser

---

## Exemplos Práticos

### Fallback de Página Offline

```javascript
// sw.js
const OFFLINE_URL = '/offline.html';

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) =>
      cache.addAll([OFFLINE_URL, ...STATIC_ASSETS])
    )
  );
});

self.addEventListener('fetch', (event) => {
  // Intercepta apenas requisições de navegação para o fallback offline
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match(OFFLINE_URL))
    );
    return;
  }
  // Outras requisições usam cache-first
  event.respondWith(cacheFirst(event.request));
});
```

### Install Prompt

Browsers disparam um evento `beforeinstallprompt` quando decidem que seu app pode ser instalado. Você deve salvar esse evento e acioná-lo em um gesto do usuário.

```javascript
// installPrompt.js
let deferredPrompt = null;

window.addEventListener('beforeinstallprompt', (event) => {
  event.preventDefault(); // impede a mini-infobar automática
  deferredPrompt = event;
  showInstallButton(); // exibe seu botão de instalação personalizado
});

async function handleInstallClick() {
  if (!deferredPrompt) return;

  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;

  if (outcome === 'accepted') {
    console.log('Usuário aceitou o install prompt');
  } else {
    console.log('Usuário dispensou o install prompt');
  }

  deferredPrompt = null;
  hideInstallButton();
}

// Exemplo com componente React
function InstallButton() {
  const [canInstall, setCanInstall] = useState(false);
  const promptRef = useRef(null);

  useEffect(() => {
    const handler = (e) => {
      e.preventDefault();
      promptRef.current = e;
      setCanInstall(true);
    };
    window.addEventListener('beforeinstallprompt', handler);
    return () => window.removeEventListener('beforeinstallprompt', handler);
  }, []);

  const install = async () => {
    if (!promptRef.current) return;
    promptRef.current.prompt();
    const { outcome } = await promptRef.current.userChoice;
    if (outcome === 'accepted') setCanInstall(false);
    promptRef.current = null;
  };

  if (!canInstall) return null;
  return <button onClick={install}>Instalar App</button>;
}
```

### Push Notifications

Push notifications requerem três partes: seu servidor, um serviço push (gerenciado por fornecedores de browser) e o browser.

```javascript
// 1. Solicitar permissão
async function requestNotificationPermission() {
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') {
    throw new Error('Permissão de notificação negada');
  }
}

// 2. Inscrever o browser no push
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true, // obrigatório: deve exibir notificação para cada push
    applicationServerKey: urlBase64ToUint8Array(process.env.VAPID_PUBLIC_KEY),
  });

  // 3. Enviar assinatura para seu servidor
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  });

  return subscription;
}

// Helper: converter chave VAPID
function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
  const rawData = atob(base64);
  return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)));
}

// 4. Tratar push no service worker
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};
  event.waitUntil(
    self.registration.showNotification(data.title ?? 'Notificação', {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/badge-96.png',
      data: { url: data.url },
      actions: [
        { action: 'open', title: 'Abrir' },
        { action: 'dismiss', title: 'Dispensar' },
      ],
    })
  );
});

// 5. Tratar clique na notificação
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'dismiss') return;

  const url = event.notification.data?.url ?? '/';
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((windowClients) => {
      const existing = windowClients.find((c) => c.url === url && 'focus' in c);
      if (existing) return existing.focus();
      return clients.openWindow(url);
    })
  );
});
```

**Push no servidor (Node.js com web-push)**

```javascript
import webpush from 'web-push';

webpush.setVapidDetails(
  'mailto:admin@example.com',
  process.env.VAPID_PUBLIC_KEY,
  process.env.VAPID_PRIVATE_KEY
);

// Gerar chaves VAPID uma vez: webpush.generateVAPIDKeys()

async function sendPushNotification(subscription, payload) {
  try {
    await webpush.sendNotification(subscription, JSON.stringify(payload));
  } catch (error) {
    if (error.statusCode === 410) {
      // Assinatura expirada — remover do banco de dados
      await db.pushSubscriptions.delete(subscription.endpoint);
    }
    throw error;
  }
}
```

---

## Padrões Comuns e Boas Práticas

### Background Sync

Enfileira requisições que falharam e as reprocessa quando a conectividade for restaurada.

```javascript
// No seu código de fetch
async function saveData(data) {
  try {
    await fetch('/api/save', { method: 'POST', body: JSON.stringify(data) });
  } catch {
    // Armazenar no IndexedDB para depois
    await idb.put('sync-queue', { id: crypto.randomUUID(), data, timestamp: Date.now() });

    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register('sync-saves');
  }
}

// No service worker
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-saves') {
    event.waitUntil(syncPendingSaves());
  }
});

async function syncPendingSaves() {
  const queue = await idb.getAll('sync-queue');
  for (const item of queue) {
    await fetch('/api/save', { method: 'POST', body: JSON.stringify(item.data) });
    await idb.delete('sync-queue', item.id);
  }
}
```

### Detecção de Atualização

```javascript
// Notificar usuários quando uma nova versão estiver disponível
function registerSW() {
  navigator.serviceWorker.register('/sw.js').then((registration) => {
    registration.addEventListener('updatefound', () => {
      const newWorker = registration.installing;

      newWorker.addEventListener('statechange', () => {
        if (
          newWorker.state === 'installed' &&
          navigator.serviceWorker.controller
        ) {
          // Nova versão pronta — exibe banner de atualização
          showUpdateBanner(() => {
            newWorker.postMessage({ type: 'SKIP_WAITING' });
            window.location.reload();
          });
        }
      });
    });
  });
}

// No sw.js
self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```

### Workbox

Workbox é a biblioteca do Google para padrões de service worker. Elimina boilerplate e lida com casos extremos.

```javascript
// vite.config.ts com vite-plugin-pwa
import { defineConfig } from 'vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.example\.com\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: { maxEntries: 50, maxAgeSeconds: 300 },
              networkTimeoutSeconds: 10,
            },
          },
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
            handler: 'StaleWhileRevalidate',
            options: { cacheName: 'google-fonts-stylesheets' },
          },
        ],
      },
      manifest: {
        name: 'Meu App',
        short_name: 'App',
        theme_color: '#6366f1',
        icons: [
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
        ],
      },
    }),
  ],
});
```

---

## Anti-Padrões a Evitar

**Fazer cache de tudo sem uma estratégia.** Dados desatualizados servidos silenciosamente do cache causam bugs misteriosos. Cada tipo de recurso precisa de uma estratégia explícita escolhida com base nos requisitos de atualização.

**Solicitar permissão de notificação no carregamento da página.** Browsers bloqueiam sites que fazem isso. Sempre solicite permissão em resposta a uma ação do usuário (um clique em um botão).

**Esquecer de clonar respostas antes de fazer cache.** Objetos `Response` são streams de uso único. Chame `response.clone()` antes de colocar no cache e retornar.

**Não chamar `waitUntil`.** Handlers de eventos de service worker são encerrados assim que a função principal retorna. Envolva trabalho assíncrono em `event.waitUntil()` para estender o tempo de vida.

**Fazer deploy sem HTTPS.** Service workers exigem HTTPS (localhost é a única exceção). Planeje seu deploy adequadamente.

**Chamar `skipWaiting()` incondicionalmente no handler de install.** Isso pode quebrar clientes no meio de uma sessão quando recebem conjuntos de assets parciais com quebras. Condicione isso a uma atualização confirmada pelo usuário.

**Ignorar limitações do iOS.** O Safari tem lacunas significativas de PWA: sem push notifications até o iOS 16.4, cota de armazenamento limitada, sem background sync. Sempre teste em dispositivos iOS reais e documente quais recursos estão indisponíveis.

---

## Depuração e Resolução de Problemas

### Chrome DevTools

- **Aba Application > Service Workers** — veja o status de registro, estado do ciclo de vida, inspecione/atualize
- **Aba Application > Cache Storage** — navegue pelas respostas em cache
- **Aba Application > Manifest** — valide o manifest e a instalabilidade
- **Aba Network > checkbox Offline** — simule modo offline
- `chrome://serviceworker-internals` — visão de baixo nível de todos os SWs

### Problemas Comuns

**SW não atualizando:** O browser faz cache do sw.js. Use `updateViaCache: 'none'` na chamada de registro ou adicione um query param de cache-busting durante o desenvolvimento.

```javascript
navigator.serviceWorker.register('/sw.js?v=' + BUILD_ID, {
  updateViaCache: 'none',
});
```

**Respostas opacas poluindo o cache:** Requisições cross-origin sem CORS retornam respostas opacas (status 0). Não as faça cache — uma única resposta opaca pode preencher toda a cota do seu cache. Adicione uma verificação:

```javascript
if (response.type === 'opaque') return response; // não fazer cache
```

**Install não disparando no iOS:** O iOS exige: HTTPS, manifest válido com `start_url`, ao menos um ícone de 192x192, `display: standalone`. A falta de qualquer um desses silenciosamente impede a opção Adicionar à Tela Inicial.

**Assinatura de push perdida após atualização do browser:** Sempre verifique se uma assinatura ainda existe antes de assumir que é válida. Reinscreva silenciosamente se necessário.

---

## Cenários do Mundo Real

### App de Notas Offline-First

Arquitetura:
1. Todas as escritas vão para o IndexedDB primeiro (síncrono do ponto de vista do usuário)
2. Uma fila de background sync mantém escritas pendentes no servidor
3. SW intercepta chamadas de API: GET usa stale-while-revalidate, POST/PUT/DELETE usam network-first com fallback de background sync
4. Na reconexão, a fila de sync reprocessa em ordem com resolução de conflitos (last-write-wins ou CRDT)

### PWA para um Site de Notícias

Arquitetura:
1. Shell (HTML, CSS, JS) é cacheado na primeira instalação — carrega instantaneamente
2. Páginas HTML de artigos são cacheadas na visita (cache-first com expiração de 24h)
3. Endpoint de feed da API usa network-first com cache de 5 minutos
4. Imagens usam stale-while-revalidate com cache LRU de 100 entradas
5. Push notifications para notícias urgentes; usuário configura tópicos nas configurações

---

## Background Sync

O background sync permite que o SW repita requisições que falharam quando o usuário se reconectar, mesmo que a aba do app esteja fechada.

```typescript
// No service worker — escuta eventos de sync
self.addEventListener('sync', (event: SyncEvent) => {
  if (event.tag === 'sync-pending-posts') {
    event.waitUntil(syncPendingPosts());
  }
});

async function syncPendingPosts(): Promise<void> {
  const db = await openDB('app-store', 1);
  const pending = await db.getAll('pending-posts');

  await Promise.all(
    pending.map(async (post) => {
      try {
        await fetch('/api/posts', {
          method: 'POST',
          body: JSON.stringify(post),
          headers: { 'Content-Type': 'application/json' },
        });
        await db.delete('pending-posts', post.id);
      } catch {
        // Vai tentar novamente no próximo evento de sync
      }
    }),
  );
}

// No app — registra o sync quando o usuário envia o formulário
async function submitPostOfflineSafe(post: Post): Promise<void> {
  const db = await openDB('app-store', 1);
  await db.put('pending-posts', { ...post, id: crypto.randomUUID() });

  if ('serviceWorker' in navigator && 'SyncManager' in window) {
    const reg = await navigator.serviceWorker.ready;
    await reg.sync.register('sync-pending-posts');
    // Executa imediatamente se online, ou na próxima reconexão
  } else {
    // Fallback: tenta diretamente
    await fetch('/api/posts', { method: 'POST', body: JSON.stringify(post) });
  }
}
```

O background sync é one-shot — dispara uma vez quando o dispositivo se reconecta. Se o handler lançar um erro, o browser tentará novamente (até um limite definido pelo browser). Use-o para: envios de formulário, mensagens de chat, uploads de arquivo.

**Periodic Background Sync** (somente Chrome, requer pontuação de engajamento com o site):

```typescript
// Solicita permissão de sync periódico
const status = await navigator.permissions.query({
  name: 'periodic-background-sync' as PermissionName,
});

if (status.state === 'granted') {
  const reg = await navigator.serviceWorker.ready;
  await (reg as any).periodicSync.register('refresh-news', {
    minInterval: 24 * 60 * 60 * 1000, // mínimo de 24 horas
  });
}

// No SW
self.addEventListener('periodicsync', (event: any) => {
  if (event.tag === 'refresh-news') {
    event.waitUntil(refreshNewsCache());
  }
});
```

O sync periódico dispara mesmo quando o usuário não está usando ativamente o app, permitindo pré-busca de conteúdo. O OS controla o tempo real — trate `minInterval` como uma sugestão, não uma garantia.

---

## Share Target API

Faça sua PWA aparecer na aba de compartilhamento do OS — usuários podem compartilhar URLs, textos ou arquivos diretamente no seu app.

```json
{
  "share_target": {
    "action": "/share-target",
    "method": "POST",
    "enctype": "multipart/form-data",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url",
      "files": [{ "name": "media", "accept": ["image/*", "video/*"] }]
    }
  }
}
```

```typescript
// app/share-target/route.ts (Next.js App Router) — recebe o conteúdo compartilhado
export async function POST(req: Request) {
  const form = await req.formData();
  const title = form.get('title') as string | null;
  const url = form.get('url') as string | null;
  const file = form.get('media') as File | null;

  // Armazena no IndexedDB para o app ler, depois redireciona
  // O SW intercepta esse POST e faz cache do payload antes de redirecionar
  return Response.redirect('/editor?shared=true', 303);
}
```

```typescript
// Na página /editor — lê os dados compartilhados do IndexedDB
import { openDB } from 'idb';

export function useSharedContent() {
  const [sharedData, setSharedData] = useState<SharedData | null>(null);

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    if (params.get('shared') === 'true') {
      // Lê do IDB onde o SW armazenou
      readSharedDataFromIDB().then(setSharedData);
    }
  }, []);

  return sharedData;
}
```

**Matriz de suporte:** Chrome (desktop + Android), Edge. Não suportado no iOS Safari — a aba de compartilhamento no iOS é controlada pelo sistema e não aceita destinos web.

**Casos de uso:** Salvar artigo em app de leitura para depois, compartilhar foto em PWA de edição de imagem, compartilhar URL em app de favoritos.

---

## Anti-Padrões Adicionais

**Fazer cache de HTML dinâmico com cache-first** — páginas HTML contêm links e meta tags que mudam. Cache-first significa que os usuários verão navegação desatualizada indefinidamente. Use network-first com um timeout curto para HTML, caindo para um shell em cache.

```typescript
// Errado: cache-first para HTML
workbox.routing.registerRoute(
  ({ request }) => request.destination === 'document',
  new workbox.strategies.CacheFirst(), // usuários veem páginas desatualizadas
);

// Certo: network-first para HTML
workbox.routing.registerRoute(
  ({ request }) => request.destination === 'document',
  new workbox.strategies.NetworkFirst({
    networkTimeoutSeconds: 3, // cai para o cache após 3s
    cacheName: 'html-cache',
    plugins: [new workbox.expiration.ExpirationPlugin({ maxAgeSeconds: 60 * 60 * 24 })],
  }),
);
```

**Não versionar nomes de cache** — entradas de cache antigas persistem quando você faz deploy de uma nova versão. Sempre inclua uma versão ou hash de conteúdo nos nomes de cache, e limpe caches antigos no evento `activate`.

```typescript
// Errado: nome de cache fixo
const CACHE_NAME = 'my-app-cache'; // entradas antigas nunca são removidas

// Certo: nome de cache versionado + limpeza
const CACHE_VERSION = 'v3';
const CACHE_NAME = `my-app-cache-${CACHE_VERSION}`;

self.addEventListener('activate', (event: ExtendableEvent) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key.startsWith('my-app-cache-') && key !== CACHE_NAME)
          .map((key) => caches.delete(key)),
      ),
    ),
  );
});
```

O Workbox lida com isso automaticamente quando você usa `generateSW` ou `injectManifest` com um hash de revisão — uma razão para preferir Workbox a service workers feitos à mão.

---

## Leitura Adicional

- [web.dev — Progressive Web Apps](https://web.dev/progressive-web-apps/)
- [MDN — Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [Documentação do Workbox](https://developer.chrome.com/docs/workbox/)
- [What PWA Can Do Today](https://whatpwacando.today/) — matriz de capacidades atuais do browser
- [vite-plugin-pwa](https://vite-pwa-org.netlify.app/)

---

## Resumo

Uma PWA é construída sobre três pilares: um service worker (proxy de rede + cache), um web manifest (metadados de instalabilidade) e HTTPS. Estratégias de cache (cache-first, network-first, stale-while-revalidate) devem ser escolhidas por tipo de recurso. Push notifications requerem chaves VAPID, uma assinatura de push armazenada no servidor e um handler de evento `push` no SW. Workbox remove boilerplate e lida com casos extremos como cache de respostas opacas. Teste em hardware iOS real cedo — o suporte a PWA do Safari está significativamente atrás do Chrome.
