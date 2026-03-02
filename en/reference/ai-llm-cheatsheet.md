# AI / LLM Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Model Comparison

| Model | Context Window | Input Cost | Output Cost | Strengths |
|-------|---------------|-----------|------------|-----------|
| **Claude Opus 4.6** | 200K tokens | $15 / 1M | $75 / 1M | Reasoning, coding, long docs |
| **Claude Sonnet 4.6** | 200K tokens | $3 / 1M | $15 / 1M | Balanced speed + quality |
| **Claude Haiku 4.5** | 200K tokens | $0.80 / 1M | $4 / 1M | Fast, cheap, simple tasks |
| **GPT-4o** | 128K tokens | $2.50 / 1M | $10 / 1M | Multimodal, broad ecosystem |
| **GPT-4o mini** | 128K tokens | $0.15 / 1M | $0.60 / 1M | Cost-efficient GPT tasks |
| **o3** | 200K tokens | $10 / 1M | $40 / 1M | Deep reasoning, math, code |
| **o4-mini** | 200K tokens | $1.10 / 1M | $4.40 / 1M | Fast reasoning, budget-friendly |
| **Gemini 2.0 Flash** | 1M tokens | $0.10 / 1M | $0.40 / 1M | Massive context, speed |
| **Gemini 2.5 Pro** | 2M tokens | $1.25 / 1M | $10 / 1M | Largest context, complex tasks |
| **Llama 3.3 70B** | 128K tokens | varies | varies | Open-source, self-hostable |

> Prices approximate as of mid-2025. Verify at provider pricing pages.

---

## Message Roles

```json
// System — sets behavior, persona, constraints (not supported by all models)
{ "role": "system", "content": "You are a helpful assistant. Answer concisely." }

// User — human input
{ "role": "user", "content": "What is the capital of France?" }

// Assistant — model response (used for few-shot examples or conversation history)
{ "role": "assistant", "content": "The capital of France is Paris." }
```

```typescript
// Multi-turn conversation
const messages = [
  { role: "user",      content: "My name is Alice." },
  { role: "assistant", content: "Hello Alice! How can I help?" },
  { role: "user",      content: "What's my name?" },
];
```

---

## Key Parameters

| Parameter | Type | Range | Default | Effect |
|-----------|------|-------|---------|--------|
| `temperature` | float | 0.0–2.0 | 1.0 | Randomness. 0 = deterministic, 1+ = creative |
| `top_p` | float | 0.0–1.0 | 1.0 | Nucleus sampling. 0.9 = top 90% prob mass |
| `max_tokens` | int | 1–model max | varies | Max output tokens |
| `stop` | string[] | — | [] | Sequences that halt generation |
| `presence_penalty` | float | -2.0–2.0 | 0 | Penalize tokens already in output |
| `frequency_penalty` | float | -2.0–2.0 | 0 | Penalize frequent tokens |
| `seed` | int | any | — | Reproducible output (best-effort) |
| `stream` | bool | — | false | Stream tokens as generated |

```
Temperature guide:
  0.0       → Deterministic, factual Q&A, extraction
  0.3–0.5   → Coding, analysis, summarization
  0.7–1.0   → General conversation, writing
  1.2–1.5   → Creative writing, brainstorming
  > 1.5     → Very random, usually not useful
```

---

## Anthropic SDK

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Basic completion
const msg = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Explain recursion in one sentence." }],
});
console.log(msg.content[0].text);
console.log(msg.usage); // { input_tokens, output_tokens }

// System prompt
const msg = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  system: "You are a senior TypeScript developer. Be concise.",
  messages: [{ role: "user", content: "How do I type a generic async function?" }],
});

// Streaming
const stream = await client.messages.stream({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Write a haiku." }],
});

for await (const chunk of stream) {
  if (chunk.type === "content_block_delta") {
    process.stdout.write(chunk.delta.text);
  }
}
const finalMsg = await stream.finalMessage();

// Prompt caching (cost optimization for repeated system prompts)
const msg = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: "You are an expert. Here is a 10k-word knowledge base...",
      cache_control: { type: "ephemeral" }, // cache for 5 min
    },
  ],
  messages: [{ role: "user", content: "Summarize." }],
});
```

---

## Anthropic Tool Use

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: "get_weather",
    description: "Get current weather for a location.",
    input_schema: {
      type: "object",
      properties: {
        location: { type: "string", description: "City name" },
        unit: { type: "string", enum: ["celsius", "fahrenheit"] },
      },
      required: ["location"],
    },
  },
];

const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  tools,
  messages: [{ role: "user", content: "What's the weather in Paris?" }],
});

if (response.stop_reason === "tool_use") {
  const toolUse = response.content.find((b) => b.type === "tool_use");
  const result = await callMyTool(toolUse.name, toolUse.input);

  // Continue conversation with tool result
  const finalResponse = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    tools,
    messages: [
      { role: "user",      content: "What's the weather in Paris?" },
      { role: "assistant", content: response.content },
      { role: "user",      content: [{ type: "tool_result", tool_use_id: toolUse.id, content: result }] },
    ],
  });
}
```

---

## OpenAI SDK

```typescript
import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// Basic completion
const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system",    content: "You are helpful." },
    { role: "user",      content: "Say hello." },
  ],
});
console.log(completion.choices[0].message.content);
console.log(completion.usage); // prompt_tokens, completion_tokens, total_tokens

// Streaming
const stream = await openai.chat.completions.create({
  model: "gpt-4o",
  stream: true,
  messages: [{ role: "user", content: "Count to 5." }],
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

### Function Calling (OpenAI)

```typescript
const tools: OpenAI.ChatCompletionTool[] = [
  {
    type: "function",
    function: {
      name: "get_weather",
      description: "Get current weather for a location.",
      parameters: {
        type: "object",
        properties: {
          location: { type: "string" },
          unit: { type: "string", enum: ["celsius", "fahrenheit"] },
        },
        required: ["location"],
      },
    },
  },
];

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  tools,
  tool_choice: "auto",  // auto | none | required | { type: "function", function: { name } }
  messages: [{ role: "user", content: "Weather in London?" }],
});

const toolCall = response.choices[0].message.tool_calls?.[0];
if (toolCall) {
  const args = JSON.parse(toolCall.function.arguments);
  const result = await callMyTool(toolCall.function.name, args);

  await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
      { role: "user",      content: "Weather in London?" },
      response.choices[0].message,
      { role: "tool", tool_call_id: toolCall.id, content: JSON.stringify(result) },
    ],
  });
}
```

---

## Vercel AI SDK

```typescript
import { streamText, generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { openai } from "@ai-sdk/openai";

// Server-side streaming (Next.js App Router)
export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic("claude-sonnet-4-6"),
    system: "You are helpful.",
    messages,
  });

  return result.toDataStreamResponse();
}

// Client-side hook
import { useChat } from "ai/react";

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: "/api/chat",
  });

  return (
    <form onSubmit={handleSubmit}>
      {messages.map((m) => (
        <p key={m.id}><b>{m.role}:</b> {m.content}</p>
      ))}
      <input value={input} onChange={handleInputChange} />
      <button type="submit">Send</button>
    </form>
  );
}

// Tool use with AI SDK
import { tool } from "ai";
import { z } from "zod";

const result = await generateText({
  model: openai("gpt-4o"),
  tools: {
    getWeather: tool({
      description: "Get weather for a city",
      parameters: z.object({ city: z.string() }),
      execute: async ({ city }) => ({ temperature: 22, condition: "sunny" }),
    }),
  },
  maxSteps: 3,  // allow multiple tool calls
  prompt: "What's the weather in Paris?",
});
```

---

## RAG Pipeline

```
1. INGEST
   Raw docs → chunk (500-1000 tokens, 10-20% overlap) → embed → upsert to vector DB

2. QUERY
   User question → embed → vector similarity search (top-k, k=3–10) → rerank (optional)

3. GENERATE
   Retrieved chunks + user question → build prompt → LLM → response

```

```typescript
// Chunking strategies
// Fixed size:   simple, works for most cases
// Sentence:     better for prose, nltk/spacy
// Recursive:    splits on \n\n → \n → " " → "" (LangChain default)
// Semantic:     embed + split on similarity drops

// Popular vector DBs
// pgvector    — PostgreSQL extension (self-hosted, SQL interface)
// Pinecone    — managed, fast, expensive
// Weaviate    — open-source, hybrid search
// Qdrant      — open-source, Rust, high performance
// Chroma      — open-source, great for dev/prototyping

// Embedding models
// text-embedding-3-small  (OpenAI) — 1536 dims, cheap
// text-embedding-3-large  (OpenAI) — 3072 dims, better quality
// voyage-3-large          (Voyage) — best for retrieval
// nomic-embed-text        (free/local)

// Prompt template for RAG
const prompt = `
Answer the question using ONLY the context below.
If the answer is not in the context, say "I don't know."

<context>
${retrievedChunks.join("\n\n---\n\n")}
</context>

<question>
${userQuestion}
</question>
`;
```

---

## Prompt Patterns

```
Zero-shot:
  "Classify the sentiment of this review: <review>"

Few-shot:
  "Classify sentiment. Examples:
   'Great product!' → positive
   'Terrible experience.' → negative
   Now classify: '<review>'"

Chain of Thought (CoT):
  "Think step by step before answering."
  "Let's work through this carefully:"

ReAct (Reason + Act):
  "Think: [reasoning]
   Act: [tool call]
   Observe: [tool result]
   Think: [next reasoning]
   Answer: [final answer]"

XML tags (Anthropic recommended):
  "<documents>
     <document index='1'>...</document>
   </documents>
   <instructions>
     Summarize each document.
   </instructions>"

Role prompting:
  "You are a senior security engineer reviewing code for vulnerabilities."

Constraints:
  "Respond in exactly 3 bullet points."
  "Output valid JSON only. No explanation."
  "If you don't know, say 'UNKNOWN'. Never guess."
```

---

## Token Counting

```typescript
// Anthropic — server-side token count
const response = await client.messages.countTokens({
  model: "claude-sonnet-4-6",
  messages: [{ role: "user", content: "Hello, world!" }],
});
console.log(response.input_tokens); // exact count

// OpenAI — tiktoken (client-side)
import { encoding_for_model } from "tiktoken";

const enc = encoding_for_model("gpt-4o");
const tokens = enc.encode("Hello, world!");
console.log(tokens.length);
enc.free(); // important: free WASM memory

// Rule of thumb estimates
// 1 token ≈ 4 chars (English)
// 1 token ≈ 0.75 words
// 1 page ≈ 500 words ≈ 667 tokens
// 100K tokens ≈ 75,000 words ≈ 150-page book
```

---

## Cost Optimization

| Technique | Savings | When to Use |
|-----------|---------|-------------|
| Prompt caching (Anthropic) | 90% on cache hits | Repeated system prompts / docs |
| Batching (Anthropic Batch API) | 50% | Non-real-time workloads |
| Model routing | 5–50× | Route simple tasks to smaller models |
| Shorter system prompts | ~10–30% | Audit bloated instructions |
| Streaming | 0% cost, better UX | Interactive apps |
| max_tokens limit | varies | Prevent runaway outputs |
| Caching responses | 100% | Identical repeated queries |

```typescript
// Model routing example
function routeToModel(task: string): string {
  if (task === "classification" || task === "extraction") return "claude-haiku-4-5";
  if (task === "summarization" || task === "qa") return "claude-sonnet-4-6";
  return "claude-opus-4-6"; // reasoning, complex analysis
}
```

---

## Common Error Codes & Handling

| Code / Type | Cause | Fix |
|-------------|-------|-----|
| `401 invalid_api_key` | Wrong or missing API key | Check env var |
| `403 permission_error` | Key lacks access to model | Check API key permissions |
| `429 rate_limit_error` | Too many requests | Exponential backoff + retry |
| `429 tokens_limit` | Token quota exceeded | Check billing / upgrade plan |
| `500 api_error` | Provider-side error | Retry with backoff |
| `529 overloaded_error` | Provider overloaded | Retry with jitter |
| `context_length_exceeded` | Input too long | Truncate / chunk input |
| `stop_reason: max_tokens` | Output truncated | Increase max_tokens |
| `stop_reason: tool_use` | Model wants tool call | Handle tool use loop |

```typescript
// Retry with exponential backoff
async function withRetry<T>(fn: () => Promise<T>, retries = 3): Promise<T> {
  for (let attempt = 0; attempt < retries; attempt++) {
    try {
      return await fn();
    } catch (err: any) {
      const isRetryable = err.status === 429 || err.status === 529 || err.status >= 500;
      if (!isRetryable || attempt === retries - 1) throw err;
      const delay = Math.min(1000 * 2 ** attempt + Math.random() * 500, 30000);
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw new Error("Unreachable");
}

const response = await withRetry(() =>
  client.messages.create({ model: "claude-sonnet-4-6", max_tokens: 1024, messages })
);
```
