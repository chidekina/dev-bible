# Códigos de Status HTTP Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## 1xx — Informacional

| Código | Nome | Caso de Uso |
|--------|------|-------------|
| 100 | Continue | Servidor recebeu os headers; cliente pode prosseguir com o corpo |
| 101 | Switching Protocols | Upgrade para WebSocket ou HTTP/2 |
| 102 | Processing | Servidor recebeu a requisição, ainda processando (WebDAV) |
| 103 | Early Hints | Servidor envia headers preliminares enquanto prepara a resposta |

---

## 2xx — Sucesso

| Código | Nome | Caso de Uso |
|--------|------|-------------|
| **200** | OK | Resposta padrão de sucesso (GET, PUT, PATCH) |
| **201** | Created | Recurso criado (POST com novo recurso; incluir header `Location`) |
| **202** | Accepted | Requisição aceita para processamento assíncrono (jobs em fila) |
| **204** | No Content | Sucesso sem corpo de resposta (DELETE, PUT sem retorno) |
| 205 | Reset Content | Cliente deve reiniciar o formulário/visualização |
| 206 | Partial Content | Resposta a requisição com header `Range` (streaming, downloads retomáveis) |
| 207 | Multi-Status | Múltiplos status para múltiplas operações (WebDAV) |
| 208 | Already Reported | Membro da coleção já enumerado (WebDAV) |
| 226 | IM Used | Resposta é resultado de delta encoding |

**Padrões de API:**
```
POST   /users       → 201 Created  + Location: /users/42
GET    /users/42    → 200 OK       + body
PATCH  /users/42    → 200 OK       + body (ou 204 sem body)
DELETE /users/42    → 204 No Content
POST   /jobs        → 202 Accepted + { jobId: "..." }
```

---

## 3xx — Redirecionamento

| Código | Nome | Caso de Uso |
|--------|------|-------------|
| 300 | Multiple Choices | Múltiplas representações disponíveis |
| **301** | Moved Permanently | Recurso permanentemente em nova URL; atualize favoritos |
| **302** | Found | Redirecionamento temporário (evite — use 303 ou 307) |
| **303** | See Other | Após POST, redirecionar para GET (padrão Post/Redirect/Get) |
| 304 | Not Modified | Versão em cache está atualizada; sem corpo enviado (ETag / If-Modified-Since) |
| 307 | Temporary Redirect | Redirecionamento temporário; método não deve mudar |
| **308** | Permanent Redirect | Redirecionamento permanente; método não deve mudar |

**Guia de redirecionamentos:**

| Cenário | Usar |
|---------|------|
| Mudança de domínio, permanente | 301 |
| HTTP → HTTPS, permanente | 301 |
| Após POST de formulário | 303 |
| Endpoint de API renomeado, permanente | 308 |
| Redirecionamento de manutenção, temporário | 307 |
| Resposta em cache ainda válida | 304 |

---

## 4xx — Erros do Cliente

| Código | Nome | Caso de Uso |
|--------|------|-------------|
| **400** | Bad Request | Sintaxe malformada, corpo inválido, falha de validação |
| **401** | Unauthorized | Autenticação necessária ou credenciais inválidas (incluir `WWW-Authenticate`) |
| **403** | Forbidden | Autenticado mas sem autorização para este recurso |
| **404** | Not Found | Recurso não existe (também usado para ocultar existência de recursos proibidos) |
| **405** | Method Not Allowed | Método HTTP não suportado neste endpoint (incluir header `Allow`) |
| 406 | Not Acceptable | Servidor não consegue produzir resposta compatível com o header `Accept` |
| 407 | Proxy Authentication Required | É necessário autenticar com o proxy |
| **408** | Request Timeout | Cliente demorou demais para enviar a requisição |
| **409** | Conflict | Conflito de estado — recurso duplicado, falha de lock otimista |
| 410 | Gone | Recurso permanentemente deletado (mais forte que 404) |
| 411 | Length Required | Header `Content-Length` obrigatório |
| 412 | Precondition Failed | Pré-condição de requisição condicional não atendida (ETag divergente no PUT) |
| 413 | Content Too Large | Corpo da requisição muito grande |
| 414 | URI Too Long | URL muito longa |
| 415 | Unsupported Media Type | Content-Type não suportado |
| **416** | Range Not Satisfiable | Intervalo solicitado não disponível |
| 417 | Expectation Failed | Header `Expect` não pode ser atendido |
| **418** | I'm a Teapot | Easter egg (RFC 2324) — às vezes usado para requisições intencionalmente recusadas |
| 421 | Misdirected Request | Requisição direcionada a servidor incapaz de responder |
| **422** | Unprocessable Entity | Corpo sintaticamente correto mas semanticamente inválido (preferido para erros de validação) |
| 423 | Locked | Recurso está bloqueado (WebDAV) |
| 424 | Failed Dependency | Operação falhou porque uma operação dependente falhou |
| 425 | Too Early | Risco de ataque de replay (Early Data) |
| **426** | Upgrade Required | Cliente deve atualizar o protocolo (ex: TLS) |
| **429** | Too Many Requests | Limite de requisições excedido (incluir header `Retry-After`) |
| 431 | Request Header Fields Too Large | Headers muito grandes |
| 451 | Unavailable For Legal Reasons | Censurado por razões legais |

**Árvore de decisão para 4xx:**
```
Credenciais ausentes/inválidas?  → 401
Credenciais válidas, sem acesso? → 403
Recurso não encontrado?          → 404
Duplicado / conflito?            → 409
Validação falhou?                → 422 (preferido) ou 400
Rate limit atingido?             → 429 + Retry-After
Método HTTP errado?              → 405 + header Allow
```

---

## 5xx — Erros do Servidor

| Código | Nome | Caso de Uso |
|--------|------|-------------|
| **500** | Internal Server Error | Exceção não tratada, bug — falha genérica do servidor |
| **501** | Not Implemented | Método ou funcionalidade ainda não implementado |
| **502** | Bad Gateway | Servidor upstream retornou resposta inválida (problema de proxy reverso) |
| **503** | Service Unavailable | Servidor sobrecarregado ou em manutenção (incluir `Retry-After`) |
| **504** | Gateway Timeout | Servidor upstream não respondeu a tempo |
| 505 | HTTP Version Not Supported | Versão HTTP da requisição não suportada |
| 506 | Variant Also Negotiates | Erro de configuração na negociação de conteúdo |
| 507 | Insufficient Storage | Servidor sem armazenamento suficiente (WebDAV) |
| 508 | Loop Detected | Loop infinito detectado (WebDAV) |
| 510 | Not Extended | Extensões adicionais necessárias |
| 511 | Network Authentication Required | Deve autenticar para acessar a rede (portais cativos) |

**Notas operacionais para 5xx:**
- 500 → corrija o bug; nunca exponha stack traces
- 502/504 → verifique saúde do serviço upstream e timeouts
- 503 → adicione lógica de retry nos clientes + retorne `Retry-After`

---

## Headers Comuns Associados a Status Codes

| Status | Headers Comuns |
|--------|---------------|
| 201 | `Location: /resources/42` |
| 204 | (sem corpo, sem Content-Type) |
| 301, 302, 307, 308 | `Location: https://nova-url.com` |
| 304 | `ETag`, `Last-Modified`, `Cache-Control` |
| 401 | `WWW-Authenticate: Bearer` |
| 405 | `Allow: GET, POST` |
| 429 | `Retry-After: 60` (segundos) |
| 503 | `Retry-After: 120` |

---

## Convenções de Design de API REST

```
GET    /resources          → 200 OK         (listagem)
GET    /resources/:id      → 200 OK         (individual)
                           → 404 Not Found  (não encontrado)
POST   /resources          → 201 Created    (novo recurso)
                           → 400/422        (corpo inválido)
                           → 409 Conflict   (duplicado)
PUT    /resources/:id      → 200 OK         (substituído, retornar corpo)
                           → 204 No Content (substituído, sem corpo)
                           → 404 Not Found
PATCH  /resources/:id      → 200 OK
                           → 422            (erro de validação)
DELETE /resources/:id      → 204 No Content (sucesso)
                           → 404 Not Found
                           → 409 Conflict   (não pode deletar — em uso)
```

---

## Cache e Requisições Condicionais

| Header | Direção | Finalidade |
|--------|---------|-----------|
| `Cache-Control: no-store` | Resposta | Nunca armazenar em cache |
| `Cache-Control: no-cache` | Resposta | Armazenar mas revalidar sempre |
| `Cache-Control: max-age=3600` | Resposta | Cache por 1 hora |
| `Cache-Control: s-maxage=3600` | Resposta | TTL de cache compartilhado (CDN) |
| `ETag: "abc123"` | Resposta | Fingerprint da versão do recurso |
| `Last-Modified: Thu, 01 Jan 2025 00:00:00 GMT` | Resposta | Hora da última alteração |
| `If-None-Match: "abc123"` | Requisição | Retornar 304 se ETag corresponder |
| `If-Modified-Since: ...` | Requisição | Retornar 304 se não modificado |
| `Vary: Accept-Encoding` | Resposta | Cache separado por encoding |

---

## Upgrade para WebSocket

```
Cliente: GET /ws HTTP/1.1
         Upgrade: websocket
         Connection: Upgrade

Servidor: HTTP/1.1 101 Switching Protocols
          Upgrade: websocket
          Connection: Upgrade
          Sec-WebSocket-Accept: <hash>
```
