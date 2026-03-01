# Padrões de Integração com IA

## Visão Geral

Integrar LLMs em aplicações de produção exige mais do que chamar uma API. Você precisa de respostas em streaming para UX em tempo real, tratamento de erros elegante para rate limits e interrupções, checkpoints de human-in-the-loop para decisões de alto risco e padrões de loop de agente para tarefas de múltiplas etapas. Este capítulo cobre os padrões de produção para construir funcionalidades de IA confiáveis e amigáveis ao usuário com o Vercel AI SDK e integração direta com a API.

## Pré-requisitos

- Fundamentos de LLM
- Noções básicas de Prompt Engineering
- Tool Use / Function Calling
- TypeScript, Node.js, React

## Conceitos Fundamentais

### Respostas em Streaming

Streaming faz as respostas de IA parecerem instantâneas — o usuário vê o texto aparecer palavra por palavra em vez de esperar pela resposta inteira.

Sem streaming, uma resposta de 500 palavras pode levar 10+ segundos para aparecer. Com streaming, as primeiras palavras aparecem em 1-2 segundos.

```typescript
// Padrão Server-Sent Events (SSE) para streaming
// Node.js / Express
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

app.post('/api/chat/stream', async (req, res) => {
  const { message } = req.body;

  // Define os headers de SSE
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('Access-Control-Allow-Origin', '*');

  try {
    const stream = anthropic.messages.stream({
      model: 'claude-sonnet-4-6',
      max_tokens: 2048,
      messages: [{ role: 'user', content: message }],
    });

    for await (const event of stream) {
      if (
        event.type === 'content_block_delta' &&
        event.delta.type === 'text_delta'
      ) {
        // Formato SSE: "data: {json}\n\n"
        res.write(`data: ${JSON.stringify({ type: 'text', text: event.delta.text })}\n\n`);
      }
    }

    const finalMessage = await stream.finalMessage();
    res.write(`data: ${JSON.stringify({
      type: 'done',
      usage: finalMessage.usage,
    })}\n\n`);
  } catch (err) {
    res.write(`data: ${JSON.stringify({ type: 'error', error: (err as Error).message })}\n\n`);
  } finally {
    res.end();
  }
});
```

### Vercel AI SDK

O Vercel AI SDK fornece uma interface unificada para respostas de IA em streaming entre múltiplos provedores, com hooks React integrados.

```bash
npm install ai @ai-sdk/anthropic @ai-sdk/openai
```

```typescript
// app/api/chat/route.ts (Next.js App Router)
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-6'),
    system: 'Você é um assistente útil.',
    messages,
    maxTokens: 2048,
  });

  return result.toDataStreamResponse();
}
```

```typescript
// hooks/useChat.ts — ou use o hook integrado de 'ai/react'
import { useChat } from 'ai/react';

export function ChatInterface() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } = useChat({
    api: '/api/chat',
    onFinish: (message) => {
      console.log('Mensagem final:', message);
    },
    onError: (err) => {
      console.error('Erro no chat:', err);
    },
  });

  return (
    <div>
      <div className="messages">
        {messages.map((m) => (
          <div key={m.id} className={`message ${m.role}`}>
            {m.content}
          </div>
        ))}
        {isLoading && <div className="loading">Pensando...</div>}
        {error && <div className="error">Erro: {error.message}</div>}
      </div>

      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Envie uma mensagem..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>Enviar</button>
      </form>
    </div>
  );
}
```

### Vercel AI SDK com Ferramentas

```typescript
// app/api/chat/route.ts
import { streamText, tool } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-6'),
    messages,
    tools: {
      searchProducts: tool({
        description: 'Busca o catálogo de produtos',
        parameters: z.object({
          query: z.string().describe('Termos de busca'),
          category: z.enum(['electronics', 'clothing', 'books']).optional(),
        }),
        execute: async ({ query, category }) => {
          return await productService.search(query, category);
        },
      }),
      getWeather: tool({
        description: 'Obtém o clima atual para uma cidade',
        parameters: z.object({
          city: z.string().describe('Nome da cidade'),
        }),
        execute: async ({ city }) => {
          return await weatherApi.current(city);
        },
      }),
    },
    maxSteps: 5,  // o SDK gerencia o loop do agente automaticamente
  });

  return result.toDataStreamResponse();
}
```

### Human-in-the-Loop

Para decisões de alto risco (excluir dados, enviar e-mails, fazer compras), interrompa o loop do agente e exija aprovação humana explícita antes de prosseguir.

```typescript
interface PendingAction {
  id: string;
  toolName: string;
  toolInput: Record<string, unknown>;
  description: string;
  requestedAt: Date;
}

class HumanInLoopAgent {
  private pendingActions = new Map<string, PendingAction>();

  async run(
    userMessage: string,
    requiresApproval: (toolName: string) => boolean,
    onPendingAction: (action: PendingAction) => void
  ): Promise<string> {
    const messages: Anthropic.MessageParam[] = [
      { role: 'user', content: userMessage },
    ];

    for (let i = 0; i < 10; i++) {
      const response = await anthropic.messages.create({
        model: 'claude-sonnet-4-6',
        max_tokens: 4096,
        tools: registry.getAnthropicTools(),
        messages,
      });

      if (response.stop_reason === 'end_turn') {
        return (response.content[0] as Anthropic.TextBlock).text;
      }

      messages.push({ role: 'assistant', content: response.content });

      const toolResults: Anthropic.ToolResultBlockParam[] = [];

      for (const block of response.content.filter(
        (b): b is Anthropic.ToolUseBlock => b.type === 'tool_use'
      )) {
        if (requiresApproval(block.name)) {
          // Pausa e aguarda aprovação humana
          const actionId = crypto.randomUUID();
          const action: PendingAction = {
            id: actionId,
            toolName: block.name,
            toolInput: block.input as Record<string, unknown>,
            description: `Executar ${block.name} com: ${JSON.stringify(block.input)}`,
            requestedAt: new Date(),
          };

          this.pendingActions.set(actionId, action);
          onPendingAction(action);

          // Aguarda aprovação (polling ou canal)
          const approved = await this.waitForApproval(actionId, 60_000);

          if (!approved) {
            toolResults.push({
              type: 'tool_result',
              tool_use_id: block.id,
              content: JSON.stringify({ error: 'Ação rejeitada pelo usuário' }),
              is_error: true,
            });
            continue;
          }
        }

        const result = await registry.execute(block.name, block.input as Record<string, unknown>);
        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: JSON.stringify(result),
        });
      }

      messages.push({ role: 'user', content: toolResults });
    }

    throw new Error('Número máximo de iterações atingido');
  }

  async approve(actionId: string): Promise<void> {
    this.pendingActions.delete(actionId);
    // Sinaliza a promise aguardando
  }

  async reject(actionId: string): Promise<void> {
    this.pendingActions.delete(actionId);
    // Sinaliza a promise aguardando
  }

  private async waitForApproval(actionId: string, timeoutMs: number): Promise<boolean> {
    // Na prática: use um banco de dados ou Redis para armazenar ações pendentes
    // e faça a UI fazer polling ou use WebSockets
    return new Promise((resolve) => {
      const timeout = setTimeout(() => resolve(false), timeoutMs);
      // Escuta eventos de aprovação/rejeição...
    });
  }
}
```

### Tratamento de Erros e Retentativa

```typescript
import pRetry, { AbortError } from 'p-retry';

class RobustAIClient {
  constructor(private anthropic: Anthropic) {}

  async complete(params: Anthropic.MessageCreateParamsNonStreaming): Promise<string> {
    return pRetry(
      async () => {
        try {
          const response = await this.anthropic.messages.create(params);
          return (response.content[0] as Anthropic.TextBlock).text;
        } catch (err) {
          if (err instanceof Anthropic.APIError) {
            // Não tenta novamente erros do cliente (exceto rate limits)
            if (err.status === 400 || err.status === 401 || err.status === 404) {
              throw new AbortError(err.message);
            }
            // Tenta novamente rate limits e erros do servidor
            if (err.status === 429 || err.status >= 500) {
              throw err; // pRetry vai tentar novamente
            }
          }
          throw new AbortError((err as Error).message); // erro desconhecido, não tenta novamente
        }
      },
      {
        retries: 3,
        factor: 2,
        minTimeout: 1000,
        maxTimeout: 30000,
        onFailedAttempt: (error) => {
          console.warn(`Tentativa ${error.attemptNumber} da API de IA falhou: ${error.message}`);
        },
      }
    );
  }
}
```

### Cache de Respostas

```typescript
import { createHash } from 'crypto';
import { redis } from '../redis';

function hashParams(params: Record<string, unknown>): string {
  return createHash('sha256')
    .update(JSON.stringify(params))
    .digest('hex');
}

async function cachedCompletion(
  params: Anthropic.MessageCreateParamsNonStreaming,
  ttlSeconds = 3600
): Promise<string> {
  const cacheKey = `ai:${hashParams(params)}`;

  // Verifica cache
  const cached = await redis.get(cacheKey);
  if (cached) return cached;

  // Chama a API
  const response = await anthropic.messages.create(params);
  const text = (response.content[0] as Anthropic.TextBlock).text;

  // Armazena em cache
  await redis.setex(cacheKey, ttlSeconds, text);
  return text;
}

// Use para: geração de conteúdo estático, resposta de FAQ, classificação
// Não use para: conteúdo personalizado, dados em tempo real, consultas dinâmicas
```

## Exemplos Práticos

### Chat com Gerenciamento de Context Window

```typescript
// Gerencia o histórico de conversas para ficar dentro dos limites de contexto
const MAX_HISTORY_TOKENS = 50_000;

function trimHistory(
  history: Anthropic.MessageParam[],
  maxTokens: number
): Anthropic.MessageParam[] {
  // Sempre mantém o system prompt e as mensagens recentes
  // Trunca do meio do histórico
  let estimatedTokens = history.reduce(
    (sum, msg) =>
      sum + (typeof msg.content === 'string' ? msg.content.length : 1000) / 4,
    0
  );

  let trimmed = [...history];
  while (estimatedTokens > maxTokens && trimmed.length > 2) {
    // Remove a segunda mensagem (mantém a primeira mensagem do usuário para contexto)
    trimmed.splice(1, 1);
    estimatedTokens = trimmed.reduce(
      (sum, msg) =>
        sum + (typeof msg.content === 'string' ? msg.content.length : 1000) / 4,
      0
    );
  }

  return trimmed;
}

class ManagedChat {
  private history: Anthropic.MessageParam[] = [];

  async send(userMessage: string): Promise<string> {
    this.history.push({ role: 'user', content: userMessage });

    const trimmedHistory = trimHistory(this.history, MAX_HISTORY_TOKENS);

    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 2048,
      system: 'Você é um assistente útil.',
      messages: trimmedHistory,
    });

    const text = (response.content[0] as Anthropic.TextBlock).text;
    this.history.push({ role: 'assistant', content: text });
    return text;
  }
}
```

### Geração Estruturada com Fallback

```typescript
import { z } from 'zod';

async function generateStructured<T>(
  prompt: string,
  schema: z.ZodType<T>,
  fallback?: T
): Promise<T> {
  const schemaDescription = JSON.stringify(
    (schema as any)._def ?? schema,
    null,
    2
  );

  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 1024,
      temperature: 0,
      system: `Responda apenas com JSON válido. Sem explicação, sem markdown, sem blocos de código.`,
      messages: [{
        role: 'user',
        content: `${prompt}\n\nRetorne JSON correspondendo a este schema:\n${schemaDescription}`,
      }],
    });

    try {
      const raw = (response.content[0] as Anthropic.TextBlock).text.trim();
      const parsed = JSON.parse(raw);
      return schema.parse(parsed);
    } catch (err) {
      if (attempt === 2) {
        if (fallback !== undefined) return fallback;
        throw new Error(`Falha ao parsear output estruturado após 3 tentativas: ${err}`);
      }
      // Tenta novamente: a próxima tentativa inclui o erro
    }
  }

  throw new Error('Inalcançável');
}

// Uso
const ProductSchema = z.object({
  name: z.string(),
  price: z.number().positive(),
  category: z.enum(['electronics', 'clothing', 'books']),
  tags: z.array(z.string()),
});

const product = await generateStructured(
  'Gere uma listagem de produto para um teclado sem fio',
  ProductSchema
);
```

### Processamento em Lote em Segundo Plano

```typescript
import PQueue from 'p-queue';

const queue = new PQueue({
  concurrency: 5,      // máximo de 5 chamadas de API simultâneas
  intervalCap: 10,     // máximo de 10 chamadas por intervalo
  interval: 1000,      // por segundo (respeita rate limits)
});

async function classifyBatch(items: Array<{ id: string; text: string }>) {
  const results: Array<{ id: string; category: string }> = [];

  await Promise.all(
    items.map((item) =>
      queue.add(async () => {
        const category = await classifyText(item.text);
        results.push({ id: item.id, category });
      })
    )
  );

  return results;
}

// Processa 10.000 itens com conformidade de rate limit
const allItems = await db.query('SELECT id, content FROM articles WHERE classified = false');
const classified = await classifyBatch(allItems.rows);

await db.transaction(async (tx) => {
  for (const { id, category } of classified) {
    await tx.query(
      'UPDATE articles SET category = $1, classified = true WHERE id = $2',
      [category, id]
    );
  }
});
```

## Padrões Comuns e Boas Práticas

### Abortar ao Desconectar o Cliente

```typescript
// Next.js App Router: sinal de abort integrado
export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-6'),
    messages,
    abortSignal: req.signal,  // aborta automaticamente quando o cliente desconecta
  });

  return result.toDataStreamResponse();
}

// Abort manual com Express
app.post('/chat', async (req, res) => {
  const controller = new AbortController();

  req.on('close', () => {
    if (!res.writableEnded) {
      controller.abort();
    }
  });

  const stream = anthropic.messages.stream(
    { ... },
    { signal: controller.signal }
  );
  // ...
});
```

### Observabilidade

```typescript
// Envolva todas as chamadas de IA com logging e métricas
async function trackedCompletion(
  params: Anthropic.MessageCreateParamsNonStreaming,
  context: { userId?: string; feature: string }
): Promise<string> {
  const start = Date.now();
  const requestId = crypto.randomUUID();

  logger.info('Requisição de IA', {
    requestId,
    model: params.model,
    feature: context.feature,
    userId: context.userId,
    inputLength: JSON.stringify(params.messages).length,
  });

  try {
    const response = await anthropic.messages.create(params);
    const text = (response.content[0] as Anthropic.TextBlock).text;
    const latency = Date.now() - start;

    logger.info('Resposta de IA', {
      requestId,
      latency,
      inputTokens: response.usage.input_tokens,
      outputTokens: response.usage.output_tokens,
      stopReason: response.stop_reason,
    });

    metrics.record('ai.request.latency', latency, { feature: context.feature });
    metrics.record('ai.tokens.input', response.usage.input_tokens, { feature: context.feature });
    metrics.record('ai.tokens.output', response.usage.output_tokens, { feature: context.feature });

    return text;
  } catch (err) {
    logger.error('Requisição de IA falhou', { requestId, error: (err as Error).message });
    metrics.increment('ai.request.error', { feature: context.feature });
    throw err;
  }
}
```

### Feature Flag para Seleção de Modelo

```typescript
import { flags } from '../feature-flags';

async function getModel(userId: string): Promise<string> {
  // Teste A/B: 10% dos usuários recebem o novo modelo
  const useNewModel = await flags.evaluate('new-ai-model', userId);
  return useNewModel ? 'claude-opus-4-6' : 'claude-sonnet-4-6';
}

// Ou roteamento de modelo por complexidade de tarefa
function selectModel(tokenCount: number, taskComplexity: 'simple' | 'complex'): string {
  if (taskComplexity === 'simple' || tokenCount < 1000) {
    return 'claude-haiku-4-5';  // 3x mais barato para tarefas simples
  }
  return 'claude-sonnet-4-6';
}
```

## Anti-Padrões a Evitar

**Bloquear a thread de requisição durante o streaming**
```typescript
// Ruim: coleta o stream inteiro antes de responder
const text = await stream.finalText();
res.json({ text });

// Bom: faz streaming diretamente para o cliente
for await (const chunk of stream) { res.write(chunk); }
```

**Não tratar rate limits**
```typescript
// Ruim: falha no 429
const response = await anthropic.messages.create(params);

// Bom: tenta novamente com backoff
const response = await pRetry(() => anthropic.messages.create(params), { retries: 3 });
```

**Armazenar histórico de conversas sem limites**
Sem truncamento, o histórico de conversas cresce indefinidamente e eventualmente excede o context window.

**Ignorar o `stop_reason`**
```typescript
// Ruim: assume que a resposta está completa
const text = response.content[0].text;

// Bom: verifica se foi truncada
if (response.stop_reason === 'max_tokens') {
  logger.warn('Resposta foi truncada — aumente max_tokens ou reduza o input');
}
```

**Sem timeout nas chamadas de IA**
Chamadas à API de IA podem levar 30 a 60 segundos para outputs grandes. Sem timeout, as requisições ficam penduradas indefinidamente.

```typescript
const timeoutSignal = AbortSignal.timeout(60_000);  // timeout de 60 segundos
const response = await anthropic.messages.create(params, { signal: timeoutSignal });
```

## Depuração e Resolução de Problemas

### Depurando Problemas de Streaming

```typescript
// Registra cada evento no stream para depuração
const stream = anthropic.messages.stream({ ... });

for await (const event of stream) {
  console.log('EVENTO DE STREAM:', event.type, JSON.stringify(event).slice(0, 100));
}
```

### Inspecionando Uso e Custo

```typescript
const response = await anthropic.messages.create(params);
console.log('Uso:', response.usage);
// { input_tokens: 450, output_tokens: 230 }
// Estimativa de custo (Sonnet): 450/1M * $3 + 230/1M * $15 = $0,00480
```

### Modo Debug do Vercel AI SDK

```typescript
const result = streamText({
  model: anthropic('claude-sonnet-4-6'),
  messages,
  onChunk: ({ chunk }) => {
    console.log('CHUNK:', chunk);
  },
  onFinish: ({ text, usage, finishReason }) => {
    console.log('CONCLUÍDO:', { textLength: text.length, usage, finishReason });
  },
});
```

## Cenários do Mundo Real

### Cenário 1: App de Chat com IA no Next.js

```typescript
// app/api/chat/route.ts
import { streamText, tool } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';
import { getUser } from '@/lib/auth';
import { ratelimit } from '@/lib/ratelimit';

export async function POST(req: Request) {
  const user = await getUser(req);
  if (!user) return new Response('Unauthorized', { status: 401 });

  // Rate limit: 20 requisições por minuto por usuário
  const { success } = await ratelimit.limit(user.id);
  if (!success) return new Response('Too Many Requests', { status: 429 });

  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-6'),
    system: `Você é um assistente útil para ${user.name}. Hoje é ${new Date().toDateString()}.`,
    messages,
    maxTokens: 2048,
    abortSignal: req.signal,
    tools: {
      searchDocs: tool({
        description: 'Busca na documentação',
        parameters: z.object({ query: z.string() }),
        execute: async ({ query }) => searchDocs(query, user.orgId),
      }),
    },
    onFinish: async ({ usage }) => {
      // Rastreia uso no banco de dados
      await db.insert(aiUsage).values({
        userId: user.id,
        inputTokens: usage.promptTokens,
        outputTokens: usage.completionTokens,
        model: 'claude-sonnet-4-6',
      });
    },
  });

  return result.toDataStreamResponse();
}
```

### Cenário 2: Processamento de Documentos Assíncrono

```typescript
// Processa documentos em segundo plano com fila de jobs (BullMQ)
import { Queue, Worker } from 'bullmq';

const summaryQueue = new Queue('document-summary', { connection: redis });

// Enfileira
async function queueDocumentSummary(docId: string) {
  await summaryQueue.add('summarize', { docId }, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
  });
}

// Worker
const worker = new Worker(
  'document-summary',
  async (job) => {
    const { docId } = job.data;
    const doc = await db.query('SELECT content FROM documents WHERE id = $1', [docId]);

    const summary = await anthropic.messages.create({
      model: 'claude-haiku-4-5',  // custo-eficiente para lote
      max_tokens: 500,
      temperature: 0,
      system: 'Resuma o documento em 3 a 5 tópicos. Seja conciso.',
      messages: [{ role: 'user', content: doc.rows[0].content }],
    });

    const text = (summary.content[0] as Anthropic.TextBlock).text;
    await db.query(
      'UPDATE documents SET summary = $1, summarized_at = NOW() WHERE id = $2',
      [text, docId]
    );
  },
  { connection: redis, concurrency: 3 }  // 3 workers simultâneos
);
```

## Leituras Complementares

- [Documentação do Vercel AI SDK](https://sdk.vercel.ai/docs)
- [Guia de Streaming da Anthropic](https://docs.anthropic.com/en/api/streaming)
- [Construindo Apps de IA com Next.js — Vercel](https://vercel.com/blog/ai-sdk-3-generative-ui)
- [BullMQ para Filas de Jobs](https://docs.bullmq.io/) — processamento assíncrono de IA
- [Rate Limiting com Upstash](https://upstash.com/docs/oss/sdks/ts/ratelimit) — rate limits por usuário

## Resumo

A integração de IA em produção exige mais do que chamar a API:

**Streaming** — sempre use streaming para funcionalidades voltadas ao usuário; use SSE ou o `toDataStreamResponse()` do Vercel AI SDK. O primeiro token aparece em 1-2 segundos vs. 10+ segundos sem streaming.

**Vercel AI SDK** — `streamText`, `generateText`, `streamObject` com hooks React integrados (`useChat`, `useCompletion`). Gerencia o plumbing de SSE automaticamente.

**Tratamento de erros** — tente novamente no 429 (rate limit) e 5xx (erros do servidor) com exponential backoff. Use `AbortError` para parar de tentar novamente em erros do cliente.

**Human-in-the-loop** — pause o loop do agente antes de ações destrutivas; armazene a ação pendente, notifique o usuário, aguarde aprovação/rejeição.

**Observabilidade** — registre cada requisição com: requestId, modelo, feature, userId, latência, tokens de entrada/saída. Use esses dados para otimizar custos e depurar problemas de qualidade.

**Caching** — armazene em cache completions determinísticos (temperature=0, prompts estáticos) no Redis. Não faça cache de respostas personalizadas ou em tempo real.

**Rate limiting** — implemente rate limits por usuário antes da chamada de IA para prevenir abuso e controlar custos.

**Sinais de abort** — sempre passe `req.signal` para que a chamada de IA seja cancelada quando o cliente desconectar, economizando dinheiro em requisições abandonadas.
