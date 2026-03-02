# AI / LLM Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Comparativo de Modelos

| Modelo | Janela de Contexto | Custo Input | Custo Output | Pontos Fortes |
|--------|-------------------|-------------|--------------|---------------|
| **Claude Opus 4.6** | 200K tokens | $15 / 1M | $75 / 1M | Raciocínio, código, documentos longos |
| **Claude Sonnet 4.6** | 200K tokens | $3 / 1M | $15 / 1M | Equilíbrio entre velocidade e qualidade |
| **Claude Haiku 4.5** | 200K tokens | $0,80 / 1M | $4 / 1M | Rápido, barato, tarefas simples |
| **GPT-4o** | 128K tokens | $2,50 / 1M | $10 / 1M | Multimodal, ecossistema amplo |
| **GPT-4o mini** | 128K tokens | $0,15 / 1M | $0,60 / 1M | Custo-benefício para tarefas GPT |
| **o3** | 200K tokens | $10 / 1M | $40 / 1M | Raciocínio profundo, matemática, código |
| **o4-mini** | 200K tokens | $1,10 / 1M | $4,40 / 1M | Raciocínio rápido, econômico |
| **Gemini 2.0 Flash** | 1M tokens | $0,10 / 1M | $0,40 / 1M | Contexto massivo, velocidade |
| **Gemini 2.5 Pro** | 2M tokens | $1,25 / 1M | $10 / 1M | Maior contexto disponível |
| **Llama 3.3 70B** | 128K tokens | varia | varia | Open-source, auto-hospedável |

> Preços aproximados de meados de 2025. Verifique na página de preços de cada provedor.

---

## Papéis das Mensagens

```json
// System — define comportamento, persona, restrições (nem todos os modelos suportam)
{ "role": "system", "content": "Você é um assistente útil. Responda de forma concisa." }

// User — entrada do usuário
{ "role": "user", "content": "Qual é a capital da França?" }

// Assistant — resposta do modelo (usado em exemplos few-shot ou histórico de conversa)
{ "role": "assistant", "content": "A capital da França é Paris." }
```

```typescript
// Conversa multi-turno
const messages = [
  { role: "user",      content: "Meu nome é Alice." },
  { role: "assistant", content: "Olá, Alice! Como posso ajudar?" },
  { role: "user",      content: "Qual é o meu nome?" },
];
```

---

## Parâmetros Principais

| Parâmetro | Tipo | Intervalo | Padrão | Efeito |
|-----------|------|-----------|--------|--------|
| `temperature` | float | 0,0–2,0 | 1,0 | Aleatoriedade. 0 = determinístico, 1+ = criativo |
| `top_p` | float | 0,0–1,0 | 1,0 | Nucleus sampling. 0,9 = top 90% da massa de probabilidade |
| `max_tokens` | int | 1–máx. do modelo | varia | Máximo de tokens na resposta |
| `stop` | string[] | — | [] | Sequências que interrompem a geração |
| `presence_penalty` | float | -2,0–2,0 | 0 | Penaliza tokens já presentes na saída |
| `frequency_penalty` | float | -2,0–2,0 | 0 | Penaliza tokens frequentes |
| `seed` | int | qualquer | — | Saída reproduzível (melhor esforço) |
| `stream` | bool | — | false | Transmitir tokens conforme são gerados |

```
Guia de temperatura:
  0,0       → Determinístico, perguntas factuais, extração de dados
  0,3–0,5   → Código, análise, resumos
  0,7–1,0   → Conversa geral, redação
  1,2–1,5   → Escrita criativa, brainstorming
  > 1,5     → Muito aleatório, geralmente não útil
```

---

## SDK da Anthropic

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Completion básica
const msg = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Explique recursão em uma frase." }],
});
console.log(msg.content[0].text);
console.log(msg.usage); // { input_tokens, output_tokens }

// System prompt
const msg = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  system: "Você é um desenvolvedor TypeScript sênior. Seja conciso.",
  messages: [{ role: "user", content: "Como tipar uma função genérica assíncrona?" }],
});

// Streaming
const stream = await client.messages.stream({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Escreva um haiku." }],
});

for await (const chunk of stream) {
  if (chunk.type === "content_block_delta") {
    process.stdout.write(chunk.delta.text);
  }
}
const finalMsg = await stream.finalMessage();

// Cache de prompt (otimização de custo para system prompts repetidos)
const msg = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: "Você é um especialista. Aqui está uma base de conhecimento de 10k palavras...",
      cache_control: { type: "ephemeral" }, // cache por 5 min
    },
  ],
  messages: [{ role: "user", content: "Faça um resumo." }],
});
```

---

## Tool Use com Anthropic

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: "get_weather",
    description: "Obter o clima atual de uma localidade.",
    input_schema: {
      type: "object",
      properties: {
        location: { type: "string", description: "Nome da cidade" },
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
  messages: [{ role: "user", content: "Como está o tempo em Paris?" }],
});

if (response.stop_reason === "tool_use") {
  const toolUse = response.content.find((b) => b.type === "tool_use");
  const result = await chamarMinhaTool(toolUse.name, toolUse.input);

  // Continuar conversa com resultado da tool
  const finalResponse = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    tools,
    messages: [
      { role: "user",      content: "Como está o tempo em Paris?" },
      { role: "assistant", content: response.content },
      { role: "user",      content: [{ type: "tool_result", tool_use_id: toolUse.id, content: result }] },
    ],
  });
}
```

---

## SDK da OpenAI

```typescript
import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// Completion básica
const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system",    content: "Você é prestativo." },
    { role: "user",      content: "Diga olá." },
  ],
});
console.log(completion.choices[0].message.content);
console.log(completion.usage); // prompt_tokens, completion_tokens, total_tokens

// Streaming
const stream = await openai.chat.completions.create({
  model: "gpt-4o",
  stream: true,
  messages: [{ role: "user", content: "Conte até 5." }],
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
      description: "Obter o clima atual de uma localidade.",
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
  messages: [{ role: "user", content: "Clima em Londres?" }],
});

const toolCall = response.choices[0].message.tool_calls?.[0];
if (toolCall) {
  const args = JSON.parse(toolCall.function.arguments);
  const result = await chamarMinhaTool(toolCall.function.name, args);

  await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
      { role: "user",      content: "Clima em Londres?" },
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

// Streaming server-side (Next.js App Router)
export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic("claude-sonnet-4-6"),
    system: "Você é prestativo.",
    messages,
  });

  return result.toDataStreamResponse();
}

// Hook client-side
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
      <button type="submit">Enviar</button>
    </form>
  );
}

// Tool use com AI SDK
import { tool } from "ai";
import { z } from "zod";

const result = await generateText({
  model: openai("gpt-4o"),
  tools: {
    getWeather: tool({
      description: "Obter clima de uma cidade",
      parameters: z.object({ city: z.string() }),
      execute: async ({ city }) => ({ temperature: 22, condition: "ensolarado" }),
    }),
  },
  maxSteps: 3,  // permitir múltiplas chamadas de tool
  prompt: "Como está o tempo em Paris?",
});
```

---

## Pipeline RAG

```
1. INGESTÃO
   Docs brutos → chunking (500-1000 tokens, 10-20% de sobreposição) → embedding → upsert no vector DB

2. CONSULTA
   Pergunta do usuário → embedding → busca por similaridade vetorial (top-k, k=3–10) → reranking (opcional)

3. GERAÇÃO
   Chunks recuperados + pergunta → montar prompt → LLM → resposta

```

```typescript
// Estratégias de chunking
// Tamanho fixo:   simples, funciona para a maioria dos casos
// Por sentença:   melhor para prosa, nltk/spacy
// Recursivo:      divide em \n\n → \n → " " → "" (padrão do LangChain)
// Semântico:      embedding + divisão onde a similaridade cai

// Vector DBs populares
// pgvector    — extensão do PostgreSQL (auto-hospedado, interface SQL)
// Pinecone    — gerenciado, rápido, caro
// Weaviate    — open-source, busca híbrida
// Qdrant      — open-source, Rust, alta performance
// Chroma      — open-source, ótimo para dev/prototipagem

// Modelos de embedding
// text-embedding-3-small  (OpenAI) — 1536 dims, barato
// text-embedding-3-large  (OpenAI) — 3072 dims, melhor qualidade
// voyage-3-large          (Voyage) — melhor para retrieval
// nomic-embed-text        (gratuito/local)

// Template de prompt para RAG
const prompt = `
Responda à pergunta usando APENAS o contexto abaixo.
Se a resposta não estiver no contexto, diga "Não sei."

<context>
${chunksRecuperados.join("\n\n---\n\n")}
</context>

<question>
${perguntaUsuario}
</question>
`;
```

---

## Padrões de Prompt

```
Zero-shot:
  "Classifique o sentimento desta avaliação: <avaliação>"

Few-shot:
  "Classifique o sentimento. Exemplos:
   'Ótimo produto!' → positivo
   'Experiência horrível.' → negativo
   Agora classifique: '<avaliação>'"

Chain of Thought (CoT):
  "Pense passo a passo antes de responder."
  "Vamos resolver isso com cuidado:"

ReAct (Raciocinar + Agir):
  "Pensar: [raciocínio]
   Agir: [chamada de ferramenta]
   Observar: [resultado da ferramenta]
   Pensar: [próximo raciocínio]
   Resposta: [resposta final]"

Tags XML (recomendado pela Anthropic):
  "<documents>
     <document index='1'>...</document>
   </documents>
   <instructions>
     Faça um resumo de cada documento.
   </instructions>"

Role prompting:
  "Você é um engenheiro de segurança sênior revisando código em busca de vulnerabilidades."

Restrições:
  "Responda em exatamente 3 pontos."
  "Retorne apenas JSON válido. Sem explicações."
  "Se não souber, diga 'DESCONHECIDO'. Nunca chute."
```

---

## Contagem de Tokens

```typescript
// Anthropic — contagem server-side
const response = await client.messages.countTokens({
  model: "claude-sonnet-4-6",
  messages: [{ role: "user", content: "Olá, mundo!" }],
});
console.log(response.input_tokens); // contagem exata

// OpenAI — tiktoken (client-side)
import { encoding_for_model } from "tiktoken";

const enc = encoding_for_model("gpt-4o");
const tokens = enc.encode("Olá, mundo!");
console.log(tokens.length);
enc.free(); // importante: liberar memória WASM

// Estimativas práticas
// 1 token ≈ 4 caracteres (inglês)
// 1 token ≈ 0,75 palavras
// 1 página ≈ 500 palavras ≈ 667 tokens
// 100K tokens ≈ 75.000 palavras ≈ livro de 150 páginas
```

---

## Otimização de Custos

| Técnica | Economia | Quando Usar |
|---------|----------|-------------|
| Cache de prompt (Anthropic) | 90% em cache hits | System prompts / docs repetidos |
| Batch API (Anthropic) | 50% | Workloads sem tempo real |
| Roteamento de modelo | 5–50× | Tarefas simples → modelos menores |
| System prompts mais curtos | ~10–30% | Auditar instruções infladas |
| Streaming | 0% custo, melhor UX | Apps interativos |
| Limitar max_tokens | varia | Evitar respostas descontroladas |
| Cache de respostas | 100% | Consultas idênticas repetidas |

```typescript
// Exemplo de roteamento de modelo
function rotearParaModelo(tarefa: string): string {
  if (tarefa === "classificacao" || tarefa === "extracao") return "claude-haiku-4-5";
  if (tarefa === "resumo" || tarefa === "qa") return "claude-sonnet-4-6";
  return "claude-opus-4-6"; // raciocínio, análise complexa
}
```

---

## Códigos de Erro Comuns e Tratamento

| Código / Tipo | Causa | Solução |
|---------------|-------|---------|
| `401 invalid_api_key` | Chave errada ou ausente | Verificar variável de ambiente |
| `403 permission_error` | Chave sem acesso ao modelo | Verificar permissões da API key |
| `429 rate_limit_error` | Muitas requisições | Backoff exponencial + retry |
| `429 tokens_limit` | Cota de tokens excedida | Verificar faturamento / upgrade |
| `500 api_error` | Erro no servidor do provedor | Retry com backoff |
| `529 overloaded_error` | Provedor sobrecarregado | Retry com jitter |
| `context_length_exceeded` | Input muito longo | Truncar / dividir input em chunks |
| `stop_reason: max_tokens` | Resposta truncada | Aumentar max_tokens |
| `stop_reason: tool_use` | Modelo quer chamar tool | Tratar loop de tool use |

```typescript
// Retry com backoff exponencial
async function comRetry<T>(fn: () => Promise<T>, tentativas = 3): Promise<T> {
  for (let tentativa = 0; tentativa < tentativas; tentativa++) {
    try {
      return await fn();
    } catch (err: any) {
      const eRetentavel = err.status === 429 || err.status === 529 || err.status >= 500;
      if (!eRetentavel || tentativa === tentativas - 1) throw err;
      const delay = Math.min(1000 * 2 ** tentativa + Math.random() * 500, 30000);
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw new Error("Inacessível");
}

const response = await comRetry(() =>
  client.messages.create({ model: "claude-sonnet-4-6", max_tokens: 1024, messages })
);
```
