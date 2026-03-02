# Desafios de Segurança

> Exercícios práticos de segurança cobrindo autenticação, autorização, código seguro e hardening. Cada exercício reflete um cenário real de produção. Stack assumida: Node.js 20+, TypeScript, Fastify, Zod, PostgreSQL. Nível alvo: intermediário a avançado.

---

## Exercício 1 — Hash de Senha com bcrypt (Fácil)

**Cenário:** Você está construindo um sistema de cadastro e login. As senhas devem ser armazenadas com segurança para que uma vazamento de banco de dados não exponha as credenciais dos usuários.

**Requisitos:**
- Implemente `hashPassword(plain: string): Promise<string>` usando bcrypt com fator de custo 12.
- Implemente `verifyPassword(plain: string, hash: string): Promise<boolean>`.
- Ambas as funções devem estar em um módulo `src/lib/password.ts`.
- Nunca registre ou retorne a senha em texto puro em nenhum ponto da pilha de chamadas.
- O fator de custo deve ser configurável via variável de ambiente `BCRYPT_ROUNDS` (padrão: 12, mínimo: 10).

**Critérios de Aceite:**
- [ ] Armazenar a mesma senha duas vezes produz dois hashes diferentes (unicidade do salt).
- [ ] `verifyPassword('correta', hash)` retorna `true`; `verifyPassword('errada', hash)` retorna `false`.
- [ ] Reduzir `BCRYPT_ROUNDS` abaixo de 10 lança um erro de configuração na inicialização — não no momento do hash.
- [ ] Testes unitários cobrem: verificação correta, senha errada, string de hash adulterada.
- [ ] Nenhuma senha em texto puro aparece em nenhuma saída de log (verificado por um teste que captura a saída do `console`).

**Dicas:**
1. Use o pacote `bcryptjs` (JS puro, sem bindings nativos) ou `bcrypt` (nativo, mais rápido em produção).
2. `bcrypt.hash(password, rounds)` retorna um hash com salt. O salt é embutido na string do hash — não precisa armazená-lo separadamente.
3. Para a proteção de configuração: leia `BCRYPT_ROUNDS` no momento do carregamento do módulo, converta para int e lance erro se `< 10`.
4. Nos testes, defina `BCRYPT_ROUNDS=10` para manter os testes rápidos sem comprometer a correção do algoritmo.

---

## Exercício 2 — Middleware de Auth JWT (Médio)

**Cenário:** Construa um sistema completo de autenticação JWT: emissão de tokens no login, verificação em rotas protegidas e implementação de rotação de refresh token.

**Requisitos:**
- `POST /auth/login`: emite `{ accessToken, refreshToken }`. Access token expira em 15 minutos; refresh token expira em 7 dias.
- `POST /auth/refresh`: aceita um refresh token válido, emite um novo access token **e** um novo refresh token (rotação). Invalida o refresh token anterior.
- Um hook `preHandler` do Fastify verifica o header `Authorization: Bearer <token>` nas rotas protegidas.
- Refresh tokens são armazenados no Redis. Na rotação, exclua o token antigo e defina o novo.
- No logout (`POST /auth/logout`), exclua o refresh token do Redis.

**Critérios de Aceite:**
- [ ] Usar um access token expirado em uma rota protegida retorna `401` com `{ code: "TOKEN_EXPIRED" }`.
- [ ] Usar um refresh token revogado (após rotação) retorna `401` — prevenção de ataque de replay.
- [ ] O refresh token no Redis usa o formato de chave `rt:<userId>:<tokenId>` com TTL de 7 dias.
- [ ] O hook `preHandler` passa `req.user` (payload decodificado) ao handler da rota.
- [ ] Teste de integração cobre o ciclo completo de rotação: login → refresh → tentativa de reuso do refresh token antigo → falha.

**Dicas:**
1. Use `@fastify/jwt` para simplificar sign/verify. Configure com `secret`, `sign: { expiresIn: '15m' }`.
2. Incorpore um `jti` (JWT ID, um `crypto.randomUUID()`) em ambos os tokens. Use `jti` como sufixo da chave Redis para verificações de revogação.
3. Endpoint de refresh: verifique o refresh token recebido, confirme que o `jti` existe no Redis, então exclua atomicamente a chave antiga e defina a nova.
4. Decodifique sem verificação primeiro para extrair o `jti` para a verificação de revogação apenas se precisar do tipo de erro — sempre verifique a assinatura também.

---

## Exercício 3 — Middleware de Token CSRF para Fastify (Médio)

**Cenário:** Seu aplicativo Fastify serve um frontend renderizado no servidor. Você precisa proteger endpoints que alteram estado contra ataques Cross-Site Request Forgery.

**Requisitos:**
- Implemente um plugin Fastify que gera um token CSRF e o define como um cookie (`csrf-token`, `HttpOnly: false` para que o JS possa lê-lo).
- Em requisições `POST`, `PUT`, `PATCH`, `DELETE`, valide o header `X-CSRF-Token` contra o valor do cookie.
- O token deve ser uma string hexadecimal aleatória de 32 bytes criptograficamente segura, regenerada por sessão.
- Caminhos isentos (ex: `/health`, `/webhooks/*`) da validação CSRF.
- O plugin deve ser configurável: `{ exemptPaths: string[], cookieName: string, headerName: string }`.

**Critérios de Aceite:**
- [ ] Uma requisição `POST` sem o header `X-CSRF-Token` retorna `403`.
- [ ] Uma requisição `POST` com valor de header incorreto retorna `403`.
- [ ] Uma requisição `GET` sem header CSRF passa (CSRF não é necessário para métodos seguros).
- [ ] A comparação do token usa `crypto.timingSafeEqual` — não `===`.
- [ ] `/webhooks/stripe` está isento e aceita `POST` sem token CSRF.

**Dicas:**
1. Registre o plugin com `fastify.register(csrfPlugin, { exemptPaths: ['/health', '/webhooks/*'] })`.
2. No hook `onRequest`: ignore métodos seguros (`GET`, `HEAD`, `OPTIONS`). Ignore caminhos isentos.
3. Geração de token: `crypto.randomBytes(32).toString('hex')`.
4. Comparação de tempo constante: ambos os valores devem ter o mesmo tamanho em bytes — normalize antes de comparar.
5. Armazene o token esperado na sessão (não apenas no cookie) para prevenir ataques cookie-to-header de contextos same-site.

---

## Exercício 4 — Auditoria de Segurança em Dockerfile (Médio)

**Cenário:** Um colega submeteu o seguinte Dockerfile. Identifique todos os problemas de segurança e produza uma versão endurecida.

**Dockerfile fornecido (intencionalmente falho):**
```dockerfile
FROM node:latest

WORKDIR /app

COPY . .

RUN npm install

ENV NODE_ENV=production
ENV DATABASE_URL=postgres://admin:secret@db:5432/myapp
ENV JWT_SECRET=supersecretkey123

RUN npm run build

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

**Requisitos:**
- Identifique pelo menos 6 problemas distintos de segurança/boas práticas no Dockerfile acima.
- Produza um Dockerfile endurecido que corrija todos os problemas identificados.
- Use uma build multi-estágio para separar dependências de build das de runtime.
- A imagem final deve rodar como usuário não-root.

**Critérios de Aceite:**
- [ ] Nenhum segredo é hardcoded na imagem (segredos são injetados em runtime via env ou secrets manager).
- [ ] Usa uma tag de imagem base fixada (ex: `node:20.11-alpine3.19`), não `latest`.
- [ ] A imagem final usa usuário não-root (`USER node` ou um `appuser` customizado).
- [ ] `npm install` no estágio final instala apenas dependências de produção (`--omit=dev`).
- [ ] O estágio de build e o de runtime são separados — `node_modules` de dev não está na imagem final.
- [ ] Arquivo `.dockerignore` está incluído, excluindo no mínimo: `.git`, `node_modules`, `.env*`, `*.log`.

**Dicas:**
1. Problemas a encontrar: tag `latest`, segredos hardcoded, usuário root, sem `.dockerignore`, dependências de dev em prod, sem health check, arquivos desnecessários copiados.
2. Multi-estágio: `FROM node:20-alpine AS builder` → instala todas as deps + build → `FROM node:20-alpine AS runtime` → copia apenas `dist/` + `node_modules` de prod.
3. Não-root: Alpine inclui um usuário `node`. Adicione `USER node` antes de `CMD`.
4. Segredos: remova as linhas `ENV` com segredos completamente. Documente que devem ser passados via `docker run -e` ou secrets manager em runtime.

---

## Exercício 5 — Helmet.js com CSP Estrita (Fácil)

**Cenário:** Sua API Express/Fastify também serve um dashboard de administração. Adicione headers de segurança incluindo uma Content Security Policy estrita.

**Requisitos:**
- Integre `helmet` (Express) ou `@fastify/helmet` (Fastify).
- Configure uma CSP que permita: scripts e estilos apenas de `'self'` e seu CDN (`https://cdn.seudominio.com`), imagens de `'self'` e `data:`, fontes de `'self'`.
- Bloqueie todo embedding via `<frame>` e `<iframe>` (`X-Frame-Options: DENY`).
- Habilite `Strict-Transport-Security` com max-age de 1 ano e `includeSubDomains`.
- Desabilite o header `X-Powered-By`.

**Critérios de Aceite:**
- [ ] Header `Content-Security-Policy` está presente em todas as respostas.
- [ ] Scripts inline são bloqueados pela CSP (sem `'unsafe-inline'`).
- [ ] `X-Frame-Options: DENY` está configurado.
- [ ] `Strict-Transport-Security: max-age=31536000; includeSubDomains` está configurado.
- [ ] Header `X-Powered-By` está ausente em todas as respostas.
- [ ] Teste que carregar a página em um browser headless com os headers ativos não gera violações de CSP para assets legítimos.

**Dicas:**
1. `@fastify/helmet` encapsula o Helmet — passe a config CSP via `contentSecurityPolicy: { directives: { ... } }`.
2. CSP `default-src 'self'` é a base — adicione substituições específicas para `script-src`, `style-src`, `img-src`, `font-src`.
3. Para permitir o CDN: `script-src: ["'self'", "https://cdn.seudominio.com"]`.
4. Para SPAs que usam `eval` (alguns bundlers fazem isso): prefira a abordagem baseada em nonce em vez de `'unsafe-eval'`.

---

## Exercício 6 — Rate Limiter com Janela Deslizante (Médio)

**Cenário:** Implemente um middleware de rate limiting usando o algoritmo de janela deslizante com Redis. Deve ser preciso e performático.

**Requisitos:**
- Limite: 100 requisições por janela deslizante de 60 segundos por cliente (identificado por IP ou API key).
- Retorne `429` com headers `Retry-After` (segundos até a requisição mais antiga sair da janela) e `X-RateLimit-Remaining`.
- O algoritmo deve ser atômico (sem condições de corrida sob requisições concorrentes).
- Falha no Redis deve falhar aberto — requisições são permitidas com header `X-RateLimit-Bypassed: true`.

**Critérios de Aceite:**
- [ ] Um cliente enviando 101 requisições em 60 segundos recebe `429` na 101ª.
- [ ] Após 61 segundos, o cliente pode enviar mais 100 requisições (a janela deslizou além das entradas mais antigas).
- [ ] Duas requisições concorrentes no limite de 100 resultam em exatamente uma aceita (atomicidade via script Lua).
- [ ] O valor de `Retry-After` é preciso — não apenas um `60` fixo, mas o tempo real até um slot se abrir.
- [ ] Teste unitário não requer instância real de Redis (use `ioredis-mock` ou injete uma store fake).

**Dicas:**
1. Janela deslizante com sorted set: `ZADD key <now_ms> <uuid>` + `ZREMRANGEBYSCORE key 0 <now_ms - windowMs>` + `ZCOUNT key -inf +inf` — envolva os três em um script Lua para atomicidade.
2. `Retry-After`: obtenha o score da entrada mais antiga (`ZRANGE key 0 0 WITHSCORES`) e compute `oldest + windowMs - now`.
3. TTL da chave: após o script Lua, execute `EXPIRE key windowSeconds` para limpeza automática de clientes inativos.
4. O script Lua retorna tanto o count quanto o score mais antigo para calcular `Retry-After` em uma única viagem ao Redis.

---

## Exercício 7 — Validação de Entrada com Zod em Todas as Rotas (Fácil)

**Cenário:** Uma API Fastify existente não possui validação de entrada. O `req.body` bruto é passado diretamente para a camada de banco de dados, causando erros 500 em entradas malformadas e potenciais vetores de injeção.

**Requisitos:**
- Adicione schemas Zod para corpo da requisição, query params e params de URL em cada rota.
- Retorne erros estruturados `422 Unprocessable Entity` com detalhes por campo em caso de falha de validação.
- Conecte Zod ao Fastify usando `fastify-type-provider-zod` para que os schemas gerem tipos TypeScript automaticamente.
- Todos os inputs de string devem ser `.trim()`'d e limitados em tamanho com limites adequados.
- IDs em params de URL devem validar como UUIDs (`z.string().uuid()`).

**Critérios de Aceite:**
- [ ] `POST /users` com um campo obrigatório faltando retorna `422` com `{ errors: [{ field: "email", message: "Required" }] }`.
- [ ] `GET /users/:id` com `id = "nao-e-uuid"` retorna `422`, não um erro de banco de dados.
- [ ] Os tipos TypeScript dos handlers de rotas são inferidos do schema Zod — sem type assertions manuais.
- [ ] Um campo de string com 10.000 caracteres é rejeitado com um erro descritivo.
- [ ] `GET /items?page=-1` é rejeitado (page deve ser um inteiro positivo).

**Dicas:**
1. Instale `fastify-type-provider-zod` e chame `fastify.withTypeProvider<ZodTypeProvider>()` antes de registrar rotas.
2. Registre o schema: `fastify.route({ schema: { body: createUserSchema, params: z.object({ id: z.string().uuid() }) }, handler })`.
3. Handler de erro customizado: em `fastify.setErrorHandler`, detecte `ZodError` e mapeie os issues para o formato estruturado.
4. Limites de tamanho: `z.string().max(255)` para nomes, `z.string().max(10000)` para conteúdo. Faça trim com `.trim()` encadeado.

---

## Exercício 8 — Correção de Vulnerabilidades de SQL Injection (Médio)

**Cenário:** Um desenvolvedor júnior escreveu o seguinte construtor de queries. Encontre as vulnerabilidades de SQL injection e reescreva as funções afetadas usando queries parametrizadas.

**Código fornecido (intencionalmente vulnerável):**
```typescript
async function getUserByEmail(email: string) {
  const query = `SELECT * FROM users WHERE email = '${email}'`;
  return db.query(query);
}

async function searchProducts(name: string, category: string) {
  const query = `
    SELECT * FROM products
    WHERE name LIKE '%${name}%'
    AND category = '${category}'
    ORDER BY created_at DESC
  `;
  return db.query(query);
}

async function updateUserRole(userId: string, role: string) {
  const query = `UPDATE users SET role = '${role}' WHERE id = ${userId}`;
  return db.query(query);
}
```

**Requisitos:**
- Identifique todos os pontos de injeção nas três funções acima.
- Reescreva cada função usando queries parametrizadas (sem interpolação de string de input do usuário).
- Adicione validação Zod para os inputs de cada função antes da query ser executada.
- Escreva um teste que demonstre que o código original era vulnerável (use um payload como `'; DROP TABLE users; --`).

**Critérios de Aceite:**
- [ ] `getUserByEmail("' OR '1'='1")` retorna zero resultados (não todos os usuários).
- [ ] `searchProducts("'; DROP TABLE products; --", "shoes")` não remove a tabela.
- [ ] `updateUserRole("1 OR 1=1", "admin")` não atualiza todos os usuários.
- [ ] As três funções usam placeholders `$1`, `$2` (ou binding de parâmetros de ORM) — sem template literals com dados do usuário.
- [ ] O campo `role` em `updateUserRole` é validado contra uma lista de permissão (`['user', 'admin', 'moderator']`).

**Dicas:**
1. Query parametrizada no PostgreSQL: `db.query('SELECT * FROM users WHERE email = $1', [email])`.
2. `LIKE` com parâmetros: `db.query("SELECT * FROM products WHERE name LIKE $1", [`%${name}%`])` — os wildcards `%` vão no valor bindado, não na string SQL.
3. Nunca use interpolação de string para nomes de coluna em `ORDER BY` — use um mapa de lista de permissão: `const allowed = { created_at: 'created_at', name: 'name' }`.
4. Zod para `role`: `z.enum(['user', 'admin', 'moderator'])`.

---

## Exercício 9 — Dependabot + npm audit no GitHub Actions (Fácil)

**Cenário:** Seu projeto não possui escaneamento automático de vulnerabilidades de dependências. Configure o Dependabot para PRs automáticos e adicione `npm audit` como gate de CI.

**Requisitos:**
- Crie um `.github/dependabot.yml` que verifica dependências npm semanalmente com alvo no branch `main`.
- Adicione um job `security-audit` em `.github/workflows/ci.yml` que roda `npm audit --audit-level=high` e falha na build em vulnerabilidades altas ou críticas.
- O job de auditoria deve rodar em todo `push` para `main` e em todo `pull_request`.
- Configure o Dependabot para atualizar automaticamente apenas versões `patch` e `minor`; versões `major` requerem revisão manual.
- Adicione um step `license-checker` que falha se alguma dependência usa uma licença não-permissiva (GPL, AGPL).

**Critérios de Aceite:**
- [ ] `.github/dependabot.yml` é YAML válido e referencia o ecossistema de pacotes correto (`npm`).
- [ ] O job de CI `security-audit` roda `npm audit --audit-level=high` e sai com código não-zero na falha.
- [ ] A saída JSON de `npm audit --json` é enviada como artifact do workflow para depuração.
- [ ] Um PR que introduz um pacote com CVE alto conhecido é bloqueado pelo gate de CI.
- [ ] Regras `ignore` do Dependabot são definidas para pacotes que você fixa intencionalmente (documente o motivo em um comentário).

**Dicas:**
1. Config mínima do `dependabot.yml`:
   ```yaml
   version: 2
   updates:
     - package-ecosystem: "npm"
       directory: "/"
       schedule:
         interval: "weekly"
   ```
2. Para o step de auditoria: `run: npm audit --audit-level=high`. O código de saída é não-zero quando vulnerabilidades no nível ou acima são encontradas.
3. Salvar resultados da auditoria: `npm audit --json > audit-report.json` depois `uses: actions/upload-artifact` com `path: audit-report.json`.
4. Verificação de licença: use o pacote `license-checker` — `npx license-checker --failOn "GPL;AGPL"`.

---

## Exercício 10 — Wrapper de Secrets Manager (Médio)

**Cenário:** Seu app lê segredos de variáveis de ambiente. Para produção, segredos são armazenados no HashiCorp Vault. Construa uma abstração que falhe graciosamente.

**Requisitos:**
- Implemente `getSecret(key: string): Promise<string>` que:
  1. Em desenvolvimento (`NODE_ENV !== 'production'`): lê de `process.env`.
  2. Em produção: busca do Vault usando o cliente `node-vault`.
- Faça cache dos segredos resolvidos em memória (Map) pelo tempo de vida do processo — não re-busque em cada chamada.
- Se o segredo não for encontrado em nenhuma fonte, lance um `SecretNotFoundError` tipado.
- Exponha uma função `preloadSecrets(keys: string[]): Promise<void>` para carregamento antecipado na inicialização.

**Critérios de Aceite:**
- [ ] Em desenvolvimento, `getSecret('DATABASE_URL')` lê `process.env.DATABASE_URL`.
- [ ] Em produção, o cliente Vault é inicializado uma vez (singleton) usando as vars de ambiente `VAULT_ADDR` e `VAULT_TOKEN`.
- [ ] Chamar `getSecret` duas vezes para a mesma chave só acessa o Vault uma vez (verificado por spy/mock no cliente Vault).
- [ ] `preloadSecrets(['DB_URL', 'JWT_SECRET'])` resolve todos os segredos antes do servidor HTTP iniciar.
- [ ] `SecretNotFoundError` inclui o nome da chave na mensagem para depuração rápida.

**Dicas:**
1. Padrão: um `const cache = new Map<string, string>()` no nível do módulo. Verifique o cache primeiro, depois busque da fonte.
2. Convenção de caminho no Vault: segredos ficam em `secret/data/<appName>/<key>`. O cliente `node-vault` retorna `data.data[key]`.
3. Carregamento antecipado: `await Promise.all(keys.map(getSecret))` — se algum lançar, a sequência de inicialização deve abortar.
4. Nos testes: defina `NODE_ENV=test`, popule `process.env`, e verifique que o cliente Vault nunca é chamado.

---

## Exercício 11 — Middleware de RBAC (Médio)

**Cenário:** Sua API tem múltiplos papéis (`guest`, `user`, `moderator`, `admin`) com diferentes permissões. Implemente um middleware de Role-Based Access Control que protege rotas declarativamente.

**Requisitos:**
- Defina um mapa de permissões: cada papel tem um conjunto de permissões (ex: `user:read`, `post:create`, `admin:delete`).
- Implemente `requirePermission(permission: string)` como uma factory de `preHandler` do Fastify.
- Os papéis são hierárquicos: `admin` herda todas as permissões de `moderator`, `moderator` herda todas as de `user`.
- O papel do usuário autenticado vem do payload decodificado do JWT (`req.user.role`).
- Retorne `403 Forbidden` (não `401`) quando um usuário válido não possui a permissão necessária.

**Critérios de Aceite:**
- [ ] Um `user` acessando uma rota `admin:delete` recebe `403`.
- [ ] Um `admin` pode acessar todas as rotas independente da permissão necessária.
- [ ] Adicionar `requirePermission('post:create')` a uma rota bloqueia corretamente usuários `guest`.
- [ ] Permissões para um papel desconhecido default para vazio (fail-secure, não fail-open).
- [ ] Teste unitário: verifique a hierarquia completa — `moderator` obtém permissões de nível `user` sem atribuição explícita.

**Dicas:**
1. Defina permissões com um objeto simples: `const ROLE_PERMISSIONS: Record<Role, Set<string>> = { admin: new Set([...all]), moderator: new Set([...]), ... }`.
2. Herança hierárquica: na definição, distribua permissões do papel pai nos filhos. Computado uma vez no carregamento do módulo — não reavaliado por requisição.
3. Factory `requirePermission`: `(perm: string) => async (req, reply) => { if (!ROLE_PERMISSIONS[req.user.role]?.has(perm)) reply.code(403)... }`.
4. Sempre verifique se `req.user` existe antes de acessar `req.user.role` — se o middleware de auth falhou em defini-lo, retorne `401`.

---

## Exercício 12 — Checklist de Auditoria de Headers de Segurança (Fácil)

**Cenário:** Você está conduzindo uma revisão de segurança de uma aplicação web em produção. Use o checklist abaixo para auditar os headers de resposta HTTP e documente as descobertas.

**Requisitos:**
- Faça uma requisição `GET` para a URL alvo (use `curl -I` ou um browser headless).
- Para cada header no checklist, marque: Presente / Ausente / Mal Configurado e explique por que importa.
- Produza um script de remediação (shell ou Node.js) que adiciona todos os headers faltantes a um servidor Fastify.
- Execute novamente a auditoria após aplicar as correções para confirmar que todos os headers estão presentes e corretamente configurados.

**Checklist:**
```
[ ] Content-Security-Policy        — previne XSS
[ ] Strict-Transport-Security      — força HTTPS
[ ] X-Content-Type-Options: nosniff — previne MIME sniffing
[ ] X-Frame-Options: DENY          — previne clickjacking
[ ] Referrer-Policy                — controla vazamento do referrer
[ ] Permissions-Policy             — desabilita APIs de browser não utilizadas
[ ] Cache-Control em rotas de auth — previne cache de respostas sensíveis
[ ] X-Powered-By ausente           — oculta a tecnologia usada
```

**Critérios de Aceite:**
- [ ] O relatório de auditoria lista cada header com status e uma explicação de uma frase.
- [ ] Pelo menos 3 headers são intencionalmente mal configurados em um servidor de teste para você encontrar e corrigir.
- [ ] O script de remediação é idempotente — executá-lo duas vezes não duplica headers.
- [ ] Após a remediação, a saída de `curl -I` mostra todos os 8 itens marcados.
- [ ] `Cache-Control: no-store` é aplicado especificamente às rotas `/auth/*`, não globalmente.

**Dicas:**
1. Auditoria rápida: `curl -sI https://seuapp.com | grep -i 'content-security\|strict-transport\|x-content\|x-frame\|referrer\|permissions\|x-powered'`.
2. No Fastify, adicione headers globalmente via `fastify.addHook('onSend', (req, reply, payload, done) => { reply.header(...); done() })`.
3. `Referrer-Policy: strict-origin-when-cross-origin` é um padrão seguro que previne vazamento de URL completa para terceiros.
4. `Permissions-Policy: camera=(), microphone=(), geolocation=()` desabilita APIs comuns que seu app provavelmente não precisa.
