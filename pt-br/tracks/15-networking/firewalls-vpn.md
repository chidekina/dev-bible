# Firewalls e VPNs

Firewalls controlam qual tráfego é permitido. VPNs criam túneis criptografados que protegem e roteiam tráfego.

## Firewalls

### Tipos de Firewall

**Filtro de pacotes (stateless)**: inspeciona cada pacote independentemente
```
Rápido mas não entende conexões — um pacote de resposta parece
idêntico a um pacote de ataque sem contexto.
```

**Inspeção stateful (rastreamento de conexão)**: rastreia conexões TCP
```
Sabe que TCP na porta 54321 pertence a uma conexão de saída
estabelecida — permite sem regra explícita.
```

**Firewall de camada de aplicação (L7)**: inspeciona conteúdo
```
Detecta: SQL injection em parâmetros HTTP
Bloqueia: URLs específicas, tipos de arquivo, métodos HTTP
Usado como: WAF (Web Application Firewall)
```

### iptables (Linux)

```bash
# Listar regras atuais
iptables -L -n -v

# Permitir conexões estabelecidas (stateful)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Permitir SSH de IP específico
iptables -A INPUT -p tcp -s 203.0.113.5 --dport 22 -j ACCEPT

# Permitir HTTP e HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Negar tudo por padrão (coloque por último)
iptables -P INPUT DROP

# Redirecionamento de porta (DNAT)
iptables -t nat -A PREROUTING -p tcp --dport 8080 \
  -j DNAT --to-destination 192.168.1.10:80

# Salvar regras
iptables-save > /etc/iptables/rules.v4
```

### UFW (Ubuntu)

```bash
ufw status
ufw enable
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny from 192.168.1.100
```

### Security Groups na Nuvem

```
AWS Security Group — Servidor Web:
Entrada:
  HTTP   :80   de 0.0.0.0/0
  HTTPS  :443  de 0.0.0.0/0
  SSH    :22   de 10.0.0.0/8 (interno apenas)
Saída:
  Todo tráfego

Diferenças do iptables:
- Stateful por padrão
- Apenas regras de permissão (sem negação explícita)
```

## VPNs — Redes Privadas Virtuais

### Como Funciona o Túneo VPN

```
Sem VPN:
Você → ISP → Internet → Servidor
     (texto visível ao ISP)

Com VPN:
Você → [Túneo criptografado] → Servidor VPN → Internet → Servidor
     (ISP vê apenas tráfego VPN)
```

### Protocolos VPN

**WireGuard** (moderno, recomendado):
```
- Código mínimo (~4000 linhas)
- Criptografia moderna (ChaCha20, Curve25519, BLAKE2)
- Muito rápido
- Integrado ao kernel Linux 5.6+
- Baseado em UDP (porta 51820)
```

**OpenVPN** (maduro, amplamente usado):
```
- Baseado em TLS, TCP ou UDP
- Muito configurável
- Mais lento que WireGuard
```

### Configuração WireGuard

```ini
# /etc/wireguard/wg0.conf (servidor)
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <chave-privada-servidor>

[Peer]
PublicKey = <chave-publica-cliente>
AllowedIPs = 10.8.0.2/32
```

```ini
# /etc/wireguard/wg0.conf (cliente)
[Interface]
Address = 10.8.0.2/24
PrivateKey = <chave-privada-cliente>
DNS = 1.1.1.1

[Peer]
PublicKey = <chave-publica-servidor>
Endpoint = 203.0.113.1:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
# Iniciar WireGuard
wg-quick up wg0

# Gerar par de chaves
wg genkey | tee chave-privada | wg pubkey > chave-publica
```

## Prática

1. Verifique quais portas estão abertas no seu VPS com `nmap -sS -p 1-1000 <ip-vps>` (de outra máquina).
2. Adicione regra UFW para permitir apenas seu IP via SSH. Teste, depois verifique que outros são bloqueados.
3. Leia o resumo do whitepaper do WireGuard — por que o código mínimo é vantagem de segurança?
