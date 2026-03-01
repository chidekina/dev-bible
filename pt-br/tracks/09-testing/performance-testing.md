# Testes de Performance

## Visão Geral

Testes de performance medem como um sistema se comporta sob carga. Eles respondem perguntas que os testes funcionais não conseguem: "A API consegue lidar com 500 requisições por segundo?" "O tempo de resposta piora com tráfego sustentado?" "A partir de qual ponto o sistema começa a lançar erros?" Este capítulo cobre os tipos de testes de performance, como escrever load tests com k6, como analisar os resultados e como definir SLOs que orientam quando agir.

---

## Pré-requisitos

- Track 09: Fundamentos de Testes
- Entendimento básico de APIs HTTP
- Familiaridade com padrões assíncronos

---

## Conceitos Fundamentais

### Tipos de testes de performance

| Tipo | O que mede | Duração típica |
|------|-----------|----------------|
| **Load test** | Tráfego normal esperado | 5–30 minutos |
| **Stress test** | Ponto de ruptura (aumenta até a falha) | 10–60 minutos |
| **Soak test** | Degradação ao longo do tempo (memory leaks, esgotamento do connection pool) | Horas |
| **Spike test** | Recuperação após pico súbito de tráfego | 5–15 minutos |
| **Smoke test** | Sanidade básica: a API consegue lidar com carga mínima? | 1–2 minutos |

### SLOs (Service Level Objectives)

Defina o que significa "bom" antes de executar os testes:

```
p95 tempo de resposta < 200ms sob 500 RPS
Taxa de erro < 0,1% sob carga normal
p99 tempo de resposta < 1s durante pico (2x tráfego normal)
Sistema volta ao baseline em até 5 minutos após o pico
```

Testes de performance validam SLOs; sem SLOs, os resultados dos testes são números sem significado.

---

## Exemplos Práticos

### Configuração do k6 e load test básico

```bash
# Instalar o k6 (Linux/macOS)
brew install k6          # macOS
# ou baixar de https://k6.io/docs/get-started/installation/
```

```javascript
// tests/load/api.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Métricas customizadas
const errorRate = new Rate('error_rate');
const responseTime = new Trend('response_time_ms');

export const options = {
  // Sobe para 100 VUs (usuários virtuais) em 30 segundos
  // Mantém 100 VUs por 1 minuto
  // Desce em 30 segundos
  stages: [
    { duration: '30s', target: 100 },
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 },
  ],

  // Teste falha se esses thresholds forem ultrapassados
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // 95% abaixo de 200ms, 99% abaixo de 500ms
    error_rate: ['rate<0.01'],                        // menos de 1% de taxa de erro
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('http://api.example.com/api/products', {
    headers: { Authorization: `Bearer ${__ENV.TEST_TOKEN}` },
  });

  // Registra métricas
  responseTime.add(res.timings.duration);
  errorRate.add(res.status !== 200);

  // Assertions
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has products array': (r) => JSON.parse(r.body).products !== undefined,
  });

  sleep(1); // tempo de pensar entre requisições
}
```

```bash
# Executar o load test
k6 run tests/load/api.js --env TEST_TOKEN=your-token

# Saída
✓ status is 200          1000/1000
✓ response time < 500ms  990/1000
✗ has products array     995/1000

checks.........................: 99.83% ✓ 2985 ✗ 5
data_received..................: 12 MB 120 kB/s
data_sent......................: 3.5 MB 35 kB/s
http_req_blocked...............: avg=1.2ms  p(95)=3.4ms
http_req_duration..............: avg=87ms   p(95)=180ms   p(99)=420ms
http_reqs......................: 10000  100/s
```

### Stress test — encontrando o ponto de ruptura

```javascript
// tests/load/stress.js
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // carga normal
    { duration: '5m', target: 100 },   // mantém no normal
    { duration: '2m', target: 500 },   // sobe para 5x
    { duration: '5m', target: 500 },   // mantém em 5x
    { duration: '2m', target: 1000 },  // sobe para 10x
    { duration: '5m', target: 1000 },  // mantém em 10x
    { duration: '5m', target: 0 },     // desce
  ],
  thresholds: {
    http_req_failed: ['rate<0.10'],    // threshold mais tolerante para stress
    http_req_duration: ['p(99)<2000'], // 99% abaixo de 2s
  },
};
```

### Soak test — encontrando memory leaks

```javascript
// tests/load/soak.js
export const options = {
  stages: [
    { duration: '5m', target: 100 },  // sobe
    { duration: '4h', target: 100 },  // mantém por 4 horas
    { duration: '5m', target: 0 },    // desce
  ],
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],
  },
};
```

Durante um soak test, fique de olho em:
- Memória aumentando gradualmente (leak no processo Node.js)
- Tempo de resposta aumentando gradualmente (esgotamento do connection pool)
- Erros 500 intermitentes (exceções não capturadas acumulando)

### Spike test

```javascript
// tests/load/spike.js
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // baseline
    { duration: '10s', target: 1400 }, // pico súbito
    { duration: '3m', target: 1400 },  // mantém o pico
    { duration: '10s', target: 100 },  // volta ao baseline
    { duration: '3m', target: 100 },   // período de recuperação
  ],
};
```

### k6 com múltiplos endpoints e cenários realistas

```javascript
// tests/load/realistic.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';

const BASE_URL = __ENV.BASE_URL ?? 'http://localhost:3000';

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '3m', target: 50 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    'group_duration{group:::browse products}': ['p(95)<300'],
    'group_duration{group:::checkout}': ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const token = __ENV.TEST_TOKEN;
  const headers = { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' };

  group('browse products', () => {
    const list = http.get(`${BASE_URL}/api/products?limit=20`, { headers });
    check(list, { 'list status 200': (r) => r.status === 200 });
    sleep(1);

    const products = JSON.parse(list.body).products;
    const product = products[Math.floor(Math.random() * products.length)];

    const detail = http.get(`${BASE_URL}/api/products/${product.id}`, { headers });
    check(detail, { 'detail status 200': (r) => r.status === 200 });
    sleep(2);
  });

  group('checkout', () => {
    const order = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify({ productId: 'prod-1', quantity: 1 }),
      { headers }
    );
    check(order, { 'order created': (r) => r.status === 201 });
    sleep(1);
  });
}
```

### Interpretando os resultados

```bash
# Métricas principais da saída do k6
http_req_duration................: avg=87ms min=12ms med=72ms max=1200ms p(90)=145ms p(95)=180ms p(99)=420ms

# p50 (mediana): 50% das requisições completam em 72ms ou menos
# p90: 90% completam em 145ms ou menos
# p95: 95% completam em 180ms ou menos  ← corresponde à sua meta de SLO
# p99: 99% completam em 420ms ou menos
# max: 1200ms — uma requisição foi muito lenta (investigue os outliers)
```

Monitore essas métricas em um dashboard durante o teste. Um pico no p99 enquanto o p50 permanece estável geralmente indica um problema de fila ou contenção de lock.

### Profiling do Node.js durante load tests

Enquanto o k6 está rodando, faça profiling do seu processo Node.js para encontrar gargalos:

```bash
# Inicia a API com profiling habilitado
node --prof src/index.js

# Após o load test, processa o perfil
node --prof-process isolate-*.log > profile.txt

# Ou use o clinic.js para uma interface mais amigável
npm install -g clinic
clinic doctor -- node src/index.js
# Depois execute o k6 contra ele
clinic flame -- node src/index.js
```

---

## Padrões Comuns e Boas Práticas

- **Defina SLOs antes de executar os testes** — p95 < 200ms, taxa de erro < 0,1%
- **Use cenários realistas** — mix de endpoints em proporções realistas, com tempo de pensar
- **Teste em um ambiente semelhante à produção** — staging com os mesmos recursos que produção
- **Monitore a infraestrutura durante os testes** — CPU, memória, conexões de DB, não apenas métricas HTTP
- **Execute smoke tests no CI** (1-2 minutos, poucos VUs) e load tests completos antes de releases importantes

---

## Anti-Padrões a Evitar

- Testar contra localhost — seu laptop não é um servidor; os resultados não se transferem
- Sem tempo de pensar entre requisições (`sleep`) — usuários reais não martelam APIs a 1000 RPS cada
- Ignorar taxas de erro e focar apenas no tempo de resposta
- Executar testes de performance em produção — você vai impactar usuários reais

---

## Depuração e Resolução de Problemas

**"Tempos de resposta sobem em intervalos específicos"**
Verifique pausas de garbage collection (GC do Node.js) ou jobs agendados (cron, processamento de fila) rodando durante o teste. Pausas de GC do Node.js podem ser investigadas com `--expose-gc` e monitoramento.

**"Conexões de DB se esgotam sob carga"**
O connection pool está muito pequeno ou as queries estão demorando muito. Adicione PgBouncer, aumente o tamanho do pool ou otimize queries lentas (veja Track 08: Database Scaling).

**"Métricas do k6 parecem boas mas usuários relatam lentidão"**
O k6 testa o servidor — latência de rede entre o usuário e o servidor não está incluída. Verifique tempos de CDN e resolução de DNS. Use o Lighthouse ou WebPageTest para performance real do usuário.

---

## Leitura Complementar

- [Documentação do k6](https://k6.io/docs/)
- [k6 thresholds e SLOs](https://k6.io/docs/using-k6/thresholds/)
- [clinic.js — profiling de Node.js](https://clinicjs.org/)
- [Google SRE Book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- Track 10: [Ferramentas de Profiling](../10-performance/profiling-tools.md)

---

## Resumo

Testes de performance consistem em definir SLOs e depois verificar se o sistema os atende sob carga realista. O k6 é a ferramenta certa para testes de performance de backend Node.js — baseado em scripts, compatível com CI e produz as métricas de percentil (p95, p99) que importam para conformidade com SLOs. Execute smoke tests no CI, load tests completos antes de releases e soak tests ao investigar problemas de memória ou connection pool. Sempre teste em um ambiente semelhante à produção com padrões de tráfego realistas, e monitore métricas de infraestrutura junto com métricas HTTP durante o teste.
