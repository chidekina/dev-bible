# Desafios de IA / LLM

> Exercícios práticos para construir funcionalidades de IA em produção: pipelines RAG, uso de ferramentas, streaming, engenharia de prompts e sistemas agênticos. Stack assumida: Node.js 20+, TypeScript, Anthropic SDK, Vercel AI SDK, pgvector, PostgreSQL. Nível alvo: intermediário.

---

## Exercício 1 — Construir um Pipeline RAG com pgvector (Difícil)

**Cenário:** Construa um pipeline de Retrieval-Augmented Generation (RAG) que responde perguntas sobre a documentação interna da sua empresa armazenada no PostgreSQL com a extensão `pgvector`.

**Requisitos:**
- Pipeline de ingestão: leia documentos Markdown, divida em chunks (500 tokens, sobreposição de 50 tokens), gere embeddings usando a API Anthropic ou OpenAI `text-embedding-3-small` e armazene na tabela `document_chunks`.
- Pipeline de consulta: faça embedding da pergunta do usuário, encontre os 5 chunks mais similares usando similaridade de cosseno, injete-os como contexto em um prompt e retorne a resposta do LLM.
- Schema de `document_chunks`: `id`, `document_id`, `content` (texto), `embedding` (vector(1536)), `metadata` (JSONB).
- Citação: cada resposta deve incluir o nome do documento fonte e o índice do chunk.
- Perguntas sem resposta (nenhum contexto relevante acima de um threshold de similaridade de 0,75) devem retornar `"Não tenho informações sobre isso."` em vez de alucinar.

**Critérios de Aceite:**
- [ ] `POST /rag/ingest` aceita um arquivo Markdown e armazena todos os chunks com embeddings.
- [ ] `POST /rag/query` retorna uma resposta com citações dentro de 3 segundos.
- [ ] Uma pergunta sem documentos relevantes retorna a resposta padrão "Não tenho informações".
- [ ] `EXPLAIN ANALYZE` na query de similaridade mostra índice IVFFlat ou HNSW sendo usado.
- [ ] A sobreposição de chunk está implementada corretamente — os últimos 50 tokens do chunk N são os primeiros 50 tokens do chunk N+1.

**Dicas:**
1. Setup do pgvector: `CREATE EXTENSION IF NOT EXISTS vector; ALTER TABLE document_chunks ADD COLUMN embedding vector(1536);`.
2. Query de similaridade de cosseno: `SELECT content, 1 - (embedding <=> $1::vector) AS similarity FROM document_chunks ORDER BY embedding <=> $1::vector LIMIT 5`.
3. Índice IVFFlat: `CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`. Execute `VACUUM ANALYZE` após insert em massa.
4. Threshold de similaridade: filtre resultados com `WHERE 1 - (embedding <=> $1) > 0.75` antes de injetar no prompt.
5. Injeção de contexto: `Você é um assistente útil. Responda apenas com base no seguinte contexto:\n\n${chunks.map(c => c.content).join('\n\n')}\n\nPergunta: ${question}`.

---

## Exercício 2 — Implementar Uso de Ferramentas com Anthropic SDK (Médio)

**Cenário:** Construa um assistente alimentado por Claude que pode responder perguntas chamando ferramentas externas: uma API de clima e uma calculadora. O assistente decide quando chamar ferramentas e quando responder diretamente.

**Requisitos:**
- Defina duas ferramentas: `get_weather({ city: string }): { temperature, condition }` e `calculate({ expression: string }): { result: number }`.
- A ferramenta `calculate` avalia expressões matemáticas seguras (sem `eval` — use um parser como `mathjs`).
- Implemente o loop agêntico: envie mensagem → se Claude retornar chamadas de ferramentas, execute-as, envie resultados de volta → repita até Claude retornar uma resposta de texto.
- Trate erros de ferramentas graciosamente: se `get_weather` lançar, envie um resultado de erro de volta para Claude para que ele possa informar o usuário.
- A conversa é multi-turno — mantenha o histórico de mensagens entre as requisições.

**Critérios de Aceite:**
- [ ] "Como está o tempo em São Paulo?" dispara uma chamada de ferramenta `get_weather` com `city: "São Paulo"`.
- [ ] "Quanto é 2^10 + 5?" dispara uma chamada de ferramenta `calculate` com a expressão correta.
- [ ] Uma falha na API de clima resulta em Claude respondendo "Não consegui obter as informações de clima para essa cidade".
- [ ] O loop agêntico termina quando Claude retorna `stop_reason: "end_turn"`.
- [ ] Não mais de 5 iterações de chamada de ferramenta por mensagem do usuário (proteção contra loop infinito).

**Dicas:**
1. Definição de ferramenta:
   ```typescript
   const tools = [{
     name: 'get_weather',
     description: 'Obtenha o clima atual de uma cidade',
     input_schema: { type: 'object', properties: { city: { type: 'string' } }, required: ['city'] }
   }];
   ```
2. Loop agêntico: após uma resposta `tool_use`, crie uma mensagem `tool_result` e chame a API novamente.
3. Resultado de ferramenta para erros: `{ type: 'tool_result', tool_use_id: toolUse.id, is_error: true, content: errorMessage }`.
4. `mathjs` eval: `import { evaluate } from 'mathjs'; evaluate('2^10 + 5')` — seguro, sem execução de código arbitrário.

---

## Exercício 3 — Fazer Streaming de uma Resposta de Chat com Vercel AI SDK (Médio)

**Cenário:** Construa um endpoint de chat com streaming que retorna tokens para o cliente conforme são gerados, para que a UI possa exibir a resposta progressivamente.

**Requisitos:**
- `POST /chat` aceita `{ messages: Message[], model: string }` e faz streaming da resposta usando o `streamText` do Vercel AI SDK.
- A resposta deve usar o protocolo de data stream do AI SDK (compatível com o hook `useChat` do React).
- Trate erros de streaming graciosamente — se a API do LLM retornar um erro no meio do stream, envie um chunk de erro e feche o stream.
- Implemente um limite de `maxTokens` de 2048 por resposta.
- Registre o total de tokens usados (prompt + completion) após o stream completar.

**Critérios de Aceite:**
- [ ] O cliente recebe o primeiro token dentro de 500ms da requisição.
- [ ] O hook `useChat` em um frontend React renderiza tokens em streaming corretamente.
- [ ] Um erro da API do LLM no meio do stream é enviado ao cliente como `{ error: "Erro do LLM: ..." }` no stream, não como resposta 500.
- [ ] Após o stream, `POST /analytics/usage` é chamado com `{ promptTokens, completionTokens, model }`.
- [ ] `maxTokens: 2048` é imposto — respostas são cortadas no limite com stop reason `max_tokens`.

**Dicas:**
1. Streaming com Vercel AI SDK:
   ```typescript
   import { streamText } from 'ai';
   import { anthropic } from '@ai-sdk/anthropic';
   const result = streamText({ model: anthropic('claude-sonnet-4-6'), messages, maxTokens: 2048 });
   return result.toDataStreamResponse();
   ```
2. Log de tokens: `result.usage.then(usage => logUsage(usage))` — a promise resolve após o stream completar.
3. Tratamento de erros: envolva em try/catch. No erro, use `result.toDataStreamResponse()` com um transformador de erro ou escreva manualmente no stream de resposta.
4. Fastify + streams: use `reply.raw` para escrever a resposta de stream diretamente.

---

## Exercício 4 — Implementar Cache de Prompts para Reduzir Custos (Médio)

**Cenário:** Seu sistema RAG envia o mesmo prompt de sistema e contexto de documentação em cada requisição. Use o cache de prompts da Anthropic para evitar reprocessar prefixos não alterados.

**Requisitos:**
- Marque o prompt de sistema e o contexto do documento como cacheáveis usando `cache_control: { type: "ephemeral" }`.
- Meça a diferença de custo entre requisições com e sem cache usando o campo `usage` da resposta.
- Implemente um aquecimento de cache: na inicialização do app, envie uma requisição dummy para garantir que o cache esteja pronto antes do tráfego de usuários.
- Registre `cache_read_input_tokens` e `cache_creation_input_tokens` em cada requisição.
- O cache deve reduzir os custos em pelo menos 80% em queries repetidas com o mesmo contexto.

**Critérios de Aceite:**
- [ ] Primeira requisição mostra `cache_creation_input_tokens > 0` e `cache_read_input_tokens = 0`.
- [ ] Requisições subsequentes com o mesmo contexto mostram `cache_read_input_tokens > 0` e `cache_creation_input_tokens = 0`.
- [ ] Cálculo de custo: `promptTokens * 3/1000000 (sem cache) vs. cacheReadTokens * 0.3/1000000 (com cache)` — verifique a redução de custo de 10x.
- [ ] O bloco de prompt de sistema tem `cache_control: { type: "ephemeral" }` como último campo no bloco de conteúdo.
- [ ] O aquecimento de cache completa antes da primeira requisição de usuário.

**Dicas:**
1. Conteúdo elegível para cache: o prompt de sistema + blocos de contexto grandes. Deve ter pelo menos 1024 tokens para ser cacheado.
2. Bloco de conteúdo com cache control:
   ```typescript
   { type: 'text', text: systemPrompt, cache_control: { type: 'ephemeral' } }
   ```
3. Apenas o último bloco `cache_control` no array de mensagens é usado para cache — coloque-o no maior prefixo estável.
4. TTL do cache é 5 minutos por padrão — projete o aquecimento para re-disparar se o servidor ficar ocioso por mais de 4 minutos.

---

## Exercício 5 — Construir um Roteador de Modelos (Médio)

**Cenário:** Nem todas as queries de usuários requerem um modelo de fronteira caro. Construa um roteador que envia queries simples para um modelo rápido e barato e queries complexas para um modelo capaz.

**Requisitos:**
- Classifique cada query como `simple` ou `complex` usando uma heurística leve ou uma chamada de classificador barato.
- Queries simples (consultas factuais, saudações, respostas curtas): roteie para `claude-haiku-4-5`.
- Queries complexas (raciocínio multi-etapas, geração de código, escrita longa): roteie para `claude-sonnet-4-6`.
- Registre o modelo usado, motivo da decisão e custo em tokens para cada query.
- Permita que usuários sobrescrevam com `?model=sonnet` ou `?model=haiku` como query params.

**Critérios de Aceite:**
- [ ] "Qual é a capital do Brasil?" é roteada para Haiku.
- [ ] "Escreva um middleware Fastify que implementa auth JWT com rotação de refresh token" é roteado para Sonnet.
- [ ] `?model=haiku` força Haiku independente da complexidade da query.
- [ ] Decisão de roteamento é registrada: `{ query_type: 'simple', model: 'claude-haiku-4-5', reason: 'query factual curta' }`.
- [ ] O classificador em si usa Haiku (não Sonnet) para manter o custo de classificação mínimo.

**Dicas:**
1. Heurística simples: comprimento da query < 50 palavras E sem tokens parecidos com código E sem palavras-chave complexas → `simple`. Caso contrário → use um classificador.
2. Prompt do classificador: `Classifique esta query como 'simple' ou 'complex'. Responda apenas com a palavra. Query: "${query}"`. Use `maxTokens: 10` baixo.
3. Comparação de custo: Haiku é ~20x mais barato por token que Sonnet. Rotear 70% das queries para Haiku reduz significativamente os custos de LLM.
4. Param de override: `const model = req.query.model === 'haiku' ? 'claude-haiku-4-5' : req.query.model === 'sonnet' ? 'claude-sonnet-4-6' : routedModel`.

---

## Exercício 6 — Escrever uma Suite de Avaliação de Prompts (Difícil)

**Cenário:** O prompt da sua funcionalidade de IA foi alterado no sprint passado. Você não tem como saber se o novo prompt é melhor ou pior. Construa uma suite de avaliação LLM-as-judge.

**Requisitos:**
- Crie um dataset de 20 casos de teste: `{ input: string, expectedBehavior: string }`.
- Para cada caso de teste, execute o prompt e colete a resposta.
- Use um LLM judge (uma chamada Claude separada) para pontuar cada resposta: `{ score: 1-5, reasoning: string }`.
- Compare duas versões de prompt (A e B) e produza um resumo: `{ promptA: { avgScore, passRate }, promptB: { avgScore, passRate } }`.
- Um "pass" é definido como pontuação de 4 ou 5 do judge.

**Critérios de Aceite:**
- [ ] O script de avaliação roda com `npx ts-node eval.ts --promptA prompts/v1.txt --promptB prompts/v2.txt`.
- [ ] A saída é um arquivo JSON com pontuações por caso de teste e um resumo comparativo.
- [ ] O prompt do judge instrui o LLM a ser um avaliador rigoroso: `"Você é um avaliador de QA rigoroso. Pontue esta resposta de 1-5 com base em [critérios]. Responda com JSON: { score, reasoning }"`.
- [ ] O judge usa `claude-haiku-4-5` para minimizar o custo de avaliação.
- [ ] O dataset de teste cobre casos extremos: queries ambíguas, inputs fora do tópico, queries que requerem recusa.

**Dicas:**
1. Chamada do judge: envie `{ input, expectedBehavior, actualResponse }` para o modelo judge. Faça parse do JSON da resposta.
2. Execute avaliações em paralelo: `await Promise.all(testCases.map(tc => evaluateTestCase(tc)))` — muito mais rápido que sequencial.
3. Taxa de aprovação: `(scores.filter(s => s >= 4).length / scores.length) * 100`.
4. Previna alucinação do judge: use um schema JSON ou `z.object({ score: z.number().min(1).max(5), reasoning: z.string() })` para validar a saída do judge.

---

## Exercício 7 — Implementar Retry + Fallback para Erros da API LLM (Médio)

**Cenário:** Sua integração LLM trava silenciosamente em erros de API (rate limits, timeouts, sobrecarga). Implemente uma estratégia robusta de retry e fallback.

**Requisitos:**
- Retente em: `429 Too Many Requests` (rate limit), `529 Overloaded`, erros de servidor `500/503`.
- Não retente em: `400 Bad Request` (prompt inválido), `401 Unauthorized` (chave de API inválida), `404`.
- Backoff exponencial: 1s, 2s, 4s (máx 3 tentativas por requisição).
- Fallback: após 3 tentativas com Sonnet, caia para Haiku uma vez antes de desistir.
- Com todas as tentativas esgotadas: retorne um erro amigável `{ error: "Serviço de IA temporariamente indisponível, por favor tente novamente." }`.

**Critérios de Aceite:**
- [ ] Uma resposta 429 dispara um retry após ~1 segundo.
- [ ] Uma resposta 400 não é retentada — é retornada imediatamente como erro.
- [ ] Após 3 tentativas com falha no Sonnet, a mesma requisição é tentada uma vez com Haiku.
- [ ] Todas as tentativas de retry e o resultado final são registrados com campos estruturados: `{ attempt, model, status, delay }`.
- [ ] Um teste usando um servidor API mock verifica o cronograma de retry sem chamadas de API reais.

**Dicas:**
1. Códigos de status retentáveis: `[429, 500, 503, 529]`. Não-retentáveis: `[400, 401, 403, 404]`.
2. Cálculo do backoff: `const delay = Math.min(1000 * 2 ** attempt, 30000)`. Adicione jitter: `delay + Math.random() * 1000`.
3. Padrão de fallback:
   ```typescript
   try {
     return await callWithRetry('claude-sonnet-4-6', prompt);
   } catch {
     return await callWithRetry('claude-haiku-4-5', prompt, { maxAttempts: 1 });
   }
   ```
4. Teste com um servidor mock que retorna `429` para as primeiras 3 requisições, depois `200` — verifique o número correto de retries.

---

## Exercício 8 — Construir um Pipeline de Chunking de Documentos (Médio)

**Cenário:** Construa um pipeline robusto de chunking de documentos que divide documentos em chunks adequados para embeddings enquanto preserva limites semânticos.

**Requisitos:**
- Entrada: documentos Markdown com headings, parágrafos e blocos de código.
- Estratégia de chunk: divida nos limites de heading primeiro. Se uma seção tiver > 500 tokens, divida nos limites de parágrafo. Se um parágrafo tiver > 500 tokens, divida nos limites de frase.
- Preserve metadados por chunk: `{ documentId, heading, chunkIndex, totalChunks, tokenCount }`.
- Sobreposição: cada chunk (exceto o primeiro) começa com a última frase do chunk anterior.
- Saída: `Chunk[]` com conteúdo e metadados.

**Critérios de Aceite:**
- [ ] Blocos de código nunca são divididos no meio (blocos de código são tratados como unidades atômicas).
- [ ] O `tokenCount` de cada chunk é preciso (use `tiktoken` ou `@anthropic-ai/tokenizer`).
- [ ] A sobreposição está presente: a última frase do chunk N aparece como a primeira frase do chunk N+1.
- [ ] Um teste processa um documento de 10.000 tokens e verifica que todos os chunks têm ≤ 500 tokens.
- [ ] A função `splitSection(section)` é recursiva: se tokens <= 500, retorna como está; caso contrário, divide em parágrafos e aplica recursivamente.

**Dicas:**
1. Parse da estrutura Markdown com `remark` ou regex manual: identifique headings (`^#{1,6} `), cercas de código (` ``` `) e parágrafos (dupla quebra de linha).
2. Contagem de tokens: `import { encode } from 'gpt-tokenizer'; encode(text).length` — compatível com contagens de tokens da Anthropic (aproximadamente).
3. Blocos de código atômicos: quando você encontra uma cerca de código, colete o bloco inteiro como uma unidade e não o divida, mesmo se exceder o limite de tokens.
4. Recursão: `splitSection(section)` → se tokens <= 500, retorne como está; caso contrário, divida em parágrafos e aplique recursivamente.

---

## Exercício 9 — Adicionar Contagem de Tokens Antes de Enviar para a API (Fácil)

**Cenário:** Sua aplicação de chat ocasionalmente envia prompts que excedem a janela de contexto do modelo, causando erros de API crípticos. Adicione contagem de tokens para validar e truncar prompts antes de enviar.

**Requisitos:**
- Conte tokens no histórico de mensagens antes de enviar para a API.
- Se a contagem de tokens exceder 80% da janela de contexto do modelo, trunce as mensagens mais antigas (não o prompt de sistema ou a última mensagem do usuário).
- Registre um aviso quando truncamento ocorrer: `{ originalMessages: N, truncatedTo: M, tokens: T }`.
- Exponha um endpoint `GET /chat/token-count` que retorna a contagem de tokens atual para um dado array de mensagens.
- Suporte múltiplos modelos com diferentes tamanhos de janela de contexto: `claude-sonnet-4-6` (200k), `claude-haiku-4-5` (200k).

**Critérios de Aceite:**
- [ ] Um histórico de mensagens de 250.000 tokens é truncado para 160.000 tokens (80% de 200k) antes de enviar.
- [ ] O prompt de sistema é sempre preservado (nunca truncado).
- [ ] A última mensagem do usuário é sempre preservada (nunca truncada).
- [ ] `GET /chat/token-count` retorna `{ messages: N, tokens: T, model: "claude-sonnet-4-6", withinLimit: true }`.
- [ ] Um teste unitário verifica a lógica de truncamento com um histórico de mensagens mock.

**Dicas:**
1. API `countTokens` da Anthropic: `anthropic.messages.countTokens({ model, messages, system })` retorna `{ input_tokens }`.
2. Alternativamente, use `tiktoken` localmente para contagem de tokens gratuita (aproximada para modelos Claude).
3. Algoritmo de truncamento: mantenha o prompt de sistema + última mensagem do usuário. Então adicione gulodosamente mensagens do mais novo ao mais antigo até atingir o limite.
4. Threshold de 80%: `const limit = MODEL_CONTEXT_WINDOWS[model] * 0.8`.

---

## Exercício 10 — Construir um Loop de Agente Simples com Uso de Ferramentas e Memória (Difícil)

**Cenário:** Construa um agente autônomo simples que pode navegar em um banco de dados fictício de produtos, adicionar itens a uma lista de compras e responder perguntas sobre a lista. O agente tem memória persistente entre turnos.

**Requisitos:**
- Ferramentas: `search_products({ query: string })`, `add_to_list({ productId: string, quantity: number })`, `get_list()`, `remove_from_list({ productId: string })`.
- Memória: a lista de compras persiste entre os turnos da conversa (armazenada no Redis ou em um Map em memória).
- O loop do agente roda até a tarefa ser concluída ou o usuário dizer "pronto".
- Implemente uma proteção de máximo de iterações (10 iterações por turno do usuário).
- O agente deve explicar seu raciocínio antes de chamar ferramentas (chain-of-thought via blocos `thinking` ou um `scratchpad` no prompt de sistema).

**Critérios de Aceite:**
- [ ] "Encontre maçãs orgânicas e adicione 3 à minha lista" resulta em uma chamada `search_products` seguida de `add_to_list`.
- [ ] "O que está na minha lista?" dispara `get_list` e retorna uma resposta formatada.
- [ ] Após 10 chamadas de ferramentas sem end_turn, o loop do agente para e retorna `"Limite de tarefas atingido."`.
- [ ] A lista de compras persiste: adicionar itens na mensagem 1 e consultar na mensagem 3 retorna a mesma lista.
- [ ] Um teste cobre o loop completo: busca → adiciona → get_list → verifica.

**Dicas:**
1. Loop do agente:
   ```typescript
   while (iterations < maxIterations) {
     const response = await claude.messages.create({ tools, messages });
     if (response.stop_reason === 'end_turn') break;
     const toolResults = await executeTools(response.content);
     messages.push({ role: 'assistant', content: response.content });
     messages.push({ role: 'user', content: toolResults });
     iterations++;
   }
   ```
2. Memória: `const lists = new Map<userId, CartItem[]>()`. `get_list` e `add_to_list` leem/escrevem neste map.
3. Execução de ferramenta: filtre `response.content` por blocos `type === 'tool_use'`. Execute cada ferramenta e colete blocos `tool_result`.
4. Chain-of-thought: adicione ao prompt de sistema `"Antes de chamar uma ferramenta, explique brevemente seu raciocínio em uma frase."` — isso produz comportamento mais previsível.
