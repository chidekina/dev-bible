# Fundamentos de LLM

## Visão Geral

Large Language Models (LLMs) são redes neurais treinadas em enormes corpora de texto para prever o próximo token em uma sequência. Esse objetivo de treinamento simples produz modelos capazes de seguir instruções complexas, raciocinar sobre problemas, gerar código e muito mais. Mas entender os mecanismos — como os tokens funcionam, o que limita o context window, o que a temperature controla, por que as alucinações acontecem — é essencial para construir aplicações confiáveis sobre eles.

Este capítulo cobre os conceitos fundamentais que todo desenvolvedor precisa antes de integrar APIs de LLM em sistemas de produção.

## Pré-requisitos

- Python ou TypeScript básico
- Familiaridade com REST APIs
- Entendimento conceitual de machine learning (não obrigatório, mas útil)

## Conceitos Fundamentais

### Tokens

LLMs não leem palavras — eles leem **tokens**. Um token é um pedaço de texto, com média de 3 a 4 caracteres no inglês. A tokenização é o primeiro passo no processamento de qualquer texto.

Regras práticas:
- 1 token ≈ 4 caracteres de texto em inglês
- 1 token ≈ 0,75 palavras
- 100 tokens ≈ 75 palavras
- 1 página de texto ≈ 700-800 tokens
- Código é mais denso em tokens do que prosa

```typescript
// Contando tokens antes de enviar para a API
// Use a biblioteca tiktoken (tokenizador da OpenAI) ou @anthropic-ai/tokenizer

import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

// Claude: use a API countTokens
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

Modelos diferentes usam tokenizadores diferentes (BPE, SentencePiece, etc.). A contagem de tokens para o mesmo texto varia entre modelos — sempre use o tokenizador específico do modelo.

Por que os tokens importam:
- **Precificação** — APIs cobram por token (entrada e saída separadamente)
- **Context window** — o total de tokens no contexto é limitado
- **Latência** — mais tokens = mais tempo para gerar
- **Atenção** — o modelo presta atenção a todos os tokens; contextos muito longos podem diluir a atenção em partes específicas

### Context Window

O **context window** é o número total de tokens que um modelo pode processar em uma requisição — a soma dos tokens de entrada (mensagens de sistema + usuário) e saída.

| Modelo | Context Window |
|--------|----------------|
| GPT-4o | 128K tokens |
| Claude Opus 4.6 | 200K tokens |
| Claude Sonnet 4.6 | 200K tokens |
| Gemini 1.5 Pro | 1M tokens |
| GPT-4o mini | 128K tokens |

Quando sua entrada se aproxima do limite do context:
1. A API retorna um erro (comprimento de contexto excedido)
2. O modelo pode truncar partes mais antigas da conversa
3. A qualidade degrada perto do limite — o modelo "perde" informação do meio de contextos muito longos

Estratégia para o context window:
```typescript
const MAX_CONTEXT_TOKENS = 180_000;  // deixa margem abaixo do limite do modelo
const MAX_OUTPUT_TOKENS = 8_000;
const MAX_INPUT_TOKENS = MAX_CONTEXT_TOKENS - MAX_OUTPUT_TOKENS;

async function safeChat(messages: Message[]): Promise<string> {
  // Estima a contagem de tokens
  const tokenCount = await countTokens(JSON.stringify(messages));

  if (tokenCount > MAX_INPUT_TOKENS) {
    // Trunca as mensagens mais antigas (mantém a mensagem de sistema + contexto recente)
    const truncated = truncateMessages(messages, MAX_INPUT_TOKENS);
    return chat(truncated);
  }

  return chat(messages);
}
```

### Papéis das Mensagens

As APIs de LLM usam um formato de conversa estruturado com três papéis:

**System** — Instruções, persona e restrições definidas pelo desenvolvedor. Processado primeiro; tem "autoridade" maior do que as mensagens do usuário. Nem todos os modelos tratam os system prompts da mesma forma.

**User** — A mensagem do humano. Entrada para o modelo.

**Assistant** — As respostas anteriores do modelo. Incluí-las na conversa habilita o diálogo multi-turno.

```typescript
const messages = [
  {
    role: 'system',
    content: `Você é um engenheiro TypeScript sênior.
Suas respostas devem:
- Usar TypeScript com strict mode
- Incluir tratamento de erros com Result types
- Ser concisas e diretas
- Não incluir ressalvas ou avisos desnecessários`,
  },
  {
    role: 'user',
    content: 'Escreva uma função que busca um usuário por ID no banco de dados.',
  },
  // Se estiver continuando uma conversa, adicione a resposta anterior do assistente:
  // { role: 'assistant', content: '...' },
  // { role: 'user', content: 'Adicione suporte a paginação' },
];
```

### Temperature e Parâmetros de Sampling

**Temperature** controla a aleatoriedade na seleção de tokens. O modelo computa uma distribuição de probabilidade sobre todos os possíveis próximos tokens; a temperature escala essa distribuição.

| Temperature | Comportamento | Caso de Uso |
|-------------|---------------|-------------|
| 0 | Determinístico (sempre escolhe o token de maior probabilidade) | Geração de código, extração de dados, classificação |
| 0,1-0,3 | Muito focado | Output estruturado, respostas precisas |
| 0,5-0,7 | Equilibrado | Q&A geral, análise, sumarização |
| 0,8-1,0 | Criativo, variado | Brainstorming, escrita criativa, opções diversas |
| >1,0 | Muito aleatório, frequentemente incoerente | Raramente útil |

**top_p** (nucleus sampling) — amostra apenas dos tokens cuja probabilidade cumulativa excede top_p. Alternativa à temperature; muitos usam apenas a temperature.

**top_k** — considera apenas os K tokens mais prováveis em cada etapa. Menos comumente exposto.

**max_tokens** — número máximo de tokens de saída. O modelo para quando termina OU quando atinge esse limite. Uma resposta truncada é frequentemente pior do que nenhuma resposta — defina esse valor alto o suficiente.

```typescript
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  temperature: 0,        // determinístico para extração de código
  messages: [
    { role: 'user', content: 'Extraia todos os comentários TODO deste código como JSON' },
  ],
});
```

### Por Que as Alucinações Acontecem

LLMs alucinam — geram informações que soam plausíveis mas são factualmente incorretas — porque:

1. **O objetivo de treinamento é prever o próximo token**, não verificar a verdade. Uma resposta errada dita com confiança e uma resposta certa dita com confiança têm a mesma aparência para a função de perda.

2. **Sem acesso a conhecimento externo** (a menos que ferramentas sejam fornecidas). O "conhecimento" do modelo é congelado no momento do treinamento e codificado nos pesos, não recuperado de fontes autoritativas.

3. **Limitação de atenção** — em contextos muito longos, a atenção do modelo a fatos específicos pode ser diluída. Quanto mais distante um fato estiver da posição atual de geração, maior a chance de ser lembrado incorretamente.

4. **Tensão entre seguir instruções e veracidade** — se você pedir ao modelo uma resposta e ele não souber a resposta, frequentemente tentará satisfazer a instrução em vez de dizer "não sei".

Estratégias de mitigação:
- **RAG** — recupera fatos relevantes de fontes autoritativas e os inclui no prompt
- **Grounding** — inclua documentos-fonte e peça ao modelo que os cite
- **Temperature baixa** — reduz a criatividade, mas também as alucinações
- **Permissão explícita de "não sei"** — "Se você não souber a resposta, diga que não sabe. Não adivinhe."
- **Etapas de verificação** — peça ao modelo que verifique seu próprio trabalho, ou use uma segunda chamada de LLM para verificar
- **Output estruturado** — formatos de saída restritos reduzem a superfície de alucinações

### Prompt Caching

LLMs modernos suportam **prompt caching** — o KV cache para um longo prefixo estático é salvo no servidor, fazendo com que requisições repetidas com o mesmo prefixo sejam mais baratas e rápidas.

```typescript
// Anthropic: marque prefixos longos e estáticos para caching
const response = await anthropic.messages.create({
  model: 'claude-opus-4-6',
  max_tokens: 1024,
  system: [
    {
      type: 'text',
      text: '...seu system prompt inteiro de 10.000 tokens...',
      cache_control: { type: 'ephemeral' },  // armazena esse prefixo em cache
    },
  ],
  messages: [{ role: 'user', content: userMessage }],
});

// Verifica o uso de cache na resposta
console.log(response.usage);
// { input_tokens: 1000, cache_creation_input_tokens: 10000, cache_read_input_tokens: 0 }
// Nas chamadas seguintes:
// { input_tokens: 1000, cache_creation_input_tokens: 0, cache_read_input_tokens: 10000 }
// Leituras de cache custam ~90% menos do que tokens de entrada completos
```

## Exemplos Práticos

### Chamada Básica à API com TypeScript

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,  // nunca hardcode
});

async function chat(userMessage: string): Promise<string> {
  const message = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: userMessage }],
  });

  // Extrai o texto da resposta
  const content = message.content[0];
  if (content.type !== 'text') {
    throw new Error(`Tipo de resposta inesperado: ${content.type}`);
  }
  return content.text;
}

// Uso
const reply = await chat('Explique o que é um context window em 2 frases.');
console.log(reply);
```

### Conversa Multi-Turno

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

// Uso
const conv = new Conversation('Você é um assistente de programação útil.');
console.log(await conv.send('Como faço para inverter uma string em TypeScript?'));
console.log(await conv.send('Pode me mostrar o mesmo em Python?'));
```

### Comparação de Temperature

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

await compareTemperatures('Descreva uma cor em uma frase.');
// Observe: temp=0 dá a mesma resposta toda vez
// temp=1.0 dá respostas variadas, às vezes criativas
```

### Calculando o Custo Antes de uma Requisição

```typescript
interface CostEstimate {
  inputTokens: number;
  maxOutputTokens: number;
  estimatedCostUSD: number;
}

// Precificação do Claude Sonnet 4.6 (em 2025)
const PRICING = {
  'claude-sonnet-4-6': {
    inputPer1M: 3.00,    // USD por 1M tokens de entrada
    outputPer1M: 15.00,  // USD por 1M tokens de saída
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

## Padrões Comuns e Boas Práticas

### Escolhendo o Modelo Certo

| Caso de Uso | Modelo Recomendado | Motivo |
|-------------|-------------------|--------|
| Classificação simples, extração | Haiku / GPT-4o mini | Rápido, barato, suficiente para tarefas estruturadas |
| Geração de código, raciocínio complexo | Sonnet / GPT-4o | Equilíbrio entre qualidade e custo |
| Raciocínio multi-etapa, pesquisa | Opus / o1 | Capacidade máxima |
| Chat em tempo real (streaming) | Sonnet | Velocidade + qualidade |
| Processamento em lote de grandes volumes | Haiku | Eficiência de custo |

### Tratamento de Erros

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
          throw new Error(`Requisição inválida: ${err.message}`);
        case 401:
          throw new Error('Chave de API inválida');
        case 429:
          // Rate limit — implementar exponential backoff
          await new Promise((r) => setTimeout(r, 5000));
          return robustChat(message); // tenta novamente uma vez
        case 500:
        case 529:
          // Servidor sobrecarregado — tenta novamente com backoff
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

### Rate Limiting e Retentativas

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
        console.warn(`Tentativa ${error.attemptNumber} falhou: ${error.message}`);
      },
    }
  );
}
```

## Anti-Padrões a Evitar

**Hardcoding de chaves de API**
```typescript
// NUNCA faça isso
const client = new Anthropic({ apiKey: 'sk-ant-...' });

// Sempre use variáveis de ambiente
const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
```

**Ignorar max_tokens**
```typescript
// Ruim: se o modelo parar no meio da resposta, você recebe output truncado silenciosamente
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 100,  // muito baixo para tarefas complexas
  messages: [...],
});
// Verifique: response.stop_reason === 'max_tokens' significa truncado
if (response.stop_reason === 'max_tokens') {
  console.warn('Resposta foi truncada');
}
```

**Não contabilizar tokens de saída nas estimativas de custo**
Tokens de saída custam 3 a 15 vezes mais do que tokens de entrada por 1M. Sempre inclua max_tokens nas estimativas de custo.

**Tratar o output do LLM como entrada confiável**
```typescript
// Ruim: executar código ou SQL gerado pelo LLM diretamente
const code = await chat('Escreva SQL para obter todos os usuários');
await db.raw(code);  // prompt injection → SQL injection

// Bom: valide e sanitize todo output do LLM
// Use output estruturado + validação de schema em vez de strings brutas
```

**Definir temperature = 1 em produção**
Temperature alta é divertida para demos, mas não confiável em produção. Use ≤ 0,3 para qualquer coisa que precise de consistência.

## Depuração e Resolução de Problemas

### Depurando Problemas de Prompt

```typescript
// Registre a requisição e resposta completas para depuração
async function debugChat(message: string) {
  console.log('REQUISIÇÃO:', JSON.stringify({ message }, null, 2));

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: message }],
  });

  console.log('RESPOSTA:', JSON.stringify(response, null, 2));
  console.log('USO:', response.usage);
  console.log('MOTIVO DE PARADA:', response.stop_reason);
  // stop_reason: 'end_turn' = normal, 'max_tokens' = truncado, 'stop_sequence' = atingiu seq de parada
}
```

### Depurando Context Window

```bash
# Erro: "prompt is too long"
# Solução: conte os tokens antes de enviar
const tokens = await countTokens(systemPrompt + userMessage);
console.log(`Total de tokens de entrada: ${tokens}`);
# Máximo típico para Claude: 200K tokens
# Verifique os limites do modelo na documentação da API
```

### Problemas de Qualidade nas Respostas

Se as respostas forem ruins:
1. **Muito verbosas**: adicione "Seja conciso" ou "Responda em 2 frases"
2. **Formato errado**: use output estruturado (veja `prompt-engineering.md`)
3. **Alucinando**: adicione "Mencione apenas fatos que você tem certeza" ou use RAG
4. **Fora do tema**: refine o system prompt, adicione exemplos negativos
5. **Inconsistente**: reduza a temperature

## Cenários do Mundo Real

### Cenário 1: Pipeline de Classificação de Texto

```typescript
type Sentiment = 'positive' | 'negative' | 'neutral';

async function classifySentiment(text: string): Promise<Sentiment> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5',    // rápido + barato para classificação
    max_tokens: 10,               // apenas uma palavra necessária
    temperature: 0,               // determinístico
    system: 'Classifique o sentimento do texto fornecido. Responda com exatamente uma palavra: positive, negative, ou neutral.',
    messages: [{ role: 'user', content: text }],
  });

  const result = (response.content[0] as Anthropic.TextBlock).text
    .trim()
    .toLowerCase() as Sentiment;

  if (!['positive', 'negative', 'neutral'].includes(result)) {
    throw new Error(`Classificação inesperada: ${result}`);
  }

  return result;
}

// Classificação em lote
async function classifyBatch(texts: string[]): Promise<Sentiment[]> {
  return Promise.all(texts.map((t) => classifySentiment(t)));
}
```

### Cenário 2: Streaming para UX em Tempo Real

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

// Uso no Express
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

## Leituras Complementares

- [Referência da API Anthropic](https://docs.anthropic.com/en/api/getting-started)
- [Referência da API OpenAI](https://platform.openai.com/docs/api-reference)
- [Guia de Prompt Engineering da Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Lilian Weng: Large Language Model](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family-v2/) — embasamento técnico aprofundado
- [Tokenizer Playground — Anthropic](https://console.anthropic.com/tokenizer)

## Resumo

LLMs são preditores do próximo token treinados em texto. Para desenvolvedores, os conceitos críticos são:

- **Tokens** — a unidade de precificação, latência e limites de contexto (~4 chars por token, ~750 tokens por página)
- **Context window** — orçamento total de tokens de entrada + saída; a qualidade degrada perto dos limites
- **Papéis** — system (instruções), user (input humano), assistant (turnos anteriores do modelo)
- **Temperature** — 0 para tarefas determinísticas (código, extração); 0,5-0,7 para conversacional; ≥ 0,8 para criativo
- **max_tokens** — sempre defina explicitamente; verifique `stop_reason === 'max_tokens'` para truncamento
- **Alucinações** — inerentes à predição do próximo token; mitigue com RAG, grounding, temperature baixa e instruções explícitas de "diga que não sabe"
- **Prompt caching** — armazene em cache prefixos estáticos longos para 90% de redução de custo em chamadas repetidas

Sempre valide e sanitize o output do LLM. Nunca o trate como entrada confiável para sistemas downstream.
