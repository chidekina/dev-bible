# Como a Web Funciona

> Todo desenvolvedor sênior deve ser capaz de traçar o caminho de um toque de tecla até um pixel renderizado. Este arquivo cobre toda a jornada: resolução DNS, handshake TCP, negociação TLS, semântica HTTP, renderização do browser e estratégias de cache.

---

## 1. O Que É e Por Que Importa

Quando um usuário digita `https://google.com` e pressiona Enter, uma cadeia de pelo menos dez operações técnicas distintas acontece antes de um único pixel aparecer. A maioria dos desenvolvedores aprende HTTP e segue em frente, mas entender toda a stack — de pacotes UDP do DNS até composição de camadas no browser — é o que separa engenheiros que depuram mistérios de produção daqueles que chutam.

Conhecer este material permite:
- Depurar "por que meu site está lento?" isolando qual camada é o gargalo
- Corrigir bugs de cache de CDN que servem assets desatualizados após um deploy
- Entender por que HTTPS importa e o que o TLS realmente faz
- Otimizar Time-to-First-Byte (TTFB) e Core Web Vitals
- Raciocinar corretamente sobre CORS, cookies e redirects

---

## 2. Conceitos Fundamentais

### DNS — Domain Name System

DNS traduz nomes legíveis por humanos (`google.com`) em endereços IP (`142.250.80.46`). É um sistema distribuído e hierárquico.

**Os quatro atores em uma consulta DNS:**

1. **Recursive Resolver** (seu provedor ou 8.8.8.8): O servidor que o seu SO consulta primeiro. Ele faz o trabalho pesado, consultando outros servidores em seu nome e armazenando resultados em cache.
2. **Root Name Server**: Existem 13 clusters lógicos de servidores raiz no mundo todo. Eles sabem quais servidores são autoritativos para TLDs (`.com`, `.org`, `.io`).
3. **TLD Name Server**: O servidor TLD `.com` sabe qual servidor de nomes autoritativo mantém registros para `google.com`.
4. **Authoritative Name Server**: A autoridade final. Este servidor mantém o registro A real (IPv4) ou AAAA (IPv6) para `google.com`.

**Tipos de registros DNS:**
- `A` — mapeia nome para endereço IPv4
- `AAAA` — mapeia nome para endereço IPv6
- `CNAME` — alias de nome canônico (`www.exemplo.com → exemplo.com`)
- `MX` — servidor de troca de e-mail
- `TXT` — texto arbitrário (usado para SPF, DKIM, verificação de domínio)
- `NS` — registros de servidor de nomes
- `SOA` — Start of Authority, metadados de zona

**TTL (Time To Live):** Todo registro DNS tem um TTL em segundos. Os caches respeitam esse TTL. Um TTL de 300 significa que resolvers re-consultam após 5 minutos. Durante uma migração de DNS, reduzir o TTL antecipadamente (para 60s) permite que as mudanças se propaguem mais rápido.

---

### TCP/IP — Transmission Control Protocol

TCP é um protocolo de transporte orientado a conexão e confiável. Antes de qualquer dado ser trocado, um **handshake de três vias** estabelece a conexão:

```
Cliente                         Servidor
  |                               |
  |—— SYN (seq=x) ——————————————>|   "Quero me conectar"
  |                               |
  |<—— SYN-ACK (seq=y, ack=x+1) —|   "Ok, recebi"
  |                               |
  |—— ACK (seq=x+1, ack=y+1) ——>|   "Recebi sua resposta"
  |                               |
  |   ← conexão estabelecida →    |
```

Este handshake adiciona um **round-trip time (RTT)** antes de qualquer dado fluir. Em uma rede móvel com RTT de 100ms, são 100ms antes mesmo do TLS começar.

TCP também cuida de:
- **Segmentação**: dividindo dados em pacotes
- **Ordenação**: remontando pacotes fora de ordem
- **Controle de fluxo**: receptor informa ao remetente a velocidade de envio (tamanho da janela)
- **Controle de congestionamento**: algoritmo de slow-start aumenta a largura de banda gradualmente

---

### TLS — Transport Layer Security

TLS (sucessor do SSL) criptografa a conexão. TLS 1.3 (atual) requer apenas **1 RTT** para o handshake (TLS 1.2 precisava de 2 RTTs).

**Handshake TLS 1.3:**

```
Cliente                                Servidor
  |                                      |
  |—— ClientHello ——————————————————>   |
  |   (cifras suportadas, key share,     |
  |    versão TLS, nonce aleatório)      |
  |                                      |
  |<—— ServerHello ——————————————————   |
  |   (cifra escolhida, key share,       |
  |    certificado, Finished)            |
  |                                      |
  |—— Finished ————————————————————>    |
  |                                      |
  |   ← dados criptografados fluem →    |
```

Conceitos-chave:
- **Certificado**: um documento X.509 assinado por uma Autoridade Certificadora (CA) que prova que o servidor é dono do domínio. O browser tem uma lista integrada de CAs confiáveis.
- **Troca de Chaves**: usando Diffie-Hellman Ephemeral (DHE) ou Elliptic Curve DHE (ECDHE), ambos os lados derivam um segredo compartilhado sem jamais transmiti-lo — sigilo futuro perfeito.
- **Suite de Cifra**: ex. `TLS_AES_256_GCM_SHA384` — especifica o algoritmo de criptografia (AES-256-GCM) e a função de hash (SHA-384).

---

### HTTP — HyperText Transfer Protocol

HTTP é um protocolo de requisição/resposta sobre TCP+TLS.

**Anatomia de uma requisição:**
```
GET /api/users HTTP/1.1          ← método + caminho + versão
Host: api.example.com            ← header obrigatório no HTTP/1.1
Accept: application/json
Authorization: Bearer eyJ...
                                 ← linha em branco separa headers do corpo
```

**Anatomia de uma resposta:**
```
HTTP/1.1 200 OK                  ← versão + código de status + frase
Content-Type: application/json
Content-Length: 142
Cache-Control: max-age=3600
ETag: "abc123"
                                 ← linha em branco
{ "users": [...] }               ← corpo
```

**Métodos HTTP:**

| Método  | Idempotente | Seguro | Corpo |
|---------|-------------|--------|-------|
| GET     | Sim         | Sim    | Não (tecnicamente permitido, ignorado por convenção) |
| POST    | Não         | Não    | Sim |
| PUT     | Sim         | Não    | Sim |
| PATCH   | Não         | Não    | Sim |
| DELETE  | Sim         | Não    | Opcional |
| HEAD    | Sim         | Sim    | Não |
| OPTIONS | Sim         | Sim    | Não |

**Classes de Código de Status:**
- `1xx` — Informacional (100 Continue, 101 Switching Protocols)
- `2xx` — Sucesso (200 OK, 201 Created, 204 No Content)
- `3xx` — Redirecionamento (301 Moved Permanently, 302 Found, 304 Not Modified)
- `4xx` — Erro do Cliente (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- `5xx` — Erro do Servidor (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)

---

## 3. Como Funciona

### Passo a passo: "O que acontece quando você digita google.com"

**Passo 1 — Verificação de cache do browser**
Antes de qualquer coisa na rede, o browser verifica seu próprio cache DNS e depois o cache DNS do SO (`/etc/hosts` no Linux/macOS, o arquivo hosts do Windows).

**Passo 2 — Resolução DNS**
```
Browser → Resolver do SO → Recursive resolver (8.8.8.8)
                               ↓ (cache miss)
                           Servidor raiz (.com NS?)
                               ↓
                           Servidor TLD (google.com NS?)
                               ↓
                           NS autoritativo (216.239.32.10)
                               ↓
                           Registro A: 142.250.80.46
                               ↑
                           Retornado e armazenado em cache (TTL: 300s)
```

**Passo 3 — Handshake TCP**
Com o endereço IP em mãos, o browser abre um socket TCP para a porta 443 (HTTPS) ou 80 (HTTP). O handshake de três vias é completado em 1 RTT.

**Passo 4 — Handshake TLS**
ClientHello → ServerHello + Certificado + Finished → Client Finished. TLS 1.3 adiciona apenas 1 RTT. Custo total até aqui: 2 RTTs (1 TCP + 1 TLS).

**Passo 5 — Requisição HTTP**
O browser envia a requisição GET incluindo headers como `Accept-Encoding: gzip, br`, cookies, `User-Agent` e `Referer`.

**Passo 6 — Processamento no servidor**
O servidor (nginx / edge CDN → servidor de aplicação → banco de dados) constrói a resposta. TTFB (Time to First Byte) é medido aqui — do momento em que a requisição sai do browser até o primeiro byte da resposta chegar.

**Passo 7 — Transmissão da resposta**
O servidor envia os headers primeiro, depois o corpo. Com HTTP/2, múltiplos recursos podem ser transmitidos concorrentemente sobre a mesma conexão.

**Passo 8 — Pipeline de renderização do browser**

1. Analisa HTML → constrói árvore **DOM** (Document Object Model)
2. Analisa CSS → constrói árvore **CSSOM** (CSS Object Model)
3. Executa JavaScript síncrono (bloqueia a análise a menos que use `defer`/`async`)
4. Combina DOM + CSSOM → **Render Tree** (apenas nós visíveis; `display: none` excluído)
5. **Layout** (a.k.a. reflow): calcula tamanho e posição de cada nó
6. **Paint**: preenche pixels (texto, cores, imagens, bordas)
7. **Composite**: combina camadas pintadas na GPU (transforms, opacity são compostos de forma barata)

> JavaScript bloqueia a análise HTML por padrão. Use `defer` (roda após a análise) ou `async` (roda assim que baixado) em tags `<script>` para evitar bloqueio de renderização.

---

### HTTP/1.1 vs HTTP/2 vs HTTP/3

**HTTP/1.1 (1997):**
- Uma requisição por vez por conexão TCP — head-of-line blocking
- Solução alternativa: browsers abrem 6 conexões TCP paralelas por domínio
- Headers enviados como texto simples, sem compressão
- Keep-Alive: conexões podem ser reutilizadas entre múltiplas requisições, mas as requisições ainda são seriais por conexão

**HTTP/2 (2015):**
- **Multiplexing**: múltiplas requisições e respostas simultaneamente sobre uma única conexão TCP, usando frames binários e IDs de stream
- **Compressão de headers** (HPACK): headers são comprimidos usando um dicionário compartilhado — grande economia em headers repetidos como `Cookie` e `User-Agent`
- **Server push**: servidor pode proativamente enviar recursos (ex: CSS, fontes) que sabe que o cliente precisará — na prática, raramente usado bem e removido em algumas implementações
- Ainda usa TCP por baixo — um único pacote TCP perdido pausa TODOS os streams (head-of-line blocking TCP)

**HTTP/3 (2022):**
- Roda sobre **QUIC** (Quick UDP Internet Connections) em vez de TCP
- QUIC é construído sobre UDP com confiabilidade, ordenação e controle de congestionamento implementados no espaço do usuário
- **Sem head-of-line blocking**: cada stream QUIC é independente; um pacote perdido bloqueia apenas seu próprio stream
- **Retomada 0-RTT**: clientes conectados anteriormente podem enviar dados com o primeiro pacote
- **Migração de conexão**: conexões QUIC sobrevivem a mudanças de endereço IP (handoffs móveis entre WiFi e 4G)

> HTTP/3 agora é suportado por todos os principais browsers e CDNs. Cloudflare, Fastly e AWS CloudFront suportam. Verifique com `curl --http3`.

---

### Cache

**Diretivas Cache-Control:**

| Diretiva | Significado |
|----------|-------------|
| `max-age=3600` | Cache por 3600 segundos a partir de quando foi recebido |
| `no-cache` | Pode armazenar em cache, mas **revalide** com o servidor antes de cada uso |
| `no-store` | **Não** armazene em absoluto (dados sensíveis) |
| `must-revalidate` | Uma vez obsoleto, deve revalidar — não sirva obsoleto mesmo offline |
| `public` | Caches compartilhados (CDN) podem armazenar em cache |
| `private` | Apenas cache do browser (não CDN) — ex: páginas personalizadas |
| `immutable` | Conteúdo nunca mudará; sem necessidade de revalidação |
| `s-maxage=3600` | Como max-age mas apenas para caches compartilhados (CDN); substitui max-age para CDN |
| `stale-while-revalidate=60` | Sirva conteúdo obsoleto enquanto revalida em segundo plano |

**ETag e Last-Modified (requisições condicionais):**
```http
# Primeira requisição — servidor inclui validadores
GET /app.js  →  200 OK
ETag: "abc123"
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT

# Requisição subsequente — browser envia validadores
GET /app.js
If-None-Match: "abc123"
If-Modified-Since: Mon, 01 Jan 2024 00:00:00 GMT

# Servidor: nada mudou
→  304 Not Modified   (sem corpo, browser usa sua cópia em cache)

# Servidor: arquivo mudou
→  200 OK + novo ETag  (resposta completa com novo conteúdo)
```

**Cache de CDN:**
Nós de borda de CDN (PoPs — Points of Presence) armazenam conteúdo em cache geograficamente próximo aos usuários. O TTL de cache do CDN é controlado por `Cache-Control: s-maxage=3600` (substitui `max-age` para caches compartilhados). Quando um usuário em São Paulo solicita um arquivo em cache em um PoP de São Paulo, a resposta vem de ~5ms de distância em vez de um servidor de origem distante.

---

## 4. Exemplos de Código

### curl — inspecionar headers e timing
```bash
# Ver apenas headers de resposta (sem corpo)
curl -I https://google.com

# Seguir redirects, mostrar info detalhada de TLS e conexão
curl -vL https://google.com 2>&1 | head -80

# Medir detalhamento de timing
curl -o /dev/null -s -w "\
DNS lookup:        %{time_namelookup}s\n\
TCP connect:       %{time_connect}s\n\
TLS handshake:     %{time_appconnect}s\n\
TTFB:              %{time_starttransfer}s\n\
Total:             %{time_total}s\n" \
  https://google.com

# Enviar um POST JSON com autenticação
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# Verificar qual versão HTTP está sendo usada
curl -vso /dev/null https://cloudflare.com 2>&1 | grep "< HTTP"
```

### Requisição HTTP/1.1 bruta (via netcat)
```bash
# Enviar manualmente uma requisição HTTP para ver a resposta bruta
printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" \
  | nc example.com 80
```

### Exemplos de headers Cache-Control
```http
# Assets estáticos com hash do conteúdo no nome do arquivo (ex: app.a3f8e1.js)
# Cache por 1 ano, nunca revalidar — nome do arquivo garante frescor
Cache-Control: public, max-age=31536000, immutable

# Respostas de API — personalizadas, sem cache CDN, browser pode cache por 5 minutos
Cache-Control: private, max-age=300

# Páginas HTML — browser sempre revalida (usa ETag/304 para economizar banda)
Cache-Control: no-cache

# Dados sensíveis (sessões, tokens, banco) — nunca armazene em lugar algum
Cache-Control: no-store

# Feed de notícias — sirva obsoleto enquanto busca novo em segundo plano
Cache-Control: public, max-age=60, stale-while-revalidate=300

# CDN armazena por 1 hora, browser por 5 minutos
Cache-Control: public, s-maxage=3600, max-age=300
```

### Servidor HTTP/1.1 mínimo em Node.js (TCP bruto)
```javascript
const net = require('net');

const server = net.createServer((socket) => {
  let raw = '';

  socket.on('data', (chunk) => {
    raw += chunk.toString();

    // Aguarda até ter os headers completos (linha em branco)
    if (!raw.includes('\r\n\r\n')) return;

    const [requestLine] = raw.split('\r\n');
    const [method, path] = requestLine.split(' ');

    let statusCode = 200;
    let body;

    if (path === '/') {
      body = JSON.stringify({ message: 'Olá, Mundo!', method, path });
    } else {
      statusCode = 404;
      body = JSON.stringify({ error: 'Não Encontrado', path });
    }

    const response = [
      `HTTP/1.1 ${statusCode} ${statusCode === 200 ? 'OK' : 'Not Found'}`,
      'Content-Type: application/json',
      `Content-Length: ${Buffer.byteLength(body)}`,
      'Connection: close',
      '',
      body,
    ].join('\r\n');

    socket.write(response);
    socket.end();
  });
});

server.listen(3000, () => {
  console.log('Servidor HTTP bruto em http://localhost:3000');
  console.log('Teste: curl http://localhost:3000/');
  console.log('Teste: curl http://localhost:3000/desconhecido');
});
```

### Headers CORS
```http
# Preflight do browser para POST cross-origin
OPTIONS /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type

# Resposta do servidor — concede permissão
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Confundir `no-cache` com `no-store`**: `no-cache` significa "PODE armazená-lo, mas DEVE revalidar antes de servir". `no-store` significa "não armazene em lugar algum". Muitos desenvolvedores usam `no-cache` quando querem dizer `no-store` para dados sensíveis, acidentalmente armazenando em cache senhas e tokens em caches intermediários.

> ⚠️ **Assumir que DNS é instantâneo**: Consultas DNS podem levar 20–200ms. O cache com TTL ajuda, mas o primeiro visitante após a expiração do TTL sempre paga o tempo total de consulta. Solução: use hints `<link rel="dns-prefetch">` e `<link rel="preconnect">` no HTML para origens de terceiros.

> ⚠️ **Tratar HTTP e HTTPS como equivalentes**: HTTPS não é apenas "HTTP criptografado". Muda a semântica dos cookies (o flag `Secure` só funciona em HTTPS), habilita certas APIs do browser (geolocalização, Service Workers, Web Bluetooth só disponíveis em origens seguras) e afeta o ranking de SEO. Nunca sirva conteúdo misto (recursos HTTP em uma página HTTPS — browsers modernos bloqueiam silenciosamente).

> ⚠️ **Confundir 301 com 302**: 301 (Movido Permanentemente) é armazenado em cache por browsers e mecanismos de busca indefinidamente, passando equidade de link SEO para a nova URL. 302 (Found / Redirect Temporário) não é armazenado em cache e não transfere valor de SEO. Usar 301 incorretamente para um redirect temporário significa que você não pode desfazê-lo facilmente — o browser ignora por meses.

> ⚠️ **Ignorar head-of-line blocking no HTTP/2**: Desenvolvedores às vezes pensam que HTTP/2 resolve completamente todos os problemas de latência. Um único pacote TCP perdido ainda pausa todos os streams naquela conexão. HTTP/3 sobre QUIC resolve isso. Além disso, server push foi removido do Chrome em 2022 — não dependa dele.

> ⚠️ **Definir `max-age` sem hash de conteúdo**: Se você armazena em cache `app.js` por 1 ano mas faz deploy de uma nova versão na mesma URL, usuários ficam presos com o arquivo antigo até o cache expirar. Solução: inclua um hash de conteúdo no nome do arquivo (`app.a3f8e1b2.js`) e defina `immutable`. Todos os bundlers modernos (Webpack, Vite, esbuild) fazem isso por padrão.

---

## 6. Quando Usar / Não Usar

**Use cache agressivo (`max-age=31536000, immutable`):**
- Assets estáticos com hashes de conteúdo no nome do arquivo
- Fontes, bundles de vendedor, imagens

**Use `no-cache` (sempre revalidar, usar 304 para economizar banda):**
- Páginas HTML (para que usuários sempre obtenham um shell atualizado que referencia os assets hashados mais recentes)
- Respostas de API que podem mudar

**Use `no-store`:**
- Respostas com dados sensíveis (bancário, saúde, tokens de auth, PII)

**Use `private`:**
- Conteúdo personalizado que CDNs nunca devem armazenar em cache (dashboards de usuário, páginas de conta)

**Use `stale-while-revalidate`:**
- Feeds, dashboards, leaderboards — tolera brevíssima desatualização para velocidade

**Use `s-maxage`:**
- Quando o TTL do CDN deve diferir do TTL do browser (cache CDN mais longo, cache do browser mais curto)

---

## 7. Cenário do Mundo Real

### O bug de "JS desatualizado após deploy"

**Situação:** Você faz deploy de uma nova versão do seu single-page app. Alguns usuários continuam vendo o bundle JavaScript antigo e experimentam uma UI quebrada misturando estruturas de código antigas e novas.

**Investigação:**
```bash
# Verificar quais headers de cache o CDN está retornando
curl -I https://seuapp.com/assets/app.js

# HTTP/2 200 OK
# cache-control: public, max-age=86400
# etag: "hash-antigo-abc123"
# age: 43210
# x-cache: HIT
```

O arquivo está sendo servido do cache CDN com TTL de 24 horas. Você fez deploy de um novo arquivo na mesma URL, mas o CDN não buscará a nova versão por mais ~11 horas (86400 - 43210 = ~43190 segundos).

**Causa raiz:** Seu build gera `app.js` (sem hash de conteúdo no nome) e você definiu cache CDN de 24 horas sem uma forma de invalidar.

**Correção 1 — Nomes de arquivo com hash de conteúdo + cache imutável (solução correta a longo prazo):**

Configure seu bundler para gerar nomes com hash:
```javascript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash][extname]',
      },
    },
  },
};
```

Depois defina os headers de cache:
```nginx
# nginx — páginas HTML sempre revalidam
location ~* \.html$ {
  add_header Cache-Control "no-cache";
}

# JS/CSS/imagens com hash no nome — cache 1 ano, imutável
location ~* \.[0-9a-f]{8}\.(js|css|woff2|png|webp)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}
```

**Correção 2 — Purga de cache CDN no deploy (correção emergencial ou sistemas legados):**
```bash
# Cloudflare — purga arquivo específico após deploy
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache" \
  -H "Authorization: Bearer ${CF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://seuapp.com/app.js"]}'

# Ou purga tudo (grosseiro mas eficaz)
--data '{"purge_everything":true}'
```

**A arquitetura correta:**
```
index.html      →  Cache-Control: no-cache
app.a3f8e1.js   →  Cache-Control: public, max-age=31536000, immutable
vendor.8c2d1f.js→  Cache-Control: public, max-age=31536000, immutable
style.4b9e2a.css→  Cache-Control: public, max-age=31536000, immutable
/api/*          →  Cache-Control: private, no-store
```

---

## 8. Perguntas de Entrevista

**P1: Descreva tudo o que acontece quando você digita uma URL e pressiona Enter.**

R: O browser verifica seu próprio cache DNS e depois o cache do SO e `/etc/hosts`. Em caso de cache miss, o SO consulta o recursive resolver configurado (ex: 8.8.8.8) via UDP. O resolver percorre a hierarquia DNS: servidores raiz → servidores de nomes TLD → servidor de nomes autoritativo para o domínio, retornando o registro A/AAAA e armazenando em cache por TTL. Com o IP, o browser abre um socket TCP via handshake de três vias (SYN → SYN-ACK → ACK), custando 1 RTT. Para HTTPS, segue um handshake TLS 1.3 (ClientHello, ServerHello + certificado + Finished, client Finished), custando mais 1 RTT. O browser envia a requisição HTTP GET. O servidor processa e retorna uma resposta. O browser analisa o HTML em um DOM, analisa CSS em um CSSOM, executa JavaScript, une DOM+CSSOM em uma render tree, executa layout para calcular posições, pinta pixels e compõe camadas na GPU.

---

**P2: O que é o handshake TLS e por que ele importa?**

R: TLS fornece criptografia, autenticação e integridade para a conexão. No TLS 1.3, o handshake é 1 RTT: o cliente envia um ClientHello com suites de cifra suportadas e um compartilhamento de chave pública Diffie-Hellman. O servidor responde com sua cifra escolhida, seu compartilhamento de chave DH, um certificado assinado por uma CA confiável (provando sua identidade) e uma mensagem Finished. Ambos os lados derivam independentemente o mesmo segredo compartilhado da troca DH — nenhum segredo é jamais transmitido. A partir daí, todos os dados são criptografados. Importa porque sem TLS, qualquer intermediário de rede (provedor, roteador de café, governo) pode ler ou silenciosamente modificar o tráfego.

---

**P3: Qual é a diferença entre HTTP/2 e HTTP/1.1?**

R: HTTP/2 usa framing binário em vez de texto simples, habilitando multiplexing — múltiplos streams de requisição/resposta concorrentes sobre uma única conexão TCP, eliminando a limitação de requisições seriais do HTTP/1.1. Também adiciona compressão de headers HPACK (grande economia para headers repetidos como cookies) e server push opcional. HTTP/1.1 exigia hacks como domain sharding e agrupamento de recursos para compensar sua serialização por conexão. A limitação principal que HTTP/2 não corrige é o head-of-line blocking TCP — um único pacote perdido paralisa todos os streams. HTTP/3 + QUIC resolve isso usando UDP com confiabilidade independente por stream.

---

**P4: O que `Cache-Control: no-cache` significa exatamente?**

R: Significa que a resposta PODE ser armazenada em cache, mas o cache DEVE revalidar com o servidor de origem antes de servi-la em requisições subsequentes. Se o ETag ou Last-Modified do servidor indica que nada mudou, ele retorna 304 Not Modified (sem corpo) e a versão em cache é servida — economizando banda sem sacrificar frescor. Não significa "não armazene em cache". Se você quer evitar o armazenamento completamente (para dados sensíveis), use `Cache-Control: no-store`.

---

**P5: O que é um CDN e como funciona?**

R: Uma Content Delivery Network é uma rede globalmente distribuída de servidores chamados nós de borda ou Points of Presence (PoPs). Quando um usuário solicita um recurso, roteamento DNS ou Anycast os direciona para o nó de borda geograficamente mais próximo. Se o edge tem o conteúdo em cache (porque a resposta tinha `Cache-Control: public`), ele responde imediatamente sem chegar ao servidor de origem — reduzindo tanto a latência (geograficamente mais próximo) quanto a carga do servidor de origem. Quando o cache está frio ou obsoleto, o edge busca da origem, armazena a resposta e serve requisições futuras do cache.

---

**P6: O que é CORS e por que existe?**

R: CORS (Cross-Origin Resource Sharing) é um mecanismo do browser que aplica a Same-Origin Policy: JavaScript só pode ler respostas da mesma origem (esquema + hostname + porta) que a página, a menos que o servidor permita explicitamente. Existe para impedir que scripts maliciosos em `evil.com` leiam silenciosamente respostas autenticadas de `seubanco.com` usando os cookies da vítima. CORS permite que servidores listem origens específicas na whitelist via `Access-Control-Allow-Origin`. A restrição é apenas no browser — requisições servidor-a-servidor e curl nunca estão sujeitas a CORS.

---

**P7: Qual é a diferença entre um redirect 301 e 302?**

R: 301 Moved Permanently — tanto browsers quanto mecanismos de busca armazenam em cache este redirect indefinidamente e transferem "equity de link" SEO para a URL destino. Uma vez em cache, o browser pula o redirect completamente e vai diretamente para a nova URL. 302 Found (Redirect Temporário) — browsers não armazenam em cache; mecanismos de busca mantêm a URL original indexada. Use 301 ao mover conteúdo permanentemente (migração de domínio, reestruturação de URL). Use 302 para situações temporárias (testes A/B, páginas de manutenção). Também relevante: 307 (Redirect Temporário) e 308 (Redirect Permanente) preservam o método HTTP original (um POST continua POST), enquanto 301/302 tipicamente convertem POST para GET.

---

**P8: O que é o TTL do DNS e como afeta deploys?**

R: TTL (Time To Live) é o número de segundos que um registro DNS deve ser armazenado em cache por resolvers antes de re-consultarem. Um TTL de 3600 significa que sua mudança de endereço IP de domínio pode não ser visível por até 1 hora após você atualizá-lo, porque todos os resolvers que já o armazenaram em cache continuarão usando o valor antigo. Para migrações: reduza o TTL para 60 segundos pelo menos 24 horas antes da migração (para deixar os caches com TTL longo expirarem), faça a mudança DNS, verifique que funciona, depois eleve o TTL de volta ao normal (3600+). Após mudanças, use `dig @8.8.8.8 example.com` para verificar a propagação sem bater no cache local.

---

## 9. Exercícios

**Exercício 1 — Trace uma requisição completa com curl:**
```bash
# Medir o detalhamento de timing para vários sites e comparar
for site in github.com google.com cloudflare.com; do
  echo "=== $site ==="
  curl -o /dev/null -s -w "\
  DNS:   %{time_namelookup}s\n\
  TCP:   %{time_connect}s\n\
  TLS:   %{time_appconnect}s\n\
  TTFB:  %{time_starttransfer}s\n\
  Total: %{time_total}s\n\n" \
    "https://$site"
done
```

Objetivos:
1. Use `curl -I` para ver apenas headers de resposta — identifique Cache-Control, ETag e Content-Type
2. Use `curl -v` para ver a cadeia de certificados TLS e a versão exata do TLS usado
3. Compare `--http1.1` vs `--http2` para a mesma URL — observe diferenças nos headers
4. Encontre um site que serve HTTP/3 e use `curl --http3` se seu curl suportar

---

**Exercício 2 — Implemente um servidor HTTP/1.1 mínimo em Node.js:**

Construa um servidor TCP bruto (usando o módulo `net`, não `http`) que:
- Analisa a primeira linha de uma requisição HTTP para extrair método e caminho
- Retorna `200 OK` com corpo JSON `{ "path": "..." }` para `GET /`
- Retorna `404 Not Found` com erro JSON para caminhos desconhecidos
- Retorna `405 Method Not Allowed` para requisições não-GET
- Define corretamente o header `Content-Length` (não Content-Transfer-Encoding)
- Pode ser testado end-to-end com `curl http://localhost:3000/`

Dica: Headers HTTP e corpo são separados por `\r\n\r\n`. A linha de requisição é `MÉTODO CAMINHO HTTP/1.1`. Todos os finais de linha em HTTP são `\r\n`, não apenas `\n`.

---

**Exercício 3 — Analise uma waterfall no Chrome DevTools:**

1. Abra Chrome DevTools → aba Network, marque "Disable cache", recarregue
2. Visite uma página com muito conteúdo (site de notícias ou homepage de e-commerce)
3. Responda a estas perguntas:
   - Qual é o TTFB para o documento HTML?
   - Há recursos que bloqueiam a renderização (procure por barras longas de "Waiting" em CSS/JS antes do primeiro paint)?
   - Quais arquivos são servidos com nomes de arquivo com hash vs nomes genéricos (`app.js` vs `app.a3f8e1.js`)?
   - As imagens estão otimizadas como WebP ou AVIF, ou ainda são JPEG/PNG?
   - O site usa HTTP/2 ou HTTP/3? (coluna Protocol)
4. Habilite o cache e recarregue — quais recursos retornam 304? Quais retornam do cache de disco/memória?

---

## 10. Leituras Complementares

- **MDN — HTTP**: https://developer.mozilla.org/pt-BR/docs/Web/HTTP — referência completa para todos os conceitos HTTP
- **"High Performance Browser Networking" por Ilya Grigorik** (gratuito online): https://hpbn.co — o livro definitivo sobre TCP, TLS, HTTP/1.1, HTTP/2, WebSockets e WebRTC
- **RFC 9114 — HTTP/3**: https://www.rfc-editor.org/rfc/rfc9114
- **"How Browsers Work"**: https://web.dev/articles/howbrowserswork — detalhamento do pipeline de renderização
- **web.dev — Core Web Vitals**: https://web.dev/vitals/ — framework de métricas de performance do Google (LCP, INP, CLS)
- **Cloudflare Learning Center**: https://www.cloudflare.com/learning/ — excelentes explicações de DNS, CDN e TLS com diagramas
- **"What happens when..."** (GitHub): https://github.com/alex/what-happens-when — mergulho profundo exaustivo mantido pela comunidade em cada camada
- **Guia HTTP/2 do Google**: https://developers.google.com/web/fundamentals/performance/http2
- **Referência do comando dig**: `man dig` — use `dig +trace example.com` para ver toda a resolução DNS
