# AI Integration Patterns

## Overview

Integrating LLMs into production applications requires more than calling an API. You need streaming responses for real-time UX, graceful error handling for rate limits and outages, human-in-the-loop checkpoints for high-stakes decisions, and agent loop patterns for multi-step tasks. This chapter covers the production patterns for building reliable, user-friendly AI features with the Vercel AI SDK and direct API integration.

## Prerequisites

- LLM Fundamentals
- Prompt Engineering basics
- Tool Use / Function Calling
- TypeScript, Node.js, React

## Core Concepts

### Streaming Responses

Streaming makes AI responses feel instant — the user sees text appear word by word instead of waiting for the entire response.

Without streaming, a 500-word response might take 10+ seconds to appear. With streaming, the first words appear within 1-2 seconds.

```typescript
// Server-Sent Events (SSE) pattern for streaming
// Node.js / Express
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

app.post('/api/chat/stream', async (req, res) => {
  const { message } = req.body;

  // Set SSE headers
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
        // SSE format: "data: {json}\n\n"
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

The Vercel AI SDK provides a unified interface for streaming AI responses across multiple providers, with built-in React hooks.

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
    system: 'You are a helpful assistant.',
    messages,
    maxTokens: 2048,
  });

  return result.toDataStreamResponse();
}
```

```typescript
// hooks/useChat.ts — or use the built-in hook from 'ai/react'
import { useChat } from 'ai/react';

export function ChatInterface() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } = useChat({
    api: '/api/chat',
    onFinish: (message) => {
      console.log('Final message:', message);
    },
    onError: (err) => {
      console.error('Chat error:', err);
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
        {isLoading && <div className="loading">Thinking...</div>}
        {error && <div className="error">Error: {error.message}</div>}
      </div>

      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Send a message..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>Send</button>
      </form>
    </div>
  );
}
```

### Vercel AI SDK with Tools

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
        description: 'Search the product catalog',
        parameters: z.object({
          query: z.string().describe('Search terms'),
          category: z.enum(['electronics', 'clothing', 'books']).optional(),
        }),
        execute: async ({ query, category }) => {
          return await productService.search(query, category);
        },
      }),
      getWeather: tool({
        description: 'Get current weather for a city',
        parameters: z.object({
          city: z.string().describe('City name'),
        }),
        execute: async ({ city }) => {
          return await weatherApi.current(city);
        },
      }),
    },
    maxSteps: 5,  // SDK handles the agent loop automatically
  });

  return result.toDataStreamResponse();
}
```

### Human-in-the-Loop

For high-stakes decisions (deleting data, sending emails, making purchases), interrupt the agent loop and require explicit human approval before proceeding.

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
          // Pause and wait for human approval
          const actionId = crypto.randomUUID();
          const action: PendingAction = {
            id: actionId,
            toolName: block.name,
            toolInput: block.input as Record<string, unknown>,
            description: `Execute ${block.name} with: ${JSON.stringify(block.input)}`,
            requestedAt: new Date(),
          };

          this.pendingActions.set(actionId, action);
          onPendingAction(action);

          // Wait for approval (poll or use a channel)
          const approved = await this.waitForApproval(actionId, 60_000);

          if (!approved) {
            toolResults.push({
              type: 'tool_result',
              tool_use_id: block.id,
              content: JSON.stringify({ error: 'Action was rejected by user' }),
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

    throw new Error('Max iterations reached');
  }

  async approve(actionId: string): Promise<void> {
    this.pendingActions.delete(actionId);
    // Signal waiting promise
  }

  async reject(actionId: string): Promise<void> {
    this.pendingActions.delete(actionId);
    // Signal waiting promise
  }

  private async waitForApproval(actionId: string, timeoutMs: number): Promise<boolean> {
    // In practice: use a database or Redis to store pending actions
    // and have the UI poll or use WebSockets
    return new Promise((resolve) => {
      const timeout = setTimeout(() => resolve(false), timeoutMs);
      // Listen for approval/rejection events...
    });
  }
}
```

### Error Handling and Retry

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
            // Don't retry client errors (except rate limits)
            if (err.status === 400 || err.status === 401 || err.status === 404) {
              throw new AbortError(err.message);
            }
            // Retry rate limits and server errors
            if (err.status === 429 || err.status >= 500) {
              throw err; // pRetry will retry this
            }
          }
          throw new AbortError((err as Error).message); // unknown error, don't retry
        }
      },
      {
        retries: 3,
        factor: 2,
        minTimeout: 1000,
        maxTimeout: 30000,
        onFailedAttempt: (error) => {
          console.warn(`AI API attempt ${error.attemptNumber} failed: ${error.message}`);
        },
      }
    );
  }
}
```

### Caching Responses

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

  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) return cached;

  // Call API
  const response = await anthropic.messages.create(params);
  const text = (response.content[0] as Anthropic.TextBlock).text;

  // Cache the result
  await redis.setex(cacheKey, ttlSeconds, text);
  return text;
}

// Use for: static content generation, FAQ answering, classification
// Don't use for: personalized content, real-time data, dynamic queries
```

## Hands-On Examples

### Chat with Context Window Management

```typescript
// Manage conversation history to stay within context limits
const MAX_HISTORY_TOKENS = 50_000;

function trimHistory(
  history: Anthropic.MessageParam[],
  maxTokens: number
): Anthropic.MessageParam[] {
  // Always keep system prompt and recent messages
  // Trim from the middle of history
  let estimatedTokens = history.reduce(
    (sum, msg) =>
      sum + (typeof msg.content === 'string' ? msg.content.length : 1000) / 4,
    0
  );

  let trimmed = [...history];
  while (estimatedTokens > maxTokens && trimmed.length > 2) {
    // Remove the second message (keep first user message for context)
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
      system: 'You are a helpful assistant.',
      messages: trimmedHistory,
    });

    const text = (response.content[0] as Anthropic.TextBlock).text;
    this.history.push({ role: 'assistant', content: text });
    return text;
  }
}
```

### Structured Generation with Fallback

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
      system: `Respond with valid JSON only. No explanation, no markdown, no code fences.`,
      messages: [{
        role: 'user',
        content: `${prompt}\n\nReturn JSON matching this schema:\n${schemaDescription}`,
      }],
    });

    try {
      const raw = (response.content[0] as Anthropic.TextBlock).text.trim();
      const parsed = JSON.parse(raw);
      return schema.parse(parsed);
    } catch (err) {
      if (attempt === 2) {
        if (fallback !== undefined) return fallback;
        throw new Error(`Failed to parse structured output after 3 attempts: ${err}`);
      }
      // Retry: the next attempt includes the error
    }
  }

  throw new Error('Unreachable');
}

// Usage
const ProductSchema = z.object({
  name: z.string(),
  price: z.number().positive(),
  category: z.enum(['electronics', 'clothing', 'books']),
  tags: z.array(z.string()),
});

const product = await generateStructured(
  'Generate a product listing for a wireless keyboard',
  ProductSchema
);
```

### Background Batch Processing

```typescript
import PQueue from 'p-queue';

const queue = new PQueue({
  concurrency: 5,      // max 5 concurrent API calls
  intervalCap: 10,     // max 10 calls per interval
  interval: 1000,      // per second (respects rate limits)
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

// Process 10,000 items with rate limit compliance
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

## Common Patterns & Best Practices

### Abort on Client Disconnect

```typescript
// Next.js App Router: built-in abort signal
export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-6'),
    messages,
    abortSignal: req.signal,  // auto-aborts when client disconnects
  });

  return result.toDataStreamResponse();
}

// Manual abort with Express
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

### Observability

```typescript
// Wrap all AI calls with logging and metrics
async function trackedCompletion(
  params: Anthropic.MessageCreateParamsNonStreaming,
  context: { userId?: string; feature: string }
): Promise<string> {
  const start = Date.now();
  const requestId = crypto.randomUUID();

  logger.info('AI request', {
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

    logger.info('AI response', {
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
    logger.error('AI request failed', { requestId, error: (err as Error).message });
    metrics.increment('ai.request.error', { feature: context.feature });
    throw err;
  }
}
```

### Feature Flag for Model Selection

```typescript
import { flags } from '../feature-flags';

async function getModel(userId: string): Promise<string> {
  // A/B test: 10% of users get the new model
  const useNewModel = await flags.evaluate('new-ai-model', userId);
  return useNewModel ? 'claude-opus-4-6' : 'claude-sonnet-4-6';
}

// Or model routing by task complexity
function selectModel(tokenCount: number, taskComplexity: 'simple' | 'complex'): string {
  if (taskComplexity === 'simple' || tokenCount < 1000) {
    return 'claude-haiku-4-5';  // 3x cheaper for simple tasks
  }
  return 'claude-sonnet-4-6';
}
```

## Anti-Patterns to Avoid

**Blocking the request thread during streaming**
```typescript
// Bad: collects entire stream before responding
const text = await stream.finalText();
res.json({ text });

// Good: stream directly to client
for await (const chunk of stream) { res.write(chunk); }
```

**Not handling rate limits**
```typescript
// Bad: fails on 429
const response = await anthropic.messages.create(params);

// Good: retry with backoff
const response = await pRetry(() => anthropic.messages.create(params), { retries: 3 });
```

**Storing raw conversation history without limits**
Without trimming, conversation history grows indefinitely and eventually exceeds the context window.

**Ignoring the `stop_reason`**
```typescript
// Bad: assumes response is complete
const text = response.content[0].text;

// Good: check if truncated
if (response.stop_reason === 'max_tokens') {
  logger.warn('Response was truncated — increase max_tokens or reduce input');
}
```

**No timeout on AI calls**
AI API calls can take 30-60 seconds for large outputs. Without a timeout, requests hang indefinitely.

```typescript
const timeoutSignal = AbortSignal.timeout(60_000);  // 60 second timeout
const response = await anthropic.messages.create(params, { signal: timeoutSignal });
```

## Debugging & Troubleshooting

### Debugging Streaming Issues

```typescript
// Log every event in the stream for debugging
const stream = anthropic.messages.stream({ ... });

for await (const event of stream) {
  console.log('STREAM EVENT:', event.type, JSON.stringify(event).slice(0, 100));
}
```

### Inspecting Usage and Cost

```typescript
const response = await anthropic.messages.create(params);
console.log('Usage:', response.usage);
// { input_tokens: 450, output_tokens: 230 }
// Cost estimate (Sonnet): 450/1M * $3 + 230/1M * $15 = $0.00480
```

### Vercel AI SDK Debug Mode

```typescript
const result = streamText({
  model: anthropic('claude-sonnet-4-6'),
  messages,
  onChunk: ({ chunk }) => {
    console.log('CHUNK:', chunk);
  },
  onFinish: ({ text, usage, finishReason }) => {
    console.log('DONE:', { textLength: text.length, usage, finishReason });
  },
});
```

## Real-World Scenarios

### Scenario 1: Next.js AI Chat App

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

  // Rate limit: 20 requests per minute per user
  const { success } = await ratelimit.limit(user.id);
  if (!success) return new Response('Too Many Requests', { status: 429 });

  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-6'),
    system: `You are a helpful assistant for ${user.name}. Today is ${new Date().toDateString()}.`,
    messages,
    maxTokens: 2048,
    abortSignal: req.signal,
    tools: {
      searchDocs: tool({
        description: 'Search the documentation',
        parameters: z.object({ query: z.string() }),
        execute: async ({ query }) => searchDocs(query, user.orgId),
      }),
    },
    onFinish: async ({ usage }) => {
      // Track usage in database
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

### Scenario 2: Async Document Processing

```typescript
// Process documents in background with job queue (BullMQ)
import { Queue, Worker } from 'bullmq';

const summaryQueue = new Queue('document-summary', { connection: redis });

// Enqueue
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
      model: 'claude-haiku-4-5',  // cost-efficient for batch
      max_tokens: 500,
      temperature: 0,
      system: 'Summarize the document in 3-5 bullet points. Be concise.',
      messages: [{ role: 'user', content: doc.rows[0].content }],
    });

    const text = (summary.content[0] as Anthropic.TextBlock).text;
    await db.query(
      'UPDATE documents SET summary = $1, summarized_at = NOW() WHERE id = $2',
      [text, docId]
    );
  },
  { connection: redis, concurrency: 3 }  // 3 concurrent workers
);
```

## Further Reading

- [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- [Anthropic Streaming Guide](https://docs.anthropic.com/en/api/streaming)
- [Building AI Apps with Next.js — Vercel](https://vercel.com/blog/ai-sdk-3-generative-ui)
- [BullMQ for Job Queues](https://docs.bullmq.io/) — async AI processing
- [Upstash Rate Limiting](https://upstash.com/docs/oss/sdks/ts/ratelimit) — per-user rate limits

## Summary

Production AI integration requires more than calling the API:

**Streaming** — always stream for user-facing features; use SSE or the Vercel AI SDK's `toDataStreamResponse()`. First token appears in 1-2 seconds vs 10+ seconds for non-streaming.

**Vercel AI SDK** — `streamText`, `generateText`, `streamObject` with built-in React hooks (`useChat`, `useCompletion`). Handles SSE plumbing automatically.

**Error handling** — retry on 429 (rate limit) and 5xx (server errors) with exponential backoff. Use `AbortError` to stop retrying on client errors.

**Human-in-the-loop** — pause the agent loop before destructive actions; store the pending action, notify the user, wait for approval/rejection.

**Observability** — log every request with: requestId, model, feature, userId, latency, input/output tokens. Use this data to optimize costs and debug quality issues.

**Caching** — cache deterministic completions (temperature=0, static prompts) in Redis. Don't cache personalized or real-time responses.

**Rate limiting** — implement per-user rate limits before the AI call to prevent abuse and control costs.

**Abort signals** — always pass `req.signal` so the AI call is cancelled when the client disconnects, saving money on abandoned requests.
