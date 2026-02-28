# AI Cost Optimization

## Overview

LLM API costs can grow faster than any other infrastructure cost. A feature that costs $0.01 per request is fine in development; at 100,000 requests per day, that's $1,000/day or $365,000/year. Understanding where costs come from and how to reduce them is essential for sustainable AI product development.

This chapter covers token counting, prompt caching, model routing, batching, and budget alerting — practical techniques to reduce costs by 50-90% without sacrificing quality.

## Prerequisites

- LLM Fundamentals (tokens, context window, pricing)
- AI Integration Patterns (streaming, caching)
- TypeScript

## Core Concepts

### Where Costs Come From

LLM costs are driven by two variables: **token count** and **model tier**.

```
Cost = (input_tokens × input_price_per_1M) + (output_tokens × output_price_per_1M)
```

| Model | Input ($/1M) | Output ($/1M) | Relative Cost |
|-------|-------------|--------------|---------------|
| GPT-4o | $2.50 | $10.00 | High |
| GPT-4o mini | $0.15 | $0.60 | Low |
| Claude Opus 4.6 | $15.00 | $75.00 | Very High |
| Claude Sonnet 4.6 | $3.00 | $15.00 | Medium |
| Claude Haiku 4.5 | $0.80 | $4.00 | Low |

Key insight: **output tokens cost 3-5x more than input tokens** on most models. Minimize verbose outputs.

### Token Counting

Always count tokens before sending requests — especially for batch jobs, where you can estimate total cost in advance.

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { encoding_for_model, TiktokenModel } from 'tiktoken';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Anthropic: use the API's countTokens endpoint (most accurate)
async function countInputTokens(
  messages: Anthropic.MessageParam[],
  system?: string
): Promise<number> {
  const response = await anthropic.beta.messages.countTokens({
    model: 'claude-sonnet-4-6',
    system,
    messages,
  });
  return response.input_tokens;
}

// OpenAI: use tiktoken (local, no API call needed)
function countTokensOpenAI(text: string, model: TiktokenModel = 'gpt-4o'): number {
  const enc = encoding_for_model(model);
  const count = enc.encode(text).length;
  enc.free();
  return count;
}

// Rough estimation (use when speed matters more than accuracy)
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);  // ≈ 4 chars per token for English
}

// Cost estimation
const PRICING: Record<string, { input: number; output: number }> = {
  'claude-sonnet-4-6': { input: 3.00, output: 15.00 },
  'claude-haiku-4-5': { input: 0.80, output: 4.00 },
  'gpt-4o': { input: 2.50, output: 10.00 },
  'gpt-4o-mini': { input: 0.15, output: 0.60 },
};

function estimateCost(
  model: string,
  inputTokens: number,
  outputTokens: number
): number {
  const pricing = PRICING[model];
  if (!pricing) throw new Error(`Unknown model: ${model}`);
  return (inputTokens / 1_000_000) * pricing.input +
         (outputTokens / 1_000_000) * pricing.output;
}

// Example: 1000 input tokens, 500 output tokens on Sonnet
console.log(estimateCost('claude-sonnet-4-6', 1000, 500).toFixed(6));
// $0.010500 per request
// At 100K requests/day: $1,050/day
```

### Prompt Caching

Prompt caching is the highest-leverage cost optimization for applications with long, repeated system prompts. The provider caches the KV attention states for a prefix; subsequent requests that reuse that prefix pay only a fraction.

**Anthropic prompt caching:**
- Cache reads cost ~10% of normal input tokens (90% discount)
- Cache writes cost 25% more than normal (one-time cost)
- Minimum cacheable prefix: 1,024 tokens
- Cache TTL: 5 minutes (ephemeral) — refreshed on each cache hit

```typescript
// Mark long static prefixes with cache_control
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  system: [
    {
      type: 'text',
      // Large system prompt (2,000+ tokens for meaningful cache savings)
      text: `You are an expert customer support agent with deep knowledge of our product.

PRODUCT DOCUMENTATION:
${largeProductDocs}  // 5,000 tokens of docs

COMPANY POLICIES:
${companyPolicies}   // 2,000 tokens of policies`,
      cache_control: { type: 'ephemeral' },  // cache this prefix
    },
  ],
  messages: [
    { role: 'user', content: userQuestion },  // dynamic, not cached
  ],
});

// First request: pays full price for system prefix + cache write cost
// Subsequent requests within 5 minutes: pays 10% for cached prefix
// console.log(response.usage)
// { input_tokens: 200, cache_creation_input_tokens: 7000, cache_read_input_tokens: 0 }
// Next call:
// { input_tokens: 200, cache_creation_input_tokens: 0, cache_read_input_tokens: 7000 }
```

**Calculating cache savings:**
```typescript
function analyzeCacheUsage(usage: Anthropic.Usage) {
  const fullInputTokens = usage.input_tokens + (usage.cache_read_input_tokens ?? 0);
  const cacheHitRate = usage.cache_read_input_tokens
    ? usage.cache_read_input_tokens / fullInputTokens
    : 0;

  const actualCost =
    (usage.input_tokens / 1_000_000) * 3.00 +
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) * 3.75 +
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) * 0.30;

  const wouldCostWithout = (fullInputTokens / 1_000_000) * 3.00;
  const savings = wouldCostWithout - actualCost;

  console.log(`Cache hit rate: ${(cacheHitRate * 100).toFixed(1)}%`);
  console.log(`Actual cost: $${actualCost.toFixed(6)}`);
  console.log(`Would cost without cache: $${wouldCostWithout.toFixed(6)}`);
  console.log(`Savings: $${savings.toFixed(6)} (${((savings/wouldCostWithout)*100).toFixed(1)}%)`);
}
```

### Model Routing

Not all tasks need the most expensive model. Route different request types to the most cost-effective model for the task.

```typescript
type TaskComplexity = 'simple' | 'moderate' | 'complex';

interface ModelConfig {
  model: string;
  maxInputTokens: number;
  costPer1MInput: number;
}

const MODELS: Record<TaskComplexity, ModelConfig> = {
  simple: {
    model: 'claude-haiku-4-5',
    maxInputTokens: 10_000,
    costPer1MInput: 0.80,
  },
  moderate: {
    model: 'claude-sonnet-4-6',
    maxInputTokens: 100_000,
    costPer1MInput: 3.00,
  },
  complex: {
    model: 'claude-opus-4-6',
    maxInputTokens: 200_000,
    costPer1MInput: 15.00,
  },
};

function selectModel(task: {
  type: 'classification' | 'extraction' | 'generation' | 'reasoning';
  inputTokens: number;
  requiresLatestKnowledge: boolean;
}): string {
  // Simple classification/extraction → Haiku
  if (
    (task.type === 'classification' || task.type === 'extraction') &&
    task.inputTokens < 5_000
  ) {
    return MODELS.simple.model;
  }

  // Complex multi-step reasoning → Opus
  if (task.type === 'reasoning' && task.inputTokens > 50_000) {
    return MODELS.complex.model;
  }

  // Default to Sonnet for balanced cost/quality
  return MODELS.moderate.model;
}

// Route in application code
async function smartComplete(
  task: { type: 'classification' | 'extraction' | 'generation' | 'reasoning'; content: string },
  systemPrompt: string
): Promise<string> {
  const inputTokens = estimateTokens(systemPrompt + task.content);
  const model = selectModel({ ...task, inputTokens, requiresLatestKnowledge: false });

  const response = await anthropic.messages.create({
    model,
    max_tokens: 1024,
    system: systemPrompt,
    messages: [{ role: 'user', content: task.content }],
  });

  return (response.content[0] as Anthropic.TextBlock).text;
}
```

### Batching

The Anthropic Batch API and OpenAI Batch API process requests asynchronously with 50% cost reduction. Use for jobs where you don't need real-time responses.

```typescript
// Anthropic Batch API
async function batchClassify(texts: Array<{ id: string; text: string }>) {
  // Create batch request
  const requests = texts.map((item) => ({
    custom_id: item.id,
    params: {
      model: 'claude-haiku-4-5',
      max_tokens: 20,
      temperature: 0,
      system: 'Classify sentiment as POSITIVE, NEGATIVE, or NEUTRAL.',
      messages: [{ role: 'user', content: item.text }],
    },
  }));

  const batch = await anthropic.beta.messages.batches.create({
    requests: requests as any,
  });

  console.log(`Batch ID: ${batch.id}`);

  // Poll for completion
  let batchResult = batch;
  while (batchResult.processing_status !== 'ended') {
    await new Promise((r) => setTimeout(r, 10_000)); // check every 10s
    batchResult = await anthropic.beta.messages.batches.retrieve(batch.id);
    console.log(`Status: ${batchResult.processing_status} | Counts: ${JSON.stringify(batchResult.request_counts)}`);
  }

  // Retrieve results
  const results: Array<{ id: string; sentiment: string }> = [];
  for await (const result of await anthropic.beta.messages.batches.results(batch.id)) {
    if (result.result.type === 'succeeded') {
      const text = (result.result.message.content[0] as Anthropic.TextBlock).text.trim();
      results.push({ id: result.custom_id, sentiment: text });
    }
  }

  return results;
}
```

### Response Caching

Cache LLM responses for identical or near-identical inputs:

```typescript
import { createHash } from 'crypto';
import { redis } from '../redis';

interface CacheConfig {
  ttlSeconds: number;
  keyPrefix: string;
}

class LLMCache {
  constructor(private config: CacheConfig) {}

  private cacheKey(params: object): string {
    const hash = createHash('sha256')
      .update(JSON.stringify(params))
      .digest('hex');
    return `${this.config.keyPrefix}:${hash}`;
  }

  async get(params: object): Promise<string | null> {
    return redis.get(this.cacheKey(params));
  }

  async set(params: object, response: string): Promise<void> {
    await redis.setex(this.cacheKey(params), this.config.ttlSeconds, response);
  }
}

const cache = new LLMCache({ ttlSeconds: 3600, keyPrefix: 'llm' });

async function cachedComplete(
  params: Anthropic.MessageCreateParamsNonStreaming
): Promise<string> {
  // Only cache deterministic calls
  if (params.temperature && params.temperature > 0) {
    const response = await anthropic.messages.create(params);
    return (response.content[0] as Anthropic.TextBlock).text;
  }

  const cached = await cache.get(params);
  if (cached) {
    console.log('[CACHE HIT]');
    return cached;
  }

  const response = await anthropic.messages.create(params);
  const text = (response.content[0] as Anthropic.TextBlock).text;
  await cache.set(params, text);
  return text;
}
```

### Budget Alerts

```typescript
import { redis } from '../redis';

class AIBudgetTracker {
  constructor(
    private dailyLimitUSD: number,
    private monthlyLimitUSD: number
  ) {}

  async track(model: string, inputTokens: number, outputTokens: number): Promise<void> {
    const cost = estimateCost(model, inputTokens, outputTokens);
    const today = new Date().toISOString().slice(0, 10);
    const month = today.slice(0, 7);

    await Promise.all([
      redis.incrbyfloat(`ai:cost:daily:${today}`, cost),
      redis.incrbyfloat(`ai:cost:monthly:${month}`, cost),
    ]);

    // Set expiry
    await redis.expire(`ai:cost:daily:${today}`, 86400 * 2);   // 2 days
    await redis.expire(`ai:cost:monthly:${month}`, 86400 * 40); // 40 days

    // Check limits
    const [dailyCost, monthlyCost] = await Promise.all([
      redis.get(`ai:cost:daily:${today}`),
      redis.get(`ai:cost:monthly:${month}`),
    ]);

    const daily = parseFloat(dailyCost ?? '0');
    const monthly = parseFloat(monthlyCost ?? '0');

    if (daily > this.dailyLimitUSD * 0.8) {
      console.warn(`AI daily budget at ${((daily/this.dailyLimitUSD)*100).toFixed(0)}%: $${daily.toFixed(2)}/$${this.dailyLimitUSD}`);
    }

    if (daily > this.dailyLimitUSD) {
      throw new Error(`Daily AI budget exceeded: $${daily.toFixed(2)} > $${this.dailyLimitUSD}`);
    }
  }

  async getDailySpend(): Promise<number> {
    const today = new Date().toISOString().slice(0, 10);
    const val = await redis.get(`ai:cost:daily:${today}`);
    return parseFloat(val ?? '0');
  }
}

const budget = new AIBudgetTracker(100, 2000); // $100/day, $2000/month

// Wrap all AI calls
async function trackedComplete(params: Anthropic.MessageCreateParamsNonStreaming): Promise<string> {
  const response = await anthropic.messages.create(params);
  await budget.track(params.model, response.usage.input_tokens, response.usage.output_tokens);
  return (response.content[0] as Anthropic.TextBlock).text;
}
```

## Hands-On Examples

### Cost Dashboard Query

```typescript
// Aggregate cost data from your database
interface CostSummary {
  date: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  costUSD: number;
  requestCount: number;
}

async function getDailyCosts(days = 30): Promise<CostSummary[]> {
  const results = await db.execute(sql`
    SELECT
      DATE(created_at) as date,
      model,
      SUM(input_tokens) as input_tokens,
      SUM(output_tokens) as output_tokens,
      SUM(cost_usd) as cost_usd,
      COUNT(*) as request_count
    FROM ai_requests
    WHERE created_at >= NOW() - INTERVAL '${days} days'
    GROUP BY DATE(created_at), model
    ORDER BY date DESC, cost_usd DESC
  `);
  return results.rows as CostSummary[];
}

// Find expensive features
async function getFeatureCosts(): Promise<Array<{ feature: string; costUSD: number }>> {
  const results = await db.execute(sql`
    SELECT
      feature,
      SUM(cost_usd) as cost_usd,
      AVG(cost_usd) as avg_cost_per_request,
      COUNT(*) as requests
    FROM ai_requests
    WHERE created_at >= NOW() - INTERVAL '7 days'
    GROUP BY feature
    ORDER BY cost_usd DESC
    LIMIT 20
  `);
  return results.rows as any[];
}
```

### Prompt Compression

Long prompts cost more. Compress them without losing meaning:

```typescript
// Remove unnecessary whitespace and markdown from prompts
function compressPrompt(prompt: string): string {
  return prompt
    .replace(/\n{3,}/g, '\n\n')      // max 2 consecutive newlines
    .replace(/[ \t]+/g, ' ')          // collapse multiple spaces
    .replace(/^[ \t]+/gm, '')         // trim line indentation
    .trim();
}

// Before:
const bloated = `
You are a helpful assistant.

    Please analyze the following text:

    The user has provided this content:

    {{content}}

    Please provide a detailed analysis of the above content.
    Make sure to include all relevant information.
`;

// After compression: ~40% fewer tokens
const compressed = `You are a helpful assistant. Analyze:

{{content}}`;
```

### Output Length Control

```typescript
// Control output length by specifying constraints in the prompt
const systemPrompt = `Summarize in exactly 3 bullet points.
Each bullet: max 15 words.
No preamble, no conclusion.`;

// vs the expensive default
const verbosePrompt = `Please provide a comprehensive summary...`;

// Measure the difference
const verboseTokens = estimateTokens(verboseOutput);
const constrainedTokens = estimateTokens(constrainedOutput);
console.log(`Output token reduction: ${verboseTokens - constrainedTokens} tokens`);
// At 100K requests/month, even 200 token reduction = 20M tokens = $300 saved on Sonnet
```

## Common Patterns & Best Practices

### The Cost Audit Process

Run this analysis monthly:

```typescript
async function costAudit() {
  const costs = await getDailyCosts(30);

  // 1. Find most expensive models
  const byModel = costs.reduce((acc, row) => {
    acc[row.model] = (acc[row.model] ?? 0) + row.costUSD;
    return acc;
  }, {} as Record<string, number>);
  console.log('Cost by model (30d):', byModel);

  // 2. Find most expensive features
  const featureCosts = await getFeatureCosts();
  console.log('Top expensive features:', featureCosts.slice(0, 5));

  // 3. Check cache hit rates
  // If cache hit rate < 50% for high-volume features, investigate

  // 4. Check average output length by feature
  // Long average outputs = either model is verbose or max_tokens too high

  // 5. Check model distribution
  // If 80% of simple classification requests use Sonnet, route to Haiku instead
}
```

### Cost vs Quality Tradeoff Table

| Optimization | Cost Reduction | Quality Impact | Effort |
|-------------|----------------|----------------|--------|
| Haiku for simple tasks | 70-80% | None for simple tasks | Low |
| Prompt caching | 60-90% on cached tokens | None | Low |
| Response caching | 100% on cache hits | None | Medium |
| Batch API | 50% | Adds latency | Low |
| Output length control | 20-60% | Minimal | Low |
| Prompt compression | 10-30% | Minimal | Medium |
| Fine-tuning | 50-80% | Can improve | High |

### Precomputing for Common Queries

```typescript
// If many users ask the same questions, precompute answers
const COMMON_QUERIES = [
  'How do I reset my password?',
  'What is your refund policy?',
  'How do I cancel my subscription?',
  // ... top 100 FAQs from analytics
];

async function precomputeFAQAnswers() {
  for (const query of COMMON_QUERIES) {
    const answer = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 500,
      temperature: 0,
      system: 'Answer the user question about our product. Be concise.',
      messages: [{ role: 'user', content: query }],
    });

    const text = (answer.content[0] as Anthropic.TextBlock).text;
    await redis.set(`faq:${hashText(query)}`, text);
  }
}

// At query time: check precomputed answers before calling API
async function answerQuestion(question: string): Promise<string> {
  const precomputed = await redis.get(`faq:${hashText(question)}`);
  if (precomputed) return precomputed;

  // Fall through to API for non-FAQ questions
  return callAPI(question);
}
```

## Anti-Patterns to Avoid

**Sending the same large system prompt uncached**
```typescript
// Bad: pays full price for 5,000 token system prompt on every request
await anthropic.messages.create({
  system: longSystemPrompt,  // 5000 tokens, no caching
  messages: [...],
});

// Good: cache the static part
await anthropic.messages.create({
  system: [{ type: 'text', text: longSystemPrompt, cache_control: { type: 'ephemeral' } }],
  messages: [...],
});
// Savings: 90% on the 5,000 token prefix for every repeat request
```

**Setting max_tokens too high by default**
```typescript
// Bad: always allows 4096 output tokens even for simple classifications
max_tokens: 4096

// Good: set based on expected output length
// Classification: 10-50 tokens
// Short answers: 100-500 tokens
// Full documents: 2000-4096 tokens
```

**Using Opus for everything**
```typescript
// Bad: $15/1M input for a task Haiku handles at $0.80/1M
model: 'claude-opus-4-6'  // for sentiment classification

// Good: match model to task complexity
model: 'claude-haiku-4-5'  // for classification, extraction, simple Q&A
```

**Ignoring caching for batch jobs**
```typescript
// Bad: 10,000 requests with the same system prompt, no caching
for (const item of items) {
  await anthropic.messages.create({ system: prompt, messages: [item] });
}

// Good: use Batch API (50% discount) + prompt caching (90% on prompt)
// Result: ~95% cost reduction on the prompt portion
```

**No monitoring until the bill arrives**
Set up budget alerts at 50%, 80%, and 100% of your daily/monthly limit. Discover cost spikes in hours, not at month end.

## Debugging & Troubleshooting

### Unexpected High Costs

```bash
# 1. Check token counts per request
SELECT
  model,
  AVG(input_tokens) avg_input,
  AVG(output_tokens) avg_output,
  MAX(input_tokens) max_input,
  feature
FROM ai_requests
WHERE created_at > NOW() - INTERVAL '1 day'
GROUP BY model, feature
ORDER BY avg_input DESC;

# 2. Find unexpectedly long outputs
SELECT id, feature, output_tokens, created_at
FROM ai_requests
WHERE output_tokens > 3000
ORDER BY created_at DESC;

# 3. Check if caching is working
SELECT
  SUM(cache_read_input_tokens) / NULLIF(SUM(input_tokens + cache_read_input_tokens), 0) as cache_hit_rate
FROM ai_requests
WHERE created_at > NOW() - INTERVAL '1 day'
  AND feature = 'customer-support';
```

### Cache Miss Investigation

```typescript
// If cache hit rate is low, check why
async function debugCacheMisses() {
  // Common reasons:
  // 1. System prompt changes between requests (templates with dynamic content)
  console.log('Check: is the system prompt truly static across requests?');

  // 2. Prefix too short (< 1024 tokens for Anthropic caching)
  const promptTokens = await countInputTokens([], systemPrompt);
  console.log(`System prompt tokens: ${promptTokens} (need 1024+ for caching)`);

  // 3. Cache TTL expired (5 min for Anthropic)
  console.log('Check: are requests being made within 5 minutes of each other?');
}
```

## Real-World Scenarios

### Scenario 1: Customer Support at Scale

10,000 support tickets per day. Initial implementation: GPT-4o with 3,000-token system prompt = ~$650/day.

**Optimization journey:**
```
Step 1: Prompt caching (same 3,000-token prompt on every request)
  → System prompt: from $0.0075 to $0.00075 per request (-90%)
  → Total: $650 → $280/day

Step 2: Route simple queries to Haiku (70% of tickets are simple)
  → Haiku at $0.80/1M vs GPT-4o at $2.50/1M input
  → Total: $280 → $115/day

Step 3: Cache repeat FAQ answers in Redis (30% of tickets are FAQs)
  → 3,000 requests skip the API entirely
  → Total: $115 → $80/day

Step 4: Batch API for non-urgent tickets (queue to process at night)
  → 50% discount on 40% of volume
  → Total: $80 → $64/day

Final result: $650 → $64/day (90% cost reduction)
```

### Scenario 2: Cost Tracking Middleware

```typescript
// Middleware that tracks every AI call
export async function withCostTracking<T>(
  fn: () => Promise<{ result: T; usage: Anthropic.Usage; model: string }>,
  context: { feature: string; userId?: string }
): Promise<T> {
  const { result, usage, model } = await fn();
  const cost = estimateCost(model, usage.input_tokens, usage.output_tokens);

  // Fire-and-forget cost tracking
  db.insert(aiRequests).values({
    model,
    feature: context.feature,
    userId: context.userId,
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    cacheReadTokens: usage.cache_read_input_tokens ?? 0,
    costUsd: cost,
  }).catch((err) => logger.error('Failed to track AI cost', err));

  return result;
}

// Usage
const text = await withCostTracking(
  async () => {
    const response = await anthropic.messages.create(params);
    return {
      result: (response.content[0] as Anthropic.TextBlock).text,
      usage: response.usage,
      model: params.model,
    };
  },
  { feature: 'customer-support', userId: req.user.id }
);
```

## Further Reading

- [Anthropic Pricing](https://www.anthropic.com/pricing)
- [Anthropic Prompt Caching Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Anthropic Batch API](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [LiteLLM — Model-agnostic proxy with cost tracking](https://docs.litellm.ai/)
- [Helicone — LLM observability with cost analytics](https://www.helicone.ai/)

## Summary

LLM costs are driven by token count × model price. Control both to stay within budget.

**Highest impact optimizations (in order):**

1. **Prompt caching** (60-90% savings on repeated static prompts) — mark long system prompts with `cache_control: ephemeral`. Requires 1,024+ token prefix, works within 5-minute window.

2. **Model routing** (70-80% savings on simple tasks) — use Haiku for classification/extraction, Sonnet for balanced tasks, Opus only for complex reasoning.

3. **Response caching** (100% savings on cache hits) — cache deterministic (temp=0) responses for FAQ-style queries in Redis with appropriate TTL.

4. **Batch API** (50% savings) — use for non-real-time jobs: classification pipelines, document processing, overnight batch work.

5. **Output length control** (20-60% savings) — specify word/sentence/bullet limits in system prompt. Output tokens cost 3-5x more than input.

6. **Prompt compression** (10-30% savings) — remove unnecessary whitespace, redundant instructions, verbose formatting.

**Monitoring:**
- Log every AI call: model, feature, input/output tokens, cost
- Set budget alerts at 80% and 100% of daily/monthly limits
- Run monthly cost audits: identify expensive features and model distribution mismatches
- Track cache hit rates; low hit rates on high-volume features indicate a caching gap
