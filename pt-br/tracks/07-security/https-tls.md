# HTTPS & TLS

## Visão Geral

HTTPS é HTTP sobre TLS (Transport Layer Security). Ele oferece três garantias: confidencialidade (dados são criptografados em trânsito), integridade (dados não podem ser adulterados sem detecção) e autenticação (você está se comunicando com o servidor que pensa que está). Em 2025, servir qualquer coisa por HTTP puro é inaceitável — os navegadores o marcam como inseguro, mecanismos de busca o penalizam, e os usuários corretamente desconfiam. Este capítulo explica como o TLS funciona, como configurá-lo corretamente e como impô-lo.

---

## Pré-requisitos

- Compreensão básica de HTTP (requisições, respostas, headers)
- Familiaridade com DNS e configuração de servidor
- Node.js e Fastify básicos

---

## Conceitos Fundamentais

### O handshake TLS

Quando um navegador se conecta a `https://example.com`:

1. **ClientHello** — o cliente envia as versões TLS suportadas e cipher suites
2. **ServerHello** — o servidor escolhe a melhor cipher suite e envia seu certificado
3. **Verificação do certificado** — o cliente verifica se o certificado está assinado por uma CA confiável e se o domínio bate
4. **Troca de chaves** — cliente e servidor estabelecem um secret compartilhado (ECDHE é comum)
5. **Sessão estabelecida** — todos os dados subsequentes são criptografados com o secret compartilhado

O handshake completo leva ~1 RTT com TLS 1.3 (o padrão atual) vs. ~2 RTT com TLS 1.2.

### Tipos de certificado

| Tipo | Validação | Caso de uso |
|------|-----------|-------------|
| DV (Domain Validation) | Prova que você controla o domínio | A maioria dos sites — Let's Encrypt fornece esses gratuitamente |
| OV (Organization Validation) | Prova que sua organização é real | Sites corporativos |
| EV (Extended Validation) | Verificação completa de identidade | Bancos, serviços financeiros |

### Versões TLS

| Versão | Status |
|--------|--------|
| SSL 3.0, TLS 1.0, TLS 1.1 | Depreciadas — não use |
| TLS 1.2 | Aceitável, mas não preferida |
| TLS 1.3 | Padrão atual — use esta |

---

## Exemplos Práticos

### Let's Encrypt com Certbot (servidor Linux)

```bash
# Instale o certbot
sudo apt install certbot python3-certbot-nginx

# Obtenha um certificado para o seu domínio
sudo certbot --nginx -d example.com -d www.example.com

# O Certbot renova automaticamente via cron — verifique
sudo certbot renew --dry-run

# Os arquivos do certificado ficam em:
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

### Configuração TLS no Nginx (proxy reverso)

Em produção, encerre o TLS em um proxy reverso (Nginx, Traefik ou um load balancer) em vez de no Node.js. O proxy lida com TLS; o Node.js lida com HTTP em uma porta interna.

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    server_name example.com www.example.com;
    # Redireciona todo HTTP para HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Hardening TLS
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS — instrui navegadores a sempre usar HTTPS neste domínio
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # OCSP stapling — acelera a validação de certificado
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Traefik com Let's Encrypt automático (Docker)

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=you@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
      # Redirecionamento global de HTTP para HTTPS
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "letsencrypt:/letsencrypt"

  api:
    image: myapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=myresolver"
      - "traefik.http.services.api.loadbalancer.server.port=3000"

volumes:
  letsencrypt:
```

### HSTS no Node.js com Helmet

Quando você controla a camada HTTP (por exemplo, rodando Fastify diretamente sem proxy em desenvolvimento):

```typescript
import helmet from '@fastify/helmet';

await fastify.register(helmet, {
  hsts: {
    maxAge: 63072000,        // 2 anos em segundos
    includeSubDomains: true,
    preload: true,           // submete às listas de preload dos navegadores
  },
});
```

### Detectando requisições inseguras

Quando atrás de um proxy que encerra TLS, verifique `X-Forwarded-Proto` para redirecionar HTTP para HTTPS:

```typescript
fastify.addHook('onRequest', async (request, reply) => {
  if (
    process.env.NODE_ENV === 'production' &&
    request.headers['x-forwarded-proto'] === 'http'
  ) {
    const httpsUrl = `https://${request.hostname}${request.url}`;
    return reply.redirect(301, httpsUrl);
  }
});
```

### Certificate pinning (clientes mobile/API)

Certificate pinning vincula seu cliente a um certificado ou chave pública específicos, prevenindo ataques MITM mesmo com um certificado válido assinado por uma CA confiável.

```typescript
// Cliente HTTPS Node.js com pinning
import https from 'https';
import crypto from 'crypto';

const EXPECTED_PIN = 'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB='; // SHA-256 do SubjectPublicKeyInfo em base64

function createPinnedAgent(): https.Agent {
  return new https.Agent({
    checkServerIdentity: (host, cert) => {
      const pubKeyDer = cert.pubkey;
      const hash = crypto
        .createHash('sha256')
        .update(pubKeyDer)
        .digest('base64');

      const pin = `sha256/${hash}`;
      if (pin !== EXPECTED_PIN) {
        throw new Error(`Pin do certificado não confere: recebeu ${pin}`);
      }
      return undefined; // sem erro
    },
  });
}

const agent = createPinnedAgent();
const response = await fetch('https://api.example.com/data', { agent } as RequestInit);
```

---

## Padrões Comuns e Boas Práticas

- **Redirecione HTTP para HTTPS** com um redirect 301 permanente — nunca sirva conteúdo por HTTP puro
- **HSTS com preload** — assim que seu site estiver estável em HTTPS, submeta às listas de preload dos navegadores
- **Apenas TLS 1.3** — desative TLS 1.0 e 1.1 no nível do proxy reverso
- **Use cipher suites ECDHE** — proporcionam perfect forward secrecy para que sessões passadas não possam ser descriptografadas se a chave for comprometida futuramente
- **Habilite OCSP stapling** — acelera a validação de certificado e melhora a privacidade
- **Automatize a renovação de certificados** — certificados Let's Encrypt expiram a cada 90 dias; Certbot e Traefik lidam com a renovação automaticamente
- **Monitore a expiração de certificados** — configure alertas com 30 dias de antecedência como rede de segurança

---

## Anti-Padrões a Evitar

- Servir conteúdo na porta 80 sem redirecionar para HTTPS
- Usar certificados auto-assinados em produção (use Let's Encrypt — é gratuito)
- Ignorar a expiração de certificados até que os usuários reportem erros de conexão
- Definir HSTS com `maxAge` muito curto e `preload: true` — uma vez na lista de preload, remover o HSTS bloqueia os usuários
- Desabilitar a verificação de certificados em desenvolvimento com `NODE_TLS_REJECT_UNAUTHORIZED=0` e esquecer de remover

---

## Debugging e Resolução de Problemas

**Erros "Certificate expired"**
Verifique a expiração: `openssl s_client -connect example.com:443 | openssl x509 -noout -dates`
Certifique-se de que o cron de renovação está rodando: `sudo certbot renew --dry-run`

**"SSL handshake error" no Node.js**
Geralmente causado por versões TLS incompatíveis. Verifique se o servidor suporta TLS 1.2+. Para se conectar a um servidor legado: `NODE_OPTIONS=--tls-min-v1.0` (apenas contorno temporário).

**"HSTS preload check failed"**
Requisitos para preload: `max-age` deve ser pelo menos 31536000 (1 ano), `includeSubDomains` deve estar configurado, `preload` deve estar configurado, e o domínio deve redirecionar HTTP para HTTPS. Verifique em [hstspreload.org](https://hstspreload.org).

---

## Cenários do Mundo Real

**Cenário: Rotação de certificado sem downtime**

Certbot e Traefik lidam com isso automaticamente. Para configurações manuais:
1. Obtenha o novo certificado antes do antigo expirar
2. Atualize a configuração do Nginx/proxy para apontar para os novos arquivos de certificado
3. Recarregue o Nginx (`nginx -s reload`) — sem downtime, conexões existentes continuam

**Cenário: Certificado wildcard para subdomínios**

```bash
# Use o challenge DNS para certificados wildcard (requer acesso à API do provedor DNS)
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.cloudflare.ini \
  -d "*.example.com" \
  -d "example.com"
```

---

## Leitura Complementar

- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Documentação do Let's Encrypt](https://letsencrypt.org/docs/)
- [TLS 1.3 explained — Cloudflare blog](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- [Lista de preload HSTS](https://hstspreload.org/)
- Track 07: [Security Headers](security-headers.md)

---

## Resumo

HTTPS não é opcional em 2025. O setup prático é simples: encerre TLS em um proxy reverso (Nginx ou Traefik), use Let's Encrypt para certificados gratuitos com renovação automática, redirecione todo HTTP para HTTPS com um 301 e configure headers HSTS para impor HTTPS. TLS 1.3 é o padrão atual — desative 1.0 e 1.1 no nível do proxy. A parte mais difícil do HTTPS não é a configuração inicial, mas a disciplina operacional: monitorar a expiração de certificados, automatizar renovações e manter as configurações do proxy atualizadas à medida que as recomendações de cipher suites evoluem.
