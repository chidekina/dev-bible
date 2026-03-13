# Diagnóstico de Redes

Abordagem sistemática para diagnosticar problemas de rede — de "está acessível?" até "por que essa requisição HTTP está lenta?".

## Metodologia

```
1. Identifique o sintoma precisamente (timeout? recusado? dados errados?)
2. Estreite a camada: física → rede → transporte → aplicação
3. Teste conectividade em cada camada
4. Isole: é o cliente, a rede ou o servidor?
5. Confirme que a correção funcionou e entenda por quê
```

## Ferramentas Essenciais

### ping — Conectividade Camada 3

```bash
ping google.com
ping 8.8.8.8
ping -c 4 google.com

# Interpretar:
# ttl=117: partiu de 128 (servidor Windows) → 11 hops
# time=12.4ms: latência round-trip
# perda de pacotes: qualquer perda é preocupante em LAN

# Se falhar:
# Sem resposta: host inativo, ou ICMP bloqueado
# "Destination unreachable": sem rota
# "Name resolution failure": problema de DNS
```

### traceroute — Descoberta de Rota

```bash
traceroute google.com
tracepath google.com   # sem root

# Cada linha é um hop (roteador):
 1  192.168.1.1    0.4 ms   # seu roteador
 2  10.10.0.1      5.2 ms   # primeiro hop do ISP
 3  * * *                   # firewall bloqueia ICMP
 4  72.14.215.165  8.1 ms
 5  142.250.80.46  12.3 ms  # destino
```

### dig — Diagnóstico DNS

```bash
dig example.com.br
dig +short example.com.br
dig @8.8.8.8 example.com.br     # DNS Google
dig @1.1.1.1 example.com.br     # DNS Cloudflare
dig +trace example.com.br       # caminho completo
dig -x 93.184.216.34            # lookup reverso
```

### curl — Diagnóstico HTTP

```bash
# Requisição detalhada
curl -v https://example.com

# Breakdown de tempo
curl -w @- -o /dev/null -s https://example.com <<'EOF'
    time_namelookup:  %{time_namelookup}s
       time_connect:  %{time_connect}s
    time_appconnect:  %{time_appconnect}s
 time_starttransfer:  %{time_starttransfer}s
         time_total:  %{time_total}s
EOF

# time_namelookup: resolução DNS
# time_connect: handshake TCP completo
# time_appconnect: handshake TLS completo
# time_starttransfer: primeiro byte recebido (TTFB)
```

### ss / netstat — Inspeção de Sockets

```bash
# Listar todas as portas ouvindo
ss -tulnp
#  -t TCP  -u UDP  -l ouvindo  -n sem resolver nomes  -p processo

# Conexões TCP ativas
ss -tnp

# Contagem por estado
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# O que está usando a porta 3000?
ss -tulnp | grep :3000
lsof -i :3000
```

### tcpdump — Captura de Pacotes

```bash
# Capturar tráfego na porta 80
tcpdump -i any port 80

# Capturar tráfego de/para host específico
tcpdump -i any host 93.184.216.34

# Salvar para Wireshark
tcpdump -i any -w captura.pcap

# Ver conteúdo em ASCII
tcpdump -i any -A port 8080

# Capturar consultas DNS
tcpdump -i any port 53

# Capturar pacotes SYN (novas conexões)
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0'
```

### nc (netcat) — Teste de Conectividade Raw

```bash
# Testar se porta está aberta
nc -zv example.com 443
nc -zv -w 3 example.com 443  # timeout de 3s

# Servidor TCP simples (para testes)
nc -l 8080

# Enviar requisição HTTP manualmente
echo -e 'GET / HTTP/1.0\r\nHost: example.com\r\n\r\n' | nc example.com 80
```

## Cenários Comuns

### "Connection refused"

```bash
# Porta acessível mas nada ouvindo
nc -zv localhost 3000

# Depurar:
ss -tulnp | grep :3000   # algo está ouvindo?
ps aux | grep node        # processo rodando?
journalctl -u meuapp -f  # verificar logs
```

### "Connection timed out"

```bash
# Pacote sendo descartado (firewall) — sem RST
# Demora muito para falhar

# Depurar:
ping host.alvo            # host acessível?
traceroute host.alvo      # onde para?
iptables -L -n | grep <porta>  # regras de firewall?
```

### "Could not resolve hostname"

```bash
# Falha de DNS
dig google.com              # resolve normalmente?
dig @8.8.8.8 google.com    # funciona com DNS externo?
cat /etc/resolv.conf        # resolver configurado?
resolvectl status           # status do systemd-resolved
```

### Problemas de rede no Docker

```bash
# Container não acessa internet
docker exec meucontainer ping 8.8.8.8
docker exec meucontainer ping google.com

# Inspecionar rede do container
docker inspect meucontainer | jq '.[0].NetworkSettings'

# Container não acessível do host
docker ps                          # porta mapeada?
ss -tulnp | grep docker-proxy      # proxy ouvindo?
curl http://localhost:8080
```

## Debug de Latência

```bash
# Identificar fase lenta com timing do curl
curl -w "dns=%{time_namelookup} connect=%{time_connect} \
tls=%{time_appconnect} ttfb=%{time_starttransfer} \
total=%{time_total}\n" -o /dev/null -s https://api.exemplo.com

# dns alto → problema no resolver DNS
# connect alto → latência de rede, servidor distante
# tls alto → problema na cadeia de cert, OCSP lento
# ttfb alto → processamento lento no servidor
```

## Referência Rápida

```bash
# Conectividade
ping host                 # ICMP
traceroute host           # Rota até host
nc -zv host porta         # Teste de porta TCP

# DNS
dig host                  # Lookup DNS
dig +trace host           # Caminho completo
dig @8.8.8.8 host         # Consultar resolver específico

# Sockets
ss -tulnp                 # Portas ouvindo + processo
ss -tnp                   # Conexões ativas
lsof -i :porta            # O que usa esta porta?

# HTTP
curl -v https://host      # Requisição HTTP detalhada
curl -w timing...         # Breakdown de tempo

# Pacotes
tcpdump -i any port X     # Capturar na porta
tcpdump -i any host X     # Capturar para/de host
```

## Prática

1. Execute o comando de timing do curl em 3 sites diferentes. Compare DNS, connect e TLS.
2. Use `tcpdump -i any port 53` em um terminal e `dig google.com` em outro. Observe os pacotes.
3. Execute `ss -tnp` e identifique quais processos têm conexões abertas e para onde.
