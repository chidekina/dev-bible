# Track 13 — IA & LLMs

Integre large language models em aplicações de produção. Entender como os LLMs funcionam, como criar prompts eficazes e como gerenciar custos e confiabilidade é o que separa demos experimentais de funcionalidades de IA prontas para produção.

**Tempo estimado:** 2–3 semanas

---

## Tópicos

1. [Fundamentos de LLM](llm-fundamentals.md) — tokens, context windows, temperature, embeddings, famílias de modelos
2. [Prompt Engineering](prompt-engineering.md) — system prompts, exemplos few-shot, chain-of-thought, output estruturado
3. [RAG & Embeddings](rag-embeddings.md) — retrieval-augmented generation, bancos de dados vetoriais, busca semântica
4. [Tool Use & Function Calling](tool-use-function-calling.md) — schemas de ferramentas, parsing de tool calls, loops agênticos
5. [Padrões de Integração com IA](ai-integration-patterns.md) — streaming de respostas, tratamento de erros, fallbacks, human-in-the-loop
6. [Conceitos de Fine-Tuning](fine-tuning-concepts.md) — quando fazer fine-tune vs. usar prompt, LoRA, preparação de dataset
7. [Otimização de Custos com IA](ai-cost-optimization.md) — prompt caching, roteamento de modelos, orçamentos de tokens, observabilidade

---

## Pré-requisitos

- Track 01 — Foundations (TypeScript, padrões assíncronos)
- Track 03 — Backend (REST APIs, bancos de dados para armazenamento de RAG)
- Track 08 — System Design (caching, filas para pipelines de IA)

---

## O Que Você Vai Construir

- Um pipeline RAG que indexa um site de documentação em um banco de dados vetorial (pgvector) e responde perguntas sobre ele
- Um agente com tool calling que usa function calling para consultar um banco de dados e enviar e-mails
- Um endpoint de chat com streaming construído com Fastify que faz proxy das respostas do Anthropic SDK
- Um dashboard de monitoramento de custos que rastreia o uso de tokens por usuário e aplica limites de orçamento diário
