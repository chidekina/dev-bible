# Track 13 — AI & LLMs

Integrate large language models into production applications. Understanding how LLMs work, how to prompt them effectively, and how to manage cost and reliability separates experimental demos from production-grade AI features.

**Estimated time:** 2–3 weeks

---

## Topics

1. [LLM Fundamentals](llm-fundamentals.md) — tokens, context windows, temperature, embeddings, model families
2. [Prompt Engineering](prompt-engineering.md) — system prompts, few-shot examples, chain-of-thought, structured output
3. [RAG & Embeddings](rag-embeddings.md) — retrieval-augmented generation, vector databases, semantic search
4. [Tool Use & Function Calling](tool-use-function-calling.md) — tool schemas, parsing tool calls, agentic loops
5. [AI Integration Patterns](ai-integration-patterns.md) — streaming responses, error handling, fallbacks, human-in-the-loop
6. [Fine-Tuning Concepts](fine-tuning-concepts.md) — when to fine-tune vs prompt, LoRA, dataset preparation
7. [AI Cost Optimization](ai-cost-optimization.md) — prompt caching, model routing, token budgets, observability

---

## Prerequisites

- Track 01 — Foundations (TypeScript, async patterns)
- Track 03 — Backend (REST APIs, databases for RAG storage)
- Track 08 — System Design (caching, queues for AI pipelines)

---

## What You'll Build

- A RAG pipeline that indexes a documentation site into a vector database (pgvector) and answers questions about it
- A tool-calling agent that uses function calling to query a database and send emails
- A streaming chat API endpoint built with Fastify that proxies responses from the Anthropic SDK
- A cost monitoring dashboard that tracks token usage per user and enforces daily budget limits
