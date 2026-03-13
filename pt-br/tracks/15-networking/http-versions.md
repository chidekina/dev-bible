# Versões do HTTP

O HTTP evoluiu significativamente. Entender as diferenças entre versões explica por que a performance web moderna é como é.

## HTTP/1.1 (1997 — ainda amplamente usado)

Melhorias principais:
- **Conexões persistentes**: reutiliza conexões TCP (`Connection: keep-alive`)
- **Transferência em chunks**: stream de respostas sem conhecer Content-Length
- **Header Host**: múltiplos domínios em um IP
- **Controle de cache**: `Cache-Control`, `ETag`, `If-Modified-Since`

### Limitações do HTTP/1.1

```
Bloqueio Head-of-Line:
Requisição 1 (grande) ─────────────────► Resposta 1 (lenta)
Requisição 2 (pequena) ─── aguardando ──► Resposta 2

Navegadores contornavam isso com 6 conexões TCP paralelas por domínio.
```

## HTTP/2 (2015)

Mesma semântica (métodos, headers, status codes), transmissão completamente diferente.

### Recursos Principais

**Framing binário**: usa frames binários em vez de texto — mais eficiente

**Multiplexing**: múltiplas requisições/respostas intercaladas em uma TCP

```
Stream 1: ─── H ─── D1 ─── D2 ───────────── END
Stream 3: ────────── H ─── D1 ────── END
Stream 5: ────────────────── H ─── D ─── END
          ─────────────────────────────► (uma conexão TCP)
```

**Compressão de headers (HPACK)**: headers comprimidos e cacheados entre requisições (redução típica de 70-85%)

**Server Push**: servidor envia recursos proativamente antes de serem solicitados

### HTTP/2 ainda tem bloqueio TCP

```
Se um pacote TCP for perdido, TODOS os streams HTTP/2 naquela
conexão travam até a retransmissão chegar.
```

## HTTP/3 (2022)

Substitui TCP por **QUIC** (construído sobre UDP).

### Recursos do QUIC

**Sem bloqueio HOL**: cada stream QUIC tem confiabilidade independente

```
Stream 1 perdeu pacote: só stream 1 trava
Streams 2, 3: não afetados
```

**TLS 1.3 integrado**:

```
HTTP/1.1 + TLS: = 3 round trips
HTTP/3 + QUIC:  = 1 round trip (0-RTT para conexões retomadas)
```

**Migração de conexão**: identificada por ID, não por IP:porta — sobrevive à troca de WiFi para 4G

## Resumo Comparativo

| Recurso | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Protocolo | Texto | Binário | Binário |
| Transporte | TCP | TCP | QUIC (UDP) |
| Multiplexing | Não | Sim | Sim |
| Bloqueio HOL | Sim | TCP apenas | Não |
| Compressão de headers | Não | HPACK | QPACK |
| Setup de conexão | 1–3 RTT | 1–3 RTT | 0–1 RTT |

## Métodos HTTP

| Método | Idempotente | Seguro | Body | Uso |
|--------|------------|--------|------|-----|
| GET | Sim | Sim | Não | Buscar |
| POST | Não | Não | Sim | Criar |
| PUT | Sim | Não | Sim | Substituir |
| PATCH | Não | Não | Sim | Atualizar parcialmente |
| DELETE | Sim | Não | Não | Remover |

## Prática

1. Abra o DevTools na aba Network. Filtre por protocolo e veja quais recursos usam h2 vs h3.
2. Execute `curl -v https://example.com 2>&1 | grep -E 'HTTP|<'` e leia os headers da resposta.
3. Explique por que o multiplexing do HTTP/2 não resolve completamente o problema de HOL.
