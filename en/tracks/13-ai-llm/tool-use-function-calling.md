# Tool Use & Function Calling

## Overview

Tool use (also called function calling) lets LLMs interact with external systems — databases, APIs, file systems, calculators — by requesting that your code execute a defined function and return the result. The model doesn't execute code itself; it declares its intent ("I want to call `searchProducts` with query='blue shoes'") and your application executes it, returns the result, and continues the conversation.

This capability transforms LLMs from static text generators into agents that can take actions, look up information, and perform multi-step tasks.

## Prerequisites

- LLM Fundamentals (`llm-fundamentals.md`)
- Prompt Engineering basics
- TypeScript

## Core Concepts

### How Tool Use Works

```
1. You define tools (name, description, input schema)
2. User sends a message
3. LLM decides whether to use a tool
4. LLM responds with a tool_use block (not text)
5. Your code executes the tool
6. You send the tool result back to the LLM
7. LLM uses the result to generate a final response
8. Repeat from step 3 if multiple tools are needed
```

```
User: "What's the weather in Tokyo and convert 25°C to Fahrenheit?"
  ↓
LLM: [tool_use: get_weather(city="Tokyo")]
  ↓
Code: calls weather API → {temp: 25, conditions: "Sunny"}
  ↓
LLM: [tool_use: convert_temperature(celsius=25)]
  ↓
Code: 25 * 9/5 + 32 = 77
  ↓
LLM: "Tokyo is currently 25°C (77°F) and sunny."
```

### Anthropic Tool Use Spec

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Define tools using JSON Schema
const tools: Anthropic.Tool[] = [
  {
    name: 'search_products',
    description: 'Search the product catalog for items matching a query. Returns a list of matching products with IDs, names, and prices.',
    input_schema: {
      type: 'object' as const,
      properties: {
        query: {
          type: 'string',
          description: 'Search terms for finding products',
        },
        category: {
          type: 'string',
          enum: ['electronics', 'clothing', 'books', 'home'],
          description: 'Optional category filter',
        },
        maxResults: {
          type: 'number',
          description: 'Maximum number of results to return (default: 5)',
        },
      },
      required: ['query'],
    },
  },
  {
    name: 'get_product_details',
    description: 'Get detailed information about a specific product by its ID.',
    input_schema: {
      type: 'object' as const,
      properties: {
        productId: {
          type: 'string',
          description: 'The unique product identifier',
        },
      },
      required: ['productId'],
    },
  },
];
```

### Tool Implementation and Agent Loop

```typescript
// Tool implementations
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

// Tool dispatch
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
      throw new Error(`Unknown tool: ${toolName}`);
  }
}

// Agent loop
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

    // If model is done, return text response
    if (response.stop_reason === 'end_turn') {
      const textBlock = response.content.find((b) => b.type === 'text');
      return textBlock ? (textBlock as Anthropic.TextBlock).text : '';
    }

    // Process tool calls
    if (response.stop_reason === 'tool_use') {
      // Add assistant's response (with tool_use blocks) to history
      messages.push({ role: 'assistant', content: response.content });

      // Execute each tool call
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

      // Send results back to the model
      messages.push({ role: 'user', content: toolResults });
    }
  }
}
```

### OpenAI Function Calling Spec

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const tools: OpenAI.Chat.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'search_products',
      description: 'Search the product catalog',
      parameters: {
        type: 'object',
        properties: {
          query: { type: 'string', description: 'Search terms' },
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
      messages.push(choice.message);  // add assistant message with tool_calls

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

### Parallel Tool Calls

Models can request multiple tool calls in a single response when they're independent. Execute them in parallel:

```typescript
// In the tool loop, instead of sequential execution:
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

### Tool Choice Control

```typescript
// Force the model to always use tools (prevent text-only responses)
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  tools,
  tool_choice: { type: 'any' },  // must use at least one tool
  messages,
});

// Force a specific tool
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  tools,
  tool_choice: { type: 'tool', name: 'search_products' },  // must use this specific tool
  messages,
});

// Auto (default): model decides
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 4096,
  tools,
  tool_choice: { type: 'auto' },
  messages,
});
```

## Hands-On Examples

### Typed Tool System

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
    if (!tool) throw new Error(`Tool not found: ${name}`);
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
  description: 'Search the product catalog for items matching a query.',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Search terms' },
      category: { type: 'string', enum: ['electronics', 'clothing', 'books', 'home'] },
      maxResults: { type: 'number', description: 'Max results (1-20, default 5)' },
    },
    required: ['query'],
  },
  execute: async (rawInput) => {
    const input = SearchInputSchema.parse(rawInput);  // validate input
    return productService.search(input.query, input.category, input.maxResults);
  },
};

// Register tools
export const registry = new ToolRegistry()
  .register(searchProductsTool)
  .register(getProductDetailsTool)
  .register(checkInventoryTool);
```

### Agent with Maximum Iteration Guard

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
      throw new Error(`Unexpected stop reason: ${response.stop_reason}`);
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

  throw new Error(`Agent exceeded maximum iterations (${maxIterations})`);
}
```

### Streaming with Tool Use

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

## Common Patterns & Best Practices

### Tool Description Quality

Tool descriptions are part of the prompt — they directly affect when and how the model uses the tool.

```typescript
// Bad: vague description
{
  name: 'get_data',
  description: 'Gets data',
  input_schema: { ... }
}

// Good: specific, action-oriented, describes when to use it and what it returns
{
  name: 'search_orders',
  description: `Search customer orders by various criteria.
Use this when the user asks about order status, history, or tracking.
Returns order ID, status, items, total, and estimated delivery date.
For a single order by ID, use get_order_details instead.`,
  input_schema: { ... }
}
```

### Handling Tool Errors Gracefully

```typescript
// Don't throw from tool implementations — return structured errors
async function getOrderStatus(args: { orderId: string }) {
  const order = await db.query('SELECT * FROM orders WHERE id = $1', [args.orderId]);

  if (!order.rows.length) {
    return {
      found: false,
      error: `No order found with ID ${args.orderId}`,
    };
  }

  return {
    found: true,
    order: order.rows[0],
  };
}

// The model will gracefully handle the error in its response:
// "I searched for order #12345 but couldn't find it in our system."
```

### Limiting Tool Exposure

Don't give the model more tools than it needs for the task — reduces confusion, latency, and cost.

```typescript
// Context-dependent tool selection
function getToolsForContext(userRole: 'customer' | 'admin' | 'readonly') {
  const baseTool = [searchProductsTool, getProductDetailsTool];

  if (userRole === 'admin') {
    return [...baseTool, updatePriceTool, deleteProductTool, viewAnalyticsTool];
  }

  if (userRole === 'customer') {
    return [...baseTool, searchOrdersTool, initiateReturnTool];
  }

  return baseTool;  // readonly: search only
}
```

### Confirmation for Destructive Actions

```typescript
// Add a confirm_action tool for irreversible operations
const confirmActionTool: ToolDefinition<{ action: string; details: string }, { confirmed: boolean }> = {
  name: 'confirm_action',
  description: 'Request user confirmation before performing a destructive or irreversible action.',
  inputSchema: {
    type: 'object',
    properties: {
      action: { type: 'string', description: 'Human-readable description of the action' },
      details: { type: 'string', description: 'What will be changed or deleted' },
    },
    required: ['action', 'details'],
  },
  execute: async (input) => {
    // This returns a pending state — actual confirmation happens in the UI
    // Return the pending confirmation for the UI to handle
    return { confirmed: false, pendingConfirmation: input };
  },
};
```

## Anti-Patterns to Avoid

**Allowing the model to call dangerous tools without guardrails**
```typescript
// Bad: model can delete anything
const deleteUserTool = {
  name: 'delete_user',
  execute: async ({ userId }) => await db.delete(users).where(eq(users.id, userId)),
};

// Good: validate authorization before executing
const deleteUserTool = {
  name: 'delete_user',
  execute: async ({ userId }, context: { adminId: string }) => {
    if (!await isAdmin(context.adminId)) {
      throw new Error('Unauthorized: admin access required');
    }
    await db.delete(users).where(eq(users.id, userId));
  },
};
```

**Ignoring the iteration limit**
Agent loops can run indefinitely if the model keeps calling tools. Always set a maximum.

**Not validating tool inputs**
```typescript
// Bad: trust the model's input directly
async function chargeCard(args: any) {
  await stripe.charge({ amount: args.amount, currency: args.currency });
}

// Good: validate with Zod before executing
const ChargeSchema = z.object({
  amount: z.number().positive().max(10000),  // cents, max $100
  currency: z.enum(['usd', 'eur', 'brl']),
  customerId: z.string().regex(/^cus_/),
});

async function chargeCard(rawArgs: unknown) {
  const args = ChargeSchema.parse(rawArgs);  // throws if invalid
  await stripe.charge(args);
}
```

**Passing entire database records as tool results**
```typescript
// Bad: returns 50 fields, consumes many tokens
return await db.query('SELECT * FROM users WHERE id = $1', [id]);

// Good: return only what's needed
const result = await db.query('SELECT id, name, email, plan FROM users WHERE id = $1', [id]);
return result.rows[0];
```

## Debugging & Troubleshooting

### Logging Tool Calls

```typescript
async function executeWithLogging(
  name: string,
  input: unknown
): Promise<unknown> {
  const start = Date.now();
  console.log(`[TOOL] ${name}`, JSON.stringify(input));

  try {
    const result = await registry.execute(name, input);
    console.log(`[TOOL] ${name} completed in ${Date.now() - start}ms`, JSON.stringify(result).slice(0, 200));
    return result;
  } catch (err) {
    console.error(`[TOOL] ${name} failed:`, (err as Error).message);
    throw err;
  }
}
```

### When the Model Doesn't Use Tools

1. **Improve descriptions** — be more explicit about when the tool should be used
2. **Set `tool_choice: 'any'`** — force tool use for debugging
3. **Check tool schema** — malformed JSON Schema may cause the model to skip the tool
4. **Simplify schema** — complex nested schemas confuse models; flatten when possible

### When the Model Loops Infinitely

1. Add maximum iteration guard
2. Check if tool results are being returned correctly — model may not be getting results
3. Check for `is_error: true` — model may be stuck retrying a broken tool
4. Add a `finish_task` tool that the model must call to complete:

```typescript
const finishTaskTool = {
  name: 'finish_task',
  description: 'Call this when you have gathered all necessary information and are ready to respond to the user.',
  input_schema: {
    type: 'object' as const,
    properties: {
      summary: { type: 'string', description: 'Summary of findings to share with the user' },
    },
    required: ['summary'],
  },
};
```

## Real-World Scenarios

### Scenario 1: Customer Support Agent

```typescript
const supportTools = [
  {
    name: 'lookup_order',
    description: 'Look up a customer order by order ID or email. Returns order status, items, tracking, and delivery estimate.',
    input_schema: {
      type: 'object' as const,
      properties: {
        orderId: { type: 'string', description: 'Order ID (e.g., ORD-12345)' },
        customerEmail: { type: 'string', description: 'Customer email (alternative to orderId)' },
      },
    },
  },
  {
    name: 'initiate_refund',
    description: 'Initiate a refund for an order. Only use after confirming with the customer and verifying the order is eligible.',
    input_schema: {
      type: 'object' as const,
      properties: {
        orderId: { type: 'string' },
        reason: { type: 'string', enum: ['defective', 'not_received', 'wrong_item', 'changed_mind'] },
        amount: { type: 'number', description: 'Refund amount in cents' },
      },
      required: ['orderId', 'reason', 'amount'],
    },
  },
  {
    name: 'create_support_ticket',
    description: 'Create a support ticket for issues that need human review.',
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
    // Inject context
    `You are a helpful customer support agent for Acme Store.
Customer ID: ${customerId}
Current date: ${new Date().toISOString()}
Always verify order ownership before sharing details or processing refunds.`
  );
}
```

### Scenario 2: Data Analysis Agent

```typescript
const analysisTools = [
  {
    name: 'query_database',
    description: 'Run a SQL query against the analytics database to retrieve data.',
    input_schema: {
      type: 'object' as const,
      properties: {
        sql: { type: 'string', description: 'SQL query (SELECT only, no mutations)' },
        limit: { type: 'number', description: 'Max rows to return (default 100)' },
      },
      required: ['sql'],
    },
  },
  {
    name: 'calculate_stats',
    description: 'Calculate descriptive statistics (mean, median, std dev, percentiles) for a dataset.',
    input_schema: {
      type: 'object' as const,
      properties: {
        data: {
          type: 'array',
          items: { type: 'number' },
          description: 'Array of numbers to analyze',
        },
      },
      required: ['data'],
    },
  },
];

// Execute with SQL injection prevention
async function executeQueryTool(args: { sql: string; limit?: number }) {
  // Validate it's a SELECT statement only
  const normalized = args.sql.trim().toUpperCase();
  if (!normalized.startsWith('SELECT')) {
    throw new Error('Only SELECT queries are allowed');
  }

  // Add LIMIT if not present
  const query = normalized.includes('LIMIT')
    ? args.sql
    : `${args.sql} LIMIT ${args.limit ?? 100}`;

  const result = await analyticsDb.query(query);
  return { rowCount: result.rowCount, rows: result.rows };
}
```

## Further Reading

- [Anthropic Tool Use Documentation](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [Building Effective Agents (Anthropic)](https://www.anthropic.com/research/building-effective-agents)
- [ReAct: Reasoning + Acting](https://arxiv.org/abs/2210.03629) — foundational paper on tool-using agents
- [LangChain Agents](https://python.langchain.com/docs/modules/agents/) — higher-level agent framework

## Summary

Tool use transforms LLMs from text generators into agents that can take actions.

The pattern:
1. Define tools with name, description, and JSON Schema input definition
2. Send messages with tools to the LLM
3. When `stop_reason === 'tool_use'`, execute the requested tools in parallel
4. Send results back and loop until `stop_reason === 'end_turn'`
5. Always set a maximum iteration limit (10 is a good default)

TypeScript implementation checklist:
- Validate all tool inputs with Zod before executing
- Return structured errors (don't throw) so the model can recover gracefully
- Return only needed fields from tools — minimize token consumption
- Log all tool calls and results for debugging
- Context-scope tools — give the model only the tools relevant to the current task
- Add confirmation steps before destructive or irreversible tool calls
- Use `Promise.all` for parallel tool calls when the model requests multiple tools at once

Tool description quality matters as much as implementation quality — a well-described tool is used correctly; a vague one is misused or ignored.
