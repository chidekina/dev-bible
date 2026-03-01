# Conceitos de Fine-Tuning

## Visão Geral

Fine-tuning treina um LLM pré-treinado no seu próprio dataset para adaptá-lo a uma tarefa, domínio ou estilo específico. O resultado é um modelo que realiza sua tarefa específica melhor, mais rápido e com menor custo do que um modelo de propósito geral com um longo system prompt.

Mas fine-tuning frequentemente é a ferramenta errada. Antes de recorrer a ele, você deve esgotar o prompt engineering e o RAG — ambos são mais rápidos, mais baratos e mais fáceis de atualizar. Fine-tuning faz sentido quando você tem uma tarefa bem definida, centenas ou milhares de exemplos e evidências claras de que o prompting sozinho não é suficiente.

## Pré-requisitos

- Fundamentos de LLM (tokens, context window, temperature)
- Noções básicas de Prompt Engineering
- Python (a maioria das ferramentas de fine-tuning é Python-first)
- Acesso a infraestrutura de treinamento (GPU ou API cloud)

## Conceitos Fundamentais

### Quando Fazer Fine-Tune vs. Quando Usar Prompt

| Cenário | Abordagem |
|---------|-----------|
| Tarefa geral com um prompt claro | Prompt engineering |
| Conhecimento específico de domínio | RAG (retrieval-augmented generation) |
| Formato/estilo de saída específico | Prompt engineering com exemplos (few-shot) |
| Tarefa complexa demais para prompting | Fine-tuning |
| Reduzir tamanho do prompt por custo | Fine-tuning |
| Dados proprietários que não devem estar em prompts | Fine-tuning |
| Necessidade de tom/estilo consistente em escala | Fine-tuning |
| Informações recentes (pós-corte de treinamento) | RAG, não fine-tuning |

**Árvore de decisão:**
1. Um system prompt forte + exemplos few-shot resolvem? → Use prompting
2. A tarefa exige conhecimento de domínio? → Tente RAG primeiro
3. O prompting é lento/caro no seu volume? → Considere fine-tuning
4. Você tem 500+ exemplos de alta qualidade? → Fine-tuning pode ser viável
5. A tarefa muda com frequência? → Fique com prompting (fine-tuning não se atualiza facilmente)

### Full Fine-Tuning vs. Fine-Tuning com Eficiência de Parâmetros

**Full fine-tuning** — atualiza todos os pesos do modelo. Caro (requer dezenas de GPUs para modelos grandes), risco de esquecimento catastrófico. Raramente necessário para casos de uso de aplicação.

**Parameter-efficient fine-tuning (PEFT)** — congela a maioria dos pesos e treina apenas um pequeno conjunto de parâmetros adicionais. Reduz drasticamente os requisitos de memória e computação.

**LoRA (Low-Rank Adaptation)** — o método PEFT mais popular. Em vez de atualizar diretamente a matriz de pesos W, o LoRA adiciona duas pequenas matrizes A e B tal que a atualização é W + BA (onde B é alta e estreita, A é baixa e larga). O rank `r` controla quantos parâmetros são treinados.

```
Modelo completo: 7B pesos (14GB em fp16)
Adaptador LoRA: 10-50MB de pesos treinados
Na inferência: W_effective = W_original + B * A
```

**QLoRA (Quantized LoRA)** — combina quantização de 4 bits dos pesos congelados com adaptadores LoRA. Permite o fine-tuning de modelos 7B em uma única GPU de 24GB.

```python
# Configuração QLoRA com Hugging Face + bitsandbytes
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import prepare_model_for_kbit_training, LoraConfig, get_peft_model

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-8B-Instruct",
    quantization_config=bnb_config,
    device_map="auto",
)

model = prepare_model_for_kbit_training(model)

peft_config = LoraConfig(
    r=16,                       # rank: maior = mais parâmetros, mais capacidade
    lora_alpha=32,              # fator de escala; geralmente 2x o rank
    target_modules=["q_proj", "v_proj"],  # quais matrizes de atenção treinar
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)

model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
# trainable params: 6,815,744 || all params: 8,036,421,632 || trainable%: 0.0848
```

### Preparação do Dataset

A qualidade dos dados de treinamento é o fator mais importante. 500 exemplos de alta qualidade superam 5.000 com ruído.

**Formato do dataset** (instruction tuning):
```json
[
  {
    "messages": [
      {
        "role": "system",
        "content": "Você é um agente de suporte ao cliente da Acme Corp. Seja conciso e útil."
      },
      {
        "role": "user",
        "content": "Como faço para redefinir minha senha?"
      },
      {
        "role": "assistant",
        "content": "Para redefinir sua senha:\n1. Acesse a página de login\n2. Clique em 'Esqueci minha senha'\n3. Informe seu e-mail\n4. Verifique sua caixa de entrada pelo link de redefinição\n\nO link expira em 24 horas. Me avise se precisar de ajuda!"
      }
    ]
  }
]
```

**Checklist de qualidade dos dados:**
- [ ] Formato consistente (mesma estrutura de system prompt ao longo de todo o dataset)
- [ ] Outputs corretos (revisados por especialistas de domínio, não apenas gerados por outro modelo)
- [ ] Cobertura diversa (diferentes formulações, casos extremos, níveis de dificuldade)
- [ ] Sem duplicatas (deduplicação com correspondência fuzzy)
- [ ] Sem PII nos dados de treinamento (conformidade GDPR/LGPD)
- [ ] Distribuição equilibrada (representação igual de todas as classes/casos)

**Tamanhos mínimos de dataset:**
| Tarefa | Mínimo | Bom |
|--------|--------|-----|
| Adaptação de estilo/formato | 100-200 | 500+ |
| Q&A específico de domínio | 500 | 2.000+ |
| Classificação | 200-500 por classe | 1.000+ por classe |
| Geração de código | 1.000 | 5.000+ |

### API de Fine-Tuning da OpenAI

O caminho mais simples para fine-tuning — sem gerenciamento de GPU necessário.

```python
# Python (fine-tuning é primariamente um fluxo de trabalho em Python)
from openai import OpenAI
import json

client = OpenAI(api_key="...")

# 1. Prepara os dados de treinamento
training_data = [
    {
        "messages": [
            {"role": "system", "content": "Classifique o sentimento da avaliação."},
            {"role": "user", "content": "Ótimo produto, funciona perfeitamente!"},
            {"role": "assistant", "content": "POSITIVE"}
        ]
    },
    # ... centenas de exemplos a mais
]

# Salva como JSONL
with open("training.jsonl", "w") as f:
    for example in training_data:
        f.write(json.dumps(example) + "\n")

# 2. Faz upload do arquivo de treinamento
with open("training.jsonl", "rb") as f:
    file_response = client.files.create(file=f, purpose="fine-tune")

training_file_id = file_response.id
print(f"Uploaded: {training_file_id}")

# 3. Cria o job de fine-tuning
job = client.fine_tuning.jobs.create(
    training_file=training_file_id,
    model="gpt-4o-mini-2024-07-18",  # modelo base mais barato
    hyperparameters={
        "n_epochs": 3,          # número de passagens de treinamento
        "batch_size": 4,
        "learning_rate_multiplier": 1.8,
    },
    suffix="sentiment-v1",      # adicionado ao nome do modelo
)

print(f"Job ID: {job.id}")
```

```typescript
// Monitorando o job de fine-tuning (TypeScript)
import OpenAI from 'openai';
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function pollJob(jobId: string) {
  while (true) {
    const job = await openai.fineTuning.jobs.retrieve(jobId);
    console.log(`Status: ${job.status} | Erro: ${job.error?.message ?? 'nenhum'}`);

    if (job.status === 'succeeded') {
      console.log(`Modelo fine-tuned: ${job.fine_tuned_model}`);
      return job.fine_tuned_model;
    }

    if (job.status === 'failed' || job.status === 'cancelled') {
      throw new Error(`Job ${job.status}: ${job.error?.message}`);
    }

    await new Promise((r) => setTimeout(r, 30_000)); // verifica a cada 30s
  }
}

// Usando o modelo fine-tuned
const fineTunedModel = await pollJob('ftjob-abc123');
const response = await openai.chat.completions.create({
  model: fineTunedModel,
  messages: [
    { role: 'system', content: 'Classifique o sentimento da avaliação.' },
    { role: 'user', content: 'Produto mediano, nada especial.' },
  ],
  max_tokens: 10,
  temperature: 0,
});
```

### Métricas de Avaliação

**Para tarefas de classificação:**
- **Accuracy** — % de previsões corretas
- **Precision/Recall/F1** — para classes desbalanceadas
- **Matriz de confusão** — veja onde o modelo erra

**Para tarefas de geração:**
- **BLEU score** — sobreposição de n-gramas com referência (bom para tradução, ruim para texto aberto)
- **ROUGE** — sobreposição baseada em recall (bom para sumarização)
- **BERTScore** — similaridade semântica usando embeddings
- **Avaliação humana** — cara, mas mais confiável
- **LLM-as-judge** — usa um modelo mais forte para avaliar os outputs com uma rubrica

```python
# Script de avaliação em Python
from sklearn.metrics import accuracy_score, classification_report
import json

def evaluate_model(model_name: str, test_data: list[dict]) -> dict:
    predictions = []
    true_labels = []

    for example in test_data:
        user_message = example["messages"][-2]["content"]
        true_label = example["messages"][-1]["content"].strip()

        response = client.chat.completions.create(
            model=model_name,
            messages=example["messages"][:-1],  # exclui o turno do assistente
            max_tokens=20,
            temperature=0,
        )
        prediction = response.choices[0].message.content.strip()

        predictions.append(prediction)
        true_labels.append(true_label)

    accuracy = accuracy_score(true_labels, predictions)
    report = classification_report(true_labels, predictions)

    return {
        "accuracy": accuracy,
        "report": report,
        "n_examples": len(test_data),
    }

# Compara base vs. fine-tuned
base_metrics = evaluate_model("gpt-4o-mini", test_data)
finetuned_metrics = evaluate_model("ft:gpt-4o-mini-2024-07-18:acme::abc123", test_data)

print(f"Accuracy do modelo base: {base_metrics['accuracy']:.3f}")
print(f"Accuracy do modelo fine-tuned: {finetuned_metrics['accuracy']:.3f}")
print(f"Melhoria: +{(finetuned_metrics['accuracy'] - base_metrics['accuracy']):.3f}")
```

## Exemplos Práticos

### Pipeline TypeScript: Geração de Dados de Treinamento

Quando você não tem milhares de exemplos rotulados, pode gerar dados sintéticos a partir de um pequeno conjunto semente:

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { writeFileSync } from 'fs';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

interface TrainingExample {
  messages: Array<{ role: string; content: string }>;
}

const SEED_EXAMPLES = [
  {
    input: 'Meu pedido não chegou depois de 2 semanas',
    output: JSON.stringify({
      category: 'SHIPPING',
      urgency: 'HIGH',
      action: 'Verifique o rastreamento da transportadora e inicie investigação se o pacote estiver perdido',
    }),
  },
  // ... 10-20 exemplos semente
];

async function generateSyntheticData(
  count: number,
  systemPrompt: string
): Promise<TrainingExample[]> {
  const examples: TrainingExample[] = [];
  const seedContext = SEED_EXAMPLES.map(
    (e) => `Input: ${e.input}\nOutput: ${e.output}`
  ).join('\n\n');

  for (let i = 0; i < count; i++) {
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 500,
      temperature: 0.8,  // temperature alta para variedade
      messages: [{
        role: 'user',
        content: `Gere uma nova mensagem realista de suporte ao cliente e sua classificação correta.
A mensagem deve ser diferente destes exemplos, mas seguir o mesmo padrão:

${seedContext}

Retorne JSON: {"input": "mensagem do cliente", "output": "JSON de classificação"}`,
      }],
    });

    try {
      const { input, output } = JSON.parse(
        (response.content[0] as Anthropic.TextBlock).text
      );

      examples.push({
        messages: [
          { role: 'system', content: systemPrompt },
          { role: 'user', content: input },
          { role: 'assistant', content: output },
        ],
      });
    } catch {
      // Ignora exemplos malformados
    }

    if (i % 50 === 0) console.log(`Gerados ${i}/${count} exemplos`);
  }

  return examples;
}

async function main() {
  const systemPrompt = 'Classifique tickets de suporte ao cliente. Retorne JSON com category, urgency e action.';
  const syntheticData = await generateSyntheticData(500, systemPrompt);

  // Divide 80/20 treinamento/validação
  const splitAt = Math.floor(syntheticData.length * 0.8);
  const trainData = syntheticData.slice(0, splitAt);
  const validData = syntheticData.slice(splitAt);

  // Escreve como JSONL
  writeFileSync('train.jsonl', trainData.map((e) => JSON.stringify(e)).join('\n'));
  writeFileSync('valid.jsonl', validData.map((e) => JSON.stringify(e)).join('\n'));
  console.log(`Escrito ${trainData.length} exemplos de treinamento, ${validData.length} de validação`);
}

main();
```

### Orientação de Hiperparâmetros

```python
# Para a maioria das tarefas de classificação e extração:
hyperparameters = {
    "n_epochs": 3,              # 1-5; mais épocas = maior risco de overfitting
    "batch_size": 4,            # 4-32; batches maiores = gradientes mais estáveis
    "learning_rate_multiplier": 1.8,  # 0,1-2,0; maior = mais rápido mas instável
}

# Para adaptação de estilo/formato:
hyperparameters = {
    "n_epochs": 2,              # menos épocas para evitar esquecer capacidades base
    "batch_size": 8,
    "learning_rate_multiplier": 1.0,
}

# Para datasets pequenos (<200 exemplos):
hyperparameters = {
    "n_epochs": 5,              # mais épocas compensa menos dados
    "batch_size": 2,
    "learning_rate_multiplier": 0.5,  # LR menor para poucos dados
}
```

## Padrões Comuns e Boas Práticas

### O Ciclo de Iteração de Fine-Tuning

```
1. Comece com prompting — estabeleça uma baseline
2. Colete falhas — casos onde o prompting falha
3. Construa conjunto de avaliação — 100+ exemplos com rótulos corretos
4. Fine-tune nas falhas — não em todos os dados, apenas nos casos difíceis
5. Avalie — meça a melhoria no conjunto de avaliação
6. Leve ao shadow traffic — compare com a baseline de prompt
7. Itere — adicione mais dados para os modos de falha restantes
```

### Boas Práticas de Formatação de Dados

```python
# 1. System prompt consistente
SYSTEM_PROMPT = "Você é um classificador de suporte ao cliente. Retorne um objeto JSON."
# Use EXATAMENTE o mesmo system prompt em todos os exemplos e na inferência

# 2. Formato de saída consistente
# Ruim: alguns exemplos retornam {"category":"X"}, outros retornam "Categoria: X"
# Bom: sempre retorne o mesmo formato

# 3. Inclua exemplos negativos
negative_examples = [
    {
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": "Olá, como vai?"},
            {"role": "assistant", "content": '{"category": "OTHER", "urgency": "LOW"}'},
        ]
    }
]
# O modelo precisa saber o que fazer com inputs fora da distribuição
```

### Prevenindo Overfitting

```python
# Sinais de overfitting:
# - Loss de treinamento diminuindo, loss de validação aumentando
# - O modelo memoriza exemplos de treinamento em vez de generalizar
# - Desempenho ruim em dados não vistos

# Mitigações:
# 1. Dados de treinamento mais diversos
# 2. Menos épocas (2-3 em vez de 5+)
# 3. Learning rate menor
# 4. Adicionar data augmentation (parafraseie exemplos)
# 5. Use um modelo base maior (mais parâmetros = menos overfitting por exemplo)
```

## Anti-Padrões a Evitar

**Fine-tuning para injetar conhecimento**
```
# Ruim: tentar ensinar fatos ao modelo
"P: Qual é a política de devolução da Acme Corp? R: Devoluções em 30 dias sem perguntas."

# Fine-tuning ensina comportamento, não fatos
# Para conhecimento: use RAG — recupere de um document store na inferência
# Fatos embutidos no fine-tuning ficam desatualizados e são difíceis de atualizar
```

**Usar dados gerados de baixa qualidade sem revisão**
```python
# Ruim: gerar 10.000 exemplos com GPT-4, fazer fine-tune sem revisão
# Resultado: o modelo aprende os erros e vieses do GPT-4

# Bom: gerar exemplos, depois ter humanos revisando e corrigindo uma amostra
# Revisão de amostra aleatória: verifique 10% dos dados gerados
# Se >5% de taxa de erro, regenere com prompts melhores ou colete dados reais
```

**Pular a baseline de avaliação**
Sempre meça o desempenho do modelo base antes do fine-tuning. Se o modelo base já atinge 90% de accuracy, você precisa do fine-tuning por outra razão (custo, latência) e não por qualidade.

**Fazer fine-tuning em todo o dataset de treinamento sem split de validação**
Sempre reserve 15-20% para validação para detectar overfitting durante o treinamento.

**Não versionar dados de treinamento e modelos**
```
# Boa prática:
dataset_v1.jsonl         → ftjob-abc123 → ft:gpt-4o-mini-...:v1
dataset_v2.jsonl         → ftjob-def456 → ft:gpt-4o-mini-...:v2
# Mantenha todas as versões — pode ser necessário fazer rollback
```

## Depuração e Resolução de Problemas

### Desempenho Ruim Após Fine-Tuning

```python
# Checklist de diagnóstico:
# 1. Verifique a qualidade dos dados de treinamento
#    - Amostre 20 exemplos e revise manualmente
#    - Os outputs estão corretos?
#    - O formato é consistente?

# 2. Verifique o volume de dados
#    - Poucos exemplos? → colete mais ou use augmentação sintética
#    - Muitos exemplos com ruído? → filtre por qualidade

# 3. Verifique a configuração de teste
#    - Está usando o mesmo system prompt do treinamento?
#    - A temperature está definida como 0 para avaliação determinística?

# 4. Verifique distribuição mismatch entre treino e teste
#    - Os dados de avaliação representam os dados de produção?
#    - Os exemplos de avaliação se parecem com os exemplos de treinamento?
```

### Verificando Data Leakage

```python
from difflib import SequenceMatcher

def check_leakage(train_data, test_data, threshold=0.85):
    leaks = []
    for i, test_ex in enumerate(test_data):
        test_input = test_ex["messages"][1]["content"]
        for j, train_ex in enumerate(train_data):
            train_input = train_ex["messages"][1]["content"]
            ratio = SequenceMatcher(None, test_input, train_input).ratio()
            if ratio > threshold:
                leaks.append((i, j, ratio))

    if leaks:
        print(f"AVISO: {len(leaks)} possíveis data leaks encontrados")
        for test_i, train_j, sim in leaks[:5]:
            print(f"  Test[{test_i}] ≈ Train[{train_j}] (similaridade: {sim:.3f})")
    else:
        print("Nenhum data leakage significativo detectado")
```

## Cenários do Mundo Real

### Cenário 1: Fine-Tuning para Triagem de Suporte ao Cliente

Um time de suporte recebe 10.000 tickets por mês. A triagem manual leva 2 dias. Fine-tuning de um modelo barato para fazer isso em milissegundos.

**Processo:**
1. Exporta 3 meses de tickets históricos (5.000+ exemplos) com categorias atribuídas por humanos
2. Limpa: remove PII, normaliza nomes de categorias, deduplica
3. Divide 80/20 treinamento/validação
4. Fine-tune no `gpt-4o-mini` (base barata, inferência rápida)
5. Avalia: compara accuracy com modelo base com system prompt
6. Se accuracy ≥ 95% na validação retida, promove para produção

**Resultados esperados:**
- `gpt-4o-mini` base com system prompt: 78% de accuracy
- `gpt-4o-mini` fine-tuned: 94% de accuracy
- Custo de inferência: 80% mais barato do que usar GPT-4o com exemplos few-shot
- Latência: 200ms vs. 2-3s para prompting few-shot

### Cenário 2: Adaptação de Formato para Geração de Código

Uma empresa usa um estilo específico de TypeScript (Result types, padrões de erro específicos). O prompting few-shot requer 2.000 tokens de exemplos em cada requisição. Fine-tuning amortiza esse custo.

**Construção do dataset:**
- 200 pares de: especificação → código TypeScript seguindo os padrões da empresa
- Gerado pelo GPT-4 a partir do código interno + revisado por engenheiros sêniores
- Foco em: Result types, tratamento de erros, convenções de nomenclatura, estrutura de pastas

**Resultados esperados:**
- Remove 2.000 tokens de exemplos few-shot de cada requisição (redução de 60% nos custos)
- O modelo aplica consistentemente as convenções sem lembrete
- Ainda usa RAG para contexto específico da codebase (não fine-tuning)

## Leituras Complementares

- [Guia de Fine-Tuning da OpenAI](https://platform.openai.com/docs/guides/fine-tuning)
- [Documentação do Hugging Face PEFT](https://huggingface.co/docs/peft)
- [Artigo LoRA (Hu et al., 2021)](https://arxiv.org/abs/2106.09685)
- [Artigo QLoRA (Dettmers et al., 2023)](https://arxiv.org/abs/2305.14314)
- [Axolotl — Framework de Fine-Tuning](https://github.com/OpenAccess-AI-Collective/axolotl) — treinamento simplificado de LoRA/QLoRA
- [Fine-Tuning do Claude na Anthropic](https://docs.anthropic.com/en/docs/about-claude/models) — planos enterprise

## Resumo

Fine-tuning adapta um LLM pré-treinado a uma tarefa específica treinando nos seus dados. Escolha-o quando prompting e RAG não são suficientes, quando você precisa de redução de custo/latência em escala, ou quando a consistência de formato de saída é crítica.

Abordagens principais:
- **Full fine-tuning** — todos os pesos atualizados; caro; raramente necessário
- **LoRA** — treina pequenas matrizes adaptadoras; 100x menos parâmetros; mesma qualidade
- **QLoRA** — LoRA + quantização de 4 bits; permite modelos 7B em GPU única
- **Fine-tuning via API OpenAI/Anthropic** — gerenciado, sem necessidade de GPU, melhor ponto de partida

Dados são tudo:
- 500 exemplos de alta qualidade > 5.000 com ruído
- Sempre reserve 20% para validação
- Use exatamente o mesmo system prompt no treinamento e na inferência
- Revise os dados gerados antes de usá-los como dados de treinamento

Processo de avaliação:
- Estabeleça a baseline do modelo base antes do fine-tuning
- Meça em um conjunto de teste retido (não o conjunto de validação usado durante o treinamento)
- Para classificação: accuracy, F1, matriz de confusão
- Para geração: LLM-as-judge ou avaliação humana

Regra de decisão: tente prompting primeiro, depois RAG, depois fine-tuning. Fine-tuning é a ferramenta de último recurso — poderosa, mas cara de manter.
