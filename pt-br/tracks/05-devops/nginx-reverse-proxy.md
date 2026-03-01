# Nginx & Reverse Proxy

## Visão Geral

Nginx (pronuncia-se "engine-x") é um servidor web de alta performance e reverse proxy. Em deploys modernos, ele fica na frente da sua aplicação e lida com preocupações que as aplicações não deveriam gerenciar diretamente: terminação SSL, roteamento de requisições, servir arquivos estáticos, load balancing, rate limiting e gerenciamento de conexões.

Um **reverse proxy** é um servidor que recebe requisições de clientes e as encaminha para um ou mais services de backend. Os clientes falam com o Nginx; o Nginx fala com a sua aplicação. Essa indireção provê segurança (esconde a topologia interna), flexibilidade (troca backends de forma transparente) e performance (lida com milhares de conexões concorrentes eficientemente).

Este capítulo cobre o Nginx como reverse proxy para aplicações Node.js/TypeScript em produção, incluindo HTTPS com Let's Encrypt, load balancing e integração com Docker.

---

## Pré-requisitos

- Um servidor Linux (Ubuntu 22.04 / 24.04 recomendado)
- Entendimento básico de HTTP, DNS e portas
- Docker e Docker Compose (para as seções integradas ao Docker)

```bash
# Instalar Nginx (Ubuntu)
sudo apt update
sudo apt install -y nginx

# Verificar
nginx -v
systemctl status nginx
```

---

## Conceitos Fundamentais

### Como Funciona um Reverse Proxy

```
Cliente (browser)
    |
    | HTTPS :443
    v
Nginx (reverse proxy)
    |
    | HTTP :3000 (interno)
    v
Aplicação Node.js
    |
    | TCP :5432 (interno)
    v
Banco PostgreSQL
```

O cliente nunca alcança o Node.js diretamente. Todo tráfego passa pelo Nginx, que:
1. Termina o TLS (descriptografa o HTTPS)
2. Valida a requisição
3. Encaminha para a aplicação upstream
4. Retorna a resposta

### Conceitos-Chave do Nginx

| Conceito | Descrição |
|---------|-----------|
| **server block** | Virtual host — lida com requisições para um domínio específico |
| **location block** | Roteia caminhos de URL específicos para diferentes handlers |
| **upstream** | Define um pool de servidores backend para proxying/load balancing |
| **proxy_pass** | Encaminha uma requisição para um servidor upstream |
| **listen** | Porta e protocolo que o server block aceita |
| **server_name** | Nome(s) de domínio que este block gerencia |

### Estrutura de Arquivos de Configuração

```
/etc/nginx/
  nginx.conf               ← config principal (inclui os outros)
  conf.d/                  ← configs de site drop-in
    myapp.conf
  sites-available/         ← configs de site disponíveis
    myapp
  sites-enabled/           ← symlinks para sites-available
    myapp -> ../sites-available/myapp
  snippets/                ← fragmentos de config reutilizáveis
    ssl-params.conf
```

Convenção Debian/Ubuntu: use `sites-available` + `sites-enabled`. Outras distros: use `conf.d/` diretamente.

---

## Exemplos Práticos

### Exemplo 1: Reverse Proxy Básico

`/etc/nginx/sites-available/myapp`:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;

    location / {
        proxy_pass http://localhost:3000;

        # Encaminhar informações reais do cliente para a aplicação
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }
}
```

Habilitar e testar:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t           # testar sintaxe da configuração
sudo systemctl reload nginx
```

### Exemplo 2: HTTPS com Let's Encrypt (Certbot)

```bash
# Instalar Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obter e instalar certificado (modifica a config do Nginx automaticamente)
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Verificar renovação automática
sudo certbot renew --dry-run
```

Configuração HTTPS manual (após certbot instalar o certificado):
```nginx
# Redirect HTTP → HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;
    return 301 https://$host$request_uri;
}

# Servidor HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.com www.myapp.com;

    # Certificado SSL (gerenciado pelo certbot)
    ssl_certificate     /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    # Configuração SSL moderna
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # HSTS (6 meses, incluir subdomínios)
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;

    # Headers de segurança
    add_header X-Content-Type-Options   "nosniff" always;
    add_header X-Frame-Options          "SAMEORIGIN" always;
    add_header X-XSS-Protection         "1; mode=block" always;
    add_header Referrer-Policy          "strict-origin-when-cross-origin" always;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # Configurações de buffer para performance
        proxy_buffering    on;
        proxy_buffer_size  4k;
        proxy_buffers      8 4k;
    }
}
```

### Exemplo 3: Servindo Arquivos Estáticos + API

```nginx
server {
    listen 443 ssl http2;
    server_name myapp.com;

    # ... config ssl ...

    root /var/www/myapp/dist;
    index index.html;

    # Assets estáticos — servir diretamente com cache agressivo
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # API — proxy para o backend
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Fallback para SPA — servir index.html para todas as rotas não-arquivo
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Exemplo 4: Load Balancing com Múltiplas Instâncias

```nginx
# Definir pool upstream
upstream api_pool {
    least_conn;   # enviar para o servidor com menos conexões ativas

    server localhost:3001;
    server localhost:3002;
    server localhost:3003;

    # Verificação de saúde (requer Nginx Plus, ou alternativa open-source)
    keepalive 32;  # manter conexões persistentes com os upstreams
}

server {
    listen 443 ssl http2;
    server_name myapp.com;

    location /api/ {
        proxy_pass http://api_pool;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Habilitar keepalive para upstream
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

Métodos de load balancing:
```nginx
upstream api_pool {
    # round_robin (padrão) — distribui igualmente
    server localhost:3001;
    server localhost:3002;

    # least_conn — menor número de conexões ativas
    least_conn;

    # ip_hash — mesmo cliente sempre vai para o mesmo servidor (session affinity)
    ip_hash;

    # weighted — servidor 1 recebe 3x mais tráfego que o servidor 2
    server localhost:3001 weight=3;
    server localhost:3002 weight=1;

    # backup — usado apenas se todos os servidores primários estiverem fora
    server localhost:3003 backup;
}
```

### Exemplo 5: Suporte a WebSocket

```nginx
server {
    listen 443 ssl http2;
    server_name myapp.com;

    location /ws/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host       $host;

        # Conexões WebSocket são de longa duração
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

### Exemplo 6: Rate Limiting

```nginx
# Definir zona de rate limit (10 MB de armazenamento, 10 requisições/segundo por IP)
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    listen 443 ssl http2;
    server_name myapp.com;

    # Aplicar rate limit geral
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://localhost:3000;
    }

    # Limite mais estrito em endpoints de autenticação
    location /api/auth/ {
        limit_req zone=login burst=3 nodelay;
        limit_req_status 429;
        proxy_pass http://localhost:3000;
    }
}
```

---

## Padrões e Boas Práticas

### Snippet de Headers de Proxy Reutilizável

`/etc/nginx/snippets/proxy-headers.conf`:
```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_http_version 1.1;
proxy_set_header Connection "";
```

Use em server blocks:
```nginx
location /api/ {
    include snippets/proxy-headers.conf;
    proxy_pass http://localhost:3000;
}
```

### Nginx no Docker com Alternativa Traefik

Para deploys Docker, o Nginx pode rodar como container:

```yaml
# compose.yml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - api
    networks:
      - frontend

  api:
    image: myapi:latest
    expose:
      - "3000"   # exposto apenas à rede Docker, não ao host
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
    internal: true
```

`nginx/conf.d/myapp.conf`:
```nginx
server {
    listen 80;
    server_name myapp.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.com;

    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    location / {
        proxy_pass http://api:3000;   # resolução por nome de service Docker
        include /etc/nginx/snippets/proxy-headers.conf;
    }
}
```

### Compressão Gzip

```nginx
# No bloco http do nginx.conf
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_proxied expired no-cache no-store private auth;
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/x-javascript
    image/svg+xml;
gzip_comp_level 6;
```

### Limite de Tamanho do Body do Cliente

```nginx
# O padrão é 1m — aumentar para uploads de arquivo
client_max_body_size 50m;
```

---

## Anti-padrões a Evitar

### Proxy para `0.0.0.0`

```nginx
# RUIM — conecta a todas as interfaces, ambíguo
proxy_pass http://0.0.0.0:3000;

# BOM — explícito
proxy_pass http://127.0.0.1:3000;
```

### Não Encaminhar IP Real

Sem `X-Real-IP` e `X-Forwarded-For`, os logs da sua aplicação mostrarão o IP do Nginx (`127.0.0.1`) em vez do IP real do cliente, quebrando analytics, rate limiting e logs de segurança.

Sua aplicação Node.js/Fastify também precisa estar configurada para confiar no proxy:
```typescript
// Fastify
const app = Fastify({
  trustProxy: true  // lê X-Forwarded-For
});
```

### Deixar a Config Padrão do Nginx Ativa

```bash
# Remover o site padrão que revela versão/config do Nginx
sudo rm /etc/nginx/sites-enabled/default
```

### Sem Timeouts Definidos

Sem timeouts, um upstream lento pode manter conexões do Nginx abertas indefinidamente, eventualmente esgotando as conexões dos workers:

```nginx
proxy_connect_timeout  10s;
proxy_send_timeout     60s;
proxy_read_timeout     60s;
```

### Expor Porta do Backend no Host

```yaml
services:
  api:
    # RUIM — expõe porta 3000 no host, bypassando o nginx
    ports:
      - "3000:3000"

    # BOM — apenas o nginx consegue alcançá-la
    expose:
      - "3000"
```

---

## Debug & Troubleshooting

### Testar Configuração

```bash
sudo nginx -t           # testar sintaxe
sudo nginx -T           # testar + fazer dump da config final
sudo nginx -T | grep -A5 "server_name myapp"
```

### Reload vs Restart

```bash
sudo systemctl reload nginx    # reload gracioso — sem downtime (use este)
sudo systemctl restart nginx   # mata todas as conexões — evitar em produção
```

### Verificar Logs

```bash
# Log de acesso (toda requisição)
sudo tail -f /var/log/nginx/access.log

# Log de erros (problemas)
sudo tail -f /var/log/nginx/error.log

# Filtrar por erros
sudo grep -i "error\|warn" /var/log/nginx/error.log | tail -50

# Verificar se o upstream está alcançável
curl -v http://localhost:3000/health
```

### Erros Comuns

**502 Bad Gateway** — Nginx não consegue alcançar o upstream:
```bash
# A aplicação está rodando?
systemctl status myapp
# Ou
docker compose ps

# O Nginx consegue alcançá-la?
curl http://localhost:3000/health
```

**413 Request Entity Too Large** — aumentar tamanho do body:
```nginx
client_max_body_size 50m;
```

**504 Gateway Timeout** — upstream demorou demais:
```nginx
proxy_read_timeout 120s;  # aumentar para operações lentas
```

**Erros de handshake SSL/TLS** — verificar certificado:
```bash
sudo certbot certificates
openssl s_client -connect myapp.com:443 -servername myapp.com
```

### Análise de Performance

```bash
# Verificar conexões ativas
nginx -s status   # se o módulo status estiver habilitado
# Ou via stub_status:
curl http://localhost/nginx-status

# Verificar processos worker
ps aux | grep nginx

# Monitorar log de acesso para respostas lentas
awk '{print $NF}' /var/log/nginx/access.log | sort -n | tail -20
```

---

## Cenários do Mundo Real

### Cenário 1: Servidor Multi-App (Múltiplos Domínios)

```nginx
# App 1: myapp.com → porta 3000
server {
    listen 443 ssl http2;
    server_name myapp.com;
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    location / { proxy_pass http://localhost:3000; include snippets/proxy-headers.conf; }
}

# App 2: api.myapp.com → porta 4000
server {
    listen 443 ssl http2;
    server_name api.myapp.com;
    ssl_certificate /etc/letsencrypt/live/api.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem;
    location / { proxy_pass http://localhost:4000; include snippets/proxy-headers.conf; }
}

# App 3: blog.myapp.com → porta 8080
server {
    listen 443 ssl http2;
    server_name blog.myapp.com;
    ssl_certificate /etc/letsencrypt/live/blog.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.myapp.com/privkey.pem;
    location / { proxy_pass http://localhost:8080; include snippets/proxy-headers.conf; }
}

# Redirecionar todo HTTP para HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}
```

### Cenário 2: Página de Manutenção

```nginx
# Verificar se o arquivo de manutenção existe, servir ele em vez de fazer proxy
location / {
    if (-f /var/www/maintenance.html) {
        return 503;
    }
    proxy_pass http://localhost:3000;
}

error_page 503 /maintenance.html;
location = /maintenance.html {
    root /var/www;
    internal;
}
```

```bash
# Ativar modo de manutenção
touch /var/www/maintenance.html
# ... fazer seu deploy ...
rm /var/www/maintenance.html
```

### Cenário 3: Autenticação Básica para Staging

```bash
# Criar arquivo de senhas
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd staging_user
```

```nginx
server {
    listen 443 ssl http2;
    server_name staging.myapp.com;

    auth_basic "Staging Environment";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://localhost:3001;
        include snippets/proxy-headers.conf;
    }
}
```

---

## Leitura Complementar

- [Documentação do Nginx](https://nginx.org/en/docs/)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — gera config TLS endurecida para Nginx
- [Documentação do Certbot](https://certbot.eff.org/docs/)
- [Nginx Performance Tuning](https://www.nginx.com/blog/tuning-nginx/)

---

## Resumo

| Conceito | Ponto Principal |
|---------|-----------------|
| Reverse proxy | Nginx fica na frente da sua aplicação — SSL, roteamento, headers |
| `proxy_pass` | Encaminha requisições para a sua aplicação upstream |
| `proxy_set_header` | Sempre encaminhe IP real do cliente e protocolo |
| HTTPS | Use Certbot + Let's Encrypt para certificados gratuitos com renovação automática |
| Headers de segurança | HSTS, X-Frame-Options, X-Content-Type-Options |
| Rate limiting | `limit_req_zone` protege sua aplicação de abusos |
| Load balancing | Bloco `upstream` com `least_conn` para distribuição equilibrada |
| WebSockets | Requer encaminhamento do header `Upgrade` e timeouts longos |
| Gzip | Habilite para JSON, HTML, JS, CSS — economia significativa de largura de banda |
| `nginx -t` | Sempre teste a config antes de recarregar |

Nginx é a parede de carga da infraestrutura web. Aprenda sua sintaxe de configuração, entenda o contrato de headers de proxy com sua aplicação e automatize a renovação de certificados desde o primeiro dia.
