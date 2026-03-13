# Modelo OSI

O modelo OSI (Open Systems Interconnection) é um framework conceitual que padroniza a comunicação em rede em 7 camadas. Cada camada tem uma responsabilidade específica e se comunica apenas com as camadas diretamente acima e abaixo dela.

## As 7 Camadas

```
┌─────────────────────────────────────────┐
│  7 - Aplicação  (HTTP, DNS, SMTP)    │
├─────────────────────────────────────────┤
│  6 - Apresentação (TLS, encoding)    │
├─────────────────────────────────────────┤
│  5 - Sessão     (ger. de sessão)      │
├─────────────────────────────────────────┤
│  4 - Transporte (TCP, UDP)            │
├─────────────────────────────────────────┤
│  3 - Rede       (IP, roteamento)      │
├─────────────────────────────────────────┤
│  2 - Enlace     (Ethernet, MAC)       │
├─────────────────────────────────────────┤
│  1 - Física     (cabos, sinais)        │
└─────────────────────────────────────────┘
```

Mnemônico PT-BR: **F**icou **E**nergizado **R**ecebendo **T**oda **S**ua **P**rimeira **A**tividade

## Detalhamento das Camadas

### Camada 1 — Física
- Transmite bits brutos pelo meio físico
- Exemplos: cabos Ethernet, fibra ótica, ondas de rádio Wi-Fi
- Dispositivos: hubs, repetidores, placas de rede (NICs)
- Unidade: **bit**

### Camada 2 — Enlace de Dados
- Transferência de dados nó-a-nó no mesmo segmento de rede
- Adiciona endereços MAC e detecção de erros (CRC)
- Exemplos: quadros Ethernet, Wi-Fi (802.11), ARP
- Dispositivos: switches, bridges
- Unidade: **quadro (frame)**

### Camada 3 — Rede
- Endereçamento lógico e roteamento entre redes
- Determina o melhor caminho para os dados
- Exemplos: IP (v4/v6), ICMP, OSPF, BGP
- Dispositivos: roteadores
- Unidade: **pacote**

### Camada 4 — Transporte
- Comunicação ponta-a-ponta, segmentação, controle de fluxo
- Exemplos: TCP (confiável), UDP (rápido)
- Conceitos-chave: portas (0–65535), multiplexação
- Unidade: **segmento** (TCP) / **datagrama** (UDP)

### Camada 5 — Sessão
- Gerencia sessões entre aplicações
- Trata estabelecimento, manutenção e encerramento de conexões
- Na prática: frequentemente fundida com L4/L6 no TCP/IP

### Camada 6 — Apresentação
- Tradução, criptografia e compressão de dados
- Exemplos: TLS/SSL, compressão JPEG, codificação de caracteres (UTF-8)

### Camada 7 — Aplicação
- Mais próxima do usuário; fornece serviços de rede
- Exemplos: HTTP, HTTPS, DNS, SMTP, FTP, SSH, WebSocket
- Dispositivos: firewalls de aplicação, API gateways, CDNs
- Unidade: **dados / mensagem**

## Encapsulamento

Ao descer a pilha, cada camada **envolve** o payload com seu próprio cabeçalho (encapsulamento). Subindo, cada camada **desfaz** o envelope (desencapsulamento).

```
Enviando:
Dados da App
→ [Cabeçalho TCP | Dados da App]               (Segmento)
→ [Cabeçalho IP | TCP | Dados]                  (Pacote)
→ [Cabeçalho ETH | IP | TCP | Dados | CRC]      (Quadro)
→ 101010010111...                               (Bits)
```

## Modelo TCP/IP vs OSI

```
OSI                    TCP/IP
────────────           ────────────
7 Aplicação    ┐
6 Apresentação ├─ Aplicação
5 Sessão      ┘
4 Transporte   ── Transporte
3 Rede         ── Internet
2 Enlace       ┐
1 Física      ┘── Acesso à Rede
```

## Perguntas Comuns de Entrevista

**Q: Em qual camada opera um roteador?**
R: Camada 3 (Rede). Lê endereços IP para tomar decisões de roteamento.

**Q: Em qual camada opera um switch?**
R: Camada 2 (Enlace). Usa endereços MAC para encaminhar quadros na LAN.

**Q: Onde o TLS se encaixa?**
R: Conceitualmente na Camada 6 (Apresentação), mas no TCP/IP fica entre Transporte e Aplicação.

**Q: O que é um load balancer de Camada 7?**
R: Um que inspeciona conteúdo HTTP (headers, URL, cookies) para decisões de roteamento — vs um de Camada 4 que vê apenas IP/porta.

## Prática

1. Abra o Wireshark e capture uma requisição HTTP. Identifique os cabeçalhos de cada camada.
2. Execute `ping 8.8.8.8`. Em qual camada OSI o ping opera? (Dica: ICMP)
3. Explique por que um switch não precisa de endereço IP mas um roteador precisa.
