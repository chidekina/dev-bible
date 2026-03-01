# Otimização de Custos com IA

## Visão Geral

Os custos de APIs de LLM podem crescer mais rápido do que qualquer outro custo de infraestrutura. Uma funcionalidade que custa $0,01 por requisição é aceitável no desenvolvimento; com 100.000 requisições por dia, isso representa $1.000/dia ou $365.000/ano. Entender de onde os custos vêm e como reduzi-los é essencial para o desenvolvimento sustentável de produtos de IA.

Este capítulo cobre contagem de tokens, prompt caching, roteamento de modelos, batching e alertas de orçamento — técnicas práticas para reduzir custos de 50 a 90% sem sacrificar qualidade.

## Pré-requisitos

- Fundamentos de LLM (tokens, context window, precificação)
- Padrões de Integração com IA (streaming, caching)
- TypeScript

## Conceitos Fundamentais

### De Onde Vêm os Custos

Os custos de LLM são determinados por duas variáveis: **contagem de tokens** e **nível do modelo**.

```
Custo = (input_tokens × preço_entrada_por_1M) + (output_tokens × preço_saída_por_1M)
```

| Modelo | Entrada ($/1M) | Saída ($/1M) | Custo Relativo |
|--------|---------------|-------------|----------------|
| GPT-4o | $2,50 | $10,00 | Alto |
| GPT-4o mini | $0,15 | $0,60 | Baixo |
| Claude Opus 4.6 | $15,00 | $75,00 | Muito Alto |
| Claude Sonnet 4.6 | $3,00 | $15,00 | Médio |
| Claude Haiku 4.5 | $0,80 | $4,00 | Baixo |

Insight chave: **tokens de saída custam 3 a 5x mais do que tokens de entrada** na maioria dos modelos. Minimize outputs verbosos.

### Contagem de Tokens

Sempre conte tokens antes de enviar requisições — especialmente para jobs em lote, onde você pode estimar o custo total antecipadamente.

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { encoding_for_model, TiktokenModel } from 'tiktoken';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Anthropic: use o endpoint countTokens da API (mais preciso)
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

// OpenAI: use tiktoken (local, sem chamada de API necessária)
function countTokensOpenAI(text: string, model: TiktokenModel = 'gpt-4o'): number {
  const enc = encoding_for_model(model);
  const count = enc.encode(text).length;
  enc.free();
  return count;
}

// Estimativa rápida (use quando velocidade importa mais do que precisão)
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);  // ≈ 4 chars por token para inglês
}

// Estimativa de custo
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
  if (!pricing) throw new Error(`Modelo desconhecido: ${model}`);
  return (inputTokens / 1_000_000) * pricing.input +
         (outputTokens / 1_000_000) * pricing.output;
}

// Exemplo: 1000 tokens de entrada, 500 de saída no Sonnet
console.log(estimateCost('claude-sonnet-4-6', 1000, 500).toFixed(6));
// $0,010500 por requisição
// Em 100K requisições/dia: $1.050/dia
```

### Prompt Caching

Prompt caching é a otimização de custo de maior impacto para aplicações com system prompts longos e repetidos. O provedor armazena os estados de atenção KV para um prefixo; requisições subsequentes que reutilizam esse prefixo pagam apenas uma fração.

**Prompt caching da Anthropic:**
- Leituras de cache custam ~10% dos tokens de entrada normais (90% de desconto)
- Escritas de cache custam 25% a mais do que o normal (custo único)
- Prefixo mínimo armazenável: 1.024 tokens
- TTL do cache: 5 minutos (ephemeral) — renovado a cada cache hit

```typescript
// Marca prefixos estáticos longos com cache_control
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  system: [
    {
      type: 'text',
      // System prompt grande (2.000+ tokens para economias de cache significativas)
      text: `Você é um agente de suporte ao cliente especialista com profundo conhecimento do nosso produto.

DOCUMENTAÇÃO DO PRODUTO:
${largeProductDocs}  // 5.000 tokens de docs

POLÍTICAS DA EMPRESA:
${companyPolicies}   // 2.000 tokens de políticas`,
      cache_control: { type: 'ephemeral' },  // armazena esse prefixo em cache
    },
  ],
  messages: [
    { role: 'user', content: userQuestion },  // dinâmico, não armazenado em cache
  ],
});

// Primeira requisição: paga o preço completo pelo prefixo do sistema + custo de escrita de cache
// Requisições subsequentes dentro de 5 minutos: paga 10% pelo prefixo em cache
// console.log(response.usage)
// { input_tokens: 200, cache_creation_input_tokens: 7000, cache_read_input_tokens: 0 }
// Próxima chamada:
// { input_tokens: 200, cache_creation_input_tokens: 0, cache_read_input_tokens: 7000 }
```

**Calculando economias de cache:**
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

  console.log(`Taxa de cache hit: ${(cacheHitRate * 100).toFixed(1)}%`);
  console.log(`Custo real: $${actualCost.toFixed(6)}`);
  console.log(`Custaria sem cache: $${wouldCostWithout.toFixed(6)}`);
  console.log(`Economia: $${savings.toFixed(6)} (${((savings/wouldCostWithout)*100).toFixed(1)}%)`);
}
```

### Roteamento de Modelos

Nem todas as tarefas precisam do modelo mais caro. Roteie diferentes tipos de requisições para o modelo mais custo-eficiente para a tarefa.

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
  // Classificação/extração simples → Haiku
  if (
    (task.type === 'classification' || task.type === 'extraction') &&
    task.inputTokens < 5_000
  ) {
    return MODELS.simple.model;
  }

  // Raciocínio complexo multi-etapa → Opus
  if (task.type === 'reasoning' && task.inputTokens > 50_000) {
    return MODELS.complex.model;
  }

  // Padrão para Sonnet no equilíbrio custo/qualidade
  return MODELS.moderate.model;
}

// Roteamento no código da aplicação
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

A Batch API da Anthropic e a Batch API da OpenAI processam requisições de forma assíncrona com 50% de redução de custo. Use para jobs onde você não precisa de respostas em tempo real.

```typescript
// Batch API da Anthropic
async function batchClassify(texts: Array<{ id: string; text: string }>) {
  // Cria requisição em lote
  const requests = texts.map((item) => ({
    custom_id: item.id,
    params: {
      model: 'claude-haiku-4-5',
      max_tokens: 20,
      temperature: 0,
      system: 'Classifique o sentimento como POSITIVE, NEGATIVE, ou NEUTRAL.',
      messages: [{ role: 'user', content: item.text }],
    },
  }));

  const batch = await anthropic.beta.messages.batches.create({
    requests: requests as any,
  });

  console.log(`ID do Batch: ${batch.id}`);

  // Faz polling até conclusão
  let batchResult = batch;
  while (batchResult.processing_status !== 'ended') {
    await new Promise((r) => setTimeout(r, 10_000)); // verifica a cada 10s
    batchResult = await anthropic.beta.messages.batches.retrieve(batch.id);
    console.log(`Status: ${batchResult.processing_status} | Contagens: ${JSON.stringify(batchResult.request_counts)}`);
  }

  // Recupera resultados
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

### Cache de Respostas

Armazene respostas de LLM em cache para inputs idênticos ou quase idênticos:

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
  // Armazena em cache apenas chamadas determinísticas
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

### Alertas de Orçamento

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

    // Define expiração
    await redis.expire(`ai:cost:daily:${today}`, 86400 * 2);   // 2 dias
    await redis.expire(`ai:cost:monthly:${month}`, 86400 * 40); // 40 dias

    // Verifica limites
    const [dailyCost, monthlyCost] = await Promise.all([
      redis.get(`ai:cost:daily:${today}`),
      redis.get(`ai:cost:monthly:${month}`),
    ]);

    const daily = parseFloat(dailyCost ?? '0');
    const monthly = parseFloat(monthlyCost ?? '0');

    if (daily > this.dailyLimitUSD * 0.8) {
      console.warn(`Orçamento diário de IA em ${((daily/this.dailyLimitUSD)*100).toFixed(0)}%: $${daily.toFixed(2)}/$${this.dailyLimitUSD}`);
    }

    if (daily > this.dailyLimitUSD) {
      throw new Error(`Orçamento diário de IA excedido: $${daily.toFixed(2)} > $${this.dailyLimitUSD}`);
    }
  }

  async getDailySpend(): Promise<number> {
    const today = new Date().toISOString().slice(0, 10);
    const val = await redis.get(`ai:cost:daily:${today}`);
    return parseFloat(val ?? '0');
  }
}

const budget = new AIBudgetTracker(100, 2000); // $100/dia, $2000/mês

// Envolve todas as chamadas de IA
async function trackedComplete(params: Anthropic.MessageCreateParamsNonStreaming): Promise<string> {
  const response = await anthropic.messages.create(params);
  await budget.track(params.model, response.usage.input_tokens, response.usage.output_tokens);
  return (response.content[0] as Anthropic.TextBlock).text;
}
```

## Exemplos Práticos

### Consulta do Dashboard de Custos

```typescript
// Agrega dados de custo do seu banco de dados
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

// Encontra funcionalidades caras
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

### Compressão de Prompts

Prompts longos custam mais. Comprima-os sem perder significado:

```typescript
// Remove espaços em branco desnecessários e markdown dos prompts
function compressPrompt(prompt: string): string {
  return prompt
    .replace(/\n{3,}/g, '\n\n')      // máximo de 2 quebras de linha consecutivas
    .replace(/[ \t]+/g, ' ')          // colapsa múltiplos espaços
    .replace(/^[ \t]+/gm, '')         // remove indentação das linhas
    .trim();
}

// Antes:
const bloated = `
Você é um assistente útil.

    Por favor, analise o seguinte texto:

    O usuário forneceu este conteúdo:

    {{content}}

    Forneça uma análise detalhada do conteúdo acima.
    Certifique-se de incluir todas as informações relevantes.
`;

// Após compressão: ~40% menos tokens
const compressed = `Você é um assistente útil. Analise:

{{content}}`;
```

### Controle de Comprimento de Saída

```typescript
// Controle o comprimento da saída especificando restrições no prompt
const systemPrompt = `Resuma em exatamente 3 tópicos.
Cada tópico: máximo de 15 palavras.
Sem preâmbulo, sem conclusão.`;

// vs. o padrão caro
const verbosePrompt = `Por favor, forneça um resumo abrangente...`;

// Meça a diferença
const verboseTokens = estimateTokens(verboseOutput);
const constrainedTokens = estimateTokens(constrainedOutput);
console.log(`Redução de tokens de saída: ${verboseTokens - constrainedTokens} tokens`);
// Em 100K requisições/mês, mesmo 200 tokens a menos = 20M tokens = $300 economizados no Sonnet
```

## Padrões Comuns e Boas Práticas

### O Processo de Auditoria de Custos

Execute esta análise mensalmente:

```typescript
async function costAudit() {
  const costs = await getDailyCosts(30);

  // 1. Encontra modelos mais caros
  const byModel = costs.reduce((acc, row) => {
    acc[row.model] = (acc[row.model] ?? 0) + row.costUSD;
    return acc;
  }, {} as Record<string, number>);
  console.log('Custo por modelo (30d):', byModel);

  // 2. Encontra funcionalidades mais caras
  const featureCosts = await getFeatureCosts();
  console.log('Funcionalidades mais caras:', featureCosts.slice(0, 5));

  // 3. Verifica taxas de cache hit
  // Se taxa de cache hit < 50% para funcionalidades de alto volume, investigue

  // 4. Verifica comprimento médio de saída por funcionalidade
  // Saídas longas em média = modelo verboso ou max_tokens muito alto

  // 5. Verifica distribuição de modelos
  // Se 80% das requisições simples de classificação usam Sonnet, roteie para Haiku
}
```

### Tabela de Tradeoff Custo vs. Qualidade

| Otimização | Redução de Custo | Impacto na Qualidade | Esforço |
|------------|-----------------|---------------------|---------|
| Haiku para tarefas simples | 70-80% | Nenhum para tarefas simples | Baixo |
| Prompt caching | 60-90% nos tokens em cache | Nenhum | Baixo |
| Cache de respostas | 100% nos cache hits | Nenhum | Médio |
| Batch API | 50% | Adiciona latência | Baixo |
| Controle de comprimento de saída | 20-60% | Mínimo | Baixo |
| Compressão de prompt | 10-30% | Mínimo | Médio |
| Fine-tuning | 50-80% | Pode melhorar | Alto |

### Pré-computando para Consultas Comuns

```typescript
// Se muitos usuários fazem as mesmas perguntas, pré-compute as respostas
const COMMON_QUERIES = [
  'Como faço para redefinir minha senha?',
  'Qual é a política de reembolso?',
  'Como cancelo minha assinatura?',
  // ... top 100 FAQs da analytics
];

async function precomputeFAQAnswers() {
  for (const query of COMMON_QUERIES) {
    const answer = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 500,
      temperature: 0,
      system: 'Responda a pergunta do usuário sobre nosso produto. Seja conciso.',
      messages: [{ role: 'user', content: query }],
    });

    const text = (answer.content[0] as Anthropic.TextBlock).text;
    await redis.set(`faq:${hashText(query)}`, text);
  }
}

// No momento da consulta: verifica respostas pré-computadas antes de chamar a API
async function answerQuestion(question: string): Promise<string> {
  const precomputed = await redis.get(`faq:${hashText(question)}`);
  if (precomputed) return precomputed;

  // Cai para a API para perguntas fora de FAQ
  return callAPI(question);
}
```

## Anti-Padrões a Evitar

**Enviar o mesmo system prompt longo sem cache**
```typescript
// Ruim: paga o preço completo pelo system prompt de 5.000 tokens em cada requisição
await anthropic.messages.create({
  system: longSystemPrompt,  // 5000 tokens, sem cache
  messages: [...],
});

// Bom: armazena a parte estática em cache
await anthropic.messages.create({
  system: [{ type: 'text', text: longSystemPrompt, cache_control: { type: 'ephemeral' } }],
  messages: [...],
});
// Economia: 90% no prefixo de 5.000 tokens para cada requisição repetida
```

**Definir max_tokens muito alto por padrão**
```typescript
// Ruim: sempre permite 4096 tokens de saída mesmo para classificações simples
max_tokens: 4096

// Bom: defina com base no comprimento esperado de saída
// Classificação: 10-50 tokens
// Respostas curtas: 100-500 tokens
// Documentos completos: 2000-4096 tokens
```

**Usar Opus para tudo**
```typescript
// Ruim: $15/1M de entrada para uma tarefa que o Haiku faz por $0,80/1M
model: 'claude-opus-4-6'  // para classificação de sentimento

// Bom: combine modelo à complexidade da tarefa
model: 'claude-haiku-4-5'  // para classificação, extração, Q&A simples
```

**Ignorar caching para jobs em lote**
```typescript
// Ruim: 10.000 requisições com o mesmo system prompt, sem cache
for (const item of items) {
  await anthropic.messages.create({ system: prompt, messages: [item] });
}

// Bom: use a Batch API (50% de desconto) + prompt caching (90% no prompt)
// Resultado: ~95% de redução de custo na parte do prompt
```

**Sem monitoramento até a fatura chegar**
Configure alertas de orçamento em 50%, 80% e 100% do seu limite diário/mensal. Descubra picos de custo em horas, não no final do mês.

## Depuração e Resolução de Problemas

### Custos Inesperadamente Altos

```bash
# 1. Verifica contagens de tokens por requisição
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

# 2. Encontra saídas inesperadamente longas
SELECT id, feature, output_tokens, created_at
FROM ai_requests
WHERE output_tokens > 3000
ORDER BY created_at DESC;

# 3. Verifica se o caching está funcionando
SELECT
  SUM(cache_read_input_tokens) / NULLIF(SUM(input_tokens + cache_read_input_tokens), 0) as cache_hit_rate
FROM ai_requests
WHERE created_at > NOW() - INTERVAL '1 day'
  AND feature = 'customer-support';
```

### Investigando Cache Misses

```typescript
// Se a taxa de cache hit estiver baixa, verifique o motivo
async function debugCacheMisses() {
  // Razões comuns:
  // 1. O system prompt muda entre requisições (templates com conteúdo dinâmico)
  console.log('Verifique: o system prompt é realmente estático entre requisições?');

  // 2. Prefixo muito curto (< 1024 tokens para cache da Anthropic)
  const promptTokens = await countInputTokens([], systemPrompt);
  console.log(`Tokens do system prompt: ${promptTokens} (precisa de 1024+ para cache)`);

  // 3. TTL do cache expirou (5 min para Anthropic)
  console.log('Verifique: as requisições estão sendo feitas dentro de 5 minutos umas das outras?');
}
```

## Cenários do Mundo Real

### Cenário 1: Suporte ao Cliente em Escala

10.000 tickets de suporte por dia. Implementação inicial: GPT-4o com system prompt de 3.000 tokens = ~$650/dia.

**Jornada de otimização:**
```
Passo 1: Prompt caching (mesmo prompt de 3.000 tokens em cada requisição)
  → System prompt: de $0,0075 para $0,00075 por requisição (-90%)
  → Total: $650 → $280/dia

Passo 2: Roteia consultas simples para Haiku (70% dos tickets são simples)
  → Haiku a $0,80/1M vs. GPT-4o a $2,50/1M de entrada
  → Total: $280 → $115/dia

Passo 3: Cache de respostas de FAQ repetidas no Redis (30% dos tickets são FAQs)
  → 3.000 requisições ignoram completamente a API
  → Total: $115 → $80/dia

Passo 4: Batch API para tickets não urgentes (fila para processar à noite)
  → 50% de desconto em 40% do volume
  → Total: $80 → $64/dia

Resultado final: $650 → $64/dia (redução de 90% nos custos)
```

### Cenário 2: Middleware de Rastreamento de Custos

```typescript
// Middleware que rastreia cada chamada de IA
export async function withCostTracking<T>(
  fn: () => Promise<{ result: T; usage: Anthropic.Usage; model: string }>,
  context: { feature: string; userId?: string }
): Promise<T> {
  const { result, usage, model } = await fn();
  const cost = estimateCost(model, usage.input_tokens, usage.output_tokens);

  // Rastreamento de custo em segundo plano (fire-and-forget)
  db.insert(aiRequests).values({
    model,
    feature: context.feature,
    userId: context.userId,
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    cacheReadTokens: usage.cache_read_input_tokens ?? 0,
    costUsd: cost,
  }).catch((err) => logger.error('Falha ao rastrear custo de IA', err));

  return result;
}

// Uso
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

## Leituras Complementares

- [Precificação da Anthropic](https://www.anthropic.com/pricing)
- [Guia de Prompt Caching da Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Batch API da Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Batch API da OpenAI](https://platform.openai.com/docs/guides/batch)
- [LiteLLM — Proxy agnóstico de modelo com rastreamento de custos](https://docs.litellm.ai/)
- [Helicone — Observabilidade de LLM com análise de custos](https://www.helicone.ai/)

## Resumo

Os custos de LLM são determinados por contagem de tokens × preço do modelo. Controle ambos para manter-se dentro do orçamento.

**Otimizações de maior impacto (em ordem):**

1. **Prompt caching** (60-90% de economia em prompts estáticos repetidos) — marque system prompts longos com `cache_control: ephemeral`. Requer prefixo de 1.024+ tokens, funciona dentro de uma janela de 5 minutos.

2. **Roteamento de modelos** (70-80% de economia em tarefas simples) — use Haiku para classificação/extração, Sonnet para tarefas equilibradas, Opus apenas para raciocínio complexo.

3. **Cache de respostas** (100% de economia nos cache hits) — armazene respostas determinísticas (temp=0) para consultas estilo FAQ no Redis com TTL apropriado.

4. **Batch API** (50% de economia) — use para jobs não em tempo real: pipelines de classificação, processamento de documentos, trabalho em lote noturno.

5. **Controle de comprimento de saída** (20-60% de economia) — especifique limites de palavras/frases/tópicos no system prompt. Tokens de saída custam 3 a 5x mais do que tokens de entrada.

6. **Compressão de prompt** (10-30% de economia) — remova espaços em branco desnecessários, instruções redundantes, formatação verbosa.

**Monitoramento:**
- Registre cada chamada de IA: modelo, feature, tokens de entrada/saída, custo
- Configure alertas de orçamento em 80% e 100% dos limites diários/mensais
- Execute auditorias mensais de custo: identifique funcionalidades caras e distribuições incorretas de modelos
- Acompanhe as taxas de cache hit; taxas baixas em funcionalidades de alto volume indicam uma lacuna de caching
