# Katas de Arquitetura

> Exercícios de design de sistemas para desenvolver seu pensamento arquitetural. Cada kata apresenta um sistema do mundo real para projetar sob restrições realistas. Não há respostas únicas corretas — o objetivo é raciocinar sobre trade-offs, comunicar decisões e entender as implicações das suas escolhas. Nível: intermediário ao avançado.
>
> Para cada kata: passe 30–45 minutos projetando antes de ler as perguntas de discussão. Diagrame sua solução antes de escrever qualquer texto.

---

## Kata 1 — Encurtador de URL (Iniciante)

**Sistema a projetar:** Um serviço de encurtamento de URLs similar ao bit.ly. Usuários enviam uma URL longa e recebem um código curto (ex.: `sht.ly/xK3pQ`). Visitar a URL curta redireciona para a original.

**Restrições:**
- Contas de usuário não são necessárias (uso anônimo).
- Códigos curtos têm 7 caracteres alfanuméricos.
- Uma determinada URL longa sempre mapeia para o mesmo código curto.
- URLs curtas não expiram (por enquanto).

**Requisitos de escala:**
- 100 milhões de URLs criadas por dia (escrita pesada é um engano — as leituras dominam amplamente).
- Proporção leitura/escrita de 10:1 → 1 bilhão de redirects por dia.
- Latência de leitura alvo: < 10ms p99.
- Retenção de dados: indefinida.

**Perguntas de discussão:**
1. Como você gera códigos curtos? Opções: aleatório, hash (prefixo MD5/SHA1), baseado em contador com codificação base62. Quais são os riscos de colisão e os trade-offs?
2. Onde você armazena os mapeamentos? Uma tabela SQL simples com índice em `short_code` é direta. Em que escala você a fragmenta, e como?
3. O endpoint de redirect deve ser extremamente rápido. Como uma camada CDN + cache muda a arquitetura? Qual é sua estratégia de invalidação de cache?
4. Como você lida com URLs longas duplicadas (mesma URL longa → mesmo código curto)? Um hash da URL longa como chave primária? Um upsert?
5. Que dados de analytics você capturaria (cliques, referrer, geografia) e como evita tornar o caminho de redirect mais lento?
6. Projete o modelo de dados. Estime o armazenamento: 500 bytes por registro * 100M/dia * 365 dias * 5 anos.

**Esboço de solução:**
- Serviço de API (stateless, escalável horizontalmente) → escreve em um banco primário (PostgreSQL), lê de uma réplica.
- Camada de cache (Redis): `short_code → long_url` com eviction LRU. Alvo de taxa de acerto ao cache: 99%+ (distribuição de Zipf significa que um pequeno % de URLs gera a maior parte do tráfego).
- Geração de código curto: codifique em base62 um ID auto-incrementado de um contador distribuído (ou use abordagem baseada em UUID com retry em colisão).
- CDN faz cache de redirects na borda — HTTP 301 (permanente, navegador armazena) vs 302 (temporário, servidor controla). Escolha 302 para preservar a precisão dos analytics.

---

## Kata 2 — Aplicação de Chat em Tempo Real (Intermediário)

**Sistema a projetar:** Uma aplicação de chat suportando mensagens diretas (DMs) e canais em grupo, com histórico de mensagens e indicadores de presença online.

**Restrições:**
- As mensagens devem ser entregues em ordem dentro de uma conversa.
- As mensagens não devem ser perdidas (entrega at-least-once).
- Usuários offline recebem as mensagens perdidas quando se reconectam.
- Cada canal em grupo pode ter até 1.000 membros.

**Requisitos de escala:**
- 50 milhões de usuários ativos diários.
- Média de 30 mensagens por usuário por dia → 1,5 bilhão de mensagens por dia.
- Pico: 50.000 mensagens por segundo.
- Latência de entrega de mensagem P99: < 500ms.

**Perguntas de discussão:**
1. Como os clientes mantêm conexões persistentes? WebSockets, Server-Sent Events ou long polling? Quais são os trade-offs nessa escala?
2. Com 50M de clientes conectados em múltiplas instâncias de servidor, como uma mensagem enviada no servidor A chega a um cliente conectado ao servidor B?
3. Como você armazena as mensagens? Banco relacional, NoSQL (Cassandra, DynamoDB) ou combinação? Modele o schema. Como você pagina o histórico de mensagens?
4. Como você implementa presença (indicadores online/offline/digitando)? Qual TTL você usaria nas chaves de presença?
5. Notificações push para usuários offline: como você integra APNs e FCM sem acoplá-los ao fluxo central de mensagens?
6. Como você lida com a ordenação de mensagens em nós distribuídos? Vector clocks? Números de sequência atribuídos pelo servidor?

**Esboço de solução:**
- Servidores WebSocket (stateless) atrás de um load balancer com sticky sessions.
- Redis Pub/Sub como barramento de mensagens: cada servidor WebSocket assina canais. Publicar uma mensagem em `channel:<id>` faz fan-out para todos os servidores assinantes, cada um entregando aos clientes locais conectados.
- Store de mensagens: Cassandra com partition key `channel_id` e clustering key `timestamp DESC` — otimizado para queries por intervalo de tempo.
- Presença: chaves Redis `presence:<userId>` com TTL de 60s, atualizadas pelo heartbeat do cliente.
- Entrega offline: mensagens são armazenadas no banco. Ao reconectar, o cliente envia seu `lastSeenTimestamp`; o servidor consulta o banco por mensagens desde aquele ponto.

---

## Kata 3 — Sistema de Entrega de Notificações (Intermediário)

**Sistema a projetar:** Um serviço de notificações multicanal que envia notificações via email, SMS, push e canais in-app. Usado por múltiplos times internos de produto.

**Restrições:**
- Times enviam requisições de notificação via API REST ou fila de mensagens.
- Cada notificação pode atingir múltiplos canais simultaneamente.
- Usuários têm preferências por canal (opt-in, opt-out, horários silenciosos).
- Proteção contra notificações duplicadas: o mesmo evento lógico não deve enviar duas vezes.

**Requisitos de escala:**
- 10 milhões de notificações por dia em todos os canais.
- Email: latência de envio de até 100ms aceitável.
- SMS: até 500ms aceitável.
- Push: < 1s aceitável.
- Meta de taxa de entrega bem-sucedida: 99,9% (com retry).

**Perguntas de discussão:**
1. Como você modela as preferências de notificação do usuário? Uma tabela de preferências por usuário? Como você lida com herança (padrões da org → overrides do usuário)?
2. Cada canal tem um provedor diferente (SendGrid para email, Twilio para SMS, FCM para push). Como você abstrai a lógica do provedor para que trocar de provedor exija mudanças mínimas de código?
3. Como você implementa idempotência? Qual é sua chave de deduplicação e onde você a armazena?
4. Uma notificação crítica (redefinição de senha) precisa de entrega garantida. Uma notificação de marketing pode ser descartada em caso de falha. Como você lida com diferentes níveis de prioridade?
5. Como você observa falhas de entrega e aciona retries? Diferencie entre falhas transitórias (retry) e permanentes (número de telefone inválido — não faça retry).
6. Rate limiting: o Twilio tem limites por segundo. Como você suaviza picos sem perder notificações?

**Esboço de solução:**
- API de entrada → barramento de eventos (Kafka ou SQS). Desacoplado da entrega.
- Serviço de roteamento de notificações: lê eventos, verifica preferências do usuário, faz fan-out para filas por canal (fila de email, fila de SMS, fila de push).
- Workers de canal: um tipo de worker por canal. Cada worker é escalável de forma independente.
- Idempotência: `SET notif:<eventId>:<channel>:<userId> 1 NX EX 86400` no Redis antes de enviar.
- Retry: backoff exponencial com dead-letter queues após 5 tentativas. Dead-letter aciona um alerta.
- Níveis de prioridade: filas de alta prioridade separadas processadas primeiro. Marketing vai para filas padrão.

---

## Kata 4 — Processamento de Pedidos de E-Commerce (Intermediário)

**Sistema a projetar:** O sistema backend que lida com a realização de pedidos, pagamento, reserva de estoque e fulfillment para um varejista online.

**Restrições:**
- O estoque nunca deve ser vendido em excesso (dois usuários comprando o último item não devem ter ambos sucesso).
- Falha no pagamento deve desfazer a reserva de estoque.
- O sistema deve lidar com o processamento de pedidos mesmo que o provedor de pagamento esteja temporariamente indisponível (retry depois).
- Os pedidos devem ser idempotentes — se o cliente tentar novamente, não deve criar pedidos duplicados.

**Requisitos de escala:**
- 500 pedidos por segundo no pico (ex.: flash sale).
- Verificação de estoque + reserva + confirmação de pagamento em < 3 segundos end-to-end.
- Histórico de pedidos deve ser consultável por até 7 anos.

**Perguntas de discussão:**
1. Reserva de estoque: locking otimista (`UPDATE inventory SET qty = qty - N WHERE qty >= N AND version = ?`) vs. locking pessimista vs. um serviço de reserva dedicado. Quais são os trade-offs sob alta concorrência?
2. O padrão Saga vs. two-phase commit para a transação distribuída (estoque + pagamento + fulfillment). Por que o 2PC raramente é usado em microsserviços?
3. Se o pagamento for bem-sucedido mas o serviço de fulfillment estiver fora, o que acontece? Como você garante consistência eventual?
4. Como você lida com a idempotência do endpoint `POST /orders`? Onde a chave de idempotência é armazenada e por quanto tempo?
5. Cenário de flash sale: 10.000 usuários tentando simultaneamente comprar as últimas 100 unidades de um produto. Como seu design evita que o "thundering herd" sobrecarregue o banco?
6. Projete o modelo de event sourcing para um pedido (eventos: `OrderCreated`, `PaymentPending`, `PaymentConfirmed`, `InventoryReserved`, `Shipped`, `Delivered`, `Cancelled`).

**Esboço de solução:**
- Serviço de pedidos: aceita requisição, grava evento `OrderCreated`, publica no barramento de eventos.
- Serviço de estoque: escuta `OrderCreated`, reserva com locking otimista. Publica `InventoryReserved` ou `InventoryUnavailable`.
- Serviço de pagamento: escuta `InventoryReserved`, inicia pagamento. Publica `PaymentConfirmed` ou `PaymentFailed`.
- Transações compensatórias: `PaymentFailed` aciona `InventoryReleased`.
- Idempotência: header `Idempotency-Key` fornecido pelo cliente, armazenado no Redis (ou banco) com corpo da resposta em cache por 24h.

---

## Kata 5 — Rate Limiter Distribuído (Avançado)

**Sistema a projetar:** Um serviço de rate limiting usado por um API gateway para impor limites de requisição por cliente em um cluster distribuído de nós do gateway.

**Restrições:**
- Os limites devem ser aplicados globalmente (não por nó) — um cliente não pode burlar fazendo round-robin entre nós.
- Budget de latência para verificação de rate limit: < 5ms p99.
- Disponibilidade mínima de 99,9% — a infraestrutura de rate limiting não deve se tornar um ponto único de falha.
- Suportar múltiplas janelas de limite: por segundo, por minuto, por hora, por dia (hierárquico).

**Requisitos de escala:**
- 1 milhão de requisições de API por segundo para limitar.
- 100.000 clientes únicos.
- Cada verificação toca o Redis.

**Perguntas de discussão:**
1. Fixed window vs. sliding window vs. token bucket vs. leaky bucket — compare os algoritmos para este caso de uso.
2. O Redis é o store central de contadores. Como você evita que o Redis se torne um gargalo em 1M RPS? (Cluster, pipelining, contadores locais com sincronização periódica?)
3. O que acontece quando o Redis está indisponível? Fail open (permitir tudo) ou fail closed (bloquear tudo)? Quais são as implicações de negócio de cada um?
4. Limites hierárquicos: um cliente tem um limite por segundo (100 req) e um limite por dia (1M req). Como você verifica ambos atomicamente?
5. Como você lida com clock drift do Redis entre nós em um Redis Cluster? O algoritmo de sliding window quebra?
6. Um cliente reclama que suas requisições estão sendo limitadas incorretamente. Como você constrói uma trilha de auditoria?

**Esboço de solução:**
- Token bucket no Redis: script Lua (atômico) verifica e decrementa o bucket. Os reabastecimentos acontecem de forma lazy em cada requisição (calcule os tokens adicionados desde a última verificação usando o delta de timestamp).
- Redis Cluster com 6 nós (3 primários, 3 réplicas) para escalonamento horizontal e HA.
- Camada de cache local em cada nó do gateway: acumule contadores localmente em janelas de 100ms, descarregue no Redis. Troca precisão por menor pressão no Redis.
- Fail open: em indisponibilidade do Redis, permita requisições mas registre cada bypass para auditoria.
- Verificação hierárquica: único script Lua que verifica todas as janelas em uma única viagem de ida e volta.

---

## Kata 6 — Plataforma de Streaming de Vídeo (Avançado)

**Sistema a projetar:** Uma plataforma de vídeo sob demanda (VoD). Usuários fazem upload de vídeos; outros usuários os assistem com streaming de bitrate adaptativo.

**Restrições:**
- Vídeos enviados são transcodificados em múltiplas resoluções (360p, 720p, 1080p).
- O streaming usa HLS (HTTP Live Streaming) com segmentos de 6 segundos.
- A plataforma atende 50 países — a latência deve ser minimizada globalmente.
- Os vídeos não devem ter hotlink de sites externos (signed URLs).

**Requisitos de escala:**
- 50.000 uploads de vídeo por dia (média de 500MB cada).
- 10 milhões de espectadores concorrentes no pico.
- Tempo de início de stream p99 global: < 3 segundos.

**Perguntas de discussão:**
1. Pipeline de upload: upload direto do navegador para S3 (presigned URL) vs. upload através do seu backend. Quais são os trade-offs para um arquivo de 500MB?
2. Transcodificação: como você aciona e monitora os jobs de transcodificação? AWS MediaConvert vs. um pool de workers FFmpeg self-hosted. Quando você escolheria cada um?
3. Entrega HLS: por que servir manifestos `.m3u8` e segmentos `.ts` via CDN em vez de seus servidores de origem?
4. Bitrate adaptativo: o player muda de qualidade com base na largura de banda. O que o playlist master `.m3u8` contém e como o player escolhe a qualidade inicial?
5. Segurança de signed URL: como você evita que um usuário compartilhe a signed URL com outros para burlar o paywall? Que parâmetros você inclui na assinatura?
6. Como você projeta o sistema de contagem de visualizações? Um `UPDATE SET views = views + 1` ingênuo em cada evento de play seria um gargalo de escrita.

**Esboço de solução:**
- Upload: presigned URL S3 → vídeo bruto no S3 → evento S3 aciona job de transcodificação → MediaConvert produz segmentos HLS → segmentos armazenados em um bucket S3 separado → CDN CloudFront na frente.
- Signed URLs: inclua `userId`, `videoId`, expiração, hash de IP na assinatura. Verifique na origem do CDN (Lambda@Edge).
- Contagem de visualizações: escreva em um tópico Kafka a cada evento de play. Um consumer agrega contagens a cada 60 segundos e grava o rollup no banco.
- Entrega global: CloudFront com edge locations em todas as regiões alvo. Origin shield para proteger a origem S3 de cache misses.

---

## Kata 7 — Replicação de Banco de Dados Multi-Região (Avançado)

**Sistema a projetar:** Uma aplicação SaaS que deve manter dados disponíveis e consistentes em três regiões (US, EU, APAC) com SLA de uptime de 99,99%.

**Restrições:**
- Escritas são aceitas em qualquer região.
- Usuários devem sempre ler suas próprias escritas.
- Dados da EU nunca devem sair fisicamente de data centers da EU (GDPR).
- Uma interrupção regional completa deve ser suportável sem perda de dados.

**Requisitos de escala:**
- 5.000 escritas por segundo globalmente.
- Proporção leitura/escrita: 20:1.
- RTO (Recovery Time Objective): 30 segundos.
- RPO (Recovery Point Objective): 0 (zero perda de dados).

**Perguntas de discussão:**
1. O teorema CAP: com a restrição GDPR e o requisito de escritas entre regiões, que modelo de consistência você pode oferecer realisticamente?
2. Como você roteia escritas para a região correta, especialmente dada a restrição GDPR de que os dados de usuários da EU devem permanecer na EU?
3. Resolução de conflitos: dois usuários em regiões diferentes atualizam simultaneamente o mesmo registro. Como você detecta e resolve o conflito? Last-write-wins? CRDTs? Merge em nível de aplicação?
4. Consistência "leia suas próprias escritas": como você garante isso quando as leituras podem ser servidas de uma réplica local que está 200ms atrás do primário?
5. Como você testa o failover regional em produção (chaos engineering)? Que métricas dizem que o failover foi bem-sucedido?
6. Opções de banco de dados: CockroachDB (SQL distribuído), YugabyteDB, Cassandra (multi-região), Aurora Global Database — compare-os para este caso de uso.

**Esboço de solução:**
- Particionamento de dados por `tenantRegion`: dados de tenants da EU vivem apenas na EU. Sem replicação cross-region para essa partição.
- US e APAC: replicação assíncrona com detecção de conflito. Conflitos são raros (usuários diferentes) — LWW com timestamps funciona para a maioria dos casos; filas de conflito explícitas para dados críticos.
- Roteamento de leitura: roteamento sticky em nível de sessão para o primário após qualquer escrita (janela de 30 segundos). Depois disso, sirva da réplica local.
- Aurora Global Database: primário em US-EAST, réplicas de leitura em EU-WEST e APAC. Lag de replicação < 1 segundo. Failover promove uma réplica em < 30 segundos.
- GDPR: cluster Aurora separado em EU-WEST para tenants da EU. Não replicado fora da EU.

---

## Kata 8 — Search-as-a-Service (Intermediário)

**Sistema a projetar:** Um serviço de busca interno usado por múltiplos times de produto. Times enviam documentos (objetos JSON) e recebem resultados de busca de texto completo ranqueados.

**Restrições:**
- Times registram "índices" com um schema. Cada índice é isolado dos outros.
- A busca deve suportar: texto completo, filtros (match exato, range), ordenação e contagens facetadas.
- Indexação near-real-time: um documento indexado via API deve ser pesquisável em 5 segundos.
- Isolamento de leitura/escrita: picos de escrita pesados (reindexação em massa) não devem degradar a latência de busca.

**Requisitos de escala:**
- 20 times de produto, cada um com 1–50M documentos.
- 500 queries de busca por segundo no pico.
- Latência média de query alvo: < 100ms.
- Reindexação em massa: até 5M documentos por hora.

**Perguntas de discussão:**
1. Elasticsearch vs. OpenSearch vs. Typesense vs. solução customizada — quando você construiria customizado vs. adotaria um engine existente?
2. Como você lida com evolução de schema? Um time adiciona um novo campo ao índice. Isso exige uma reindexação completa? Como os dynamic mappings do Elasticsearch ajudam e prejudicam?
3. Como você implementa multi-tenancy com Elasticsearch? Um cluster por time, um índice por time (mesmo cluster) ou um índice com campo `tenantId`? Compare isolamento, custo e overhead operacional.
4. Isolamento de leitura/escrita: a reindexação (index refresh) do Elasticsearch é cara. Como funciona a reindexação zero-downtime baseada em alias?
5. Como você projeta a API de forma que os chamadores estejam desacoplados dos internos do Elasticsearch (para que você possa trocar o engine depois)?
6. Tuning de relevância: um time de produto reclama que os resultados de busca são irrelevantes. Que ferramentas e workflows você fornece para que eles ajustem a relevância sem seu envolvimento?

**Esboço de solução:**
- Um índice Elasticsearch por time (isolado, simples). Acesso baseado em alias: `alias: <team>-search` apontando para a versão atual do índice.
- Caminho de escrita: API REST → tópico Kafka por time → workers de indexação (fan-out). Workers escrevem em lote no Elasticsearch (bulk API) a cada 2 segundos — atinge a meta NRT de 5 segundos.
- Caminho de leitura: API REST → query builder → Elasticsearch. O query builder traduz a linguagem de query voltada ao time para DSL do Elasticsearch.
- Reindexação sem downtime: escreva no novo índice `v2` enquanto serve leituras no `v1`. Quando `v2` estiver pronto, troque o alias atomicamente. Depois delete `v1`.
- Isolamento multi-tenancy: acesso por índice com API key por time. A API key de cada time tem permissões apenas em seu índice.

---

## Kata 9 — Ledger de Transações Financeiras (Avançado)

**Sistema a projetar:** Um sistema de ledger para registrar transações financeiras com garantias de corretude. Usado por uma fintech para rastrear saldos de contas de usuários.

**Restrições:**
- Cada débito deve ter um crédito correspondente (contabilidade de partidas dobradas).
- Os saldos devem ser consistentes o tempo todo — sem cheque especial, a menos que a conta esteja explicitamente configurada para isso.
- Cada transação deve ser auditável: quem a autorizou, quando, de qual sistema.
- O ledger é append-only — sem atualizações ou deleções nos registros de transação.

**Requisitos de escala:**
- 5.000 transações por segundo no pico.
- Latência de query de saldo: < 50ms.
- Retenção de trilha de auditoria: 10 anos.
- Requisito regulatório: relatório de reconciliação diário deve fechar em zero.

**Perguntas de discussão:**
1. Contabilidade de partidas dobradas: modele o schema. Que tabelas você precisa? Que invariantes devem ser mantidos no nível do banco?
2. Cálculo de saldo: somar todas as transações de uma conta vs. uma coluna de saldo corrente. Quais são as implicações de performance? Como você lida com contas grandes com milhões de entradas?
3. Como você evita que transações concorrentes gerem cheque especial em uma conta? Lock pessimista em nível de linha? Lock otimista com coluna de versão? Nível de isolamento serializável?
4. O ledger é append-only. Como isso afeta backup, arquivamento e performance de query ao longo do tempo (à medida que a tabela cresce para bilhões de linhas)?
5. Event sourcing é um encaixe natural aqui. Como o sistema de ledger mapeia para o padrão de event sourcing? O que é o "aggregate" e quais são os "events"?
6. Reconciliação: ao final do dia, a soma de todos os débitos deve ser igual à soma de todos os créditos. Como você implementa essa verificação eficientemente sem travar a tabela?

**Esboço de solução:**
- Schema: `accounts(id, type, currency, created_at)`, `journal_entries(id, account_id, amount, direction ENUM(debit,credit), txn_id, created_at, metadata JSONB)`, `transactions(id, description, initiated_by, created_at)`.
- Query de saldo: saldo materializado em uma tabela `account_balances`, atualizado via trigger no banco ou código de aplicação dentro da mesma transação. `SUM()` direto como fallback para auditoria.
- Concorrência: `SELECT ... FOR UPDATE` na linha da conta antes de inserir as journal entries. Isso serializa as escritas na mesma conta.
- Arquivamento: particione `journal_entries` por ano. Anos frios (> 2 anos) movidos para object storage barato (S3), mas ainda consultáveis via Athena ou data warehouse.
- Reconciliação: job de agregação noturno (não query ao vivo) que soma por transação, verifica débito = crédito, grava um registro `reconciliation_report`.

---

## Kata 10 — Pipeline de Ingestão de Dados IoT (Avançado)

**Sistema a projetar:** Uma plataforma de ingestão de dados que recebe dados de telemetria de 5 milhões de dispositivos IoT (sensores, veículos, equipamentos industriais) em tempo real.

**Restrições:**
- Dispositivos enviam dados via MQTT ou HTTP.
- Os dados devem ser persistidos e consultáveis por ID de dispositivo e intervalo de tempo.
- Regras de detecção de anomalia executam contra o stream de entrada e acionam alertas em 10 segundos.
- Dispositivos podem enviar leituras duplicadas (at-least-once do lado do dispositivo) — deduplicação é necessária.

**Requisitos de escala:**
- 5 milhões de dispositivos, cada um enviando uma leitura a cada 30 segundos.
- Ingestão de pico: 200.000 mensagens por segundo.
- Armazenamento: 50 bytes por leitura * 200K/seg * 86.400 seg/dia = ~864 GB/dia.
- Query de série temporal p99: < 200ms para query de intervalo de 24 horas em um único dispositivo.

**Perguntas de discussão:**
1. MQTT vs. HTTP para comunicação de dispositivo: quando as conexões persistentes e os níveis QoS do MQTT o tornam a melhor escolha?
2. A 200K mensagens/segundo, como você escala a camada de ingestão sem que um único broker se torne um gargalo?
3. Bancos de dados de série temporal (TimescaleDB, InfluxDB, Apache Cassandra, ClickHouse) — compare suas características de escrita e leitura para esta carga de trabalho.
4. Deduplicação: cada mensagem tem um número de sequência gerado pelo dispositivo. Como você deduplica sem armazenar todos os números de sequência para sempre?
5. Detecção de anomalia: processamento de stream stateful (Flink, Spark Streaming, Kafka Streams) vs. regras de threshold simples aplicadas por mensagem. Quando cada um é apropriado?
6. O firmware do dispositivo envia o timestamp errado (drift do relógio do dispositivo). Como você lida e corrige isso no pipeline de ingestão?

**Esboço de solução:**
- Camada de ingestão: cluster de broker MQTT (EMQX, HiveMQ) para dispositivos MQTT; Nginx + Fastify para dispositivos HTTP. Ambos publicam no Kafka (200 partições para 200K msg/s em 1KB/msg).
- Consumers Kafka: workers de ingestão validam, deduplicam (Bloom filter por dispositivo para janela recente de 1 hora) e escrevem no banco de série temporal.
- Store de série temporal: TimescaleDB (extensão PostgreSQL) com hypertables particionadas por tempo. Políticas de compressão para dados mais antigos de 7 dias.
- Detecção de anomalia: aplicação Kafka Streams lê o tópico bruto, aplica regras em janela (ex.: temperatura > 100°C por 5 leituras consecutivas), emite para um tópico `alerts`. Consumer de alertas envia notificações.
- Deduplicação: Bloom filter por dispositivo no Redis (TTL de 1 hora). Aceite alguns falsos positivos (reenvio é idempotente — dados do dispositivo são os mesmos).
- Correção de clock drift: registre `device_timestamp` e `server_timestamp`. Use `server_timestamp` como eixo de tempo primário para queries. Exponha `clock_drift_seconds` como métrica.
