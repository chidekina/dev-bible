# Linux Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Operações com Arquivos e Diretórios

```bash
# Navegação
pwd                        # exibir diretório atual
ls -lah                    # listagem longa, todos os arquivos, tamanhos legíveis
ls -lt                     # ordenar por hora de modificação
cd -                       # ir para o diretório anterior
pushd /some/dir            # empilhar diretório
popd                       # voltar ao diretório anterior

# Criar / deletar
mkdir -p /deep/path/dir    # criar com todos os pais
touch file.txt             # criar vazio ou atualizar timestamp
rm file.txt                # remover arquivo
rm -rf dir/                # remover diretório recursivamente (cuidado!)
rmdir dir/                 # remover diretório vazio

# Copiar / mover
cp file.txt dest/
cp -r src/ dest/           # cópia recursiva
cp -a src/ dest/           # modo arquivo (preserva permissões e timestamps)
mv old.txt new.txt         # mover ou renomear
mv dir/ /new/location/

# Links
ln -s /original /link      # link simbólico
ln /original /hardlink     # hard link

# Encontrar arquivos
find . -name "*.log"
find . -type f -size +100M
find . -mtime -7           # modificados nos últimos 7 dias
find . -name "*.js" -not -path "*/node_modules/*"
find . -name "*.tmp" -exec rm {} \;
```

---

## Visualizar Arquivos

```bash
cat file.txt               # imprimir arquivo completo
less file.txt              # visualizador paginado (q para sair)
head -n 20 file.txt        # primeiras 20 linhas
tail -n 50 file.txt        # últimas 50 linhas
tail -f /var/log/app.log   # acompanhar (stream ao vivo)
tail -F /var/log/app.log   # acompanhar + reabrir se rotacionado

wc -l file.txt             # contar linhas
wc -c file.txt             # contar bytes

file mystery.bin           # detectar tipo do arquivo
xxd file.bin | head        # dump hexadecimal
stat file.txt              # metadados detalhados do arquivo
```

---

## Busca: grep e ripgrep

```bash
grep "pattern" file.txt
grep -r "pattern" .        # recursivo
grep -i "pattern" file     # sem distinção de maiúsculas
grep -n "pattern" file     # mostrar números de linha
grep -v "pattern" file     # inverter (excluir correspondências)
grep -c "pattern" file     # contar linhas correspondentes
grep -l "pattern" *.txt    # apenas nomes de arquivo
grep -A 3 -B 3 "pattern" f # 3 linhas de contexto ao redor da correspondência
grep -E "regex|pattern" f  # regex estendida (ERE)
grep -P "\d{4}" f          # regex compatível com Perl

# ripgrep (mais rápido, respeita .gitignore)
rg "pattern"
rg -i "pattern" --type ts  # arquivos TypeScript, sem distinção de maiúsculas
rg -l "pattern"            # apenas nomes de arquivo
rg -n "pattern"            # com números de linha
```

---

## Processamento de Texto: sed e awk

```bash
# sed — editor de stream
sed 's/old/new/' file              # substituir primeira ocorrência por linha
sed 's/old/new/g' file             # substituir todas as ocorrências
sed 's/old/new/gi' file            # sem distinção de maiúsculas
sed -i 's/old/new/g' file          # edição no próprio arquivo
sed -i.bak 's/old/new/g' file      # no próprio arquivo com backup
sed -n '5,10p' file                # imprimir linhas 5-10
sed '/pattern/d' file              # deletar linhas que correspondem ao padrão
sed 's/^/  /' file                 # indentar todas as linhas
sed -E 's/(foo|bar)/baz/g' file    # regex estendida

# awk — processamento de colunas/campos
awk '{print $1}' file              # imprimir primeiro campo
awk '{print $NF}' file             # imprimir último campo
awk -F: '{print $1}' /etc/passwd   # delimitador customizado
awk '/pattern/{print}' file        # imprimir linhas correspondentes
awk '{sum += $2} END {print sum}'  # somar coluna 2
awk 'NR==5' file                   # imprimir linha 5
awk 'NR>=5 && NR<=10' file         # imprimir linhas 5-10
awk '{print NR, $0}' file          # adicionar números de linha
awk '!seen[$0]++'                  # deduplicar linhas

# Combinando
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -20
```

---

## Permissões

```bash
ls -l                      # exibir permissões
chmod 755 file             # rwxr-xr-x
chmod 644 file             # rw-r--r--
chmod +x script.sh         # adicionar execução para todos
chmod -R 755 dir/          # recursivo
chmod u+x,g-w,o-rwx file  # notação simbólica

chown user:group file      # alterar dono e grupo
chown -R user:group dir/   # recursivo
chgrp developers file      # alterar apenas o grupo

# Bits de permissão
# r=4  w=2  x=1
# 7=rwx  6=rw-  5=r-x  4=r--  0=---
# Dono Grupo Outros
# ex: 755 = rwxr-xr-x

# Bits especiais
chmod u+s file             # SUID — executar como dono
chmod g+s dir/             # SGID — herdar grupo
chmod +t /tmp              # sticky — apenas dono pode deletar

# umask (máscara padrão de permissões)
umask                      # exibir atual (ex: 022)
umask 027                  # novos arquivos: 640, diretórios: 750
```

---

## Processos

```bash
# Listagem
ps aux                     # todos os processos, detalhado
ps aux | grep nginx        # encontrar processo específico
pgrep nginx                # obter PID(s) pelo nome
pgrep -l nginx             # com nome do processo

# Monitoramento
top                        # visualizador interativo de processos
htop                       # visualizador interativo mais amigável
watch -n 2 'ps aux'        # atualizar a cada 2 segundos

# Sinais
kill PID                   # SIGTERM (encerramento gracioso)
kill -9 PID                # SIGKILL (forçar encerramento)
kill -HUP PID              # SIGHUP (recarregar configuração)
killall nginx              # matar pelo nome
pkill -f "node app.js"     # matar por correspondência de comando

# Segundo plano e primeiro plano
command &                  # executar em segundo plano
Ctrl+Z                     # suspender job em primeiro plano
bg                         # retomar em segundo plano
fg                         # trazer para primeiro plano
jobs                       # listar jobs em segundo plano
disown %1                  # desanexar job do shell

# Prioridade
nice -n 10 command         # iniciar com prioridade menor (-20 a 19)
renice -n 5 -p PID         # alterar prioridade de processo em execução

# Informações
lsof -p PID                # arquivos abertos pelo processo
lsof -i :3000              # processo na porta 3000
strace -p PID              # rastrear chamadas de sistema
```

---

## Disco e Memória

```bash
df -h                      # uso de disco por filesystem
df -h /                    # ponto de montagem específico
du -sh /var/log            # tamanho de diretório
du -sh *                   # tamanho de tudo no diretório atual
du -sh * | sort -h         # ordenado por tamanho

free -h                    # uso de memória
vmstat 1                   # estatísticas de memória virtual (a cada 1s)

lsblk                      # listar dispositivos de bloco
fdisk -l                   # listar partições de disco (root)
mount | column -t          # listar filesystems montados

# Encontrar arquivos grandes
find / -type f -size +1G 2>/dev/null
du -a /var | sort -rh | head -20
```

---

## Redes

```bash
# IP e interfaces
ip addr                    # exibir interfaces e IPs
ip addr show eth0          # interface específica
ip link                    # status da camada de link
ip route                   # tabela de roteamento

# DNS
dig example.com            # consulta DNS
dig example.com A          # registro A
dig @8.8.8.8 example.com   # usar servidor DNS específico
nslookup example.com       # consulta DNS alternativa
host example.com           # consulta simples

# Conectividade
ping -c 4 google.com       # 4 pings
traceroute google.com      # caminho hop a hop
mtr google.com             # combina ping + traceroute

# Scanning de portas e sockets
ss -tlnp                   # sockets TCP em escuta
ss -ulnp                   # sockets UDP em escuta
netstat -tlnp              # equivalente mais antigo
lsof -i :8080              # o que está na porta 8080?

# HTTP
curl -v https://example.com
curl -X POST -H "Content-Type: application/json" \
  -d '{"key":"val"}' https://api.example.com/endpoint
curl -o output.html https://example.com
curl -I https://example.com          # apenas headers
curl -u user:pass https://example.com
wget https://example.com/file.zip    # baixar arquivo

# Firewall (ufw)
ufw status
ufw allow 80/tcp
ufw allow from 192.168.1.0/24 to any port 5432
ufw deny 22
ufw enable
```

---

## Systemd

```bash
# Gerenciamento de serviços
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx          # recarregar config sem reiniciar
systemctl status nginx
systemctl enable nginx          # iniciar no boot
systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx

# Estado do sistema
systemctl list-units            # todas as units ativas
systemctl list-units --failed   # units com falha
systemctl daemon-reload         # recarregar arquivos de unit (após edição)
systemctl reboot
systemctl poweroff

# Logs (journald)
journalctl -u nginx             # logs do serviço nginx
journalctl -u nginx -f          # acompanhar
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -n 50       # últimas 50 linhas
journalctl --since today
journalctl -p err               # apenas erros
journalctl --disk-usage
journalctl --vacuum-time=7d     # limpar logs com mais de 7 dias
```

### Arquivo de Unit de Serviço

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node App
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node dist/index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/opt/myapp/.env

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now myapp
```

---

## Cron

```bash
crontab -e                  # editar crontab do usuário
crontab -l                  # listar crontab do usuário
crontab -r                  # remover crontab do usuário
crontab -u www-data -e      # editar crontab de outro usuário

# Todo o sistema
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

```
# Formato do cron: MIN HORA DOM MES DOW COMANDO
# MIN  0-59
# HORA 0-23
# DOM  1-31 (dia do mês)
# MES  1-12
# DOW  0-7  (0 e 7 = domingo)

0 * * * *        /script.sh    # toda hora no :00
0 2 * * *        /backup.sh    # diariamente às 2h
0 2 * * 0        /weekly.sh    # todo domingo às 2h
0 2 1 * *        /monthly.sh   # dia 1 do mês às 2h
*/15 * * * *     /check.sh     # a cada 15 minutos
0 9-17 * * 1-5   /job.sh       # dias úteis das 9h às 17h

# Strings especiais
@reboot          /startup.sh   # no boot
@daily           /daily.sh     # diariamente
@weekly          /weekly.sh    # semanalmente
```

---

## SSH

```bash
ssh user@host
ssh -p 2222 user@host             # porta customizada
ssh -i ~/.ssh/id_rsa user@host    # chave específica
ssh -J jump@bastion user@target   # host de salto (ProxyJump)
ssh -L 8080:localhost:80 user@host # redirecionamento de porta local
ssh -R 9090:localhost:3000 user@h  # redirecionamento de porta remota
ssh -N -f user@host -L 5432:db:5432 # túnel em segundo plano, sem shell

# Gerenciamento de chaves
ssh-keygen -t ed25519 -C "voce@exemplo.com"
ssh-copy-id user@host             # instalar chave pública no servidor
cat ~/.ssh/id_ed25519.pub | ssh user@host 'cat >> ~/.ssh/authorized_keys'

# Config SSH (~/.ssh/config)
Host myserver
  HostName 192.168.1.100
  User deploy
  Port 2222
  IdentityFile ~/.ssh/id_ed25519
  ServerAliveInterval 60
```

---

## Ambiente e Shell

```bash
# Variáveis
export VAR=value            # definir e exportar
echo $VAR                   # usar variável
unset VAR                   # remover
printenv                    # imprimir todas as variáveis de ambiente
env | grep NODE             # filtrar variáveis de ambiente

# PATH
export PATH="$HOME/.local/bin:$PATH"

# Redirecionamento
command > file              # stdout para arquivo (sobrescrever)
command >> file             # stdout para arquivo (acrescentar)
command 2> file             # stderr para arquivo
command &> file             # stdout + stderr para arquivo
command 2>&1 | tee file     # stdout + stderr para arquivo e tela
command < file              # stdin a partir de arquivo

# Pipes e substituição
cmd1 | cmd2                 # redirecionar stdout para o próximo
cmd1 && cmd2                # executar cmd2 apenas se cmd1 tiver sucesso
cmd1 || cmd2                # executar cmd2 apenas se cmd1 falhar
cmd1 ; cmd2                 # executar sequencialmente independente do resultado
$(command)                  # substituição de comando
<(command)                  # substituição de processo

# One-liners úteis
history | grep ssh          # buscar no histórico
Ctrl+R                      # busca reversa no histórico
!!                          # repetir último comando
!$                          # último argumento do comando anterior
!nginx                      # último comando que começa com nginx
```

---

## Gerenciamento de Pacotes

```bash
# Debian / Ubuntu (apt)
apt update
apt upgrade
apt install nginx
apt remove nginx
apt purge nginx             # remover + arquivos de configuração
apt autoremove              # remover pacotes órfãos
apt search nginx
apt show nginx
apt list --installed

# RHEL / CentOS (dnf/yum)
dnf install nginx
dnf remove nginx
dnf update
dnf search nginx

# Verificar o que fornece um comando
apt-file search /usr/bin/dig
which dig && dpkg -S $(which dig)
```
