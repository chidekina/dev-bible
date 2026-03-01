# Tool Use & Function Calling

## Visão Geral

Tool use (também chamado de function calling) permite que LLMs interajam com sistemas externos — bancos de dados, APIs, sistemas de arquivos, calculadoras — solicitando que seu código execute uma função definida e retorne o resultado. O modelo não executa código por si mesmo; ele declara sua intenção ("Quero chamar `searchProducts` com query='sapatos azuis'") e a sua aplicação o executa, retorna o resultado e continua a conversa.

Essa capacidade transforma os LLMs de geradores de texto estáticos em agentes que podem tomar ações, buscar informações e realizar tarefas de múltiplas etapas.

## Pré-requisitos

- Fundamentos de LLM (`llm-fundamentals.md`)
- Noções básicas de Prompt Engineering
- TypeScript

## Conceitos Fundamentais

### Como o Tool Use Funciona

```
1. Você define ferramentas (nome, descrição, schema de entrada)
2. O usuário envia uma mensagem
3. O LLM decide se vai usar uma ferramenta
4. O LLM responde com um bloco tool_use (não texto)
5. Seu código executa a ferramenta
6. Você envia o resultado da ferramenta de volta ao LLM
7. O LLM usa o resultado para gerar uma resposta final
8. Repita a partir do passo 3 se múltiplas ferramentas forem necessárias
```

```
Usuário: "Qual é o tempo em Tóquio e converta 25°C para Fahrenheit?"
  ↓
LLM: [tool_use: get_weather(city="Tokyo")]
  ↓
Código: chama API de clima → {temp: 25, conditions: "Ensolarado"}
  ↓
LLM: [tool_use: convert_temperature(celsius=25)]
  ↓
Código: 25 * 9/5 + 32 = 77
  ↓
LLM: "Tóquio está atualmente a 25°C (77°F) e ensolarado."
```

### Spec de Tool Use da Anthropic

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Define ferramentas usando JSON Schema
const tools: Anthropic.Tool[] = [
  {
    name: 'search_products',
    description: 'Busca o catálogo de produtos por itens correspondentes a uma consulta. Retorna uma lista de produtos correspondentes com IDs, nomes e preços.',
    input_schema: {
      type: 'object' as const,
      properties: {
        query: {
          type: 'string',
          description: 'Termos de busca para encontrar produtos',
        },
        category: {
          type: 'string',
          enum: ['electronics', 'clothing', 'books', 'home'],
          description: 'Filtro de categoria opcional',
        },
        maxResults: {
          type: 'number',
          description: 'Número máximo de resultados a retornar (padrão: 5)',
        },
      },
      required: ['query'],
    },
  },
  {
    name: 'get_product_details',
    description: 'Obtém informações detalhadas sobre um produto específico pelo seu ID.',
    input_schema: {
      type: 'object' as const,
      properties: {
        productId: {
          type: 'string',
          description: 'O identificador único do produto',
        },
      },
      required: ['productId'],
    },
  },
];
```

### Implementação de Ferramentas e Loop do Agente

```typescript
// Implementações das ferramentas
async function searchProducts(args: { query: string; category?: string; maxResults?: number }) {
  const results = await db.query(
    `SELECT id, name, price, category FROM products
     WHERE to_tsvector('english', name || ' ' || description) @@ plainto_tsquery($1)
     ${args.category ? 'AND category = $2' : ''}
     LIMIT $${args.category ? '3' : '2'}`,
    args.category
      ? [args.query, args.category, args.maxResults ?? 5]
      : [args.query, args.maxResults ?? 5]
  );
  return results.rows;
}

async function getProductDetails(args: { productId: string }) {
  const result = await db.query('SELECT * FROM products WHERE id = $1', [args.productId]);
  return result.rows[0] ?? null;
}

// Despacho de ferramentas
async function executeTool(
  toolName: string,
  toolInput: Record<string, unknown>
): Promise<unknown> {
  switch (toolName) {
    case 'search_products':
      return searchProducts(toolInput as any);
    case 'get_product_details':
      return getProductDetails(toolInput as any);
    default:
      throw new Error(`Ferramenta desconhecida: ${toolName}`);
  }
}

// Loop do agente
async function runAgent(userMessage: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  while (true) {
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 4096,
      tools,
      messages,
    });

    // Se o modelo terminou, retorna a resposta em texto
    if (response.stop_reason === 'end_turn') {
      const textBlock = response.content.find((b) => b.type === 'text');
      return textBlock ? (textBlock as Anthropic.TextBlock).text : '';
    }

    // Processa as chamadas de ferramentas
    if (response.stop_reason === 'tool_use') {
      // Adiciona a resposta do assistente (com blocos tool_use) ao histórico
      messages.push({ role: 'assistant', content: response.content });

      // Executa cada chamada de ferramenta
      const toolResults: Anthropic.ToolResultBlockParam[] = [];

      for (const block of response.content) {
        if (block.type !== 'tool_use') continue;

        let result: unknown;
        let isError = false;

        try {
          result = await executeTool(block.name, block.input as Record<string, unknown>);
        } catch (err) {
          result = { error: (err as Error).message };
          isError = true;
        }

        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: JSON.stringify(result),
          is_error: isError,
        });
      }

      // Envia os resultados de volta ao modelo
      messages.push({ role: 'user', content: toolResults });
    }
  }
}
```

### Spec de Function Calling da OpenAI

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const tools: OpenAI.Chat.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'search_products',
      description: 'Busca o catálogo de produtos',
      parameters: {
        type: 'object',
        properties: {
          query: { type: 'string', description: 'Termos de busca' },
          category: { type: 'string', enum: ['electronics', 'clothing', 'books'] },
        },
        required: ['query'],
      },
    },
  },
];

async function runOpenAIAgent(userMessage: string): Promise<string> {
  const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  while (true) {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o',
      tools,
      messages,
    });

    const choice = response.choices[0];

    if (choice.finish_reason === 'stop') {
      return choice.message.content ?? '';
    }

    if (choice.finish_reason === 'tool_calls') {
      messages.push(choice.message);  // adiciona a mensagem do assistente com tool_calls

      for (const toolCall of choice.message.tool_calls ?? []) {
        const args = JSON.parse(toolCall.function.arguments);
        const result = await executeTool(toolCall.function.name, args);

        messages.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: JSON.stringify(result),
        });
      }
    }
  }
}
```

### Chamadas de Ferramentas em Paralelo

Os modelos podem solicitar múltiplas chamadas de ferramentas em uma única resposta quando são independentes. Execute-as em paralelo:

```typescript
// No loop de ferramentas, em vez de execução sequencial:
const toolResults = await Promise.all(
  response.content
    .filter((b): b is Anthropic.ToolUseBlock => b.type === 'tool_use')
    .map(async (block) => {
      let result: unknown;
      let isError = false;

      try {
        result = await executeTool(block.name, block.input as Record<string, unknown>);
      } catch (err) {
        result = { error: (err as Error).message };
        isError = true;
      }

      return {
        type: 'tool_result' as const,
        tool_use_id: block.id,
        content: JSON.stringify(result),
        is_error: isError,
      };
    })
);
```

### Controle de Seleção de Ferramentas

```typescript
// Força o modelo a sempre usar ferramentas (impede respostas apenas em texto)
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  tools,
  tool_choice: { type: 'any' },  // deve usar pelo menos uma ferramenta
  messages,
});

// Força uma ferramenta específica
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  tools,
  tool_choice: { type: 'tool', name: 'search_products' },  // deve usar esta ferramenta específica
  messages,
});

// Auto (padrão): o modelo decide
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  tools,
  tool_choice: { type: 'auto' },
  messages,
});
```

## Exemplos Práticos

### Sistema de Ferramentas Tipado

```typescript
// src/tools/types.ts
export interface ToolDefinition<TInput, TOutput> {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  execute: (input: TInput) => Promise<TOutput>;
}

// src/tools/registry.ts
import { z } from 'zod';

type AnyToolDef = ToolDefinition<any, any>;

export class ToolRegistry {
  private tools = new Map<string, AnyToolDef>();

  register<TInput, TOutput>(def: ToolDefinition<TInput, TOutput>) {
    this.tools.set(def.name, def);
    return this;
  }

  getAnthropicTools(): Anthropic.Tool[] {
    return Array.from(this.tools.values()).map((t) => ({
      name: t.name,
      description: t.description,
      input_schema: t.inputSchema as Anthropic.Tool['input_schema'],
    }));
  }

  async execute(name: string, input: unknown): Promise<unknown> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Ferramenta não encontrada: ${name}`);
    return tool.execute(input);
  }
}

// src/tools/product-tools.ts
const SearchInputSchema = z.object({
  query: z.string(),
  category: z.enum(['electronics', 'clothing', 'books', 'home']).optional(),
  maxResults: z.number().min(1).max(20).default(5),
});

export const searchProductsTool: ToolDefinition<z.infer<typeof SearchInputSchema>, Product[]> = {
  name: 'search_products',
  description: 'Busca o catálogo de produtos por itens correspondentes a uma consulta.',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Termos de busca' },
      category: { type: 'string', enum: ['electronics', 'clothing', 'books', 'home'] },
      maxResults: { type: 'number', description: 'Máximo de resultados (1-20, padrão 5)' },
    },
    required: ['query'],
  },
  execute: async (rawInput) => {
    const input = SearchInputSchema.parse(rawInput);  // valida o input
    return productService.search(input.query, input.category, input.maxResults);
  },
};

// Registra ferramentas
export const registry = new ToolRegistry()
  .register(searchProductsTool)
  .register(getProductDetailsTool)
  .register(checkInventoryTool);
```

### Agente com Guarda de Iteração Máxima

```typescript
async function runSafeAgent(
  userMessage: string,
  maxIterations = 10
): Promise<{ result: string; iterations: number; toolsUsed: string[] }> {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  let iterations = 0;
  const toolsUsed: string[] = [];

  while (iterations < maxIterations) {
    iterations++;

    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 4096,
      tools: registry.getAnthropicTools(),
      messages,
    });

    if (response.stop_reason === 'end_turn') {
      const text = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === 'text')
        .map((b) => b.text)
        .join('');
      return { result: text, iterations, toolsUsed };
    }

    if (response.stop_reason !== 'tool_use') {
      throw new Error(`Motivo de parada inesperado: ${response.stop_reason}`);
    }

    messages.push({ role: 'assistant', content: response.content });

    const toolResults = await Promise.all(
      response.content
        .filter((b): b is Anthropic.ToolUseBlock => b.type === 'tool_use')
        .map(async (block) => {
          toolsUsed.push(block.name);

          try {
            const result = await registry.execute(block.name, block.input);
            return {
              type: 'tool_result' as const,
              tool_use_id: block.id,
              content: JSON.stringify(result),
            };
          } catch (err) {
            return {
              type: 'tool_result' as const,
              tool_use_id: block.id,
              content: JSON.stringify({ error: (err as Error).message }),
              is_error: true,
            };
          }
        })
    );

    messages.push({ role: 'user', content: toolResults });
  }

  throw new Error(`Agente excedeu o número máximo de iterações (${maxIterations})`);
}
```

### Streaming com Tool Use

```typescript
async function streamWithTools(
  userMessage: string,
  onText: (text: string) => void
): Promise<void> {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  while (true) {
    const stream = anthropic.messages.stream({
      model: 'claude-sonnet-4-6',
      max_tokens: 4096,
      tools: registry.getAnthropicTools(),
      messages,
    });

    const toolUseBlocks: Anthropic.ToolUseBlock[] = [];

    for await (const event of stream) {
      if (
        event.type === 'content_block_delta' &&
        event.delta.type === 'text_delta'
      ) {
        onText(event.delta.text);
      }
    }

    const finalMessage = await stream.finalMessage();

    if (finalMessage.stop_reason === 'end_turn') break;

    if (finalMessage.stop_reason === 'tool_use') {
      messages.push({ role: 'assistant', content: finalMessage.content });

      const toolResults = await Promise.all(
        finalMessage.content
          .filter((b): b is Anthropic.ToolUseBlock => b.type === 'tool_use')
          .map(async (block) => ({
            type: 'tool_result' as const,
            tool_use_id: block.id,
            content: JSON.stringify(await registry.execute(block.name, block.input)),
          }))
      );

      messages.push({ role: 'user', content: toolResults });
    }
  }
}
```

## Padrões Comuns e Boas Práticas

### Qualidade da Descrição das Ferramentas

As descrições das ferramentas fazem parte do prompt — elas afetam diretamente quando e como o modelo usa a ferramenta.

```typescript
// Ruim: descrição vaga
{
  name: 'get_data',
  description: 'Obtém dados',
  input_schema: { ... }
}

// Bom: específica, orientada a ações, descreve quando usá-la e o que retorna
{
  name: 'search_orders',
  description: `Busca pedidos de clientes por vários critérios.
Use isso quando o usuário perguntar sobre status de pedido, histórico ou rastreamento.
Retorna ID do pedido, status, itens, total e data estimada de entrega.
Para um único pedido por ID, use get_order_details.`,
  input_schema: { ... }
}
```

### Tratando Erros de Ferramentas com Elegância

```typescript
// Não lance erros das implementações de ferramentas — retorne erros estruturados
async function getOrderStatus(args: { orderId: string }) {
  const order = await db.query('SELECT * FROM orders WHERE id = $1', [args.orderId]);

  if (!order.rows.length) {
    return {
      found: false,
      error: `Nenhum pedido encontrado com o ID ${args.orderId}`,
    };
  }

  return {
    found: true,
    order: order.rows[0],
  };
}

// O modelo tratará o erro elegantemente em sua resposta:
// "Busquei o pedido #12345 mas não consegui encontrá-lo em nosso sistema."
```

### Limitando a Exposição de Ferramentas

Não dê ao modelo mais ferramentas do que o necessário para a tarefa — reduz confusão, latência e custo.

```typescript
// Seleção de ferramentas dependente de contexto
function getToolsForContext(userRole: 'customer' | 'admin' | 'readonly') {
  const baseTool = [searchProductsTool, getProductDetailsTool];

  if (userRole === 'admin') {
    return [...baseTool, updatePriceTool, deleteProductTool, viewAnalyticsTool];
  }

  if (userRole === 'customer') {
    return [...baseTool, searchOrdersTool, initiateReturnTool];
  }

  return baseTool;  // readonly: apenas busca
}
```

### Confirmação para Ações Destrutivas

```typescript
// Adicione uma ferramenta confirm_action para operações irreversíveis
const confirmActionTool: ToolDefinition<{ action: string; details: string }, { confirmed: boolean }> = {
  name: 'confirm_action',
  description: 'Solicita confirmação do usuário antes de realizar uma ação destrutiva ou irreversível.',
  inputSchema: {
    type: 'object',
    properties: {
      action: { type: 'string', description: 'Descrição legível da ação' },
      details: { type: 'string', description: 'O que será alterado ou excluído' },
    },
    required: ['action', 'details'],
  },
  execute: async (input) => {
    // Retorna um estado pendente — a confirmação real acontece na UI
    return { confirmed: false, pendingConfirmation: input };
  },
};
```

## Anti-Padrões a Evitar

**Permitir que o modelo chame ferramentas perigosas sem guardrails**
```typescript
// Ruim: o modelo pode deletar qualquer coisa
const deleteUserTool = {
  name: 'delete_user',
  execute: async ({ userId }) => await db.delete(users).where(eq(users.id, userId)),
};

// Bom: valida autorização antes de executar
const deleteUserTool = {
  name: 'delete_user',
  execute: async ({ userId }, context: { adminId: string }) => {
    if (!await isAdmin(context.adminId)) {
      throw new Error('Não autorizado: acesso de administrador necessário');
    }
    await db.delete(users).where(eq(users.id, userId));
  },
};
```

**Ignorar o limite de iteração**
Loops de agente podem rodar indefinidamente se o modelo continuar chamando ferramentas. Sempre defina um máximo.

**Não validar os inputs das ferramentas**
```typescript
// Ruim: confia diretamente no input do modelo
async function chargeCard(args: any) {
  await stripe.charge({ amount: args.amount, currency: args.currency });
}

// Bom: valida com Zod antes de executar
const ChargeSchema = z.object({
  amount: z.number().positive().max(10000),  // centavos, máximo R$100
  currency: z.enum(['usd', 'eur', 'brl']),
  customerId: z.string().regex(/^cus_/),
});

async function chargeCard(rawArgs: unknown) {
  const args = ChargeSchema.parse(rawArgs);  // lança se inválido
  await stripe.charge(args);
}
```

**Passar registros completos do banco como resultados de ferramentas**
```typescript
// Ruim: retorna 50 campos, consome muitos tokens
return await db.query('SELECT * FROM users WHERE id = $1', [id]);

// Bom: retorna apenas o necessário
const result = await db.query('SELECT id, name, email, plan FROM users WHERE id = $1', [id]);
return result.rows[0];
```

## Depuração e Resolução de Problemas

### Registrando Chamadas de Ferramentas

```typescript
async function executeWithLogging(
  name: string,
  input: unknown
): Promise<unknown> {
  const start = Date.now();
  console.log(`[TOOL] ${name}`, JSON.stringify(input));

  try {
    const result = await registry.execute(name, input);
    console.log(`[TOOL] ${name} concluído em ${Date.now() - start}ms`, JSON.stringify(result).slice(0, 200));
    return result;
  } catch (err) {
    console.error(`[TOOL] ${name} falhou:`, (err as Error).message);
    throw err;
  }
}
```

### Quando o Modelo Não Usa Ferramentas

1. **Melhore as descrições** — seja mais explícito sobre quando a ferramenta deve ser usada
2. **Defina `tool_choice: 'any'`** — force o uso de ferramentas para depuração
3. **Verifique o schema da ferramenta** — JSON Schema malformado pode fazer o modelo ignorar a ferramenta
4. **Simplifique o schema** — schemas aninhados complexos confundem os modelos; simplifique quando possível

### Quando o Modelo Entra em Loop Infinito

1. Adicione guarda de iteração máxima
2. Verifique se os resultados das ferramentas estão sendo retornados corretamente — o modelo pode não estar recebendo os resultados
3. Verifique `is_error: true` — o modelo pode estar preso tentando novamente uma ferramenta quebrada
4. Adicione uma ferramenta `finish_task` que o modelo deve chamar para concluir:

```typescript
const finishTaskTool = {
  name: 'finish_task',
  description: 'Chame isso quando tiver coletado todas as informações necessárias e estiver pronto para responder ao usuário.',
  input_schema: {
    type: 'object' as const,
    properties: {
      summary: { type: 'string', description: 'Resumo das descobertas para compartilhar com o usuário' },
    },
    required: ['summary'],
  },
};
```

## Cenários do Mundo Real

### Cenário 1: Agente de Suporte ao Cliente

```typescript
const supportTools = [
  {
    name: 'lookup_order',
    description: 'Busca um pedido de cliente por ID ou e-mail. Retorna status do pedido, itens, rastreamento e estimativa de entrega.',
    input_schema: {
      type: 'object' as const,
      properties: {
        orderId: { type: 'string', description: 'ID do pedido (ex.: PED-12345)' },
        customerEmail: { type: 'string', description: 'E-mail do cliente (alternativa ao orderId)' },
      },
    },
  },
  {
    name: 'initiate_refund',
    description: 'Inicia um reembolso para um pedido. Use apenas após confirmar com o cliente e verificar que o pedido é elegível.',
    input_schema: {
      type: 'object' as const,
      properties: {
        orderId: { type: 'string' },
        reason: { type: 'string', enum: ['defective', 'not_received', 'wrong_item', 'changed_mind'] },
        amount: { type: 'number', description: 'Valor do reembolso em centavos' },
      },
      required: ['orderId', 'reason', 'amount'],
    },
  },
  {
    name: 'create_support_ticket',
    description: 'Cria um ticket de suporte para problemas que precisam de revisão humana.',
    input_schema: {
      type: 'object' as const,
      properties: {
        subject: { type: 'string' },
        description: { type: 'string' },
        priority: { type: 'string', enum: ['low', 'medium', 'high', 'urgent'] },
        orderId: { type: 'string' },
      },
      required: ['subject', 'description', 'priority'],
    },
  },
];

async function customerSupportAgent(customerMessage: string, customerId: string) {
  return runSafeAgent(
    customerMessage,
    10,
    // Injeta contexto
    `Você é um agente de suporte ao cliente prestativo da Loja Acme.
ID do cliente: ${customerId}
Data atual: ${new Date().toISOString()}
Sempre verifique a propriedade do pedido antes de compartilhar detalhes ou processar reembolsos.`
  );
}
```

### Cenário 2: Agente de Análise de Dados

```typescript
const analysisTools = [
  {
    name: 'query_database',
    description: 'Executa uma consulta SQL no banco de dados analítico para recuperar dados.',
    input_schema: {
      type: 'object' as const,
      properties: {
        sql: { type: 'string', description: 'Consulta SQL (apenas SELECT, sem mutações)' },
        limit: { type: 'number', description: 'Máximo de linhas a retornar (padrão 100)' },
      },
      required: ['sql'],
    },
  },
  {
    name: 'calculate_stats',
    description: 'Calcula estatísticas descritivas (média, mediana, desvio padrão, percentis) para um dataset.',
    input_schema: {
      type: 'object' as const,
      properties: {
        data: {
          type: 'array',
          items: { type: 'number' },
          description: 'Array de números para analisar',
        },
      },
      required: ['data'],
    },
  },
];

// Executa com prevenção de SQL injection
async function executeQueryTool(args: { sql: string; limit?: number }) {
  // Valida que é apenas uma instrução SELECT
  const normalized = args.sql.trim().toUpperCase();
  if (!normalized.startsWith('SELECT')) {
    throw new Error('Apenas consultas SELECT são permitidas');
  }

  // Adiciona LIMIT se não estiver presente
  const query = normalized.includes('LIMIT')
    ? args.sql
    : `${args.sql} LIMIT ${args.limit ?? 100}`;

  const result = await analyticsDb.query(query);
  return { rowCount: result.rowCount, rows: result.rows };
}
```

## Leituras Complementares

- [Documentação de Tool Use da Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [Guia de Function Calling da OpenAI](https://platform.openai.com/docs/guides/function-calling)
- [Building Effective Agents (Anthropic)](https://www.anthropic.com/research/building-effective-agents)
- [ReAct: Reasoning + Acting](https://arxiv.org/abs/2210.03629) — artigo fundacional sobre agentes que usam ferramentas
- [LangChain Agents](https://python.langchain.com/docs/modules/agents/) — framework de agentes de nível mais alto

## Resumo

Tool use transforma LLMs de geradores de texto em agentes que podem tomar ações.

O padrão:
1. Defina ferramentas com nome, descrição e definição de entrada em JSON Schema
2. Envie mensagens com ferramentas ao LLM
3. Quando `stop_reason === 'tool_use'`, execute as ferramentas solicitadas em paralelo
4. Envie os resultados de volta e repita até `stop_reason === 'end_turn'`
5. Sempre defina um limite máximo de iterações (10 é um bom padrão)

Checklist de implementação em TypeScript:
- Valide todos os inputs das ferramentas com Zod antes de executar
- Retorne erros estruturados (não lance exceções) para o modelo se recuperar graciosamente
- Retorne apenas os campos necessários das ferramentas — minimize o consumo de tokens
- Registre todas as chamadas e resultados de ferramentas para depuração
- Escope as ferramentas pelo contexto — dê ao modelo apenas as ferramentas relevantes para a tarefa atual
- Adicione etapas de confirmação antes de chamadas de ferramentas destrutivas ou irreversíveis
- Use `Promise.all` para chamadas de ferramentas em paralelo quando o modelo solicita múltiplas ferramentas de uma vez

A qualidade da descrição das ferramentas importa tanto quanto a qualidade da implementação — uma ferramenta bem descrita é usada corretamente; uma vaga é mal usada ou ignorada.
