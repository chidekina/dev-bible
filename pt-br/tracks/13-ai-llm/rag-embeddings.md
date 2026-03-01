# RAG & Embeddings

## Visão Geral

Retrieval-Augmented Generation (RAG) é o padrão de enriquecer um prompt de LLM com informações relevantes recuperadas de uma base de conhecimento externa. Em vez de torcer para que o modelo "lembre" fatos do treinamento, você recupera os fatos no momento da consulta e os inclui no contexto.

RAG resolve as limitações fundamentais dos LLMs: corte de conhecimento (dados de treinamento são congelados), alucinações em fatos específicos de domínio e limites do context window (você não pode colocar uma base de conhecimento inteira em um único prompt). Embeddings são a fundação matemática que torna possível a recuperação baseada em similaridade.

## Pré-requisitos

- Fundamentos de LLM (`llm-fundamentals.md`)
- Noções básicas de PostgreSQL (para pgvector)
- TypeScript

## Conceitos Fundamentais

### Embeddings

Um **embedding** é um vetor (array de números de ponto flutuante) que representa o significado semântico de um texto. Dois textos que significam coisas semelhantes têm embeddings semelhantes — seus vetores são próximos no espaço de embedding de alta dimensão.

```typescript
// Obtém um embedding para um trecho de texto
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function embed(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',  // 1536 dimensões, $0,02/1M tokens
    input: text,
  });
  return response.data[0].embedding;
}

// Similaridade: similaridade de cosseno (varia de -1 a 1, quanto maior mais similar)
function cosineSimilarity(a: number[], b: number[]): number {
  const dotProduct = a.reduce((sum, ai, i) => sum + ai * b[i], 0);
  const magnitudeA = Math.sqrt(a.reduce((sum, ai) => sum + ai * ai, 0));
  const magnitudeB = Math.sqrt(b.reduce((sum, bi) => sum + bi * bi, 0));
  return dotProduct / (magnitudeA * magnitudeB);
}

const v1 = await embed('Como faço para redefinir minha senha?');
const v2 = await embed('Passos para alterar suas credenciais de login');
const v3 = await embed('Melhores restaurantes em Paris');

console.log(cosineSimilarity(v1, v2));  // ~0,92 — muito similar
console.log(cosineSimilarity(v1, v3));  // ~0,15 — muito diferente
```

### Modelos de Embedding

| Modelo | Dimensões | Melhor Para | Custo |
|--------|-----------|-------------|-------|
| `text-embedding-3-small` (OpenAI) | 1536 | Uso geral, custo-eficiente | $0,02/1M |
| `text-embedding-3-large` (OpenAI) | 3072 | Maior precisão | $0,13/1M |
| `voyage-3-lite` (Voyage AI) | 512 | Otimizado para recuperação, muito rápido | $0,02/1M |
| `voyage-3` (Voyage AI) | 1024 | Otimizado para recuperação, alta qualidade | $0,06/1M |
| `nomic-embed-text` (local) | 768 | Self-hosted, sem custo por consulta | Gratuito |

Insight chave: modelos de embedding e LLMs são diferentes — um modelo de embedding barato é suficiente para recuperação. Use o LLM caro apenas para geração, não para embedding.

### Bancos de Dados Vetoriais

Um banco de dados vetorial armazena embeddings e suporta busca por **vizinhos mais próximos aproximados (ANN)** — encontrando os N vetores mais similares a um vetor de consulta.

Opções:
| Solução | Melhor Para |
|---------|-------------|
| **pgvector** (extensão do PostgreSQL) | PostgreSQL existente, < 1M vetores |
| **Pinecone** | Totalmente gerenciado, grande escala, sem ops |
| **Qdrant** | Self-hosted, alto desempenho |
| **Weaviate** | Recursos de ML integrados, busca híbrida |
| **Chroma** | Desenvolvimento local, prototipagem |

### Configuração do pgvector

pgvector adiciona um tipo `vector` e operadores de similaridade vetorial ao PostgreSQL.

```sql
-- Habilita a extensão
CREATE EXTENSION IF NOT EXISTS vector;

-- Cria uma tabela com uma coluna vector
CREATE TABLE documents (
  id          BIGSERIAL PRIMARY KEY,
  content     TEXT NOT NULL,
  metadata    JSONB DEFAULT '{}',
  embedding   vector(1536),   -- deve corresponder às dimensões do seu modelo de embedding
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Cria um índice para busca por similaridade rápida
-- HNSW: melhor desempenho para a maioria dos casos de uso
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Ou IVFFlat: menor memória, bom para datasets grandes
-- CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
--   WITH (lists = 100);
```

```typescript
// TypeScript com Drizzle ORM + pgvector
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

### Estratégias de Chunking

Você não pode embedar documentos inteiros — eles são grandes demais e diluem o sinal semântico. Você deve dividir os documentos em chunks antes de embedar.

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

// Chunking de tamanho fixo (simples, frequentemente suficiente)
function chunkFixed(text: string, chunkSize = 512, overlap = 50): Chunk[] {
  const chunks: Chunk[] = [];
  let start = 0;

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length);
    chunks.push({
      content: text.slice(start, end),
      metadata: { sourceId: 'doc', chunkIndex: chunks.length, startChar: start, endChar: end },
    });
    start += chunkSize - overlap; // sobreposição ajuda a manter contexto entre limites
  }
  return chunks;
}

// Chunking por sentenças (melhor qualidade)
function chunkBySentences(
  text: string,
  maxTokens = 400,
  overlapSentences = 1
): string[] {
  // Divide nos limites de sentença
  const sentences = text.match(/[^.!?]+[.!?]+/g) ?? [text];
  const chunks: string[] = [];
  let current: string[] = [];
  let tokenCount = 0;

  for (const sentence of sentences) {
    const sentenceTokens = Math.ceil(sentence.length / 4); // estimativa aproximada

    if (tokenCount + sentenceTokens > maxTokens && current.length > 0) {
      chunks.push(current.join(' '));
      // Mantém as últimas N sentenças como sobreposição
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

// Chunking de Markdown (para documentação)
function chunkMarkdown(markdown: string, maxTokens = 512): string[] {
  // Divide nos cabeçalhos — cada seção vira um chunk (com seu cabeçalho)
  const sections = markdown.split(/^#{1,3} .+$/m);
  const headers = markdown.match(/^#{1,3} .+$/mg) ?? [];

  return headers.map((header, i) => {
    const content = sections[i + 1] ?? '';
    return `${header}\n${content}`.trim();
  }).filter((chunk) => chunk.length > 50); // ignora seções vazias
}
```

### O Pipeline RAG

```
Ingestão de documentos:
  1. Carrega o documento (PDF, HTML, markdown, texto simples)
  2. Divide em chunks
  3. Embeda cada chunk
  4. Armazena chunk + embedding no banco vetorial

Tempo de consulta:
  1. Embeda a consulta do usuário
  2. Encontra os K chunks mais similares (busca ANN)
  3. Inclui os chunks no prompt do LLM como contexto
  4. Gera a resposta fundamentada no contexto recuperado
```

### Reranking

A recuperação de primeiro estágio (similaridade de embedding) é rápida, mas imprecisa. O reranking usa um modelo cross-encoder para pontuar pares consulta-chunk com mais precisão, e então reordena os resultados.

```typescript
import Anthropic from '@anthropic-ai/sdk';

// Reranker simples baseado em LLM
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
      content: `Consulta: "${query}"

Classifique estes ${chunks.length} trechos por relevância para a consulta. Retorne apenas os índices (base 0) dos ${topK} trechos mais relevantes, separados por vírgula.

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

// Em produção: use Cohere Rerank ou Voyage Rerank (especializados, muito mais rápidos)
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

## Exemplos Práticos

### Implementação Completa de RAG com pgvector

```typescript
// src/rag/index.ts
import { db } from '../db';
import { documents } from '../db/schema';
import { sql, desc } from 'drizzle-orm';
import OpenAI from 'openai';
import Anthropic from '@anthropic-ai/sdk';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ---- Indexação ----

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

// ---- Recuperação ----

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
    .filter((r) => r.similarity > 0.7)  // inclui apenas chunks suficientemente similares
    .map((r) => r.content);
}

// ---- Geração ----

export async function answerWithRAG(question: string): Promise<string> {
  // 1. Recupera o contexto relevante
  const chunks = await retrieveRelevantChunks(question, 5);

  if (chunks.length === 0) {
    return "Não tenho informações suficientes na minha base de conhecimento para responder a essa pergunta.";
  }

  const context = chunks.map((c, i) => `[${i + 1}] ${c}`).join('\n\n');

  // 2. Gera a resposta fundamentada no contexto
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    temperature: 0,
    system: `Você é um assistente útil. Responda perguntas baseando-se APENAS no contexto fornecido.
Se o contexto não contiver informações suficientes para responder, diga isso.
Não use conhecimento de fora do contexto fornecido.
Cite as fontes referenciando os números do contexto, ex.: "De acordo com [1]..."`,
    messages: [{
      role: 'user',
      content: `Contexto:\n${context}\n\nPergunta: ${question}`,
    }],
  });

  return (response.content[0] as Anthropic.TextBlock).text;
}
```

### Busca Híbrida (Vetorial + Texto Completo)

A busca vetorial pura perde correspondências exatas de palavras-chave. A busca híbrida combina similaridade vetorial com busca full-text (BM25) para melhor recuperação.

```typescript
// pgvector suporta busca híbrida com RRF (Reciprocal Rank Fusion)
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
             ROW_NUMBER() OVER (ORDER BY ts_rank(to_tsvector('portuguese', content), plainto_tsquery('portuguese', ${query})) DESC) AS rank
      FROM documents
      WHERE to_tsvector('portuguese', content) @@ plainto_tsquery('portuguese', ${query})
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

### Integração com Pinecone

```typescript
import { Pinecone } from '@pinecone-database/pinecone';

const pc = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
const index = pc.index('my-knowledge-base');

// Insere documentos
async function upsertToPinecone(
  id: string,
  content: string,
  metadata: Record<string, string>
) {
  const embedding = await embedText(content);
  await index.upsert([{
    id,
    values: embedding,
    metadata: { content, ...metadata },  // armazena o conteúdo nos metadados para recuperação
  }]);
}

// Consulta
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

## Padrões Comuns e Boas Práticas

### Diretrizes de Tamanho de Chunk

| Tipo de Conteúdo | Tamanho do Chunk | Sobreposição |
|-----------------|-----------------|--------------|
| Documentação técnica densa | 256-512 tokens | 10-20% |
| Prosa geral | 512-768 tokens | 10-15% |
| Código | Nível de função/classe | N/A |
| Entradas de FAQ | Par completo P&R | N/A |
| Texto jurídico/médico | 512 tokens | 15-25% |

Chunks maiores = mais contexto por recuperação, mais ruído no embedding
Chunks menores = correspondência mais precisa, menos contexto

### Filtragem por Metadados

Armazene metadados estruturados com cada chunk para pré-filtragem:

```typescript
// Indexa com metadados
await index.upsert([{
  id: `doc-${id}-chunk-${i}`,
  values: embedding,
  metadata: {
    content: chunk,
    source: 'documentation',
    category: 'api-reference',
    version: '2.4',
    language: 'pt-br',
    createdAt: new Date().toISOString(),
  },
}]);

// Filtra antes da busca por similaridade
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

### Expansão de Consulta

A consulta do usuário pode não usar as mesmas palavras dos documentos. Expanda a consulta antes de embedar:

```typescript
async function expandQuery(query: string): Promise<string[]> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5',
    max_tokens: 200,
    temperature: 0.3,
    messages: [{
      role: 'user',
      content: `Gere 3 formulações alternativas desta consulta de busca.
Consulta: "${query}"
Retorne apenas as 3 alternativas, uma por linha. Sem numeração ou marcadores.`,
    }],
  });

  const alternatives = (response.content[0] as Anthropic.TextBlock).text
    .trim()
    .split('\n')
    .filter(Boolean);

  return [query, ...alternatives];
}

// Busca com todas as variantes da consulta, mescla os resultados
async function retrieveWithExpansion(query: string, topK = 5) {
  const queries = await expandQuery(query);
  const allResults = await Promise.all(
    queries.map((q) => retrieveRelevantChunks(q, topK))
  );

  // Deduplica por conteúdo
  const seen = new Set<string>();
  const unique = allResults.flat().filter((chunk) => {
    if (seen.has(chunk)) return false;
    seen.add(chunk);
    return true;
  });

  return unique.slice(0, topK);
}
```

## Anti-Padrões a Evitar

**Embedar documentos completos sem chunking**
```typescript
// Ruim: documento de 10 páginas em um embedding = média semântica de todos os tópicos
const embedding = await embedText(entireDocument);  // perde especificidade

// Bom: faz chunking primeiro, embeda cada chunk
const chunks = chunkBySentences(document);
const embeddings = await Promise.all(chunks.map(embedText));
```

**Não filtrar por limiar de similaridade**
```typescript
// Ruim: retorna resultados independentemente da relevância
const results = await retrieveRelevantChunks(query, 5);
return results;

// Bom: filtra resultados de baixa relevância
const results = await retrieveRelevantChunks(query, 10);
const relevant = results.filter(r => r.similarity > 0.75);
if (relevant.length === 0) {
  return "Não tenho informações relevantes para esta pergunta.";
}
```

**Usar o mesmo modelo para embeddings e geração**
```typescript
// Desperdício: claude-sonnet para embeddings (não é um modelo de embedding)
// Use text-embedding-3-small ou voyage-3 para embeddings
// Use claude-sonnet para geração
```

**Não incluir a fonte no prompt**
```typescript
// Ruim: o modelo não pode citar fontes
const context = chunks.join('\n\n');

// Bom: numerado, citável
const context = chunks.map((c, i) => `[${i + 1}] ${c}`).join('\n\n');
system: "Cite as fontes usando a notação [N] ao fazer afirmações."
```

**Fazer chunking no meio de sentenças**
O chunking fixo por caracteres pode dividir no meio de uma sentença. Sempre use chunking consciente de sentenças ou parágrafos.

## Depuração e Resolução de Problemas

### Baixa Qualidade de Recuperação

```typescript
// Debug: exibe pontuações de similaridade para uma consulta
async function debugRetrieval(query: string) {
  const queryEmbedding = await embedText(query);

  const results = await db.execute(sql`
    SELECT content,
           1 - (embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS similarity
    FROM documents
    ORDER BY similarity DESC
    LIMIT 10
  `);

  console.log(`\nTop 10 resultados para: "${query}"`);
  for (const row of results.rows as any[]) {
    console.log(`  ${row.similarity.toFixed(3)}: ${row.content.slice(0, 100)}...`);
  }
}

// Problemas comuns:
// Todas as similaridades < 0,5: a consulta não corresponde ao vocabulário do documento → tente expansão de consulta
// Resultado principal está errado: chunk muito grande (embedding diluído) → reduza o tamanho do chunk
// Resultado óbvio ausente: documento não indexado → verifique o pipeline de indexação
```

### Verificando Cobertura do Índice

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

## Cenários do Mundo Real

### Cenário 1: Assistente de Documentação

```typescript
// Indexa documentação em markdown
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
    console.log(`Indexado: ${file} (${chunks.length} chunks)`);
  }
}

// Endpoint de resposta a perguntas
app.post('/ask', async (req, res) => {
  const { question } = req.body;
  const answer = await answerWithRAG(question);
  res.json({ answer });
});
```

### Cenário 2: Busca Semântica de Produtos

```typescript
// Ao contrário da busca por palavras-chave, a busca semântica entende a intenção
// "calçados confortáveis para longas caminhadas" → encontra "calçados esportivos acolchoados"

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

## Leituras Complementares

- [pgvector GitHub](https://github.com/pgvector/pgvector) — instalação, tipos de índice, ajuste de desempenho
- [Documentação do Pinecone](https://docs.pinecone.io/) — banco vetorial totalmente gerenciado
- [Embeddings da Voyage AI](https://docs.voyageai.com/docs/embeddings) — embeddings otimizados para recuperação
- [LangChain Text Splitters](https://python.langchain.com/docs/modules/data_connection/document_transformers/) — muitas estratégias de chunking
- [A Survey of RAG Techniques (arXiv)](https://arxiv.org/abs/2312.10997) — levantamento acadêmico abrangente

## Resumo

RAG enriquece prompts de LLM com contexto recuperado, resolvendo as limitações de corte de conhecimento, alucinações e tamanho de contexto.

Pipeline:
1. **Chunking** — divide documentos em pedaços de 256-768 tokens com 10-20% de sobreposição
2. **Embedding** — converte cada chunk em um vetor usando um modelo de embedding (text-embedding-3-small, voyage-3)
3. **Armazenamento** — salva chunk + vetor em um banco de dados vetorial (pgvector para PostgreSQL, Pinecone para gerenciado)
4. **Recuperação** — no momento da consulta, embeda a consulta e encontra os K chunks mais similares
5. **Geração** — inclui os chunks recuperados no prompt do LLM como contexto de fundamentação

Melhorias de qualidade:
- **Busca híbrida** — combina similaridade vetorial com busca full-text BM25 (fusão RRF)
- **Reranking** — usa um cross-encoder ou LLM para pontuar e reordenar os chunks recuperados
- **Expansão de consulta** — gera formulações alternativas para ampliar a recuperação
- **Filtragem por metadados** — pré-filtra por categoria, data ou outros atributos

Números-chave:
- Filtre chunks com similaridade < 0,70 para a maioria dos casos de uso
- Use 5 a 10 chunks recuperados por prompt
- text-embedding-3-small: $0,02/1M tokens — muito barato para rodar em escala
