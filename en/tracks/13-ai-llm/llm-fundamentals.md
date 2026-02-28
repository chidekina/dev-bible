# LLM Fundamentals

## Overview

Large Language Models (LLMs) are neural networks trained on massive text corpora to predict the next token in a sequence. This simple training objective produces models capable of following complex instructions, reasoning through problems, generating code, and more. But understanding the mechanisms — how tokens work, what the context window limits, what temperature controls, why hallucinations happen — is essential for building reliable applications on top of them.

This chapter covers the foundational concepts every developer needs before integrating LLM APIs into production systems.

## Prerequisites

- Basic Python or TypeScript
- Familiarity with REST APIs
- Conceptual understanding of machine learning (not required, but helpful)

## Core Concepts

### Tokens

LLMs don't read words — they read **tokens**. A token is a chunk of text, roughly 3-4 characters on average for English. Tokenization is the first step in processing any text.

Rules of thumb:
- 1 token ≈ 4 characters of English text
- 1 token ≈ 0.75 words
- 100 tokens ≈ 75 words
- 1 page of text ≈ 700-800 tokens
- Code is more token-dense than prose

```typescript
// Counting tokens before sending to API
// Use the tiktoken library (OpenAI's tokenizer) or @anthropic-ai/tokenizer

import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

// Claude: use countTokens API
async function countTokens(text: string): Promise<number> {
  const response = await client.beta.messages.countTokens({
    model: 'claude-opus-4-6',
    messages: [{ role: 'user', content: text }],
  });
  return response.input_tokens;
}

// OpenAI: use tiktoken
import { encoding_for_model } from 'tiktoken';

function countTokensOpenAI(text: string, model = 'gpt-4o'): number {
  const enc = encoding_for_model(model as any);
  const tokens = enc.encode(text);
  enc.free();
  return tokens.length;
}
```

Different models use different tokenizers (BPE, SentencePiece, etc.). Token counts for the same text will differ across models — always use the model-specific tokenizer.

Why tokens matter:
- **Pricing** — APIs charge per token (input and output separately)
- **Context window** — total tokens in context is limited
- **Latency** — more tokens = more time to generate
- **Attention** — the model attends to all tokens; very long contexts may dilute attention to specific parts

### Context Window

The **context window** is the total number of tokens a model can process in one request — the sum of input (system + user messages) and output tokens.

| Model | Context Window |
|-------|---------------|
| GPT-4o | 128K tokens |
| Claude Opus 4.6 | 200K tokens |
| Claude Sonnet 4.6 | 200K tokens |
| Gemini 1.5 Pro | 1M tokens |
| GPT-4o mini | 128K tokens |

When your input approaches the context limit:
1. The API returns an error (context length exceeded)
2. The model may truncate earlier parts of the conversation
3. Quality degrades near the limit — the model "loses" information from the middle of very long contexts

Context window strategy:
```typescript
const MAX_CONTEXT_TOKENS = 180_000;  // leave buffer below model limit
const MAX_OUTPUT_TOKENS = 8_000;
const MAX_INPUT_TOKENS = MAX_CONTEXT_TOKENS - MAX_OUTPUT_TOKENS;

async function safeChat(messages: Message[]): Promise<string> {
  // Estimate token count
  const tokenCount = await countTokens(JSON.stringify(messages));

  if (tokenCount > MAX_INPUT_TOKENS) {
    // Truncate oldest messages (keep system message + recent context)
    const truncated = truncateMessages(messages, MAX_INPUT_TOKENS);
    return chat(truncated);
  }

  return chat(messages);
}
```

### Message Roles

LLM APIs use a structured conversation format with three roles:

**System** — Instructions, persona, and constraints set by the developer. Processed first; higher "authority" than user turns. Not all models treat system prompts the same way.

**User** — The human's message. Input to the model.

**Assistant** — The model's previous responses. Including these in the conversation enables multi-turn dialogue.

```typescript
const messages = [
  {
    role: 'system',
    content: `You are a senior TypeScript engineer.
Your responses must:
- Use TypeScript with strict mode
- Include error handling with Result types
- Be concise and direct
- Not include disclaimers or caveats`,
  },
  {
    role: 'user',
    content: 'Write a function that fetches a user by ID from the database.',
  },
  // If continuing a conversation, append previous assistant response:
  // { role: 'assistant', content: '...' },
  // { role: 'user', content: 'Make it support pagination' },
];
```

### Temperature and Sampling Parameters

**Temperature** controls randomness in token selection. The model computes a probability distribution over all possible next tokens; temperature scales that distribution.

| Temperature | Behavior | Use Case |
|-------------|----------|----------|
| 0 | Deterministic (always picks highest probability token) | Code generation, data extraction, classification |
| 0.1-0.3 | Very focused | Structured output, precise answers |
| 0.5-0.7 | Balanced | General Q&A, analysis, summarization |
| 0.8-1.0 | Creative, varied | Brainstorming, creative writing, diverse options |
| >1.0 | Very random, often incoherent | Rarely useful |

**top_p** (nucleus sampling) — only sample from tokens whose cumulative probability exceeds top_p. Alternative to temperature; many use temperature alone.

**top_k** — only consider the top K most likely tokens at each step. Less commonly exposed.

**max_tokens** — maximum number of output tokens. The model stops when it finishes OR hits this limit. A truncated response is often worse than no response — set this high enough.

```typescript
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  temperature: 0,        // deterministic for code extraction
  messages: [
    { role: 'user', content: 'Extract all TODO comments from this code as JSON' },
  ],
});
```

### Why Hallucinations Happen

LLMs hallucinate — they generate plausible-sounding but factually incorrect information — because:

1. **Training objective is next-token prediction**, not truth verification. A confident-sounding wrong answer and a confident-sounding right answer look the same to the loss function.

2. **No access to external knowledge** (unless given tools). The model's "knowledge" is frozen at training time and encoded in weights, not retrieved from authoritative sources.

3. **Attention limitation** — in very long contexts, the model's attention to specific facts can be diluted. The further a fact is from the current generation position, the more likely it may be misremembered.

4. **Instruction following vs truthfulness tension** — if you ask the model to produce an answer and it doesn't know the answer, it often tries to satisfy the instruction rather than say "I don't know."

Mitigation strategies:
- **RAG** — retrieve relevant facts from authoritative sources and include them in the prompt
- **Grounding** — include source documents and ask the model to cite them
- **Low temperature** — reduces creativity but also reduces hallucinations
- **Explicit "I don't know" permission** — "If you don't know the answer, say you don't know. Don't guess."
- **Verification steps** — have the model check its own work, or use a second LLM call to verify
- **Structured output** — constrained output formats reduce hallucination surface area

### Prompt Caching

Modern LLMs support **prompt caching** — the KV cache for a long prefix is saved server-side, so repeat requests with the same prefix are cheaper and faster.

```typescript
// Anthropic: mark long static prefixes for caching
const response = await anthropic.messages.create({
  model: 'claude-opus-4-6',
  max_tokens: 1024,
  system: [
    {
      type: 'text',
      text: '...your entire 10,000 token system prompt...',
      cache_control: { type: 'ephemeral' },  // cache this prefix
    },
  ],
  messages: [{ role: 'user', content: userMessage }],
});

// Check cache usage in the response
console.log(response.usage);
// { input_tokens: 1000, cache_creation_input_tokens: 10000, cache_read_input_tokens: 0 }
// On subsequent calls:
// { input_tokens: 1000, cache_creation_input_tokens: 0, cache_read_input_tokens: 10000 }
// Cache reads are ~90% cheaper than full input tokens
```

## Hands-On Examples

### Basic API Call with TypeScript

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,  // never hardcode
});

async function chat(userMessage: string): Promise<string> {
  const message = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: userMessage }],
  });

  // Extract text from the response
  const content = message.content[0];
  if (content.type !== 'text') {
    throw new Error(`Unexpected response type: ${content.type}`);
  }
  return content.text;
}

// Usage
const reply = await chat('Explain what a context window is in 2 sentences.');
console.log(reply);
```

### Multi-Turn Conversation

```typescript
interface Message {
  role: 'user' | 'assistant';
  content: string;
}

class Conversation {
  private history: Message[] = [];

  constructor(
    private systemPrompt: string,
    private model = 'claude-sonnet-4-6'
  ) {}

  async send(userMessage: string): Promise<string> {
    this.history.push({ role: 'user', content: userMessage });

    const response = await anthropic.messages.create({
      model: this.model,
      max_tokens: 2048,
      system: this.systemPrompt,
      messages: this.history,
    });

    const text = (response.content[0] as Anthropic.TextBlock).text;
    this.history.push({ role: 'assistant', content: text });
    return text;
  }

  reset() {
    this.history = [];
  }
}

// Usage
const conv = new Conversation('You are a helpful coding assistant.');
console.log(await conv.send('How do I reverse a string in TypeScript?'));
console.log(await conv.send('Can you show me the same thing in Python?'));
```

### Temperature Comparison

```typescript
async function compareTemperatures(prompt: string) {
  const temperatures = [0, 0.3, 0.7, 1.0];
  for (const temp of temperatures) {
    const response = await anthropic.messages.create({
      model: 'claude-haiku-4-5',
      max_tokens: 100,
      temperature: temp,
      messages: [{ role: 'user', content: prompt }],
    });
    console.log(`\nTemperature ${temp}:`);
    console.log((response.content[0] as Anthropic.TextBlock).text);
  }
}

await compareTemperatures('Describe a color in one sentence.');
// Notice: temp=0 gives the same response every time
// temp=1.0 gives varied, sometimes creative responses
```

### Calculating Cost Before a Request

```typescript
interface CostEstimate {
  inputTokens: number;
  maxOutputTokens: number;
  estimatedCostUSD: number;
}

// Claude Sonnet 4.6 pricing (as of 2025)
const PRICING = {
  'claude-sonnet-4-6': {
    inputPer1M: 3.00,    // USD per 1M input tokens
    outputPer1M: 15.00,  // USD per 1M output tokens
  },
  'claude-haiku-4-5': {
    inputPer1M: 0.80,
    outputPer1M: 4.00,
  },
};

async function estimateCost(
  systemPrompt: string,
  userMessage: string,
  maxOutputTokens: number,
  model: keyof typeof PRICING
): Promise<CostEstimate> {
  const inputTokens = await countTokens(systemPrompt + userMessage);
  const pricing = PRICING[model];
  const inputCost = (inputTokens / 1_000_000) * pricing.inputPer1M;
  const outputCost = (maxOutputTokens / 1_000_000) * pricing.outputPer1M;
  return {
    inputTokens,
    maxOutputTokens,
    estimatedCostUSD: inputCost + outputCost,
  };
}
```

## Common Patterns & Best Practices

### Choosing the Right Model

| Use Case | Recommended Model | Reason |
|----------|------------------|--------|
| Simple classification, extraction | Haiku / GPT-4o mini | Fast, cheap, sufficient for structured tasks |
| Code generation, complex reasoning | Sonnet / GPT-4o | Balance of quality and cost |
| Multi-step reasoning, research | Opus / o1 | Maximum capability |
| Real-time chat (streaming) | Sonnet | Speed + quality |
| Batch processing large volumes | Haiku | Cost efficiency |

### Error Handling

```typescript
import Anthropic from '@anthropic-ai/sdk';

async function robustChat(message: string): Promise<string> {
  try {
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 1024,
      messages: [{ role: 'user', content: message }],
    });
    return (response.content[0] as Anthropic.TextBlock).text;
  } catch (err) {
    if (err instanceof Anthropic.APIError) {
      switch (err.status) {
        case 400:
          throw new Error(`Invalid request: ${err.message}`);
        case 401:
          throw new Error('Invalid API key');
        case 429:
          // Rate limit — implement exponential backoff
          await new Promise((r) => setTimeout(r, 5000));
          return robustChat(message); // retry once
        case 500:
        case 529:
          // Server overloaded — retry with backoff
          await new Promise((r) => setTimeout(r, 10000));
          return robustChat(message);
        default:
          throw err;
      }
    }
    throw err;
  }
}
```

### Rate Limiting and Retries

```typescript
import pRetry from 'p-retry';

async function chatWithRetry(message: string): Promise<string> {
  return pRetry(
    async () => {
      const response = await anthropic.messages.create({
        model: 'claude-sonnet-4-6',
        max_tokens: 1024,
        messages: [{ role: 'user', content: message }],
      });
      return (response.content[0] as Anthropic.TextBlock).text;
    },
    {
      retries: 3,
      factor: 2,
      minTimeout: 1000,
      maxTimeout: 10000,
      onFailedAttempt: (error) => {
        console.warn(`Attempt ${error.attemptNumber} failed: ${error.message}`);
      },
    }
  );
}
```

## Anti-Patterns to Avoid

**Hardcoding API keys**
```typescript
// NEVER
const client = new Anthropic({ apiKey: 'sk-ant-...' });

// Always
const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
```

**Ignoring max_tokens**
```typescript
// Bad: if the model stops mid-response, you get truncated output silently
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 100,  // too low for complex tasks
  messages: [...],
});
// Check: response.stop_reason === 'max_tokens' means truncated
if (response.stop_reason === 'max_tokens') {
  console.warn('Response was truncated');
}
```

**Not accounting for output tokens in cost estimates**
Output tokens cost 3-15x more than input tokens per 1M. Always include max_tokens in cost estimates.

**Treating LLM output as trusted input**
```typescript
// Bad: executing LLM-generated code or SQL directly
const code = await chat('Write SQL to get all users');
await db.raw(code);  // prompt injection → SQL injection

// Good: validate and sanitize all LLM output
// Use structured output + schema validation instead of raw strings
```

**Setting temperature = 1 for production**
High temperature is fun for demos but unreliable in production. Use ≤ 0.3 for anything that needs consistency.

## Debugging & Troubleshooting

### Debugging Prompt Issues

```typescript
// Log full request and response for debugging
async function debugChat(message: string) {
  console.log('REQUEST:', JSON.stringify({ message }, null, 2));

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: message }],
  });

  console.log('RESPONSE:', JSON.stringify(response, null, 2));
  console.log('USAGE:', response.usage);
  console.log('STOP REASON:', response.stop_reason);
  // stop_reason: 'end_turn' = normal, 'max_tokens' = truncated, 'stop_sequence' = hit stop seq
}
```

### Context Window Debugging

```bash
# Error: "prompt is too long"
# Solution: count tokens before sending
const tokens = await countTokens(systemPrompt + userMessage);
console.log(`Total input tokens: ${tokens}`);
# Typical max for Claude: 200K tokens
# Check model limits in the API docs
```

### Response Quality Issues

If responses are poor:
1. **Too verbose**: add "Be concise" or "Respond in 2 sentences"
2. **Wrong format**: use structured output (see `prompt-engineering.md`)
3. **Hallucinating**: add "Only state facts you are certain of" or use RAG
4. **Off-topic**: tighten system prompt, add negative examples
5. **Inconsistent**: lower temperature

## Real-World Scenarios

### Scenario 1: Text Classification Pipeline

```typescript
type Sentiment = 'positive' | 'negative' | 'neutral';

async function classifySentiment(text: string): Promise<Sentiment> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5',    // fast + cheap for classification
    max_tokens: 10,               // just one word needed
    temperature: 0,               // deterministic
    system: 'Classify the sentiment of the given text. Respond with exactly one word: positive, negative, or neutral.',
    messages: [{ role: 'user', content: text }],
  });

  const result = (response.content[0] as Anthropic.TextBlock).text
    .trim()
    .toLowerCase() as Sentiment;

  if (!['positive', 'negative', 'neutral'].includes(result)) {
    throw new Error(`Unexpected classification: ${result}`);
  }

  return result;
}

// Batch classification
async function classifyBatch(texts: string[]): Promise<Sentiment[]> {
  return Promise.all(texts.map((t) => classifySentiment(t)));
}
```

### Scenario 2: Streaming for Real-Time UX

```typescript
import Anthropic from '@anthropic-ai/sdk';

async function streamChat(message: string, onChunk: (text: string) => void) {
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
      onChunk(event.delta.text);
    }
  }

  const finalMessage = await stream.finalMessage();
  return finalMessage;
}

// Usage in Express
app.post('/chat', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  await streamChat(req.body.message, (chunk) => {
    res.write(`data: ${JSON.stringify({ text: chunk })}\n\n`);
  });

  res.write('data: [DONE]\n\n');
  res.end();
});
```

## Further Reading

- [Anthropic API Reference](https://docs.anthropic.com/en/api/getting-started)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Lilian Weng: Large Language Model](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family-v2/) — deep technical background
- [Tokenizer Playground — Anthropic](https://console.anthropic.com/tokenizer)

## Summary

LLMs are next-token predictors trained on text. For developers, the critical concepts are:

- **Tokens** — the unit of pricing, latency, and context limits (~4 chars per token, ~750 tokens per page)
- **Context window** — total input + output token budget; quality degrades near limits
- **Roles** — system (instructions), user (human input), assistant (model's prior turns)
- **Temperature** — 0 for deterministic tasks (code, extraction); 0.5-0.7 for conversational; ≥ 0.8 for creative
- **max_tokens** — always set explicitly; check `stop_reason === 'max_tokens'` for truncation
- **Hallucinations** — inherent to next-token prediction; mitigate with RAG, grounding, low temperature, and explicit "say I don't know" instructions
- **Prompt caching** — cache long static prefixes for 90% cost reduction on repeat calls

Always validate and sanitize LLM output. Never treat it as trusted input for downstream systems.
