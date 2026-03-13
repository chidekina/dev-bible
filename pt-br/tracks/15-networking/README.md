# Trilha 15 — Redes

Entenda como os dados trafegam pelas redes — desde bits físicos até protocolos de aplicação.

## Capítulos

| # | Capítulo | Tópicos |
|---|----------|----------|
| 1 | [Modelo OSI](./osi-model.md) | 7 camadas, responsabilidades, encapsulamento |
| 2 | [Pilha TCP/IP](./tcp-ip-stack.md) | Internos do TCP, handshake triplo, portas, sockets |
| 3 | [TCP vs UDP](./tcp-vs-udp.md) | Confiabilidade, controle de fluxo, casos de uso |
| 4 | [DNS, DHCP e NAT](./dns-dhcp-nat.md) | Resolução de nomes, atribuição de IP, tradução de endereços |
| 5 | [Versões do HTTP](./http-versions.md) | HTTP/1.1 vs HTTP/2 vs HTTP/3, QUIC |
| 6 | [TLS e Certificados](./tls-certificates.md) | Handshake TLS, PKI, mTLS, cadeias de certificados |
| 7 | [Firewalls e VPNs](./firewalls-vpn.md) | Filtragem de pacotes, inspeção stateful, túneis VPN |
| 8 | [Diagnóstico de Redes](./network-troubleshooting.md) | ping, traceroute, dig, netstat, curl, tcpdump |

## Pré-requisitos

- Linha de comando Linux básica (Trilha 05)
- Conceitos de redes no Docker (Trilha 05)

## Por que esta trilha importa

Cada requisição web, consulta a banco de dados e chamada entre microsserviços atravessa uma rede. Entender redes permite:
- Depurar problemas de latência e conexão
- Projetar arquiteturas seguras
- Configurar firewalls, proxies e balanceadores de carga
- Entender o que acontece quando algo falha
