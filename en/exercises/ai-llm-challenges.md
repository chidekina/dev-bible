# AI / LLM Challenges

> Practical exercises for building production AI features: RAG pipelines, tool use, streaming, prompt engineering, and agentic systems. Stack assumed: Node.js 20+, TypeScript, Anthropic SDK, Vercel AI SDK, pgvector, PostgreSQL. Target skill level: intermediate.

---

## Exercise 1 — Build a RAG Pipeline with pgvector (Hard)

**Scenario:** Build a Retrieval-Augmented Generation (RAG) pipeline that answers questions about your company's internal documentation stored in PostgreSQL with the `pgvector` extension.

**Requirements:**
- Ingest pipeline: read Markdown documents, chunk them (500 tokens, 50-token overlap), generate embeddings using the Anthropic API or OpenAI `text-embedding-3-small`, and store in a `document_chunks` table.
- Query pipeline: embed the user question, find the top-5 most similar chunks using cosine similarity, inject them as context into a prompt, and return the LLM's answer.
- `document_chunks` schema: `id`, `document_id`, `content` (text), `embedding` (vector(1536)), `metadata` (JSONB).
- Citation: each answer must include the source document name and chunk index.
- Unanswerable questions (no relevant context found above a similarity threshold of 0.75) must return `"I don't have information about that."` instead of hallucinating.

**Acceptance Criteria:**
- [ ] `POST /rag/ingest` accepts a Markdown file and stores all chunks with embeddings.
- [ ] `POST /rag/query` returns an answer with citations within 3 seconds.
- [ ] A question with no relevant docs returns the canned "I don't have information" response.
- [ ] `EXPLAIN ANALYZE` on the similarity query shows an IVFFlat or HNSW index used (not a sequential scan).
- [ ] Chunk overlap is implemented correctly — the last 50 tokens of chunk N are the first 50 tokens of chunk N+1.

**Hints:**
1. pgvector setup: `CREATE EXTENSION IF NOT EXISTS vector; ALTER TABLE document_chunks ADD COLUMN embedding vector(1536);`.
2. Cosine similarity query: `SELECT content, 1 - (embedding <=> $1::vector) AS similarity FROM document_chunks ORDER BY embedding <=> $1::vector LIMIT 5`.
3. IVFFlat index: `CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`. Run `VACUUM ANALYZE` after bulk insert.
4. Similarity threshold: filter results with `WHERE 1 - (embedding <=> $1) > 0.75` before injecting into the prompt.
5. Context injection: `You are a helpful assistant. Answer based only on the following context:\n\n${chunks.map(c => c.content).join('\n\n')}\n\nQuestion: ${question}`.

---

## Exercise 2 — Implement Tool Use with Anthropic SDK (Medium)

**Scenario:** Build a Claude-powered assistant that can answer questions by calling external tools: a weather API and a calculator. The assistant decides when to call tools and when to respond directly.

**Requirements:**
- Define two tools: `get_weather({ city: string }): { temperature, condition }` and `calculate({ expression: string }): { result: number }`.
- The `calculate` tool evaluates safe math expressions (no `eval` — use a math expression parser like `mathjs`).
- Implement the agentic loop: send message → if Claude returns tool calls, execute them, send results back → repeat until Claude returns a text response.
- Handle tool errors gracefully: if `get_weather` throws, send an error result back to Claude so it can inform the user.
- The conversation is multi-turn — maintain message history across requests.

**Acceptance Criteria:**
- [ ] "What's the weather in London?" triggers a `get_weather` tool call with `city: "London"`.
- [ ] "What is 2^10 + 5" triggers a `calculate` tool call with the correct expression.
- [ ] A failed weather API call results in Claude responding "I couldn't retrieve the weather for that city" (not an unhandled error).
- [ ] The agentic loop terminates when Claude returns `stop_reason: "end_turn"`.
- [ ] No more than 5 tool call iterations per user message (infinite loop guard).

**Hints:**
1. Tool definition:
   ```typescript
   const tools = [{
     name: 'get_weather',
     description: 'Get current weather for a city',
     input_schema: { type: 'object', properties: { city: { type: 'string' } }, required: ['city'] }
   }];
   ```
2. Agentic loop: after a `tool_use` response, create a `tool_result` message and call the API again.
3. Tool result for errors: `{ type: 'tool_result', tool_use_id: toolUse.id, is_error: true, content: errorMessage }`.
4. `mathjs` eval: `import { evaluate } from 'mathjs'; evaluate('2^10 + 5')` — safe, no arbitrary code execution.

---

## Exercise 3 — Stream a Chat Response with Vercel AI SDK (Medium)

**Scenario:** Build a streaming chat endpoint that returns tokens to the client as they are generated, so the UI can display the response progressively.

**Requirements:**
- `POST /chat` accepts `{ messages: Message[], model: string }` and streams the response using Vercel AI SDK's `streamText`.
- The response must use the AI SDK's data stream protocol (compatible with the `useChat` React hook).
- Handle stream errors gracefully — if the LLM API returns an error mid-stream, send an error chunk and close the stream.
- Implement a `maxTokens` limit of 2048 per response.
- Log the total tokens used (prompt + completion) after the stream completes.

**Acceptance Criteria:**
- [ ] The client receives the first token within 500ms of the request.
- [ ] `useChat` hook in a React frontend renders streaming tokens correctly.
- [ ] An LLM API error mid-stream is sent to the client as `{ error: "LLM error: ..." }` in the stream, not a 500 response.
- [ ] After the stream, `POST /analytics/usage` is called with `{ promptTokens, completionTokens, model }`.
- [ ] `maxTokens: 2048` is enforced — responses are cut off at the limit with a `max_tokens` stop reason.

**Hints:**
1. Vercel AI SDK streaming:
   ```typescript
   import { streamText } from 'ai';
   import { anthropic } from '@ai-sdk/anthropic';
   const result = streamText({ model: anthropic('claude-sonnet-4-6'), messages, maxTokens: 2048 });
   return result.toDataStreamResponse();
   ```
2. Token logging: `result.usage.then(usage => logUsage(usage))` — the promise resolves after the stream completes.
3. Error handling: wrap in try/catch. On error, use `result.toDataStreamResponse()` with an error transformer or manually write to the response stream.
4. Fastify + streams: use `reply.raw` to write the stream response directly, or use a Fastify plugin compatible with the Web Streams API.

---

## Exercise 4 — Implement Prompt Caching to Reduce Costs (Medium)

**Scenario:** Your RAG system sends the same system prompt and documentation context on every request. Use Anthropic's prompt caching to avoid re-processing unchanged prefixes.

**Requirements:**
- Mark the system prompt and the document context as cacheable using `cache_control: { type: "ephemeral" }`.
- Measure the cost difference between cached and uncached requests using the `usage` response field.
- Implement a cache warmup: on app startup, send a dummy request to ensure the cache is primed before user traffic arrives.
- Log `cache_read_input_tokens` and `cache_creation_input_tokens` on every request.
- The cache should reduce costs by at least 80% on repeated queries with the same context.

**Acceptance Criteria:**
- [ ] First request shows `cache_creation_input_tokens > 0` and `cache_read_input_tokens = 0`.
- [ ] Subsequent requests with the same context show `cache_read_input_tokens > 0` and `cache_creation_input_tokens = 0`.
- [ ] Cost calculation: `promptTokens * 3/1000000 (uncached) vs. cacheReadTokens * 0.3/1000000 (cached)` — verify the 10x cost reduction.
- [ ] The system prompt block has `cache_control: { type: "ephemeral" }` as the last field in the content block.
- [ ] Cache warmup completes before the first user request (enforced by startup order).

**Hints:**
1. Cache-eligible content: the system prompt + large context blocks. Must be at least 1024 tokens to be cached.
2. Content block with cache control:
   ```typescript
   { type: 'text', text: systemPrompt, cache_control: { type: 'ephemeral' } }
   ```
3. Only the last `cache_control` block in the messages array is used for caching — put it on the largest stable prefix.
4. Cache TTL is 5 minutes by default — design the warmup to re-fire if the server is idle for more than 4 minutes.

---

## Exercise 5 — Build a Model Router (Medium)

**Scenario:** Not all user queries require an expensive frontier model. Build a router that sends simple queries to a fast, cheap model and complex queries to a capable frontier model.

**Requirements:**
- Classify each query as `simple` or `complex` using a lightweight heuristic or a cheap classifier call.
- Simple queries (factual lookups, greetings, short answers): route to `claude-haiku-4-5`.
- Complex queries (multi-step reasoning, code generation, long-form writing): route to `claude-sonnet-4-6`.
- Log the model used, decision reason, and token cost for each query.
- Allow users to override with `?model=sonnet` or `?model=haiku` query params.

**Acceptance Criteria:**
- [ ] "What is the capital of France?" is routed to Haiku.
- [ ] "Write a Fastify middleware that implements JWT auth with refresh token rotation" is routed to Sonnet.
- [ ] `?model=haiku` forces Haiku regardless of query complexity.
- [ ] Routing decision is logged: `{ query_type: 'simple', model: 'claude-haiku-4-5', reason: 'short factual query' }`.
- [ ] The classifier itself uses Haiku (not Sonnet) to keep classification cost minimal.

**Hints:**
1. Simple heuristic: query length < 50 words AND no code-like tokens AND no complex keywords → `simple`. Else → use a classifier.
2. Classifier prompt: `Classify this query as 'simple' or 'complex'. Reply with only the word. Query: "${query}"`. Use a low `maxTokens: 10`.
3. Cost comparison: Haiku is ~20x cheaper per token than Sonnet. Routing 70% of queries to Haiku cuts LLM costs significantly.
4. Override param: `const model = req.query.model === 'haiku' ? 'claude-haiku-4-5' : req.query.model === 'sonnet' ? 'claude-sonnet-4-6' : routedModel`.

---

## Exercise 6 — Write a Prompt Evaluation Suite (Hard)

**Scenario:** Your AI feature's prompt was changed last sprint. You have no way to tell if the new prompt is better or worse. Build an LLM-as-judge evaluation suite.

**Requirements:**
- Create a dataset of 20 test cases: `{ input: string, expectedBehavior: string }`.
- For each test case, run the prompt and collect the response.
- Use an LLM judge (a separate Claude call) to score each response: `{ score: 1-5, reasoning: string }`.
- Compare two prompt versions (A and B) and output a summary: `{ promptA: { avgScore, passRate }, promptB: { avgScore, passRate } }`.
- A "pass" is defined as a score of 4 or 5 from the judge.

**Acceptance Criteria:**
- [ ] The evaluation script runs with `npx ts-node eval.ts --promptA prompts/v1.txt --promptB prompts/v2.txt`.
- [ ] Output is a JSON file with per-test-case scores and a summary comparison.
- [ ] Judge prompt instructs the LLM to be a strict evaluator: `"You are a strict QA evaluator. Score this response from 1-5 based on [criteria]. Respond with JSON: { score, reasoning }"`.
- [ ] The judge uses `claude-haiku-4-5` to minimize evaluation cost.
- [ ] The test dataset covers edge cases: ambiguous queries, off-topic inputs, queries requiring refusal.

**Hints:**
1. Judge call: send `{ input, expectedBehavior, actualResponse }` to the judge model. Parse JSON from the response.
2. Run evaluations in parallel: `await Promise.all(testCases.map(tc => evaluateTestCase(tc)))` — much faster than sequential.
3. Pass rate: `(scores.filter(s => s >= 4).length / scores.length) * 100`.
4. Prevent judge hallucination: use a JSON schema or `z.object({ score: z.number().min(1).max(5), reasoning: z.string() })` to validate the judge's output.

---

## Exercise 7 — Implement Retry + Fallback for LLM API Errors (Medium)

**Scenario:** Your LLM integration crashes silently on API errors (rate limits, timeouts, overload). Implement a robust retry and fallback strategy.

**Requirements:**
- Retry on: `429 Too Many Requests` (rate limit), `529 Overloaded`, `500/503` server errors.
- Do not retry on: `400 Bad Request` (invalid prompt), `401 Unauthorized` (bad API key), `404`.
- Exponential backoff: 1s, 2s, 4s (max 3 retries per request).
- Fallback: after 3 retries on Sonnet, fall back to Haiku once before giving up.
- On all retries exhausted: return a user-friendly error `{ error: "AI service temporarily unavailable, please try again." }`.

**Acceptance Criteria:**
- [ ] A 429 response triggers a retry after ~1 second.
- [ ] A 400 response is not retried — it is returned immediately as an error.
- [ ] After 3 failed Sonnet retries, the same request is attempted once with Haiku.
- [ ] All retry attempts and the final outcome are logged with structured fields: `{ attempt, model, status, delay }`.
- [ ] A test using a mock API server verifies the retry schedule without actual API calls.

**Hints:**
1. Retryable status codes: `[429, 500, 503, 529]`. Non-retryable: `[400, 401, 403, 404]`.
2. Backoff calculation: `const delay = Math.min(1000 * 2 ** attempt, 30000)`. Add jitter: `delay + Math.random() * 1000`.
3. Fallback pattern:
   ```typescript
   try {
     return await callWithRetry('claude-sonnet-4-6', prompt);
   } catch {
     return await callWithRetry('claude-haiku-4-5', prompt, { maxAttempts: 1 });
   }
   ```
4. Test with a mock server that returns `429` for the first 3 requests, then `200` — verify the correct number of retries.

---

## Exercise 8 — Build a Document Chunking Pipeline (Medium)

**Scenario:** Build a robust document chunking pipeline that splits documents into embedding-friendly chunks while preserving semantic boundaries.

**Requirements:**
- Input: Markdown documents with headings, paragraphs, and code blocks.
- Chunk strategy: split on heading boundaries first. If a section is > 500 tokens, split further on paragraph boundaries. If a paragraph is > 500 tokens, split on sentence boundaries.
- Preserve metadata per chunk: `{ documentId, heading, chunkIndex, totalChunks, tokenCount }`.
- Overlap: each chunk (except the first) starts with the last sentence of the previous chunk.
- Output: `Chunk[]` with content and metadata.

**Acceptance Criteria:**
- [ ] A section with 3 paragraphs of 200 tokens each remains as one chunk (600 tokens total... wait — re-check: if > 500, split).
- [ ] A code block is never split mid-block (code blocks are treated as atomic units).
- [ ] Each chunk's `tokenCount` is accurate (use `tiktoken` or `@anthropic-ai/tokenizer`).
- [ ] Overlap is present: chunk N's last sentence appears as the first sentence of chunk N+1.
- [ ] A test processes a 10,000-token document and verifies all chunks are ≤ 500 tokens.

**Hints:**
1. Parse Markdown structure with `remark` or manual regex: identify headings (`^#{1,6} `), code fences (` ``` `), and paragraphs (double newline).
2. Token counting: `import { encode } from 'gpt-tokenizer'; encode(text).length` — compatible with both OpenAI and Anthropic token counts (approximately).
3. Atomic code blocks: when you encounter a code fence, collect the entire block as one unit and do not split it, even if it exceeds the token limit.
4. Recursion: `splitSection(section)` → if tokens <= 500, return as-is; else split into paragraphs and recursively apply.

---

## Exercise 9 — Add Token Counting Before Sending to API (Easy)

**Scenario:** Your chat application occasionally sends prompts that exceed the model's context window, causing cryptic API errors. Add token counting to validate and truncate prompts before sending.

**Requirements:**
- Count tokens in the message history before sending to the API.
- If the token count exceeds 80% of the model's context window, truncate the oldest messages (not the system prompt or latest user message).
- Log a warning when truncation occurs: `{ originalMessages: N, truncatedTo: M, tokens: T }`.
- Expose a `GET /chat/token-count` endpoint that returns the current token count for a given message array.
- Support multiple models with different context window sizes: `claude-sonnet-4-6` (200k), `claude-haiku-4-5` (200k).

**Acceptance Criteria:**
- [ ] A message history of 250,000 tokens is truncated to 160,000 tokens (80% of 200k) before sending.
- [ ] The system prompt is always preserved (never truncated).
- [ ] The latest user message is always preserved (never truncated).
- [ ] `GET /chat/token-count` returns `{ messages: N, tokens: T, model: "claude-sonnet-4-6", withinLimit: true }`.
- [ ] A unit test verifies truncation logic with a mock message history.

**Hints:**
1. Anthropic's `countTokens` API: `anthropic.messages.countTokens({ model, messages, system })` returns `{ input_tokens }`.
2. Alternatively, use `tiktoken` locally for free token counting (approximate for Claude models).
3. Truncation algorithm: keep system prompt + last user message. Then greedily add messages from newest to oldest until the limit is reached.
4. 80% threshold: `const limit = MODEL_CONTEXT_WINDOWS[model] * 0.8`.

---

## Exercise 10 — Build a Simple Agent Loop with Tool Use and Memory (Hard)

**Scenario:** Build a simple autonomous agent that can browse a fake product database, add items to a shopping list, and answer questions about the list. The agent has persistent memory across turns.

**Requirements:**
- Tools: `search_products({ query: string })`, `add_to_list({ productId: string, quantity: number })`, `get_list()`, `remove_from_list({ productId: string })`.
- Memory: the shopping list persists across conversation turns (stored in Redis or in-memory Map).
- The agent loop runs until the task is complete or the user says "done".
- Implement a max iterations guard (10 iterations max per user turn).
- The agent must explain its reasoning before calling tools (chain-of-thought via `thinking` blocks or a `scratchpad` in the system prompt).

**Acceptance Criteria:**
- [ ] "Find me organic apples and add 3 to my list" results in a `search_products` call followed by `add_to_list`.
- [ ] "What's in my list?" triggers `get_list` and returns a formatted response.
- [ ] After 10 tool calls with no end_turn, the agent loop stops and returns `"Task limit reached."`.
- [ ] The shopping list persists: adding items in message 1 and querying in message 3 returns the same list.
- [ ] A test covers the full loop: search → add → get_list → verify.

**Hints:**
1. Agent loop:
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
2. Memory: `const lists = new Map<userId, CartItem[]>()`. `get_list` and `add_to_list` read/write this map.
3. Tool execution: filter `response.content` for `type === 'tool_use'` blocks. Execute each tool and collect `tool_result` blocks.
4. Chain-of-thought: add to system prompt `"Before calling a tool, briefly explain your reasoning in one sentence."` — this produces more predictable behavior.
