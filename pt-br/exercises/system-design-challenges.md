# Desafios de System Design

> Katas de design de sistemas para aprimorar seu raciocínio sobre sistemas distribuídos. Cada exercício apresenta um problema concreto de implementação ou design, não apenas um vago "design o Twitter". Foque em trade-offs, modos de falha e realidade operacional. Nível alvo: intermediário a avançado.
>
> Para cada kata: passe 20–30 minutos esboçando um design antes de ler as dicas. Escreva suas premissas explicitamente.

---

## Exercício 1 — Rate Limiter Distribuído (Token Bucket) (Difícil)

**Cenário:** Projete e implemente um rate limiter distribuído com algoritmo token bucket. Deve impor limites em múltiplos nós de API gateway sem compartilhar estado local.

**Requisitos:**
- Cada cliente recebe 100 tokens por minuto. Os tokens são recarregados continuamente (não em lotes no início de cada minuto).
- O limitador roda como um serviço compartilhado com Redis. Múltiplos nós de gateway o chamam em cada requisição.
- Retorne o número de tokens restantes e o tempo até o próximo token estar disponível nos headers de resposta.
- Implemente o algoritmo principal como um script Lua atômico para prevenir condições de corrida.
- Suporte burst: um cliente pode consumir até 20 tokens em uma única requisição.

**Critérios de Aceite:**
- [ ] Dois nós de gateway consumindo o último token concorrentemente resultam em exatamente um sucesso (atomicidade).
- [ ] Tokens são recarregados a 100/60 ≈ 1,67 tokens/segundo (contínuo, não em lote por minuto).
- [ ] Headers `X-RateLimit-Remaining` e `X-RateLimit-Reset` são retornados em toda resposta.
- [ ] Um cliente que foi limitado pode retomar assim que tokens estiverem disponíveis (sem penalidade de janela fixa).
- [ ] O script Lua é testável unitariamente injetando uma função `redis.call` fake.

**Perguntas de discussão:**
1. Token bucket vs. janela deslizante: qual é mais justo sob tráfego em rajadas?
2. O que acontece se o tempo de execução do script Lua exceder 5ms? Como você lida com picos de latência do Redis?
3. Como você adicionaria limites por endpoint sem duplicar a lógica?

**Dicas:**
1. Token bucket no Redis: armazene `{ tokens, lastRefillTimestamp }` como um hash. Em cada chamada, compute o tempo decorrido, adicione `elapsed * refillRate` tokens (limitado ao máximo), depois subtraia os tokens solicitados.
2. Lua garante atomicidade. O script recebe: `KEYS[1]` = chave do bucket, `ARGV[1]` = agora (ms), `ARGV[2]` = taxa de recarga, `ARGV[3]` = capacidade, `ARGV[4]` = tokens solicitados.
3. `X-RateLimit-Reset`: compute `(requested - current_tokens) / refillRate` segundos a partir de agora.

---

## Exercício 2 — Sistema de Fanout de Notificações (Médio)

**Cenário:** Projete um serviço de fanout de notificações que envia um único evento de notificação para milhões de usuários simultaneamente via push, email e SMS.

**Requisitos:**
- Um serviço produtor publica um `NotificationEvent` para uma fila com: `{ eventId, recipients: string[], channels: ['push','email','sms'], payload }`.
- Destinatários podem ser até 1 milhão de usuários (ex: um blast de marketing).
- Cada canal tem um pool de workers independente.
- As preferências do usuário (canais opted-out) devem ser verificadas antes do envio.
- Envios com falha são retentados até 3 vezes por canal por destinatário com backoff exponencial.

**Critérios de Aceite:**
- [ ] Um único evento com 1M de destinatários é totalmente entregue dentro de 10 minutos.
- [ ] Um usuário que optou por não receber não recebe notificação no canal opted-out.
- [ ] Uma falha de entrega push não afeta a entrega de email para o mesmo destinatário.
- [ ] Mensagens com dead-letter (tentativas esgotadas) são armazenadas e observáveis.
- [ ] O sistema lida com submissões de `eventId` duplicadas de forma idempotente.

**Perguntas de discussão:**
1. Como você faz fanout para 1M de destinatários sem carregar todos os registros de usuários na memória de uma vez?
2. As verificações de preferência do usuário devem acontecer antes ou depois de enfileirar jobs por destinatário?
3. Como você limita a taxa de saída por canal sem perder mensagens?

**Dicas:**
1. Padrão de fanout: o serviço de intake divide a lista de destinatários em lotes de 1.000 e enfileira um job por lote. Cada worker de lote expande, verifica preferências e enfileira jobs por canal.
2. Verificação de preferência no nível do lote: busque preferências para todos os 1.000 usuários em uma query de banco de dados (`WHERE user_id IN (...)`) antes de enfileirar jobs por canal.
3. Limitação de taxa por canal: use uma fila separada por canal com um limite de concorrência que corresponde ao limite de taxa do provedor.

---

## Exercício 3 — Fila de Jobs com Retry e Dead-Letter (Médio)

**Cenário:** Construa um sistema de fila de jobs de nível produção com lógica de retry, backoff exponencial e fila dead-letter para jobs com falha.

**Requisitos:**
- Jobs têm tipos, payloads, níveis de prioridade (`high`, `normal`, `low`) e uma configuração `maxAttempts`.
- Em caso de falha, jobs são retentados com backoff exponencial: `delay = baseDelay * 2^(attempt - 1)` (base: 1 segundo, máximo: 5 minutos).
- Após `maxAttempts`, o job vai para uma fila dead-letter (DLQ) e dispara um alerta.
- A fila deve suportar entrega at-least-once (sem descartes silenciosos).
- Exponha um endpoint `GET /jobs/:id` para inspecionar status, tentativas e motivos de falha.

**Critérios de Aceite:**
- [ ] Um job que falha 3 vezes com `maxAttempts=3` vai para a DLQ após a 3ª falha.
- [ ] Os delays de retry são aproximadamente 1s, 2s, 4s (exponencial) — não intervalos fixos.
- [ ] Reiniciar um worker no meio de um job não resulta em descarte silencioso do job.
- [ ] Entradas da DLQ incluem: `jobId`, `jobType`, `failureReason`, `attempts`, `lastAttemptAt`.
- [ ] Um teste simula um crash do worker (`process.exit`) e verifica que o job é re-enfileirado.

**Perguntas de discussão:**
1. Qual é a diferença entre entrega at-least-once e exactly-once? Quando cada uma é adequada?
2. Como você previne um job ruim (ex: payload malformado) de bloquear uma fila indefinidamente?
3. Quando um job deve ir para a DLQ imediatamente (sem tentativas)?

**Dicas:**
1. BullMQ lida com a maioria disso: `{ attempts: maxAttempts, backoff: { type: 'exponential', delay: 1000 } }`.
2. Recuperação de crash do worker: BullMQ usa um lock Redis por job. Se um worker morrer, o lock expira e outro worker pega o job.
3. DLQ: escute o evento `'failed'` do BullMQ na fila após `maxAttempts` esgotados. Mova o job para uma fila `dead-letter` separada e envie um alerta.

---

## Exercício 4 — Leaderboard com Leitura Intensiva e Cache (Médio)

**Cenário:** Projete um leaderboard para um jogo mobile com 5 milhões de usuários ativos diários. O leaderboard mostra os 100 melhores jogadores e a posição do jogador solicitante.

**Requisitos:**
- Jogadores ganham pontos via ações no jogo. Pontos são atualizados em tempo real (até 10.000 atualizações/segundo).
- `GET /leaderboard/top100`: retorna os 100 melhores jogadores com posição, username e pontuação.
- `GET /leaderboard/me`: retorna a posição e pontuação do jogador solicitante.
- A query dos top 100 deve responder em < 50ms em qualquer escala.
- A posição do jogador deve ser precisa dentro de 5 segundos de uma atualização de pontuação.

**Critérios de Aceite:**
- [ ] O top 100 é servido de um sorted set Redis, não de uma query de banco de dados.
- [ ] Atualizações de pontuação são escritas tanto no PostgreSQL (fonte da verdade) quanto no Redis (sorted set) na mesma operação.
- [ ] `ZREVRANK` é usado para busca eficiente de posição — sem contagem manual.
- [ ] Mudanças na posição de um jogador são refletidas em `GET /leaderboard/me` dentro de 5 segundos.
- [ ] Teste de carga: 10.000 requisições concorrentes para `GET /leaderboard/top100` são servidas em < 50ms p99.

**Perguntas de discussão:**
1. O que acontece com o sorted set Redis se a instância Redis for reiniciada?
2. Como você lida com empates (dois jogadores com a mesma pontuação)?
3. A 10.000 atualizações/segundo, escrever tanto no Redis quanto no PostgreSQL sincronamente é sustentável?

**Dicas:**
1. Redis sorted set: `ZADD leaderboard <score> <userId>`. `ZREVRANGE leaderboard 0 99 WITHSCORES` para o top 100. `ZREVRANK leaderboard <userId>` para a posição (índice 0).
2. Reconstrução após restart: um script de inicialização que roda `SELECT user_id, score FROM scores ORDER BY score DESC` e carrega em bulk com pipeline `ZADD`.
3. Para alta taxa de escrita: escreva no Redis sincronamente (rápido, em memória), escreva no PostgreSQL assincronamente via job BullMQ. Aceite consistência eventual para o banco.

---

## Exercício 5 — Schema Multi-Tenant SaaS com Row-Level Security (Difícil)

**Cenário:** Projete um schema PostgreSQL para uma aplicação SaaS multi-tenant. Os tenants (organizações) devem ser estritamente isolados — nenhum vazamento de dados entre tenants é aceitável.

**Requisitos:**
- Todas as tabelas com escopo de tenant têm uma coluna `tenant_id UUID NOT NULL`.
- Políticas de Row Level Security (RLS) do PostgreSQL impõem isolamento na camada de banco de dados (não apenas na camada de aplicação).
- Um usuário admin (papel: `superadmin`) pode fazer query em todos os tenants.
- A aplicação conecta como um único usuário de banco de dados (`app_user`) e define `app.tenant_id` na conexão antes de cada query.
- Migrações devem incluir criação de políticas RLS (reproduzível, não manual).

**Critérios de Aceite:**
- [ ] Uma query executada como `app_user` com `SET app.tenant_id = 'tenant-A'` não pode retornar linhas pertencentes a `tenant-B`.
- [ ] Queries de `superadmin` ignoram RLS (verificado por teste direto no banco).
- [ ] A migração que cria a tabela `projects` também habilita RLS e cria a política de isolamento.
- [ ] Teste de integração: dois tenants, três projetos cada — verifique isolamento cross-tenant via API.
- [ ] A camada de aplicação nunca filtra manualmente por `tenant_id` em métodos de serviço — o RLS cuida disso.

**Perguntas de discussão:**
1. Quais são as implicações de desempenho do RLS em escala?
2. O que acontece se um desenvolvedor esquece de definir `app.tenant_id` antes de uma query?
3. Como você lida com tabelas agnósticas de tenant (ex: `countries`, `currencies`) com RLS habilitado?

**Dicas:**
1. Política RLS: `CREATE POLICY tenant_isolation ON projects FOR ALL TO app_user USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
2. Habilite: `ALTER TABLE projects ENABLE ROW LEVEL SECURITY; ALTER TABLE projects FORCE ROW LEVEL SECURITY;`.
3. Padrão seguro: defina `app.tenant_id = ''` como padrão para `app_user`. Uma string vazia nunca corresponde a um UUID real, então um `SET` esquecido resulta em resultados vazios (seguro) em vez de vazar dados.
4. Bypass do superadmin: crie um papel de banco de dados `superadmin_user` separado com atributo `BYPASSRLS`.

---

## Exercício 6 — Padrão Circuit Breaker (Médio)

**Cenário:** Seu serviço chama uma API de pagamento externa. Quando a API de pagamento está degradada, seu serviço propaga falhas e fica lento. Implemente um circuit breaker para falhar rápido e se recuperar graciosamente.

**Requisitos:**
- Implemente uma classe `CircuitBreaker` com três estados: `CLOSED` (normal), `OPEN` (falhando rápido), `HALF_OPEN` (testando recuperação).
- Threshold: abra após 5 falhas consecutivas dentro de uma janela de 30 segundos.
- Recuperação: após 60 segundos no estado `OPEN`, transicione para `HALF_OPEN`. Permita uma requisição de sonda. Se tiver sucesso, feche o circuito. Se falhar, retorne para `OPEN`.
- Quando `OPEN`, chamadas imediatamente lançam `CircuitOpenError` sem atingir o serviço externo.
- Emita eventos em transições de estado para observabilidade.

**Critérios de Aceite:**
- [ ] Após 5 falhas consecutivas, o circuito abre e chamadas subsequentes falham imediatamente.
- [ ] Enquanto `OPEN`, zero requisições de rede são feitas ao serviço externo (verificado por mock).
- [ ] Após o timeout de recuperação, exatamente uma requisição de sonda é permitida.
- [ ] Uma sonda bem-sucedida fecha o circuito; uma sonda com falha reinicia o timer de recuperação.
- [ ] Transições de estado emitem eventos `{ from, to, timestamp }` que um logger ou sistema de métricas pode consumir.

**Perguntas de discussão:**
1. O circuit breaker deve ser por instância ou um singleton compartilhado? Quais são os trade-offs em um deploy multi-nó?
2. Qual é a diferença entre um circuit breaker e um retry? Quando você deve usar ambos juntos?
3. Como você testa um circuit breaker sem introduzir flakiness na sua suite de testes?

**Dicas:**
1. Máquina de estados: `CLOSED → OPEN` (no threshold); `OPEN → HALF_OPEN` (no timeout); `HALF_OPEN → CLOSED` (no sucesso) ou `HALF_OPEN → OPEN` (na falha).
2. Armazene: `failureCount`, `lastFailureTime`, `state`, `halfOpenProbeAllowed` (booleano).
3. Use `EventEmitter` para eventos de transição de estado.
4. Nos testes: use fake timers (`vi.useFakeTimers`) para avançar o timeout de recuperação sem esperar realmente 60 segundos.

---

## Exercício 7 — API de Pagamento Idempotente (Difícil)

**Cenário:** Sua API de pagamento deve ser idempotente: se um cliente retentar uma requisição (devido a timeout de rede), não deve cobrar duas vezes.

**Requisitos:**
- `POST /payments` aceita um header `Idempotency-Key` (UUID, gerado pelo cliente).
- Se a chave já foi vista antes e a requisição foi concluída, retorne a resposta em cache (mesmo status code + body).
- Se a chave está sendo processada atualmente, retorne `409 Conflict` — não processe uma segunda vez.
- Chaves de idempotência expiram após 24 horas.
- A chave deve estar vinculada ao usuário autenticado — chaves não são compartilhadas entre usuários.

**Critérios de Aceite:**
- [ ] Enviar `POST /payments` duas vezes com a mesma chave e mesmo usuário retorna respostas idênticas.
- [ ] O processador de pagamento (Stripe, etc.) é chamado apenas uma vez por chave de idempotência.
- [ ] Uma segunda requisição chegando enquanto a primeira está sendo processada retorna `409`, não uma cobrança duplicada.
- [ ] Usar a mesma chave com um usuário diferente retorna `403` (isolamento de chave por usuário).
- [ ] Chaves com mais de 24 horas são tratadas como novas (sem resposta em cache retornada).

**Perguntas de discussão:**
1. Onde você armazena chaves de idempotência — Redis ou PostgreSQL? Quais são os trade-offs de durabilidade?
2. O que acontece se o servidor travar após cobrar mas antes de armazenar a resposta de idempotência?
3. A idempotência deve ser imposta na camada HTTP ou na camada de serviço?

**Dicas:**
1. Formato da chave no Redis: `idempotency:<userId>:<key>`. Valor: `{ status, body }` como JSON. TTL: 86400 segundos.
2. Lock durante processamento: `SET idempotency:<userId>:<key>:lock 1 NX EX 30`. Se o lock existir (cenário 409), retorne `409`. Libere o lock com script Lua após armazenar o resultado.
3. Recuperação de escrita parcial: armazene o ID do payment intent do Stripe antes do `SET` — se ocorrer um crash, um job de recuperação pode reconciliar verificando o status do intent no Stripe.

---

## Exercício 8 — Sistema de Presença em Tempo Real (Médio)

**Cenário:** Projete o recurso "quem está online" para um editor de documentos colaborativo. Usuários devem ver quais membros da equipe estão visualizando o mesmo documento.

**Requisitos:**
- Quando um usuário abre um documento, ele se torna "presente" naquele documento.
- A presença expira após 30 segundos de inatividade (heartbeat mantém ativa).
- `GET /documents/:id/presence` retorna `[{ userId, name, avatarUrl, lastSeen }]` para usuários atualmente presentes.
- Quando a presença de um usuário muda (entra/sai), todos os outros usuários presentes recebem uma atualização em tempo real via WebSocket.
- Suporte a até 50 usuários simultâneos por documento.

**Critérios de Aceite:**
- [ ] Um usuário fechando a aba do browser é removido da lista de presença dentro de 30 segundos (expiração de TTL).
- [ ] Dois usuários no mesmo documento ambos recebem um evento `presence_update` quando um terceiro usuário entra.
- [ ] Endpoint de heartbeat: `POST /documents/:id/heartbeat` reinicia o TTL de 30 segundos.
- [ ] Escalonamento horizontal: atualizações de presença no servidor A chegam a clientes conectados ao servidor B (via Redis Pub/Sub).
- [ ] `GET /documents/:id/presence` é servido do Redis — sem query de banco de dados em cada chamada.

**Perguntas de discussão:**
1. Qual é o trade-off entre expiração baseada em TTL e eventos explícitos de "usuário saiu"?
2. Como você lida com o caso em que o browser de um usuário trava (sem evento de close do WebSocket)?
3. A 50 usuários por documento, o broadcast via Redis Pub/Sub é um gargalo?

**Dicas:**
1. Chave Redis: `presence:<docId>:<userId>` com TTL de 30 segundos. Valor: `{ name, avatarUrl }` como JSON.
2. Melhor padrão: use um Redis Set `presence:doc:<docId>` contendo `userId`s, mais chaves individuais para metadados do usuário.
3. Fanout WebSocket: no heartbeat, publique `{ type: 'presence_join', docId, userId }` para `presence:updates:<docId>`. Cada servidor WS se inscreve e entrega aos clientes locais conectados.

---

## Exercício 9 — Estratégia de Invalidação de Cache para Catálogo de Produtos (Médio)

**Cenário:** Uma API de catálogo de produtos serve 50.000 requisições/segundo. Dados de produtos raramente mudam (< 100 atualizações/dia). Projete uma estratégia de invalidação de cache que mantém os dados atualizados sem sobrecarregar o banco de dados.

**Requisitos:**
- `GET /products/:id`: servido do cache com < 10ms de latência.
- Quando um produto é atualizado, todas as versões em cache (no Redis e no CDN) devem ser invalidadas dentro de 60 segundos.
- Camadas de cache: CDN (Cloudflare) → Redis (cache de aplicação) → PostgreSQL.
- Mudanças em dados de produtos disparam invalidação de uma API de admin interna.
- Trate cache stampede na invalidação (muitos clientes requisitando a mesma chave simultaneamente após evicção).

**Critérios de Aceite:**
- [ ] `GET /products/123` após uma atualização retorna os novos dados dentro de 60 segundos.
- [ ] Um evento de invalidação de cache dispara: exclusão da chave Redis + purge do cache Cloudflare via API.
- [ ] Cache stampede é prevenido: apenas uma query de banco de dados dispara por chave fria, independente da concorrência.
- [ ] Taxa de acerto do cache permanece acima de 99% sob carga normal.
- [ ] Teste de integração: atualize um produto, espere 5 segundos, verifique que `GET /products/:id` retorna dados atualizados.

**Perguntas de discussão:**
1. Expiração baseada em TTL vs. invalidação explícita na escrita — quais são os trade-offs de consistência?
2. Como você lida com uma cascata: uma atualização de produto também afeta listagens de categoria e resultados de busca?
3. Se a API de purge do Cloudflare é lenta (latência de 2–5 segundos), como você garante que clientes não sirvam dados desatualizados?

**Dicas:**
1. Fluxo de invalidação: `PUT /admin/products/:id` → atualiza banco de dados → publica `product:updated:<id>` no Redis Pub/Sub → subscriber exclui `product:cache:<id>` + chama API de purge do Cloudflare.
2. Prevenção de stampede: expiração antecipada probabilística, ou um lock Redis em cache misses — `SET product:lock:<id> 1 NX EX 5`. Se o lock existir, espere brevemente e tente novamente do cache.
3. Purge do Cloudflare: `POST https://api.cloudflare.com/client/v4/zones/<zoneId>/purge_cache` com `{ files: ['https://api.seudominio.com/products/123'] }`.

---

## Exercício 10 — Sistema de Entrega de Webhooks com Retries (Médio)

**Cenário:** Sua plataforma envia webhooks para URLs definidas pelos clientes quando eventos ocorrem. Os servidores dos clientes são não-confiáveis — as entregas devem ser retentadas de forma confiável.

**Requisitos:**
- Quando um evento dispara, enfileire um job de entrega de webhook para a URL do endpoint registrado.
- Tentativas de entrega: imediata, depois +1min, +5min, +30min, +2h, +8h (6 tentativas no total).
- Se todas as 6 tentativas falharem, marque o webhook como permanentemente falho e notifique o cliente.
- Registre cada tentativa de entrega: `{ timestamp, httpStatus, responseBody, durationMs }`.
- Exponha `GET /webhooks/:id/deliveries` para mostrar o histórico de tentativas.

**Critérios de Aceite:**
- [ ] Um endpoint do cliente retornando `500` dispara um retry após o delay correto.
- [ ] Uma resposta `200` ou `201` marca a entrega como bem-sucedida — sem mais retries.
- [ ] Após 6 tentativas com falha, um email é enviado para o endereço registrado do cliente.
- [ ] Logs de entrega são armazenados no PostgreSQL (não apenas na fila de jobs).
- [ ] Um teste com um servidor HTTP mock verifica que o cronograma de retry está correto.

**Perguntas de discussão:**
1. Você deve retentar em respostas `4xx` (ex: `401 Unauthorized`)? Qual é o raciocínio?
2. Como você protege contra um endpoint lento do cliente bloqueando suas threads de worker?
3. Como você permite que clientes reproduzam um webhook com falha específico sem re-disparar o evento original?

**Dicas:**
1. Use BullMQ com agendamento de delay customizado: em vez do backoff exponencial embutido, enfileire a próxima tentativa como um novo job com delay fixo após cada falha.
2. Armazene tentativas de entrega: nos handlers `onFailed` e `onCompleted` do job, insira uma linha em `webhook_attempts(webhookId, attempt, status, responseStatus, durationMs)`.
3. Timeout por tentativa: use `AbortSignal.timeout(5000)` com `fetch` para limitar cada tentativa de entrega a 5 segundos.
4. Não retente em `410 Gone` ou respostas `4xx` que indicam que o endpoint está intencionalmente rejeitando a requisição.
