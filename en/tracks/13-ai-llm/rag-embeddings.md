# RAG & Embeddings

## Overview

Retrieval-Augmented Generation (RAG) is the pattern of enriching an LLM prompt with relevant information retrieved from an external knowledge base. Instead of hoping the model "remembers" facts from training, you retrieve the facts at query time and include them in the context.

RAG solves the core limitations of LLMs: knowledge cutoff (training data is frozen), hallucination on domain-specific facts, and context window limits (you can't fit an entire knowledge base in one prompt). Embeddings are the mathematical foundation that makes similarity-based retrieval possible.

## Prerequisites

- LLM Fundamentals (`llm-fundamentals.md`)
- PostgreSQL basics (for pgvector)
- TypeScript

## Core Concepts

### Embeddings

An **embedding** is a vector (array of floating-point numbers) that represents the semantic meaning of text. Two pieces of text that mean similar things have similar embeddings — their vectors are close in the high-dimensional embedding space.

```typescript
// Get an embedding for a piece of text
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function embed(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',  // 1536 dimensions, $0.02/1M tokens
    input: text,
  });
  return response.data[0].embedding;
}

// Similarity: cosine similarity (ranges from -1 to 1, higher = more similar)
function cosineSimilarity(a: number[], b: number[]): number {
  const dotProduct = a.reduce((sum, ai, i) => sum + ai * b[i], 0);
  const magnitudeA = Math.sqrt(a.reduce((sum, ai) => sum + ai * ai, 0));
  const magnitudeB = Math.sqrt(b.reduce((sum, bi) => sum + bi * bi, 0));
  return dotProduct / (magnitudeA * magnitudeB);
}

const v1 = await embed('How do I reset my password?');
const v2 = await embed('Steps to change your login credentials');
const v3 = await embed('Best restaurants in Paris');

console.log(cosineSimilarity(v1, v2));  // ~0.92 — very similar
console.log(cosineSimilarity(v1, v3));  // ~0.15 — very different
```

### Embedding Models

| Model | Dimensions | Best For | Cost |
|-------|-----------|----------|------|
| `text-embedding-3-small` (OpenAI) | 1536 | General purpose, cost-efficient | $0.02/1M |
| `text-embedding-3-large` (OpenAI) | 3072 | Higher accuracy | $0.13/1M |
| `voyage-3-lite` (Voyage AI) | 512 | Retrieval-optimized, very fast | $0.02/1M |
| `voyage-3` (Voyage AI) | 1024 | Retrieval-optimized, high quality | $0.06/1M |
| `nomic-embed-text` (local) | 768 | Self-hosted, no cost per query | Free |

Key insight: embedding models and LLMs are different — a cheap embedding model is fine for retrieval. Use the expensive LLM only for generation, not embedding.

### Vector Databases

A vector database stores embeddings and supports **approximate nearest neighbor (ANN)** search — finding the N most similar vectors to a query vector.

Options:
| Solution | Best For |
|---------|----------|
| **pgvector** (PostgreSQL extension) | Existing PostgreSQL, < 1M vectors |
| **Pinecone** | Fully managed, large scale, no ops |
| **Qdrant** | Self-hosted, high performance |
| **Weaviate** | Built-in ML features, hybrid search |
| **Chroma** | Local development, prototyping |

### pgvector Setup

pgvector adds a `vector` type and vector similarity operators to PostgreSQL.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table with a vector column
CREATE TABLE documents (
  id          BIGSERIAL PRIMARY KEY,
  content     TEXT NOT NULL,
  metadata    JSONB DEFAULT '{}',
  embedding   vector(1536),   -- must match your embedding model's dimensions
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Create an index for fast similarity search
-- HNSW: best performance for most use cases
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Or IVFFlat: lower memory, good for large datasets
-- CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
--   WITH (lists = 100);
```

```typescript
// TypeScript with Drizzle ORM + pgvector
import { pgTable, bigserial, text, jsonb, timestamp } from 'drizzle-orm/pg-core';
import { vector } from 'pgvector/drizzle-orm';

export const documents = pgTable('documents', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  content: text('content').notNull(),
  metadata: jsonb('metadata').$type<Record<string, unknown>>().default({}),
  embedding: vector('embedding', { dimensions: 1536 }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
});
```

### Chunking Strategies

You can't embed entire documents — they're too large and dilute the semantic signal. You must split documents into chunks before embedding.

```typescript
interface Chunk {
  content: string;
  metadata: {
    sourceId: string;
    chunkIndex: number;
    startChar: number;
    endChar: number;
  };
}

// Fixed-size chunking (simple, often good enough)
function chunkFixed(text: string, chunkSize = 512, overlap = 50): Chunk[] {
  const chunks: Chunk[] = [];
  let start = 0;

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length);
    chunks.push({
      content: text.slice(start, end),
      metadata: { sourceId: 'doc', chunkIndex: chunks.length, startChar: start, endChar: end },
    });
    start += chunkSize - overlap; // overlap helps maintain context across boundaries
  }
  return chunks;
}

// Sentence-aware chunking (better quality)
function chunkBySentences(
  text: string,
  maxTokens = 400,
  overlapSentences = 1
): string[] {
  // Split on sentence boundaries
  const sentences = text.match(/[^.!?]+[.!?]+/g) ?? [text];
  const chunks: string[] = [];
  let current: string[] = [];
  let tokenCount = 0;

  for (const sentence of sentences) {
    const sentenceTokens = Math.ceil(sentence.length / 4); // rough estimate

    if (tokenCount + sentenceTokens > maxTokens && current.length > 0) {
      chunks.push(current.join(' '));
      // Keep last N sentences as overlap
      current = current.slice(-overlapSentences);
      tokenCount = current.join(' ').length / 4;
    }

    current.push(sentence.trim());
    tokenCount += sentenceTokens;
  }

  if (current.length > 0) {
    chunks.push(current.join(' '));
  }

  return chunks;
}

// Markdown-aware chunking (for documentation)
function chunkMarkdown(markdown: string, maxTokens = 512): string[] {
  // Split on headers — each section becomes a chunk (with its header)
  const sections = markdown.split(/^#{1,3} .+$/m);
  const headers = markdown.match(/^#{1,3} .+$/mg) ?? [];

  return headers.map((header, i) => {
    const content = sections[i + 1] ?? '';
    return `${header}\n${content}`.trim();
  }).filter((chunk) => chunk.length > 50); // skip empty sections
}
```

### The RAG Pipeline

```
Document ingestion:
  1. Load document (PDF, HTML, markdown, plain text)
  2. Split into chunks
  3. Embed each chunk
  4. Store chunk + embedding in vector DB

Query time:
  1. Embed the user query
  2. Find K most similar chunks (ANN search)
  3. Include chunks in LLM prompt as context
  4. Generate answer grounded in retrieved context
```

### Reranking

First-stage retrieval (embedding similarity) is fast but imprecise. Reranking uses a cross-encoder model to score query-chunk pairs more accurately, then reorders the results.

```typescript
import Anthropic from '@anthropic-ai/sdk';

// Simple LLM-based reranker
async function rerankChunks(
  query: string,
  chunks: string[],
  topK = 3
): Promise<string[]> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5',
    max_tokens: 100,
    temperature: 0,
    messages: [{
      role: 'user',
      content: `Query: "${query}"

Rank these ${chunks.length} passages by relevance to the query. Return only the indices (0-based) of the top ${topK} most relevant passages, comma-separated.

${chunks.map((c, i) => `[${i}] ${c.slice(0, 200)}`).join('\n\n')}`,
    }],
  });

  const indices = (response.content[0] as Anthropic.TextBlock).text
    .trim()
    .split(',')
    .map((s) => parseInt(s.trim()))
    .filter((i) => !isNaN(i) && i >= 0 && i < chunks.length)
    .slice(0, topK);

  return indices.map((i) => chunks[i]);
}

// Production: use Cohere Rerank or Voyage Rerank (purpose-built, much faster)
import { CohereClient } from 'cohere-ai';
const cohere = new CohereClient({ token: process.env.COHERE_API_KEY });

async function rerankWithCohere(
  query: string,
  documents: string[],
  topN = 3
): Promise<string[]> {
  const response = await cohere.v2.rerank({
    model: 'rerank-v3.5',
    query,
    documents,
    topN,
  });

  return response.results.map((r) => documents[r.index]);
}
```

## Hands-On Examples

### Full RAG Implementation with pgvector

```typescript
// src/rag/index.ts
import { db } from '../db';
import { documents } from '../db/schema';
import { sql, desc } from 'drizzle-orm';
import OpenAI from 'openai';
import Anthropic from '@anthropic-ai/sdk';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ---- Indexing ----

async function embedText(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text.replace(/\n/g, ' '),
  });
  return response.data[0].embedding;
}

export async function indexDocument(
  content: string,
  metadata: Record<string, unknown> = {}
): Promise<void> {
  const chunks = chunkBySentences(content);

  for (let i = 0; i < chunks.length; i++) {
    const embedding = await embedText(chunks[i]);
    await db.insert(documents).values({
      content: chunks[i],
      metadata: { ...metadata, chunkIndex: i },
      embedding,
    });
  }
}

// ---- Retrieval ----

async function retrieveRelevantChunks(
  query: string,
  topK = 5
): Promise<string[]> {
  const queryEmbedding = await embedText(query);

  const results = await db.execute(sql`
    SELECT content, 1 - (embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS similarity
    FROM documents
    ORDER BY embedding <=> ${JSON.stringify(queryEmbedding)}::vector
    LIMIT ${topK}
  `);

  return (results.rows as any[])
    .filter((r) => r.similarity > 0.7)  // only include sufficiently similar chunks
    .map((r) => r.content);
}

// ---- Generation ----

export async function answerWithRAG(question: string): Promise<string> {
  // 1. Retrieve relevant context
  const chunks = await retrieveRelevantChunks(question, 5);

  if (chunks.length === 0) {
    return "I don't have enough information in my knowledge base to answer that question.";
  }

  const context = chunks.map((c, i) => `[${i + 1}] ${c}`).join('\n\n');

  // 2. Generate answer grounded in context
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    temperature: 0,
    system: `You are a helpful assistant. Answer questions based ONLY on the provided context.
If the context doesn't contain enough information to answer, say so.
Do not use knowledge from outside the provided context.
Cite sources by referencing the context numbers, e.g., "According to [1]..."`,
    messages: [{
      role: 'user',
      content: `Context:\n${context}\n\nQuestion: ${question}`,
    }],
  });

  return (response.content[0] as Anthropic.TextBlock).text;
}
```

### Hybrid Search (Vector + Full-Text)

Pure vector search misses exact keyword matches. Hybrid search combines vector similarity with full-text search (BM25) for better retrieval.

```typescript
// pgvector supports RRF (Reciprocal Rank Fusion) hybrid search
async function hybridSearch(
  query: string,
  topK = 5
): Promise<Array<{ content: string; score: number }>> {
  const queryEmbedding = await embedText(query);

  const results = await db.execute(sql`
    WITH vector_results AS (
      SELECT id, content,
             ROW_NUMBER() OVER (ORDER BY embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS rank
      FROM documents
      ORDER BY embedding <=> ${JSON.stringify(queryEmbedding)}::vector
      LIMIT 20
    ),
    text_results AS (
      SELECT id, content,
             ROW_NUMBER() OVER (ORDER BY ts_rank(to_tsvector('english', content), plainto_tsquery('english', ${query})) DESC) AS rank
      FROM documents
      WHERE to_tsvector('english', content) @@ plainto_tsquery('english', ${query})
      LIMIT 20
    ),
    rrf AS (
      SELECT COALESCE(v.id, t.id) AS id,
             COALESCE(v.content, t.content) AS content,
             (COALESCE(1.0 / (60 + v.rank), 0) + COALESCE(1.0 / (60 + t.rank), 0)) AS score
      FROM vector_results v
      FULL OUTER JOIN text_results t ON v.id = t.id
    )
    SELECT * FROM rrf
    ORDER BY score DESC
    LIMIT ${topK}
  `);

  return results.rows as any[];
}
```

### Pinecone Integration

```typescript
import { Pinecone } from '@pinecone-database/pinecone';

const pc = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
const index = pc.index('my-knowledge-base');

// Upsert documents
async function upsertToPinecone(
  id: string,
  content: string,
  metadata: Record<string, string>
) {
  const embedding = await embedText(content);
  await index.upsert([{
    id,
    values: embedding,
    metadata: { content, ...metadata },  // store content in metadata for retrieval
  }]);
}

// Query
async function queryPinecone(query: string, topK = 5) {
  const queryEmbedding = await embedText(query);
  const results = await index.query({
    vector: queryEmbedding,
    topK,
    includeMetadata: true,
  });
  return results.matches.map((m) => m.metadata?.content as string);
}
```

## Common Patterns & Best Practices

### Chunk Size Guidelines

| Content Type | Chunk Size | Overlap |
|-------------|-----------|---------|
| Dense technical docs | 256-512 tokens | 10-20% |
| General prose | 512-768 tokens | 10-15% |
| Code | Function/class level | N/A |
| FAQ entries | Entire Q&A pair | N/A |
| Legal/medical text | 512 tokens | 15-25% |

Larger chunks = more context per retrieval, more noise in embedding
Smaller chunks = more precise matching, less context

### Metadata Filtering

Store structured metadata with each chunk for pre-filtering:

```typescript
// Index with metadata
await index.upsert([{
  id: `doc-${id}-chunk-${i}`,
  values: embedding,
  metadata: {
    content: chunk,
    source: 'documentation',
    category: 'api-reference',
    version: '2.4',
    language: 'en',
    createdAt: new Date().toISOString(),
  },
}]);

// Filter before similarity search
const results = await index.query({
  vector: queryEmbedding,
  topK: 5,
  filter: {
    category: { $eq: 'api-reference' },
    version: { $eq: '2.4' },
  },
  includeMetadata: true,
});
```

### Query Expansion

The user's query may not use the same words as the documents. Expand the query before embedding:

```typescript
async function expandQuery(query: string): Promise<string[]> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5',
    max_tokens: 200,
    temperature: 0.3,
    messages: [{
      role: 'user',
      content: `Generate 3 alternative phrasings of this search query.
Query: "${query}"
Return only the 3 alternatives, one per line. No numbering or bullets.`,
    }],
  });

  const alternatives = (response.content[0] as Anthropic.TextBlock).text
    .trim()
    .split('\n')
    .filter(Boolean);

  return [query, ...alternatives];
}

// Search with all query variants, merge results
async function retrieveWithExpansion(query: string, topK = 5) {
  const queries = await expandQuery(query);
  const allResults = await Promise.all(
    queries.map((q) => retrieveRelevantChunks(q, topK))
  );

  // Deduplicate by content
  const seen = new Set<string>();
  const unique = allResults.flat().filter((chunk) => {
    if (seen.has(chunk)) return false;
    seen.add(chunk);
    return true;
  });

  return unique.slice(0, topK);
}
```

## Anti-Patterns to Avoid

**Embedding full documents without chunking**
```typescript
// Bad: 10-page document in one embedding = semantic average of all topics
const embedding = await embedText(entireDocument);  // loses specificity

// Good: chunk first, embed each chunk
const chunks = chunkBySentences(document);
const embeddings = await Promise.all(chunks.map(embedText));
```

**Not filtering by similarity threshold**
```typescript
// Bad: returning results regardless of relevance
const results = await retrieveRelevantChunks(query, 5);
return results;

// Good: filter out low-relevance results
const results = await retrieveRelevantChunks(query, 10);
const relevant = results.filter(r => r.similarity > 0.75);
if (relevant.length === 0) {
  return "I don't have relevant information for this question.";
}
```

**Using the same model for embeddings and generation**
```typescript
// Wasteful: claude-sonnet for embeddings (not an embedding model)
// Use text-embedding-3-small or voyage-3 for embeddings
// Use claude-sonnet for generation
```

**Not including the source in the prompt**
```typescript
// Bad: model can't cite sources
const context = chunks.join('\n\n');

// Good: numbered, citable
const context = chunks.map((c, i) => `[${i + 1}] ${c}`).join('\n\n');
system: "Cite sources using [N] notation when making claims."
```

**Chunking in the middle of sentences**
Character-based fixed chunking can split mid-sentence. Always use sentence-aware or paragraph-aware chunking.

## Debugging & Troubleshooting

### Poor Retrieval Quality

```typescript
// Debug: print similarity scores for a query
async function debugRetrieval(query: string) {
  const queryEmbedding = await embedText(query);

  const results = await db.execute(sql`
    SELECT content,
           1 - (embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS similarity
    FROM documents
    ORDER BY similarity DESC
    LIMIT 10
  `);

  console.log(`\nTop 10 results for: "${query}"`);
  for (const row of results.rows as any[]) {
    console.log(`  ${row.similarity.toFixed(3)}: ${row.content.slice(0, 100)}...`);
  }
}

// Common issues:
// All similarities < 0.5: query doesn't match document vocabulary → try query expansion
// Top result is wrong: chunk too large (diluted embedding) → reduce chunk size
// Missing obvious result: document not indexed → check indexing pipeline
```

### Checking Index Coverage

```typescript
async function checkIndexCoverage() {
  const counts = await db.execute(sql`
    SELECT
      COUNT(*) as total_chunks,
      COUNT(embedding) as indexed_chunks,
      COUNT(*) - COUNT(embedding) as missing_embeddings
    FROM documents
  `);
  console.log(counts.rows[0]);
}
```

## Real-World Scenarios

### Scenario 1: Documentation Assistant

```typescript
// Index markdown documentation
import { readdir, readFile } from 'fs/promises';
import { join } from 'path';

async function indexDocumentation(docsDir: string) {
  const files = await readdir(docsDir, { recursive: true });
  const mdFiles = files.filter((f) => f.toString().endsWith('.md'));

  for (const file of mdFiles) {
    const content = await readFile(join(docsDir, file.toString()), 'utf-8');
    const chunks = chunkMarkdown(content);

    for (let i = 0; i < chunks.length; i++) {
      const embedding = await embedText(chunks[i]);
      await db.insert(documents).values({
        content: chunks[i],
        metadata: { source: file.toString(), chunkIndex: i, type: 'documentation' },
        embedding,
      });
    }
    console.log(`Indexed: ${file} (${chunks.length} chunks)`);
  }
}

// Question answering endpoint
app.post('/ask', async (req, res) => {
  const { question } = req.body;
  const answer = await answerWithRAG(question);
  res.json({ answer });
});
```

### Scenario 2: Semantic Product Search

```typescript
// Unlike keyword search, semantic search understands intent
// "comfortable shoes for long walks" → finds "cushioned athletic footwear"

async function semanticProductSearch(query: string) {
  const queryEmbedding = await embedText(query);

  const results = await db.execute(sql`
    SELECT
      p.id,
      p.name,
      p.description,
      p.price,
      1 - (p.embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS relevance
    FROM products p
    WHERE p.in_stock = true
    ORDER BY p.embedding <=> ${JSON.stringify(queryEmbedding)}::vector
    LIMIT 10
  `);

  return (results.rows as any[])
    .filter((p) => p.relevance > 0.65)
    .map(({ id, name, description, price, relevance }) => ({
      id, name, description, price,
      relevance: parseFloat(relevance.toFixed(3)),
    }));
}
```

## Further Reading

- [pgvector GitHub](https://github.com/pgvector/pgvector) — installation, index types, performance tuning
- [Pinecone Documentation](https://docs.pinecone.io/) — fully managed vector DB
- [Voyage AI Embeddings](https://docs.voyageai.com/docs/embeddings) — retrieval-optimized embeddings
- [LangChain Text Splitters](https://python.langchain.com/docs/modules/data_connection/document_transformers/) — many chunking strategies
- [A Survey of RAG Techniques (arXiv)](https://arxiv.org/abs/2312.10997) — comprehensive academic survey

## Summary

RAG enriches LLM prompts with retrieved context, solving knowledge cutoff, hallucination, and context size limitations.

Pipeline:
1. **Chunk** — split documents into 256-768 token pieces with 10-20% overlap
2. **Embed** — convert each chunk to a vector using an embedding model (text-embedding-3-small, voyage-3)
3. **Store** — save chunk + vector in a vector database (pgvector for PostgreSQL, Pinecone for managed)
4. **Retrieve** — at query time, embed the query and find the K most similar chunks
5. **Generate** — include retrieved chunks in the LLM prompt as grounding context

Quality improvements:
- **Hybrid search** — combine vector similarity with BM25 full-text search (RRF fusion)
- **Reranking** — use a cross-encoder or LLM to rescore and reorder retrieved chunks
- **Query expansion** — generate alternative phrasings to broaden retrieval
- **Metadata filtering** — pre-filter by category, date, or other attributes

Key numbers:
- Filter out chunks with similarity < 0.70 for most use cases
- Use 5-10 retrieved chunks per prompt
- text-embedding-3-small: $0.02/1M tokens — very cheap to run at scale
