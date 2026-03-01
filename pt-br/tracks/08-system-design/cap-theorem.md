# Teorema CAP

## Visão Geral

O teorema CAP afirma que um sistema distribuído pode garantir no máximo duas das três propriedades simultaneamente: Consistência, Disponibilidade e Tolerância a Partições. Formulado por Eric Brewer em 2000 e provado por Gilbert e Lynch em 2002, é a restrição fundamental que molda toda decisão de design de bancos de dados e serviços distribuídos. Entender o CAP significa entender *por que* seu banco de dados se comporta do jeito que se comporta durante falhas, e por que "simplesmente deixar consistente e disponível" nem sempre é uma opção.

---

## Pré-requisitos

- Conceitos básicos de banco de dados (transações, leituras, escritas)
- Entendimento de redes e o que significa "partição de rede"
- Familiaridade com termos como replicação e sistemas distribuídos em alto nível

---

## Conceitos Fundamentais

### As três propriedades

**Consistência (C):** Toda leitura recebe a escrita mais recente, ou um erro. Todos os nós enxergam os mesmos dados ao mesmo tempo.

**Disponibilidade (A):** Toda requisição recebe uma resposta (não um erro), mas ela pode não conter a escrita mais recente.

**Tolerância a Partições (P):** O sistema continua operando mesmo quando ocorrem partições de rede (mensagens perdidas entre nós).

### Por que só é possível ter duas

Em um sistema distribuído que se comunica por uma rede real, partições de rede não são teóricas — elas acontecem. Hardware falha, cabos de rede se partem, data centers perdem conectividade. Isso significa que **Tolerância a Partições não é opcional** para qualquer sistema que rode em mais de uma máquina. A escolha real é entre **CP** e **AP**.

```
      ┌─────────────┐
      │    Partition │
      │   Tolerance  │
      │      (P)     │
      └──────┬───────┘
             │
    ┌─────────┴─────────┐
    │                   │
┌───▼───┐           ┌───▼───┐
│  CP   │           │  AP   │
│       │           │       │
│Consist│           │ Avail │
│ency + │           │ability│
│Parttn │           │+ Parttn│
└───────┘           └───────┘
```

### Sistemas CP

Quando uma partição ocorre, o sistema recusa-se a servir dados desatualizados. Ele retorna um erro em vez de uma resposta inconsistente. Os usuários experimentam indisponibilidade; os dados são sempre corretos.

Exemplos: HBase, ZooKeeper, etcd, maioria dos bancos relacionais em modo single-primary.

### Sistemas AP

Quando uma partição ocorre, o sistema continua atendendo requisições, potencialmente retornando dados desatualizados. Todos os nós permanecem disponíveis; os dados podem estar temporariamente inconsistentes.

Exemplos: DynamoDB (com consistência eventual), Cassandra, CouchDB, DNS.

### A extensão PACELC

O CAP trata de falhas. O PACELC estende isso para a operação normal: mesmo quando não há partição (P não está em jogo), existe um trade-off entre latência e consistência. Você quer baixa Latência (L) ou forte Consistência (C)?

---

## Exemplos Práticos

### Demonstrando consistência eventual no Redis

O Redis Cluster usa semântica AP. Durante um failover, leituras de uma replica podem estar desatualizadas:

```typescript
import { createClient } from 'redis';

const primary = createClient({ url: 'redis://primary:6379' });
const replica = createClient({ url: 'redis://replica:6380' });

await primary.connect();
await replica.connect();

// Escreve no primary
await primary.set('user:42:name', 'Alice');

// Leitura imediata da replica — pode retornar valor antigo ou null
// (a replicação é assíncrona, ~1-5ms de lag em operação normal)
const name = await replica.get('user:42:name');
console.log(name); // pode ser null ou 'Alice' dependendo do lag de replicação
```

**Mitigação para leituras que precisam ser frescas:** leia do primary. Aceite o custo de latência.

```typescript
// Para leituras críticas, sempre vá para o primary
const freshName = await primary.get('user:42:name'); // sempre consistente
```

### Consistência read-your-writes

Um padrão comum que faz a ponte entre AP e CP: após uma escrita, o mesmo cliente lê do primary por uma janela curta.

```typescript
// src/lib/db-routing.ts
import { PrismaClient } from '@prisma/client';

const primaryClient = new PrismaClient({ datasources: { db: { url: process.env.DATABASE_PRIMARY_URL } } });
const replicaClient = new PrismaClient({ datasources: { db: { url: process.env.DATABASE_REPLICA_URL } } });

// Após uma escrita, armazena um timestamp. Roteia leituras para o primary dentro da janela de lag de replicação.
const POST_WRITE_PRIMARY_WINDOW_MS = 1000;
const lastWriteByUser = new Map<string, number>();

export function markWrite(userId: string): void {
  lastWriteByUser.set(userId, Date.now());
}

export function getDbClient(userId?: string): PrismaClient {
  if (!userId) return replicaClient;

  const lastWrite = lastWriteByUser.get(userId) ?? 0;
  const timeSinceWrite = Date.now() - lastWrite;

  return timeSinceWrite < POST_WRITE_PRIMARY_WINDOW_MS
    ? primaryClient   // ainda dentro da janela pós-escrita — usa o primary
    : replicaClient;  // seguro usar a replica
}
```

### Consistência forte com transações PostgreSQL

O Postgres (single-primary) é um sistema CP. Dentro de uma transação, todas as leituras são consistentes:

```typescript
// Todas as leituras e escritas nessa transação enxergam um snapshot consistente
await db.$transaction(async (tx) => {
  const account = await tx.account.findUnique({
    where: { id: accountId },
    // SELECT ... FOR UPDATE — impede modificações concorrentes
    // No Prisma: use raw query ou locking otimista
  });

  if (!account || account.balance < amount) {
    throw new Error('Insufficient funds');
  }

  await tx.account.update({
    where: { id: accountId },
    data: { balance: { decrement: amount } },
  });

  await tx.account.update({
    where: { id: toAccountId },
    data: { balance: { increment: amount } },
  });
});
// Se qualquer etapa falhar, a transação inteira é revertida — sem transferências parciais
```

### Leituras e escritas em quorum (estilo Cassandra)

Em um cluster Cassandra com fator de replicação 3:
- Quorum de escrita: escreve em 2 de 3 nós (W=2)
- Quorum de leitura: lê de 2 de 3 nós (R=2)
- W + R > N (2+2=4 > 3) garante a leitura da escrita mais recente

```typescript
// Simulando lógica de quorum (conceitual — os drivers do Cassandra lidam com isso)
const REPLICATION_FACTOR = 3;

function quorumCount(n: number): number {
  return Math.floor(n / 2) + 1; // maioria
}

const writeQuorum = quorumCount(REPLICATION_FACTOR); // 2
const readQuorum = quorumCount(REPLICATION_FACTOR);  // 2

// Se writeQuorum + readQuorum > REPLICATION_FACTOR, as leituras sempre veem as escritas mais recentes
console.log(writeQuorum + readQuorum > REPLICATION_FACTOR); // true → consistência forte
```

---

## Padrões e Boas Práticas

**Escolha CP quando:**
- Transações financeiras (bancos, pagamentos, estoque)
- Autenticação de usuários (estado de login incorreto é um problema de segurança)
- Locks distribuídos e eleição de líderes (ZooKeeper, etcd)

**Escolha AP quando:**
- Carrinhos de compra (inconsistência temporária é aceitável)
- Feeds de redes sociais (mostrar posts levemente desatualizados é aceitável)
- DNS (staleness é aceitável por segundos ou minutos)
- Contadores de analytics (precisão eventual é suficiente)

**Projete para a falha que você espera:**
- Se seus nós estão em um único data center, partições de rede são raras — priorize consistência
- Se seus nós estão em múltiplos data centers, partições são esperadas — projete para AP

**Mitigue inconsistências AP com:**
- Sessões read-your-writes
- Leituras monotônicas (nunca retorne dados mais antigos do que o que você mostrou ao cliente antes)
- Staleness limitado (lag da replica garantidamente abaixo de N segundos)

---

## Anti-Padrões a Evitar

- Construir um sistema financeiro em um banco AP sem entender as implicações
- Assumir que seu banco SQL é sempre consistente — em um setup primary/replica, as replicas têm lag
- Escolher "consistência forte em todo lugar" sem medir o custo de latência
- Ignorar a possibilidade de partições de rede porque "isso nunca acontece" — acontece sim

---

## Debugging e Troubleshooting

**"Usuários estão vendo dados desatualizados após uma atualização"**
Você está lendo de uma replica que ainda não replicou a escrita. Soluções: roteie leituras pós-escrita para o primary, adicione monitoramento de lag da replica, ou aumente o nível de consistência ao custo de latência.

**"O lock distribuído não está funcionando — dois nós são líderes ao mesmo tempo"**
Isso é um problema de split-brain causado por uma partição. Use etcd ou ZooKeeper para locking distribuído — eles são CP e usam o algoritmo de consenso Raft para garantir apenas um líder.

**"Meu cluster Cassandra está rejeitando escritas durante uma falha de nó"**
Verifique seu nível de consistência de escrita. `QUORUM` vai falhar se muitos nós estiverem fora. `ONE` ou `ANY` vai funcionar, mas com garantias de consistência mais fracas.

---

## Cenários do Mundo Real

**Cenário: Design de sistema de processamento de pagamentos**

Requisitos: nunca cobrar duas vezes, nunca permitir uma transferência quando o saldo é insuficiente.

Decisão: **banco CP** (PostgreSQL com transações). A disponibilidade pode ser reduzida durante uma partição — uma mensagem de erro é melhor do que uma cobrança incorreta.

Arquitetura:
- Todas as escritas financeiras passam por um único primary Postgres
- Sem read replicas para verificações de saldo — sempre leia do primary
- Encapsule toda operação multi-etapa em uma transação de banco de dados
- Use constraints em nível de banco de dados como última linha de defesa

**Cenário: Design de sistema de feed social**

Requisitos: mostrar aos usuários posts recentes, aceitável mostrar posts de 1-2 segundos atrás.

Decisão: **banco AP** (Cassandra ou DynamoDB com consistência eventual). Mostrar um feed levemente desatualizado é aceitável; indisponibilidade não é.

Arquitetura:
- Escreva posts no Cassandra com `LOCAL_QUORUM` (consistente dentro de um data center)
- Leia feeds de qualquer replica
- Aceite que um post pode levar 2-3 segundos para aparecer para todos os usuários

---

## Leitura Complementar

- [Talk original do Teorema CAP por Eric Brewer](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf)
- [Gilbert e Lynch: Brewer's Conjecture and Feasibility of Consistent, Available, Partition-Tolerant Web Services](https://dl.acm.org/doi/10.1145/564585.564601)
- [Kleppmann: Designing Data-Intensive Applications](https://dataintensive.net/) — Capítulo 9
- [Teorema PACELC](https://en.wikipedia.org/wiki/PACELC_theorem)
- Track 08: [Database Scaling](database-scaling.md)
- Track 08: [Sistemas Distribuídos](distributed-systems.md)

---

## Resumo

O teorema CAP não é uma escolha de design — é uma restrição. Como partições de rede acontecem em qualquer sistema distribuído, você deve escolher entre Consistência (erros durante partições) e Disponibilidade (dados desatualizados durante partições). Para sistemas financeiros, de autenticação e de coordenação, escolha CP e aceite que erros são melhores do que dados incorretos. Para sistemas sociais, de analytics e de entrega de conteúdo, escolha AP e projete mitigações (read-your-writes, leituras monotônicas, staleness limitado) para minimizar o impacto da inconsistência. O importante é fazer a escolha deliberadamente — a maioria dos incidentes atribuídos a "corrupção de dados" são, na verdade, transições não intencionais de CP para AP sob carga.
