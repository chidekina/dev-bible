# DNS, DHCP e NAT

Três protocolos que tornam a internet utilizável: DNS traduz nomes em endereços, DHCP atribui endereços automaticamente e NAT permite que muitos dispositivos compartilhem um único IP público.

## DNS — Sistema de Nomes de Domínio

### Hierarquia DNS

```
.                              ← Raiz
├── com.
│   └── google.com.
│       ├── www.google.com. → 142.250.80.4
└── br.
    └── aethostech.com.br.
```

### Processo de Resolução

```
1. Verifica cache local → miss
2. Pergunta ao resolver do SO (/etc/resolv.conf)
3. Resolver pergunta ao Servidor Raiz → "pergunte aos TLDs .com"
4. Resolver pergunta ao TLD .com → "pergunte aos NS de example.com"
5. Resolver pergunta ao NS → "example.com = 93.184.216.34"
6. Resolver armazena em cache (TTL) e retorna ao browser
```

### Tipos de Registro DNS

| Tipo | Finalidade | Exemplo |
|------|-----------|--------|
| A | Endereço IPv4 | `api.example.com → 93.184.216.34` |
| AAAA | Endereço IPv6 | `api.example.com → 2606::1` |
| CNAME | Alias | `www → example.com` |
| MX | Servidor de e-mail | `example.com → mail.example.com` |
| TXT | Texto livre | SPF, DKIM, verificação |
| NS | Servidor de nomes | Delega zona |
| PTR | DNS reverso | IP → hostname |

### TTL e Cache

```
TTL baixo (60s): mudanças propagam rápido, mais consultas DNS
TTL alto (3600s): menos consultas, propagação lenta

Antes de alterar DNS: reduza TTL para 60s com antecedência
```

### Diagnóstico DNS

```bash
dig example.com.br
dig @8.8.8.8 example.com.br    # consultar DNS específico
dig +trace example.com.br      # rastrear resolução completa
dig -x 93.184.216.34           # lookup reverso
```

## DHCP — Protocolo de Configuração Dinâmica de Hosts

### Processo DORA

```
Cliente                    Servidor DHCP
  │──── DISCOVER (broadcast) ►│  "Alguém tem IP para mim?"
  │◄─── OFFER ────────────────│  "Aqui: 192.168.1.50 por 24h"
  │──── REQUEST (broadcast) ──►│  "Quero esse IP"
  │◄─── ACK ──────────────────│  "É seu por 86400s"
```

**D**iscover → **O**ffer → **R**equest → **A**cknowledge

### Atribuição Estática vs Dinâmica

```
Dinâmica: servidor atribui do pool → dispositivos podem ter IPs diferentes
Reserva DHCP: servidor sempre atribui mesmo IP baseado no MAC
  → Melhor dos dois mundos: automático + previsível
```

## NAT — Tradução de Endereços de Rede

```
Rede Privada              Internet
192.168.1.10:54321  ─┐
192.168.1.11:54322  ─┤  Roteador/NAT  ──► 93.184.216.34:80
192.168.1.12:54323  ─┘  203.0.113.1

Tabela NAT:
 IP Privado:Porta        →  IP Público:Porta
 192.168.1.10:54321     →  203.0.113.1:1024
 192.168.1.11:54322     →  203.0.113.1:1025
```

### Tipos de NAT

- **SNAT**: muda endereço origem — usado para tráfego de saída
- **DNAT**: muda endereço destino — usado para redirecionamento de porta
- **MASQUERADE**: SNAT dinâmico para IPs dinâmicos

### NAT no Docker

```bash
# Docker cria rede privada: 172.17.0.0/16
# Containers recebem IPs privados
# Publicação de porta cria DNAT:
docker run -p 8080:80 nginx
# → :8080 externo → container 172.17.0.2:80
```

## ARP — Protocolo de Resolução de Endereços

```
Dispositivo sabe: IP destino = 192.168.1.1
Dispositivo precisa: endereço MAC

1. Broadcast: "Quem tem 192.168.1.1?"
2. Alvo responde: "192.168.1.1 está em AA:BB:CC:DD:EE:FF"

Ver tabela ARP:
ip neigh show   # ou: arp -n
```

## Prática

1. Execute `dig +trace google.com` e identifique cada etapa da resolução.
2. Execute `ip neigh show` para ver sua tabela ARP.
3. Verifique `ip addr show` — seu IP é atribuído dinamicamente? Confirme com `ip route show`.
