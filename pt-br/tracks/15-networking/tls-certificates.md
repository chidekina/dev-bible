# TLS e Certificados

TLS (Transport Layer Security) fornece confidencialidade, integridade e autenticação para comunicação em rede. É o "S" no HTTPS.

## Por que TLS Importa

Sem TLS, qualquer um no caminho da rede pode:
- **Ler** seus dados (senhas, tokens, dados pessoais)
- **Modificar** dados em trânsito (injetar conteúdo)
- **Se passar** por servidores (site falso do banco)

## Handshake TLS 1.3

```
Cliente                         Servidor
  │──── ClientHello ─────────────►│
  │     (cifras suportadas,       │
  │      compartilhamento de      │
  │      chave, versão TLS)       │
  │◄─── ServerHello ─────────────│
  │     (cifra escolhida,         │
  │      compartilhamento chave,  │
  │      Certificado,             │
  │      Finished)                │
  │──── Finished ───────────────►│
  │    [Fluxo de dados cifrado]   │
```

### Forward Secrecy (Sigilo Futuro)

TLS 1.3 usa apenas **Diffie-Hellman Efêmero**: chaves de sessão descartadas após uso. Comprometer a chave privada do servidor não descriptografa sessões anteriores.

## PKI — Infraestrutura de Chaves Públicas

### Cadeia de Certificados

```
CA Raiz (auto-assinada, pré-instalada no SO/browser)
  └── CA Intermediária (assinada pela Raiz)
        └── Certificado do Servidor
              example.com, válido 2025-01-01 a 2026-01-01
```

### Conteúdo de um Certificado X.509

```
Subject:    CN=example.com
Issuer:     CN=Let's Encrypt R11
Válido:     2025-01-01 a 2025-04-01 (90 dias)
SANs:       example.com, www.example.com
Chave Púb: EC 256-bit (ECDSA)
```

### Etapas de Validação

```
1. Cadeia: certificado servidor → intermediária → raiz (confiável?)
2. Validade: data atual dentro do período?
3. Revogação: verificar OCSP ou CRL
4. Hostname: CN/SANs batem com o hostname solicitado?
```

## Let's Encrypt e ACME

```bash
# Desafio HTTP-01: prova controle do domínio
# ACME coloca token em /.well-known/acme-challenge/<token>
certbot certonly --webroot -w /var/www/html -d example.com.br

# Renovação automática (certificados duram 90 dias):
certbot renew --quiet

# Traefik/Caddy: gerenciam ACME automaticamente
```

## Inspeção de Certificados

```bash
# Ver detalhes do certificado
openssl s_client -connect example.com:443 </dev/null \
  | openssl x509 -noout -text

# Verificar data de expiração
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -dates
```

## mTLS — TLS Mútuo

No TLS padrão, apenas o servidor apresenta certificado. No mTLS, **ambos** são autenticados:

```
Usos: microsserviços (zero-trust), autenticação de APIs,
      VPNs, dispositivos IoT, pods Kubernetes (via Istio/Linkerd)
```

```bash
# Chamar API com certificado de cliente
curl --cert cliente.crt --key cliente.key \
     --cacert ca.crt \
     https://api.interno/endpoint
```

## Erros Comuns de TLS

| Erro | Causa | Solução |
|------|-------|----------|
| `certificate has expired` | Cert vencido | Renovar certificado |
| `certificate is not trusted` | CA desconhecida | Instalar CA, usar CA confiável |
| `hostname mismatch` | CN/SANs não batem | Obter cert com SANs corretos |
| `self-signed certificate` | Sem cadeia CA | Usar Let's Encrypt |

## Prática

1. Execute `openssl s_client -connect google.com:443` — identifique versão TLS e cadeia de certificados.
2. Encontre um site usando Let's Encrypt. Verifique o período de validade.
3. Explique o que forward secrecy significa e por que importa se a chave privada do servidor vazar.
