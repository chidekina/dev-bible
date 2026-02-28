# Linux & Shell

> O terminal é a ferramenta de poder do desenvolvedor. Proficiência com Linux e shell scripting aumenta dramaticamente sua eficiência: automatizar tarefas repetitivas, debugar servidores, gerenciar deploys e navegar bases de código complexas em alta velocidade.

---

## 1. O que é e por que importa

Linux alimenta a maioria dos servidores do mundo, infraestrutura em nuvem, sistemas de CI/CD e containers. MacOS compartilha boa parte da sua base POSIX. Entender o shell significa:
- Debugar problemas de produção em um servidor remoto sem GUI
- Escrever scripts de automação que rodam em pipelines de CI
- Gerenciar permissões de arquivo, processos e configurações de rede
- Usar ferramentas como Docker, SSH e logs do sistema com eficiência
- Construir memória muscular que te deixa dramaticamente mais rápido

---

## 2. Conceitos Fundamentais

### Filesystem Hierarchy Standard (FHS)

```
/                   root — o topo da árvore do sistema de arquivos
├── etc/            arquivos de configuração do sistema (nginx.conf, passwd, fstab)
├── var/            dados variáveis — conteúdo que muda em runtime
│   ├── log/        logs do sistema e de aplicações
│   ├── cache/      dados em cache (cache do apt, etc.)
│   └── lib/        estado persistente de aplicações
├── opt/            pacotes de software opcionais/de terceiros
├── home/           diretórios home dos usuários (/home/alice, /home/bob)
├── root/           diretório home do usuário root
├── usr/            utilitários e aplicações do usuário
│   ├── bin/        a maioria dos comandos do usuário (ls, grep, curl)
│   ├── lib/        bibliotecas compartilhadas para /usr/bin
│   └── local/      software compilado/instalado localmente
├── bin/            → symlink para /usr/bin em sistemas modernos
├── sbin/           → symlink para /usr/sbin — binários de administração do sistema
├── tmp/            arquivos temporários, limpos ao reiniciar
├── proc/           sistema de arquivos virtual — kernel expõe info de processos aqui
│   ├── 1234/       diretório por PID com mem, fd, status, etc.
│   └── cpuinfo     informações da CPU
├── dev/            arquivos de dispositivo (block devices, char devices)
│   ├── sda         primeiro disco SCSI/SATA
│   ├── null        buraco negro — descarta todas as escritas, EOF na leitura
│   └── random      fonte de números aleatórios
├── sys/            sistema de arquivos virtual — parâmetros do kernel e info de hardware
├── run/            dados em runtime (arquivos PID, sockets) — limpos ao reiniciar
└── mnt/            pontos de montagem temporários para sistemas de arquivos externos
```

---

## 3. Como Funciona

### Comandos Essenciais

**Navegação:**
```bash
pwd                    # imprime o diretório de trabalho atual
ls -la                 # lista todos os arquivos (incluindo ocultos) com permissões e tamanho
ls -lah                # igual, tamanhos de arquivo legíveis por humanos
cd /etc/nginx          # muda para caminho absoluto
cd ..                  # sobe um nível
cd -                   # vai para o diretório anterior
tree -L 2              # visualização em árvore, 2 níveis de profundidade
tree -L 3 --gitignore  # respeita .gitignore
```

**Operações com arquivos:**
```bash
cp file.txt backup.txt         # copia arquivo
cp -r src/ dst/                # copia diretório recursivamente
mv old.txt new.txt             # move/renomeia
rm file.txt                    # deleta arquivo
rm -rf dir/                    # deleta diretório recursivamente (sem confirmação)
mkdir -p path/to/nested/dir    # cria diretórios aninhados
touch file.txt                 # cria arquivo vazio ou atualiza timestamp
ln -s /path/to/target link     # cria link simbólico
stat file.txt                  # mostra metadados do arquivo (tamanho, permissões, timestamps)
```

**Visualizando arquivos:**
```bash
cat file.txt                   # imprime o arquivo inteiro
less file.txt                  # visualizador paginado (q para sair, / para buscar)
head -n 20 file.txt            # primeiras 20 linhas
tail -n 20 file.txt            # últimas 20 linhas
tail -f /var/log/app.log       # follow — imprime novas linhas conforme são escritas
tail -f -n 100 app.log         # follow últimas 100 linhas
wc -l file.txt                 # conta linhas
wc -c file.txt                 # conta bytes
```

**Buscando:**
```bash
# find — busca no sistema de arquivos
find . -name "*.log"                    # arquivos que casam com o padrão
find . -name "*.log" -mtime -7          # modificados nos últimos 7 dias
find . -name "*.ts" -not -path "*/node_modules/*"
find /var/log -size +100M               # arquivos maiores que 100MB
find . -type d -name ".git"             # apenas diretórios chamados .git
find . -empty                           # arquivos e diretórios vazios
find . -name "*.sh" -exec chmod +x {} \;  # executa comando em cada resultado

# grep — busca de conteúdo
grep -r "TODO" src/                     # busca recursivamente
grep -r "pattern" --include="*.ts"      # busca apenas em arquivos .ts
grep -r "pattern" --exclude-dir="node_modules"
grep -n "error" app.log                 # mostra números de linha
grep -v "DEBUG" app.log                 # linhas que NÃO casam (invertido)
grep -i "error" app.log                 # case-insensitive
grep -c "error" app.log                 # conta linhas que casam
grep -l "TODO" src/**/*.ts              # apenas nomes de arquivos (não as linhas)
grep -A 3 -B 3 "error" app.log         # 3 linhas de contexto antes e depois
grep -E "error|warn|fatal" app.log     # regex estendida (ERE)
```

---

### Processamento de Texto

```bash
# awk — processamento baseado em colunas
awk '{print $2}' file.txt              # imprime a segunda coluna (delimitada por espaços)
awk -F: '{print $1}' /etc/passwd       # primeira coluna, delimitada por dois-pontos
awk '$3 > 100 {print $1, $3}' data     # condicional: imprime colunas 1 e 3 se col 3 > 100
awk '{sum += $1} END {print sum}' nums # acumula soma

# sed — editor de stream (substituição é o uso mais comum)
sed 's/old/new/' file.txt              # substitui primeira ocorrência por linha
sed 's/old/new/g' file.txt             # substitui todas as ocorrências (global)
sed 's/old/new/gi' file.txt            # global, case-insensitive
sed -i 's/old/new/g' file.txt          # edição in-place (modifica o arquivo)
sed -i.bak 's/old/new/g' file.txt      # in-place, mantém backup como file.txt.bak
sed -n '10,20p' file.txt               # imprime linhas 10-20
sed '/pattern/d' file.txt              # deleta linhas que casam com o padrão

# cut — extrai colunas de texto delimitado
cut -d: -f1 /etc/passwd                # primeiro campo, delimitador de dois-pontos
cut -d',' -f2,4 data.csv               # campos 2 e 4 do CSV
cut -c1-10 file.txt                    # caracteres 1-10

# sort e uniq
sort file.txt                          # ordenação alfabética
sort -n file.txt                       # ordenação numérica
sort -rn file.txt                      # numérica reversa (maior primeiro)
sort -t: -k3 -n /etc/passwd            # ordena pelo 3º campo, delimitado por dois-pontos
sort -u file.txt                       # ordena e remove duplicatas (como sort | uniq)
uniq -c file.txt                       # prefixa linhas com a contagem
sort | uniq -c | sort -rn              # análise de frequência (linhas mais comuns)

# tr — traduz ou deleta caracteres
tr 'a-z' 'A-Z'                         # maiúsculas
tr -d '\r'                             # deleta retornos de carro (corrige quebras de linha do Windows)
tr -s ' '                              # comprime múltiplos espaços em um
```

---

### Pipes, Redirecionamento e xargs

```bash
# Pipes — envia stdout de um comando para stdin do próximo
ls -la | grep ".log"
cat app.log | grep "ERROR" | tail -20
ps aux | grep node | awk '{print $2}'  # obtém PIDs de processos node

# Redirecionamento
command > output.txt                   # stdout para arquivo (sobrescreve)
command >> output.txt                  # stdout para arquivo (acrescenta)
command 2> errors.txt                  # stderr para arquivo
command 2>&1                           # redireciona stderr para stdout (combina streams)
command > output.txt 2>&1             # tanto stdout quanto stderr para arquivo
command 2>/dev/null                    # descarta stderr
command > /dev/null 2>&1              # descarta toda a saída
command < input.txt                    # lê stdin de arquivo

# Here documents e here strings
cat << 'EOF' > config.txt
server {
  listen 80;
}
EOF

grep pattern <<< "alguma string para buscar"

# xargs — constrói comandos a partir do stdin
find . -name "*.log" | xargs rm                    # deleta arquivos encontrados
find . -name "*.log" | xargs -I{} cp {} /backup/  # copia cada um com substituição
echo "um dois três" | xargs -n1 echo              # um arg por comando
find . -name "*.ts" | xargs grep -l "TODO"        # busca em cada arquivo encontrado
cat urls.txt | xargs -P4 curl -sO                 # baixa 4 URLs em paralelo
```

---

### Permissões de Arquivo

Linux usa um modelo de três grupos: **dono**, **grupo**, **outros**.

```
-rwxr-xr--  1  alice  dev  4096  Jan 1  file.sh
 |||||||||||
 |└────────── permissões (usuário/grupo/outros)
 └─────────── tipo de arquivo (- = arquivo regular, d = diretório, l = symlink)

Detalhamento das permissões:
  rwx = leitura(4) + escrita(2) + execução(1) = 7
  r-x = leitura(4) + execução(1) = 5
  r-- = leitura(4) = 4

Então: rwxr-xr-- = 754
       rwxr-xr-x = 755  (comum para diretórios e scripts)
       rw-r--r-- = 644  (comum para arquivos)
       rw------- = 600  (arquivos de chave privada)
```

```bash
chmod 755 script.sh            # rwxr-xr-x
chmod +x script.sh             # adiciona execução para todos
chmod u+x script.sh            # adiciona execução apenas para o dono
chmod go-w file.txt            # remove escrita do grupo e outros
chmod -R 644 /var/www/         # define permissões recursivamente

chown alice file.txt           # muda o dono
chown alice:devteam file.txt   # muda dono e grupo
chown -R alice:web /var/www/   # recursivamente

# umask — máscara de permissões padrão para novos arquivos
umask                          # mostra umask atual (ex.: 022)
umask 022                      # novos diretórios: 755, novos arquivos: 644
# umask subtrai de 777 (diretórios) ou 666 (arquivos)
```

---

### Gerenciamento de Processos

```bash
# Visualizando processos
ps aux                         # todos os processos com usuário, PID, CPU, MEM
ps aux | grep node             # encontra processos node
ps -ef --forest                # visualização em árvore de processos
top                            # monitor interativo de processos (q para sair)
htop                           # monitor interativo melhor (se instalado)

# Matando processos
kill PID                       # envia SIGTERM (pedido educado para parar)
kill -9 PID                    # envia SIGKILL (força parada, não pode ser ignorado)
kill -15 PID                   # igual a kill (SIGTERM)
pkill -f "node server.js"      # mata por nome ou padrão de comando
killall node                   # mata todos os processos chamados "node"

# Plano de fundo e foreground
command &                      # roda em segundo plano
nohup command &                # roda em segundo plano, imune a hangup (persiste após logout)
nohup command > output.log 2>&1 &  # com redirecionamento de saída
Ctrl+C                         # interrompe (SIGINT) processo em foreground
Ctrl+Z                         # suspende processo em foreground
bg                             # retoma processo suspenso em segundo plano
fg                             # traz processo em segundo plano para foreground
jobs                           # lista jobs em segundo plano no shell atual

# Gerenciamento de serviço systemd
systemctl status nginx          # mostra status do serviço
systemctl start nginx           # inicia serviço
systemctl stop nginx            # para serviço
systemctl restart nginx         # reinicia serviço
systemctl reload nginx          # recarrega config sem reiniciar
systemctl enable nginx          # habilita no boot
systemctl disable nginx         # desabilita no boot
systemctl is-active nginx       # verifica se está rodando (exit code 0 = ativo)
journalctl -u nginx             # logs de um serviço
journalctl -u nginx -f          # segue logs
journalctl -u nginx --since "1 hour ago"
journalctl -n 100               # últimas 100 linhas de todos os serviços
```

---

### Comandos de Rede

```bash
# Requisições HTTP
curl https://api.example.com/data
curl -X POST -H "Content-Type: application/json" \
  -d '{"name":"Alice"}' https://api.example.com/users
curl -o file.zip https://example.com/file.zip    # baixa para arquivo
curl -I https://example.com                      # requisição HEAD (apenas headers)
curl -L https://example.com                      # segue redirecionamentos
curl -v https://example.com 2>&1 | head -50      # verboso (TLS, headers)
curl -s https://api.example.com | jq '.users[]'  # silencioso, passa para jq

wget https://example.com/file.zip                # downloader alternativo

# Inspeção de portas e sockets
ss -tlnp                                         # sockets TCP ouvindo com PID
ss -tulnp                                        # TCP + UDP ouvindo com PID
netstat -tlnp                                    # alternativa mais antiga ao ss
lsof -i :3000                                    # o que está usando a porta 3000
lsof -i -P -n | grep LISTEN                      # todos os sockets ouvindo

# DNS
dig google.com                                   # consulta DNS
dig google.com A                                 # apenas registros A
dig google.com +short                            # apenas o IP
dig @8.8.8.8 google.com                         # usa resolver específico
dig +trace google.com                            # percurso completo de resolução DNS
host google.com                                  # consulta DNS mais simples
nslookup google.com                              # ferramenta de consulta DNS mais antiga

# Diagnósticos de rede
ping -c 4 google.com                             # 4 pings ICMP
traceroute google.com                            # rota até o destino
mtr google.com                                   # ping + traceroute combinados

# Varredura
nmap -p 80,443 example.com                       # verifica portas específicas
nmap -p- localhost                               # todas as portas no localhost
```

---

### Uso de Disco

```bash
df -h                          # uso de disco em todos os sistemas de arquivos, legível
df -h /var                     # uso de disco para caminho específico
du -sh *                       # uso de disco dos itens no diretório atual
du -sh /var/log/               # tamanho total de um diretório
du -sh * | sort -h             # ordenado por tamanho (menor primeiro)
du -sh * | sort -rh            # ordenado por tamanho (maior primeiro)
du -sh /var/* | sort -rh | head  # 10 maiores itens em /var
lsblk                          # lista dispositivos de bloco e partições
```

---

### Shell Scripting

```bash
#!/bin/bash
# Sempre inicie com um shebang

# Modo estrito — sai em caso de erro, variáveis indefinidas e falhas em pipes
set -euo pipefail

# Variáveis
NAME="Mundo"
echo "Olá, $NAME"

# Substituição de comandos
DATE=$(date +%Y-%m-%d)
FILES=$(find . -name "*.log" | wc -l)
echo "Encontrei $FILES arquivos de log em $DATE"

# Condicionais
if [ -f "/etc/nginx/nginx.conf" ]; then
  echo "config do nginx existe"
elif [ -d "/etc/nginx" ]; then
  echo "diretório do nginx existe mas sem config"
else
  echo "nginx não encontrado"
fi

# Operadores de comparação para strings e números
# [ "$a" = "$b" ]   — igualdade de strings
# [ "$a" != "$b" ]  — desigualdade de strings
# [ -z "$a" ]       — string está vazia
# [ -n "$a" ]       — string não está vazia
# [ "$a" -eq "$b" ] — igualdade numérica
# [ "$a" -lt "$b" ] — menor que numérico
# [ "$a" -gt "$b" ] — maior que numérico

# Loops
for f in *.log; do
  echo "Processando $f"
  gzip "$f"
done

for i in {1..10}; do
  echo "Contagem: $i"
done

while read -r line; do
  echo "Linha: $line"
done < input.txt

# Arrays
FILES=("app.js" "server.js" "config.js")
echo "${FILES[0]}"         # primeiro elemento
echo "${#FILES[@]}"        # tamanho do array
for f in "${FILES[@]}"; do
  echo "$f"
done

# Funções
log_info() {
  echo "[INFO] $(date +%H:%M:%S) $*"
}

log_error() {
  echo "[ERROR] $(date +%H:%M:%S) $*" >&2  # stderr
}

backup_dir() {
  local src="$1"                              # variável local
  local dst="$2"
  local timestamp
  timestamp=$(date +%Y%m%d_%H%M%S)

  if [ ! -d "$src" ]; then
    log_error "Diretório de origem $src não existe"
    return 1                                 # exit code não-zero = falha
  fi

  mkdir -p "$dst"
  cp -r "$src" "$dst/${timestamp}_$(basename "$src")"
  log_info "Backup de $src para $dst feito"
}

# Verificando exit codes
backup_dir /var/app /backups
if [ $? -ne 0 ]; then                        # $? = exit code do último comando
  echo "Backup falhou!"
  exit 1
fi

# Ou use || para tratamento de erro inline
cp important.txt /backup/ || { echo "Cópia falhou"; exit 1; }

# Manipulação de strings
FILE="backup_20240115.tar.gz"
echo "${FILE%.tar.gz}"         # remove sufixo: backup_20240115
echo "${FILE#backup_}"         # remove prefixo: 20240115.tar.gz
echo "${FILE//backup/copy}"    # substitui todas: copy_20240115.tar.gz
echo "${FILE:7:8}"             # substring: 20240115
echo "${#FILE}"                # tamanho: 24
echo "${FILE^^}"               # maiúsculas (bash 4+)
```

---

### SSH

```bash
# Geração de chave
ssh-keygen -t ed25519 -C "alice@example.com"   # gera chave Ed25519 (recomendado)
ssh-keygen -t rsa -b 4096 -C "alice@example.com" # RSA 4096 (compatibilidade com mais antigos)
# Chaves criadas em ~/.ssh/id_ed25519 e ~/.ssh/id_ed25519.pub

# Copia chave pública para o servidor
ssh-copy-id alice@server.example.com
# ou manualmente:
cat ~/.ssh/id_ed25519.pub | ssh alice@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Conexão básica
ssh alice@server.example.com
ssh -p 2222 alice@server.example.com          # porta não padrão
ssh -i ~/.ssh/custom_key alice@server.example.com  # especifica chave

# ~/.ssh/config — aliases e opções
# Host meuservidor
#   HostName server.example.com
#   User alice
#   Port 2222
#   IdentityFile ~/.ssh/id_ed25519
#   ServerAliveInterval 60
#
# Após config:
ssh meuservidor

# Tunelamento SSH
# Port forwarding local — acessa serviço remoto localmente
ssh -L 5432:localhost:5432 alice@server     # acessa postgres do servidor localmente
ssh -L 8080:internal-server:80 alice@bastion  # acessa servidor interno via bastion

# Port forwarding remoto — expõe porta local no remoto
ssh -R 8080:localhost:3000 alice@server    # :8080 do servidor → :3000 seu

# Jump hosts (-J substitui ProxyJump)
ssh -J bastion.example.com alice@internal.server.local

# Transferência de arquivos
scp file.txt alice@server:~/              # copia para home remota
scp -r dir/ alice@server:~/backup/       # copia diretório
scp alice@server:~/file.txt .            # copia do remoto

# rsync — melhor para transferências grandes ou repetidas
rsync -avz --progress src/ alice@server:~/dst/
rsync -avz --delete --exclude="node_modules" ./app/ alice@server:/opt/app/
# -a = archive (preserva permissões, timestamps, symlinks)
# -v = verbose
# -z = comprime
# --delete = remove no destino arquivos que não existem na origem
```

---

## 4. Exemplos de Código

### Script de backup com timestamp
```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${1:?Uso: $0 <dir_origem> [dir_destino]}"
DEST_DIR="${2:-/backups}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="$(basename "$BACKUP_DIR")_${TIMESTAMP}"
ARCHIVE="${DEST_DIR}/${BACKUP_NAME}.tar.gz"
MAX_BACKUPS=10

log() { echo "[$(date '+%H:%M:%S')] $*"; }
die() { echo "[ERROR] $*" >&2; exit 1; }

[ -d "$BACKUP_DIR" ] || die "Diretório de origem '$BACKUP_DIR' não encontrado"
mkdir -p "$DEST_DIR" || die "Não é possível criar diretório de backup '$DEST_DIR'"

log "Fazendo backup de $BACKUP_DIR → $ARCHIVE"
tar -czf "$ARCHIVE" -C "$(dirname "$BACKUP_DIR")" "$(basename "$BACKUP_DIR")"
log "Tamanho do arquivo: $(du -sh "$ARCHIVE" | cut -f1)"

# Rotaciona backups antigos — mantém apenas MAX_BACKUPS
log "Rotacionando backups antigos (mantendo $MAX_BACKUPS)"
ls -t "$DEST_DIR"/*.tar.gz 2>/dev/null | tail -n +"$((MAX_BACKUPS + 1))" | xargs -r rm
log "Concluído. $(ls "$DEST_DIR"/*.tar.gz | wc -l) backups mantidos."
```

### Monitorar um arquivo de log e alertar em erros
```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="${1:?Uso: $0 <arquivo_log>}"
ALERT_PATTERN="${2:-ERROR}"
ALERT_CMD="${3:-echo}"  # substitua por: 'curl -s -X POST https://hooks.slack.com/...'

[ -f "$LOG_FILE" ] || { echo "Arquivo de log não encontrado: $LOG_FILE"; exit 1; }

echo "Monitorando $LOG_FILE por '$ALERT_PATTERN' ..."

tail -f -n 0 "$LOG_FILE" | while IFS= read -r line; do
  if echo "$line" | grep -qi "$ALERT_PATTERN"; then
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    MESSAGE="[$TIMESTAMP] ALERTA em $LOG_FILE: $line"
    $ALERT_CMD "$MESSAGE"
  fi
done
```

---

## 5. Erros Comuns e Armadilhas

> **`rm -rf /` (espaços importam!)**: `rm -rf /important-dir` é seguro (deleta esse diretório). `rm -rf / important-dir` (espaço extra) deleta o root primeiro! Sempre verifique os caminhos. Use `--` para separar opções de nomes de arquivos: `rm -rf -- "$dir"`.

> **Não colocar aspas em variáveis**: Variáveis sem aspas passam por word splitting e expansão de globs. `rm $file` falha ou faz coisas inesperadas se `$file` contiver espaços ou wildcards. Sempre use aspas: `rm "$file"`.
> ```bash
> # Bug — word splitting em espaços no nome do arquivo
> file="meu arquivo importante.txt"
> rm $file     # tenta deletar "meu", "arquivo", "importante.txt"
> rm "$file"   # correto — trata como argumento único
> ```

> **Não usar `set -euo pipefail`**: Sem `set -e`, scripts continuam rodando após erros. Sem `set -u`, variáveis não definidas se expandem silenciosamente para string vazia (fazendo `rm -rf "$DIR/"` virar `rm -rf /` se `$DIR` não estiver definido). Sem `set -o pipefail`, `false | true` sai com código 0.

> **Ignorar exit codes**: Todo comando sai com um código (0 = sucesso, não-zero = falha). Em scripts, verifique exit codes com `$?`, `if command`, ou `command || handle_error`.

> **Usar `cp -r` quando `rsync` seria melhor**: Para diretórios grandes, `rsync` é mais rápido (só transfere arquivos alterados), lida com transferências parciais e preserva permissões melhor. Use rsync para deploys e backups.

> **Hardcodar caminhos em vez de usar variáveis**: Use variáveis para caminhos em scripts. Se o caminho mudar, você atualiza apenas uma linha.

---

## 6. Quando Usar / Não Usar

**Use shell scripts para:**
- Orquestrar outros comandos (código de cola)
- Tarefas de automação que são fundamentalmente manipulação de arquivos e processos
- Etapas de pipeline de CI/CD
- Processamento rápido de dados ad-hoc

**NÃO use shell scripts para:**
- Lógica complexa com estruturas de dados, tratamento de erros ou regras de negócio (use Python/Node.js)
- Qualquer coisa que precise de testes unitários (frameworks de teste para shell existem mas são dolorosos)
- Quando você se encontrar escrevendo mais de ~100 linhas de lógica não trivial

**Use `find` quando:**
- Precisa de filtragem baseada em sistema de arquivos (por tipo, tamanho, data, permissões)
- Combinado com `-exec` ou `xargs` para operações em lote

**Use `grep` quando:**
- Buscando conteúdo de arquivos
- Filtrando saída de comandos

**Use `awk` quando:**
- Processando colunas de saída de texto estruturado (linhas de log, CSV, saída do `ps aux`)

**Use `sed` quando:**
- Substituição de stream em arquivos ou pipelines

---

## 7. Cenário Real

### Diagnosticando um problema em servidor de produção

Você recebe um alerta às 2h da manhã: "App está lento, algumas requisições estão dando timeout."

```bash
# 1. Verifica se o serviço está rodando
systemctl status app-server
journalctl -u app-server -n 50

# 2. Verifica CPU e memória
top
# Pressione 'c' para ver comando completo, 'M' para ordenar por memória, 'P' para CPU

# 3. Verifica espaço em disco (causa comum)
df -h
du -sh /var/log/* | sort -rh | head -10

# 4. Verifica arquivos de log grandes consumindo disco
find /var/log -size +500M -ls

# 5. Verifica quais portas estão em uso
ss -tlnp | grep :3000

# 6. Verifica excesso de conexões abertas
ss -s  # estatísticas de conexão
lsof -i :3000 | wc -l  # quantas conexões na porta 3000

# 7. Verifica erros recentes da aplicação
tail -100 /var/log/app/error.log | grep -i "error\|warn\|fatal"

# 8. Verifica logs do sistema por OOM killer
journalctl -k | grep -i "killed\|oom"
dmesg | grep -i "killed\|oom\|memory"

# 9. Verifica conexões do postgres (se aplicável)
ss -tlnp | grep :5432
# De dentro do psql:
# SELECT count(*), state FROM pg_stat_activity GROUP BY state;

# Causa raiz encontrada: disco 95% cheio por logs de acesso não rotacionados
# Correção:
find /var/log/app -name "access.log.*" -mtime +30 -delete
gzip /var/log/app/access.log
# Depois configure logrotate para rotação automática
```

---

## 8. Perguntas de Entrevista

**Q1: Como você encontra todos os arquivos modificados nas últimas 24 horas?**

R: `find . -mtime -1 -type f`. O `-mtime -1` significa "tempo de modificação menor que 1 dia atrás". Para horas: `find . -mmin -60` (modificados nos últimos 60 minutos). Para ver os tempos de modificação também: `find . -mtime -1 -type f -ls`. Para encontrar arquivos e fazer algo com eles: `find . -mtime -1 -name "*.log" -exec gzip {} \;` ou `find . -mtime -1 | xargs ls -la`.

---

**Q2: O que significa `2>&1`?**

R: Redireciona o file descriptor 2 (stderr) para onde o file descriptor 1 (stdout) está apontando atualmente. Quando você escreve `command > output.txt 2>&1`, stdout é redirecionado para o arquivo primeiro, depois stderr é redirecionado para o mesmo lugar que stdout (o arquivo). A ordem importa: `command 2>&1 > output.txt` redirecionaria stderr para o terminal (onde stdout estava antes do redirecionamento) e stdout para o arquivo. `/dev/null` é um dispositivo especial que descarta tudo que é escrito nele: `command > /dev/null 2>&1` silencia toda a saída.

---

**Q3: Explique permissões de arquivo no Linux.**

R: Todo arquivo tem três grupos de permissão: dono (user), grupo e outros. Cada grupo tem três bits de permissão: leitura (r=4), escrita (w=2), execução (x=1). A permissão numérica é a soma: rwx=7, r-x=5, r--=4. `chmod 755` define dono=rwx(7), grupo=r-x(5), outros=r-x(5). Para diretórios, execução significa "pode entrar e listar". Para arquivos, execução significa "pode rodar como programa". `chown user:group file` muda a propriedade. `umask` subtrai do padrão (666 para arquivos, 777 para diretórios) ao criar novos arquivos.

---

**Q4: Como funciona autenticação por chave SSH?**

R: Você tem um par de chaves: uma chave privada (fica na sua máquina, nunca compartilhada) e uma chave pública (colocada em `~/.ssh/authorized_keys` no servidor). Quando você se conecta, o servidor envia um desafio criptografado com sua chave pública. Apenas sua chave privada pode descriptografá-lo. Seu cliente assina o desafio com sua chave privada e envia de volta. O servidor verifica a assinatura usando a chave pública — isso prova que você tem a chave privada sem nunca transmiti-la. Nenhuma senha é trocada. A chave privada é protegida por uma passphrase (opcional mas recomendado), usada apenas para descriptografar a chave localmente.

---

**Q5: Qual a diferença entre `kill` e `kill -9`?**

R: `kill PID` (sinal padrão: SIGTERM=15) envia uma solicitação educada de encerramento. O processo a recebe e pode fazer a limpeza: fechar arquivos, limpar buffers, salvar estado, então sair. `kill -9 PID` (SIGKILL) é enviado diretamente ao kernel — o processo nunca o recebe e não pode interceptá-lo. O kernel o encerra imediatamente. Use SIGTERM primeiro (permita encerramento gracioso), espere alguns segundos, depois SIGKILL apenas se não parar. `kill -9` pode deixar arquivos abertos, conexões de banco de dados não fechadas e transações incompletas.

---

**Q6: Como você roda um processo em segundo plano para que persista após logout?**

R: Várias opções: (1) `nohup command &` — roda em segundo plano, imune a SIGHUP (sinal de hangup enviado no logout), saída vai para `nohup.out`. (2) `nohup command > app.log 2>&1 &` — redireciona saída. (3) `tmux` ou `screen` — multiplexadores de terminal que persistem sessões. (4) Serviço `systemd` — `systemctl start myapp` — melhor para serviços de produção (reinício automático, logging, gerenciamento de dependências). (5) `disown` após iniciar: `command &; disown` — desanexa job em segundo plano do shell atual sem nohup.

---

**Q7: O que é um pipe e como funciona?**

R: Um pipe (`|`) conecta o stdout de um comando ao stdin do próximo. `ps aux | grep node | awk '{print $2}'` — `ps aux` escreve no pipe, `grep node` lê dele, filtra e escreve no próximo pipe, `awk` lê e imprime a segunda coluna. Pipes são implementados no kernel como um buffer do kernel entre processos. Comandos em um pipeline rodam concorrentemente — o lado direito começa a ler enquanto o lado esquerdo começa a escrever. É por isso que `tail -f log | grep ERROR` funciona como um filtro ao vivo.

---

**Q8: Como você verifica qual processo está usando uma porta específica?**

R: `ss -tlnp | grep :3000` ou `lsof -i :3000`. `ss` (socket statistics) é o substituto moderno do `netstat`. As flags `-t` (TCP), `-l` (ouvindo), `-n` (numérico, sem resolução de hostname), `-p` (mostra processo). Em alguns sistemas, `-p` requer privilégios de root para ver processos de outros usuários: `sudo ss -tlnp | grep :3000`. Você obtém o PID e o nome do processo na saída. Então pode investigar com `ps -p PID -f` ou `kill PID`.

---

## 9. Exercícios

**Exercício 1 — Script de backup:**

Escreva um script bash `backup.sh <origem> [destino]` que:
- Cria um arquivo `.tar.gz` do diretório de origem
- Nomeia o arquivo com um timestamp (`dirorigem_YYYYMMDD_HHMMSS.tar.gz`)
- Rotaciona backups antigos (mantém apenas os 5 mais recentes)
- Imprime mensagens de progresso com timestamps
- Usa `set -euo pipefail`
- Trata diretório de origem ausente graciosamente com uma mensagem de erro clara

---

**Exercício 2 — Encontre e mate todos os processos node:**

```bash
# Escreva um one-liner que:
# 1. Encontra todos os PIDs para processos que casam com "node"
# 2. Exclui o próprio processo grep
# 3. Envia SIGTERM para cada um, então espera 3 segundos
# 4. Envia SIGKILL para qualquer um ainda rodando

# Dica 1: ps aux | grep node | grep -v grep | awk '{print $2}'
# Dica 2: kill $(comando) vs xargs kill
# Dica 3: verifica se processo ainda existe: kill -0 PID 2>/dev/null
```

---

**Exercício 3 — Script de monitoramento de log:**

Escreva `monitor.sh <arquivo_log> <padrão>` que:
- Faz tail do arquivo de log em tempo real
- Alerta (imprime para stderr com timestamp) sempre que uma linha casar com o padrão
- Conta quantos alertas foram disparados
- Imprime um resumo no Ctrl+C (trap SIGINT)
- Tem um rate-limit para evitar mais de 1 alerta por 5 segundos para o mesmo padrão

---

**Exercício 4 — Configure autenticação por chave SSH para um servidor remoto:**

1. Gere um par de chaves Ed25519: `ssh-keygen -t ed25519 -C "seu@email.com"`
2. Copie a chave pública para seu servidor: `ssh-copy-id user@servidor`
3. Adicione uma entrada em `~/.ssh/config` com um alias curto
4. Verifique se consegue conectar sem senha
5. (Bônus) Desabilite autenticação por senha no servidor: defina `PasswordAuthentication no` em `/etc/ssh/sshd_config` e reinicie o sshd

---

## 10. Leitura Adicional

- **"The Linux Command Line" de William Shotts** (gratuito online): https://linuxcommand.org/tlcl.php — o melhor livro Linux de iniciante a intermediário
- **Manual do Bash**: https://www.gnu.org/software/bash/manual/bash.html — referência completa
- **"Advanced Bash-Scripting Guide"**: https://tldp.org/LDP/abs/html/ — guia abrangente de scripting
- **explainshell.com**: https://explainshell.com — cole qualquer comando shell para ver o que cada parte faz
- **ShellCheck**: https://www.shellcheck.net — análise estática para scripts shell (captura bugs comuns)
- **páginas `man`**: `man bash`, `man find`, `man grep`, `man ssh`, `man curl` — sempre autoritativas
- **páginas tldr**: https://tldr.sh — páginas man simplificadas mantidas pela comunidade com exemplos
- **"Linux Pocket Guide" de Daniel Barrett** — referência rápida para os comandos mais comuns
- **"SSH Mastery" de Michael W. Lucas** — livro definitivo sobre configuração e uso do SSH
- **Filesystem Hierarchy Standard**: https://refspecs.linuxfoundation.org/fhs.shtml
