# TCP vs UDP

TCP e UDP são os dois principais protocolos da camada de transporte.

## Comparação Lado a Lado

| Característica | TCP | UDP |
|----------------|-----|-----|
| Conexão | Orientado a conexão (handshake) | Sem conexão |
| Confiabilidade | Entrega garantida | Melhor esforço |
| Ordenamento | Entrega em ordem | Sem garantia |
| Controle de fluxo | Sim (janela deslizante) | Não |
| Velocidade | Mais lento (overhead) | Mais rápido |
| Tamanho do cabeçalho | 20–60 bytes | 8 bytes |
| Casos de uso | HTTP, email, SSH, bancos | DNS, streaming, jogos, VoIP |

## Bloqueio Head-of-Line do TCP

```
HTTP/1.1 sobre TCP:
Requisição 1 (grande) ──────────────────────► Resposta 1 (lenta)
Requisição 2 (pequena) ─── aguardando ────────► Resposta 2

HTTP/2 resolve no nível de aplicação (multiplexing),
mas o bloqueio TCP ainda existe se um pacote for perdido.
```

## Cabeçalho UDP

```
Cabeçalho UDP (apenas 8 bytes):
┌────────────┬────────────┐
│ Porta Ori. │ Porta Dst  │
├────────────┼────────────┤
│  Tamanho   │  Checksum  │
└────────────┴────────────┘
│   Dados...              │
```

## QUIC — O Melhor dos Dois

QUIC (usado no HTTP/3) é construído sobre UDP mas implementa:
- Entrega confiável por stream (não por conexão)
- TLS 1.3 integrado (setup de conexão em 0-RTT)
- Sem bloqueio head-of-line na camada de transporte
- Migração de conexão (funciona após troca de IP — ótimo para mobile)

```
HTTP/1.1: TCP + TLS 1.2 = 3 RTTs antes dos dados
HTTP/2:   TCP + TLS 1.3 = 2 RTTs antes dos dados
HTTP/3:   QUIC (UDP)    = 1 RTT (ou 0-RTT em conexões retomadas)
```

## Guia de Decisão

```
Precisa de entrega confiável?
  Sim → TCP (HTTP, SSH, bancos, email)
  Não → Latência é crítica?
          Sim → UDP (jogos, VoIP, streaming ao vivo)
          Não → TCP ainda (mais simples de implementar corretamente)

Precisa de multicast?
  Sim → UDP
```

## Prática

1. Use `tcpdump -i any port 53` e execute `dig google.com`. Confirme que DNS usa UDP.
2. Crie um servidor UDP simples em Node.js. Envie mensagem com `nc -u`.
3. Pesquise QUIC: por que 0-RTT existe e qual trade-off de segurança ele implica?
