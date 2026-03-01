# Hardening de VPS

## Visão Geral

Um VPS recém-provisionado é uma vulnerabilidade. A configuração padrão prioriza acessibilidade em vez de segurança: a conta root aceita logins com senha, todas as portas estão acessíveis e nenhuma detecção de intrusão está rodando. Poucas horas após um novo VPS ficar online, ele já terá sido varrido por bots automatizados procurando senhas fracas e exploits conhecidos.

Hardening é o processo de reduzir sistematicamente a superfície de ataque de um servidor. Não torna um sistema invencível — torna a exploração difícil o suficiente para que os atacantes sigam em frente para alvos mais fáceis.

Este capítulo cobre as etapas essenciais de hardening para um VPS Ubuntu 22.04/24.04 em produção: gerenciamento de usuários, configuração de SSH, regras de firewall, prevenção de intrusões, atualizações automáticas e segurança em nível de aplicação para deploys Docker.

---

## Pré-requisitos

- Acesso root ou sudo a um VPS Ubuntu 22.04/24.04 recém-criado
- Um par de chaves SSH local (`ssh-keygen -t ed25519`)
- Conhecimento básico de linha de comando Linux

---

## Conceitos Fundamentais

### A Superfície de Ataque

Cada service exposto, porta aberta, conta habilitada e pacote instalado é um potencial ponto de entrada. O hardening elimina sistematicamente os desnecessários:

```
Redução da superfície de ataque:
  Contas    → desabilitar login root, usar SSH apenas com chave
  Rede      → firewall bloqueia tudo exceto 22/80/443
  Services  → desabilitar/remover daemons não utilizados
  Software  → atualizações de segurança automáticas
  Monitoramento → detectar e bloquear tentativas de força bruta
  Containers    → rodar como não-root, FS somente-leitura onde possível
```

### Defesa em Profundidade

Nenhum controle único é suficiente. Sobreponha múltiplas defesas para que contornar uma camada ainda deixe as outras de pé:

```
Camada 1: Firewall (UFW)          — bloquear tráfego indesejado
Camada 2: Hardening de SSH        — apenas chave, sem root, fail2ban
Camada 3: Hardening do SO         — pacotes mínimos, auto-updates
Camada 4: Hardening de aplicação  — containers não-root, sem portas de DB expostas
Camada 5: Monitoramento           — logs, alertas, detecção de anomalias
```

---

## Exemplos Práticos

### Passo 1: Criar um Usuário Não-Root

```bash
# Logar como root inicialmente
ssh root@YOUR_VPS_IP

# Criar usuário admin
adduser deploy
usermod -aG sudo deploy

# Configurar chave SSH para o usuário deploy
mkdir -p /home/deploy/.ssh
chmod 700 /home/deploy/.ssh

# Adicionar sua chave pública (da sua máquina local: cat ~/.ssh/id_ed25519.pub)
echo "ssh-ed25519 AAAA... your-key-comment" >> /home/deploy/.ssh/authorized_keys

chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh

# Testar antes de fechar a sessão root
# Abrir novo terminal:
ssh deploy@YOUR_VPS_IP
sudo whoami   # deve retornar "root"
```

### Passo 2: Fortalecer o SSH

`/etc/ssh/sshd_config` (ou criar `/etc/ssh/sshd_config.d/hardening.conf`):

```bash
# Criar arquivo de override (abordagem moderna — sobrevive a upgrades de pacote)
cat > /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# Desabilitar login root
PermitRootLogin no

# Autenticação apenas com chave — desabilitar senhas
PasswordAuthentication no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes

# Restringir a usuário(s) específico(s)
AllowUsers deploy

# Usar cifras e algoritmos modernos
Protocol 2
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org

# Configurações de conexão
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Desabilitar funcionalidades não utilizadas
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitEmptyPasswords no
PrintLastLog yes
EOF

# Testar a nova configuração
sshd -t

# Aplicar
systemctl restart sshd
```

**Crítico:** Antes de reiniciar o sshd, confirme que consegue logar com sua chave SSH em um terminal separado.

### Passo 3: Configurar Firewall UFW

```bash
# Instalar UFW (geralmente pré-instalado no Ubuntu)
apt install -y ufw

# Políticas padrão
ufw default deny incoming
ufw default allow outgoing

# Permitir SSH (faça isso antes de habilitar!)
ufw allow ssh          # ou: ufw allow 22/tcp

# Permitir tráfego web
ufw allow 80/tcp       # HTTP
ufw allow 443/tcp      # HTTPS

# Se precisar expor portas internas temporariamente (ex: para diagnósticos)
# ufw allow from YOUR.IP.ADDRESS to any port 5432

# Habilitar o firewall
ufw --force enable

# Verificar regras
ufw status verbose
```

Exemplo de saída:
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

### Passo 4: Instalar e Configurar Fail2Ban

O Fail2ban monitora arquivos de log e bane IPs que mostram comportamento malicioso (muitas tentativas de login falhas):

```bash
apt install -y fail2ban

# Criar override local (sobrevive a updates)
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
# Banir por 1 hora
bantime  = 3600
# Janela de tempo para contar falhas
findtime = 600
# Número de falhas antes do ban
maxretry = 5
# Backend (systemd para Ubuntu 22.04+)
backend = systemd

[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 86400  # banir força bruta SSH por 24 horas

[nginx-http-auth]
enabled  = true

[nginx-limit-req]
enabled  = true
EOF

systemctl enable fail2ban
systemctl start fail2ban

# Verificar status
fail2ban-client status
fail2ban-client status sshd
```

Verificar bans e desbanir:
```bash
# Ver bans atuais
fail2ban-client status sshd

# Desbanir um IP (se você acidentalmente se bloqueou)
fail2ban-client set sshd unbanip YOUR.IP.ADDRESS
```

### Passo 5: Atualizações Automáticas de Segurança

```bash
apt install -y unattended-upgrades apt-listchanges

# Configurar
cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

// Reiniciar se necessário (ex: atualizações de kernel)
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

// Notificação por e-mail (opcional)
// Unattended-Upgrade::Mail "admin@myapp.com";
EOF

cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
EOF

# Testar
unattended-upgrade --dry-run
```

### Passo 6: Desabilitar Services Não Utilizados

```bash
# Listar services em execução
systemctl list-units --type=service --state=running

# Services comuns para desabilitar em um servidor mínimo
systemctl disable --now snapd.service   # Daemon Snap (se não usar snaps)
systemctl disable --now avahi-daemon    # mDNS (não necessário em servidores)
systemctl disable --now cups            # Impressão (não em servidores)
systemctl disable --now bluetooth       # Bluetooth

# Verificar quais services estão escutando em portas de rede
ss -tlnp
# ou
netstat -tlnp
```

### Passo 7: Hardening de Kernel e Rede (sysctl)

```bash
cat > /etc/sysctl.d/99-security.conf << 'EOF'
# Prevenir IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Desabilitar redirecionamentos ICMP
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Desabilitar source routing de IP
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Registrar pacotes suspeitos
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Proteger contra ataques SYN flood
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Desabilitar IPv6 se não utilizado
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Não aceitar anúncios de roteadores
net.ipv6.conf.all.accept_ra = 0

# Proteger contra time-wait assassination
net.ipv4.tcp_rfc1337 = 1

# Randomizar espaço de endereço virtual (ASLR)
kernel.randomize_va_space = 2

# Desabilitar core dumps para programas setuid
fs.suid_dumpable = 0
EOF

# Aplicar imediatamente
sysctl -p /etc/sysctl.d/99-security.conf
```

### Passo 8: Segurança do Docker

```bash
# Configuração do daemon Docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "icc": false
}
EOF

systemctl restart docker
```

| Configuração | Propósito |
|-------------|-----------|
| `no-new-privileges` | Impede containers de ganhar privilégios adicionais |
| `live-restore` | Mantém containers rodando quando o daemon Docker reinicia |
| `userland-proxy` | Desabilitar para melhor performance (usar NAT do kernel) |
| `icc: false` | Desabilitar comunicação inter-container por padrão |

Segurança do Docker no compose.yml:
```yaml
services:
  api:
    image: myapi:latest
    # Nunca rodar como root
    user: "1000:1000"

    # Sistema de arquivos root somente-leitura (montar caminhos graváveis explicitamente)
    read_only: true
    tmpfs:
      - /tmp

    # Remover todas as capabilities e adicionar apenas o necessário
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # apenas se fizer bind em porta < 1024

    # Prevenir escalada de privilégio
    security_opt:
      - no-new-privileges:true

    # Limitar recursos
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
        reservations:
          memory: 128M
```

---

## Padrões e Boas Práticas

### Gerenciamento de Chaves SSH

```bash
# Gerar uma chave forte localmente
ssh-keygen -t ed25519 -C "deploy@myapp.com" -f ~/.ssh/myapp_deploy

# Adicionar ao authorized_keys do servidor
ssh-copy-id -i ~/.ssh/myapp_deploy.pub deploy@YOUR_VPS_IP

# Usar no SSH config (~/.ssh/config na sua máquina local)
Host myapp
  HostName YOUR_VPS_IP
  User deploy
  IdentityFile ~/.ssh/myapp_deploy
  IdentitiesOnly yes

# Conectar com:
ssh myapp
```

### Autenticação de Dois Fatores para SSH

```bash
# Instalar módulo PAM do Google Authenticator
apt install -y libpam-google-authenticator

# Configurar para o usuário deploy
su - deploy
google-authenticator   # seguir os prompts, salvar o QR code

# Configurar PAM
echo "auth required pam_google_authenticator.so" >> /etc/pam.d/sshd

# Atualizar sshd_config
echo "AuthenticationMethods publickey,keyboard-interactive" >> /etc/ssh/sshd_config.d/hardening.conf
sed -i 's/KbdInteractiveAuthentication no/KbdInteractiveAuthentication yes/' /etc/ssh/sshd_config.d/hardening.conf

systemctl restart sshd
```

### Port Knocking (opcional, para SSH)

Mudar o SSH para uma porta não-padrão para reduzir o ruído de logs:

```bash
# Mudar porta SSH (em sshd_config.d/hardening.conf)
Port 2222

# Atualizar UFW
ufw delete allow ssh
ufw allow 2222/tcp

# Atualizar seu ~/.ssh/config local
Host myapp
  Port 2222
```

Isso não melhora a segurança significativamente, mas reduz dramaticamente o ruído de logs de força bruta.

### /etc/hosts Imutável

Prevenir DNS rebinding e garantir resolução de hostname consistente:

```bash
# Adicionar ao /etc/hosts
127.0.0.1   localhost
YOUR_VPS_IP  myapp.com

# Tornar imutável (previne mudanças acidentais)
chattr +i /etc/hosts
```

---

## Anti-padrões a Evitar

### Desabilitar o Firewall "Só para Testar"

```bash
# Nunca faça isso em produção
ufw disable

# Se precisar permitir algo temporariamente, seja específico
ufw allow from YOUR.IP to any port 5432
# Remover quando terminar
ufw delete allow from YOUR.IP to any port 5432
```

### Manter SSH Root com Senha Habilitado

SSH root com senha é o vetor de ataque mais comum em VPS. Desabilite imediatamente:

```
PermitRootLogin no
PasswordAuthentication no
```

### Armazenar Chaves Privadas no Servidor

Nunca coloque sua chave SSH privada no VPS. O VPS guarda apenas chaves públicas no `authorized_keys`.

### Rodar Tudo como Root

Mesmo no Docker, rodar como root significa que uma fuga de container dá ao atacante root no host. Sempre use usuários não-root em containers.

### Não Rotacionar Secrets

Chaves SSH, tokens de API e senhas de banco de dados devem ser rotacionados periodicamente. Se uma chave for comprometida sem você saber, a rotação limita a janela de exposição.

---

## Debug & Troubleshooting

### Bloqueado via SSH

Se você acidentalmente se bloqueou:

1. A maioria dos provedores VPS oferece console web / acesso VNC — use-o
2. Do console, corrija o sshd_config e reinicie o sshd
3. Ou temporariamente reabilite a autenticação por senha, corrija o problema, desabilite novamente

```bash
# Do console web
sshd -T | grep -E "passwordauth|permitrootlogin|allowusers"
systemctl restart sshd
```

### Fail2ban Bloqueando Acesso Legítimo

```bash
# Verificar se seu IP está banido
fail2ban-client status sshd
iptables -L INPUT -n | grep YOUR.IP

# Desbanir
fail2ban-client set sshd unbanip YOUR.IP

# Adicionar seu IP à whitelist permanentemente
echo "ignoreip = 127.0.0.1/8 ::1 YOUR.IP" >> /etc/fail2ban/jail.local
systemctl restart fail2ban
```

### UFW Bloqueando Algo Inesperadamente

```bash
ufw status numbered           # ver regras com números
ufw status verbose            # saída detalhada
ufw delete 5                  # deletar regra pelo número

# Desabilitar temporariamente para testar se UFW é o problema
ufw disable
# ... testar ...
ufw enable
```

### Verificar Quem Tem Acesso

```bash
# Quem está logado atualmente
who
w

# Histórico de logins recentes
last -n 20
lastb -n 20   # tentativas de login falhas

# Verificar authorized_keys
cat /home/deploy/.ssh/authorized_keys

# Verificar acesso sudo
cat /etc/sudoers
ls -la /etc/sudoers.d/
```

### Auditoria de Segurança

```bash
# Instalar lynis para auditoria de segurança automatizada
apt install -y lynis
lynis audit system

# Fornecerá uma pontuação e recomendações específicas
```

---

## Cenários do Mundo Real

### Cenário 1: Resposta a uma Violação

Se você suspeitar de uma violação:

```bash
# 1. Verificar quem está logado atualmente
who
ps aux | grep -v "^\[" | sort -k2

# 2. Verificar comandos recentes
last -n 30
cat /root/.bash_history
cat /home/deploy/.bash_history

# 3. Verificar processos inesperados
ps aux | grep -E "nc|ncat|socat|python|perl|ruby" | grep -v grep

# 4. Verificar conexões de rede incomuns
ss -tlnp
ss -tnp | grep ESTABLISHED

# 5. Verificar arquivos modificados recentemente
find /etc /usr/bin /usr/sbin -newer /etc/passwd -type f 2>/dev/null
find /tmp /var/tmp -type f -newer /var/log/auth.log 2>/dev/null

# 6. Se violação confirmada: tirar snapshot, encerrar, reconstruir
```

### Cenário 2: Checklist de Hardening para Novo Servidor

```bash
#!/usr/bin/env bash
# Verificação rápida de hardening — rodar após configuração inicial

echo "=== Config SSH ==="
sshd -T | grep -E "^(passwordauth|permitrootlogin|allowusers|port)"

echo "=== Firewall ==="
ufw status

echo "=== Fail2ban ==="
systemctl is-active fail2ban

echo "=== Auto-updates ==="
systemctl is-active unattended-upgrades

echo "=== Daemon Docker ==="
cat /etc/docker/daemon.json

echo "=== Portas escutando ==="
ss -tlnp

echo "=== Últimos logins ==="
last -n 5
```

### Cenário 3: Rotacionar Chaves SSH

```bash
# 1. Gerar novo par de chaves na máquina local
ssh-keygen -t ed25519 -C "deploy-new-$(date +%Y%m)" -f ~/.ssh/myapp_new

# 2. Adicionar nova chave pública ao servidor (enquanto a chave antiga ainda funciona)
ssh deploy@myapp "echo '$(cat ~/.ssh/myapp_new.pub)' >> ~/.ssh/authorized_keys"

# 3. Testar que a nova chave funciona
ssh -i ~/.ssh/myapp_new deploy@myapp whoami

# 4. Remover chave antiga do servidor
ssh -i ~/.ssh/myapp_new deploy@myapp \
  "sed -i '/old-key-comment/d' ~/.ssh/authorized_keys"

# 5. Atualizar seu SSH config para usar a nova chave
```

---

## Leitura Complementar

- [Ubuntu Security Guide](https://ubuntu.com/security/certifications/docs/2204)
- [CIS Benchmarks for Ubuntu](https://www.cisecurity.org/benchmark/ubuntu_linux)
- [Lynis — ferramenta de auditoria de segurança](https://cisofy.com/lynis/)
- [Documentação do Fail2ban](https://www.fail2ban.org/wiki/index.php/MANUAL_0_8)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)

---

## Resumo

| Controle | Comando / Config |
|---------|-----------------|
| Usuário não-root | `adduser deploy && usermod -aG sudo deploy` |
| Apenas chaves SSH | `PasswordAuthentication no` no sshd_config |
| Desabilitar SSH root | `PermitRootLogin no` |
| Firewall | `ufw default deny incoming` + allow 22/80/443 |
| Proteção contra força bruta | `fail2ban` com `maxretry=3`, `bantime=86400` para SSH |
| Auto updates | `unattended-upgrades` com reboot às 3 AM |
| Hardening do kernel | sysctl: SYN cookies, ASLR, sem redirecionamentos ICMP |
| Segurança Docker | `no-new-privileges`, containers não-root, `icc=false` |
| Auditoria | `lynis audit system` — rodar mensalmente |

Segurança é um processo, não um estado. Faça o hardening no dia um, audite regularmente, rotacione credenciais e monitore os logs. O objetivo é ser um alvo mais difícil do que o outro VPS rodando configs padrão.
