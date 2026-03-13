# Pilha TCP/IP

TCP (Transmission Control Protocol) rodando sobre IP (Internet Protocol) é a base da maioria das comunicações na internet.

## IP — Protocolo de Internet

O IP é responsável pelo **endereçamento** e **roteamento** de pacotes. É sem conexão e não confiável — faz melhor esforço sem garantias.

### Endereçamento IPv4 e CIDR

```
Formato: X.X.X.X onde X é 0–255
Exemplo: 192.168.1.100

Faixas privadas (RFC 1918 — não roteáveis na internet):
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16

Notação CIDR:
192.168.1.0/24
  └── 24 bits para rede, 8 bits para hosts
  → 254 endereços utilizáveis

Hosts utilizáveis = 2^(32-prefixo) - 2
  /24 → 254 hosts
  /16 → 65.534 hosts
```

## TCP — Protocolo de Controle de Transmissão

O TCP fornece entrega **confiável, ordenada e verificada** de dados.

### Portas

```
Portas conhecidas (0–1023):
  22   SSH        25  SMTP
  53   DNS        80  HTTP
  443  HTTPS    5432  PostgreSQL
  3306 MySQL    6379  Redis

Portas efêmeras (49152–65535): atribuídas dinamicamente aos clientes
```

### Handshake de Três Vias (Estabelecimento de Conexão)

```
Cliente                   Servidor
  │                          │
  │──── SYN (seq=x) ────────►│  Cliente inicia
  │◄─── SYN-ACK (seq=y,      │  Servidor confirma
  │       ack=x+1) ──────────│
  │──── ACK (ack=y+1) ──────►│  Cliente confirma
  │    [Conexão ABERTA]      │
```

### Encerramento em Quatro Vias

```
Cliente                   Servidor
  │──── FIN ────────────────►│  Cliente terminou de enviar
  │◄─── ACK ─────────────────│
  │◄─── FIN ─────────────────│  Servidor terminou
  │──── ACK ────────────────►│
  │    [Conexão FECHADA]     │
```

### Controle de Fluxo — Janela Deslizante

```
O receptor anuncia o tamanho da janela (buffer disponível).
O emissor não pode ter mais bytes não confirmados que o tamanho da janela.

Janela = 3 segmentos:
Enviados+Ack  Enviados+NãoAck  Podem Enviar  Aguardam
[  1  2  ]   [  3  4  5  ]    [ 6 7 8 ]     [ 9 10... ]
```

### Controle de Congestionamento

```
Slow Start: começa com janela pequena, dobra a cada RTT
Evitação de Congestionamento: cresce linearmente após limiar
Retransmissão Rápida: 3 ACKs duplicados → retransmite imediatamente
```

## Sockets

Socket é a abstração do SO para uma conexão de rede:

```
(ip_origem, porta_origem, ip_destino, porta_destino, protocolo)

Exemplo:
192.168.1.5:54321 ──TCP──► 93.184.216.34:443
```

## Diagnóstico Comum

```bash
# Ver todas as portas ouvindo e conexões ativas
ss -tulnp

# Testar se porta está aberta em host remoto
nc -zv exemplo.com.br 443

# Ver estados das conexões TCP
ss -s
```

## Problemas Comuns

| Sintoma | Causa Provável |
|---------|----------------|
| Connection refused | Nada ouvindo naquela porta |
| Connection timed out | Firewall descartando pacotes |
| Muitos TIME_WAIT | Alta rotatividade de conexões |
| RST recebido | Peer fechou conexão forçosamente |

## Prática

1. Execute `ss -tulnp` na sua máquina. Identifique o que está ouvindo nas portas comuns.
2. Use `nc -zv google.com 80` e `nc -zv google.com 443`. Qual a diferença?
3. Explique por que TIME_WAIT existe e que problemas removê-lo causaria.
