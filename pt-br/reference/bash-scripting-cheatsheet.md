# Bash Scripting Cheatsheet

## Shebang e Opções de Segurança

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e  sai em caso de erro (qualquer comando retorna não-zero)
# -u  sai em referência a variável não definida
# -o pipefail  sai se qualquer comando em um pipe falhar (não apenas o último)

# Modo debug — imprime cada comando antes de executar
set -x

# Rastrear para arquivo
exec 2> /tmp/debug.log
set -x

# Cabeçalho recomendado para scripts de produção
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'   # divisão de palavras mais segura (sem divisão por espaço)
```

---

## Variáveis

```bash
# Atribuição (sem espaços ao redor do =)
name="Alice"
count=42
path="/usr/local/bin"

# Acesso
echo "$name"          # sempre cite referências a variáveis
echo "${name}"        # chaves explícitas (necessárias antes de texto)
echo "${name}Script"  # → "AliceScript"

# Valores padrão
echo "${var:-default}"         # usa padrão se var não definida ou vazia
echo "${var:=default}"         # define e usa padrão se não definida ou vazia
echo "${var:?mensagem de erro}" # sai com erro se não definida ou vazia
echo "${var:+substituição}"    # usa substituição se var estiver definida

# Substrings
str="Hello World"
echo "${str:6}"       # → "World" (a partir do índice 6)
echo "${str:0:5}"     # → "Hello" (5 chars a partir do índice 0)

# Manipulação de strings
echo "${str,,}"       # → "hello world" (minúsculas)
echo "${str^^}"       # → "HELLO WORLD" (maiúsculas)
echo "${str/World/Bash}"  # → "Hello Bash" (substitui primeira ocorrência)
echo "${str//l/L}"    # → "HeLLo WorLd" (substitui todas)
echo "${#str}"        # → 11 (comprimento)

# Remover prefixo/sufixo
file="path/to/script.sh"
echo "${file##*/}"    # → "script.sh"  (remove o maior prefixo correspondendo a */)
echo "${file%.*}"     # → "path/to/script" (remove o menor sufixo correspondendo a .*)
echo "${file%%/*}"    # → "path" (remove o maior sufixo correspondendo a /*)
echo "${file#*/}"     # → "to/script.sh" (remove o menor prefixo correspondendo a */)
```

---

## Arrays

```bash
# Arrays indexados
fruits=("apple" "banana" "cherry")
fruits[3]="date"

echo "${fruits[0]}"     # apple
echo "${fruits[@]}"     # todos os elementos (seguro com aspas)
echo "${#fruits[@]}"    # comprimento: 4
echo "${!fruits[@]}"    # índices: 0 1 2 3

# Adicionar ao final
fruits+=("elderberry")

# Fatia
echo "${fruits[@]:1:2}"   # banana cherry (2 elementos a partir do índice 1)

# Loop
for fruit in "${fruits[@]}"; do
  echo "$fruit"
done

# Arrays associativos (Bash 4+)
declare -A colors
colors["red"]="#ff0000"
colors["green"]="#00ff00"
colors[blue]="#0000ff"

echo "${colors[red]}"
echo "${!colors[@]}"   # chaves
echo "${colors[@]}"    # valores

# Ler saída de comando em array
mapfile -t lines < file.txt
readarray -t lines < file.txt   # mesmo que mapfile

# Dividir string em array
IFS=',' read -ra parts <<< "a,b,c"
```

---

## Funções

```bash
# Definição
greet() {
  local name="${1:?Uso: greet <nome>}"  # parâmetro obrigatório
  local greeting="${2:-Olá}"            # opcional com padrão
  echo "$greeting, $name!"
}

# Chamada
greet "Alice"           # Olá, Alice!
greet "Bob" "Oi"        # Oi, Bob!

# Valor de retorno (apenas código de saída — 0=sucesso, 1-255=erro)
is_even() {
  [[ $(( $1 % 2 )) -eq 0 ]]  # retorno implícito baseado no resultado do teste
}
is_even 4 && echo "par"

# Retornar string via stdout
get_timestamp() {
  date +%Y%m%d_%H%M%S
}
ts=$(get_timestamp)

# Variáveis locais (sempre use local dentro de funções)
calculate() {
  local x=$1
  local y=$2
  local result=$(( x + y ))
  echo "$result"
}

# Função com nameref (Bash 4.3+)
return_multiple() {
  local -n _result=$1   # nameref — modifica a variável do chamador
  _result=("value1" "value2")
}
declare -a output
return_multiple output
echo "${output[@]}"
```

---

## Condicionais

```bash
# if / elif / else
if [[ condition ]]; then
  ...
elif [[ other ]]; then
  ...
else
  ...
fi

# Testes de string
[[ "$a" == "$b" ]]      # igual
[[ "$a" != "$b" ]]      # diferente
[[ "$a" < "$b" ]]       # menor lexicograficamente
[[ -z "$a" ]]           # string vazia
[[ -n "$a" ]]           # string não-vazia
[[ "$a" =~ ^[0-9]+$ ]] # correspondência regex (captura em BASH_REMATCH)

# Testes numéricos
[[ $n -eq 0 ]]   # igual
[[ $n -ne 0 ]]   # diferente
[[ $n -lt 10 ]]  # menor que
[[ $n -le 10 ]]  # menor ou igual
[[ $n -gt 10 ]]  # maior que
[[ $n -ge 10 ]]  # maior ou igual
(( n == 0 ))     # teste aritmético (preferido para números)
(( n > 5 && n < 10 ))

# Testes de arquivo
[[ -e "$path" ]]   # existe
[[ -f "$path" ]]   # arquivo regular
[[ -d "$path" ]]   # diretório
[[ -r "$path" ]]   # legível
[[ -w "$path" ]]   # gravável
[[ -x "$path" ]]   # executável
[[ -s "$path" ]]   # arquivo não-vazio
[[ -L "$path" ]]   # link simbólico
[[ -z "$(ls -A "$dir")" ]]  # diretório está vazio

# Operadores lógicos
[[ $a && $b ]]   # E lógico
[[ $a || $b ]]   # OU lógico
[[ ! $a ]]       # NÃO lógico

# Curto-circuito
command1 && command2   # roda 2 apenas se 1 for bem-sucedido
command1 || command2   # roda 2 apenas se 1 falhar
command1 || { echo "erro"; exit 1; }  # note: chaves para comandos compostos
```

---

## Loops

```bash
# Loop for em lista
for item in a b c d; do
  echo "$item"
done

# Loop for em array
for item in "${array[@]}"; do echo "$item"; done

# Loop for em arquivos
for file in /path/to/*.log; do
  [[ -f "$file" ]] || continue   # pula se glob não correspondeu
  process "$file"
done

# Loop for estilo C
for (( i = 0; i < 10; i++ )); do
  echo "$i"
done

# Loop while
while [[ condition ]]; do
  ...
done

# Ler linhas de arquivo
while IFS= read -r line; do
  echo "$line"
done < file.txt

# Ler linhas de saída de comando
while IFS= read -r line; do
  echo "$line"
done < <(some_command)  # substituição de processo — roda no shell atual

# Loop until
until [[ $count -ge 10 ]]; do
  (( count++ ))
done

# Controle de loop
continue   # pula para próxima iteração
break      # sai do loop
break 2    # sai de 2 níveis de loops aninhados

# seq para intervalos numéricos
for i in $(seq 1 5); do echo "$i"; done
for i in $(seq 0 2 10); do echo "$i"; done   # 0 2 4 6 8 10
```

---

## Manipulação de Strings

```bash
# Remover espaços no início/fim
trim() {
  local str="$1"
  str="${str#"${str%%[![:space:]]*}"}"   # início
  str="${str%"${str##*[![:space:]]}"}"   # fim
  echo "$str"
}

# Verificar se string contém substring
if [[ "$haystack" == *"$needle"* ]]; then echo "encontrado"; fi

# Dividir string
IFS='/' read -ra parts <<< "/usr/local/bin"
# parts = ("" "usr" "local" "bin")

# Juntar array
function join_by { local IFS="$1"; shift; echo "$*"; }
join_by , "${array[@]}"

# Repetir string
printf '%0.s-' {1..40}   # 40 hífens

# Codificação de URL (requer python ou jq)
python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$str"
```

---

## Aritmética

```bash
# Expansão aritmética
result=$(( 5 + 3 ))
result=$(( a * b + c ))
result=$(( 10 / 3 ))     # divisão inteira → 3
result=$(( 10 % 3 ))     # módulo → 1
result=$(( 2 ** 8 ))     # potência → 256

# Incremento/decremento
(( count++ ))
(( count-- ))
(( count += 5 ))

# Ponto flutuante (requer bc ou awk)
result=$(echo "scale=2; 10 / 3" | bc)      # → 3.33
result=$(awk 'BEGIN { printf "%.2f", 10/3 }')
```

---

## Testes e Operações de Arquivo

```bash
# Criar diretórios com segurança
mkdir -p /path/to/nested/dir

# Arquivo/diretório temporário
tmpfile=$(mktemp)
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpfile" "$tmpdir"' EXIT   # limpeza ao sair

# Ler arquivo em variável
content=$(<file.txt)

# Verificar se comando existe
if command -v docker &>/dev/null; then
  echo "docker está instalado"
fi

# Caminho absoluto
realpath ./relative/path
# Ou apenas bash:
abs=$(cd "$(dirname "$0")"; pwd)/$(basename "$0")
```

---

## Substituição de Processo

```bash
# Usar saída de comando como entrada de arquivo
diff <(sort file1.txt) <(sort file2.txt)

# Comparar dois comandos
comm <(ls dir1) <(ls dir2)

# Evitar subshell (while read com pipe perde mudanças de variável)
while IFS= read -r line; do
  count=$((count + 1))
done < <(grep "pattern" file.txt)
echo "Encontrados $count"   # count é acessível (substituição de processo, não pipe)
```

---

## Here-Documents e Here-Strings

```bash
# Here-doc (string multilinha)
cat <<EOF
Linha 1
Linha 2 com expansão de $variable
EOF

# Here-doc sem expansão (delimitador entre aspas)
cat <<'EOF'
Linha 1 com $cifrão literal
Sem expansão aqui
EOF

# Here-doc com indentação (traço remove tabs iniciais, não espaços)
cat <<-EOF
	Conteúdo indentado — tabs removidos
	EOF

# Here-string (linha única)
grep "pattern" <<< "$variable"
read -r first rest <<< "one two three"
```

---

## Traps e Tratamento de Sinais

```bash
# Limpeza ao sair (sempre)
cleanup() {
  local exit_code=$?
  rm -f "$tmpfile"
  [[ $exit_code -ne 0 ]] && echo "Script falhou com código $exit_code" >&2
}
trap cleanup EXIT

# Trap de sinais específicos
trap 'echo "Interrompido"; exit 1' INT TERM

# Ignorar um sinal
trap '' HUP

# Resetar para padrão
trap - INT

# Sinais comuns
# INT  — Ctrl+C
# TERM — kill / desligamento do sistema
# HUP  — terminal fechado
# EXIT — qualquer saída (pseudo-sinal)
# ERR  — qualquer erro de comando (com set -e)
```

---

## Padrões Comuns

### Análise de Argumentos

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Uso: $(basename "$0") [OPÇÕES] <entrada>

Opções:
  -o, --output ARQUIVO   Arquivo de saída (padrão: output.txt)
  -v, --verbose          Ativa saída detalhada
  -h, --help             Exibe esta ajuda
EOF
}

output="output.txt"
verbose=false

while [[ $# -gt 0 ]]; do
  case "$1" in
    -o|--output) output="$2"; shift 2 ;;
    -v|--verbose) verbose=true; shift ;;
    -h|--help) usage; exit 0 ;;
    --) shift; break ;;
    -*) echo "Opção desconhecida: $1" >&2; usage >&2; exit 1 ;;
    *) break ;;
  esac
done

[[ $# -lt 1 ]] && { echo "Erro: entrada obrigatória" >&2; usage >&2; exit 1; }
input="$1"
```

### Logging

```bash
# Funções de logging
readonly LOG_LEVEL_DEBUG=0
readonly LOG_LEVEL_INFO=1
readonly LOG_LEVEL_WARN=2
readonly LOG_LEVEL_ERROR=3
LOG_LEVEL=${LOG_LEVEL:-$LOG_LEVEL_INFO}

log() {
  local level=$1; shift
  local timestamp; timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  echo "[$timestamp] [$level] $*" >&2
}

debug() { [[ $LOG_LEVEL -le $LOG_LEVEL_DEBUG ]] && log "DEBUG" "$@"; }
info()  { [[ $LOG_LEVEL -le $LOG_LEVEL_INFO  ]] && log "INFO " "$@"; }
warn()  { [[ $LOG_LEVEL -le $LOG_LEVEL_WARN  ]] && log "WARN " "$@"; }
error() { log "ERROR" "$@"; }

info "Iniciando deploy"
warn "Arquivo de config não encontrado, usando padrões"
error "Falha na conexão com o banco de dados"
```

### Loop de Retry

```bash
retry() {
  local max_attempts=${MAX_ATTEMPTS:-3}
  local delay=${RETRY_DELAY:-5}
  local attempt=1

  while (( attempt <= max_attempts )); do
    if "$@"; then
      return 0
    fi
    warn "Tentativa $attempt/$max_attempts falhou. Tentando novamente em ${delay}s..."
    sleep "$delay"
    (( attempt++ ))
    delay=$(( delay * 2 ))  # backoff exponencial
  done

  error "Comando falhou após $max_attempts tentativas: $*"
  return 1
}

retry curl -f "https://api.example.com/health"
```

### Execução Paralela

```bash
# Rodar jobs em paralelo, coletar PIDs
pids=()
for host in "${hosts[@]}"; do
  deploy_to "$host" &
  pids+=($!)
done

# Aguardar todos e verificar códigos de saída
failed=0
for pid in "${pids[@]}"; do
  if ! wait "$pid"; then
    error "Job $pid falhou"
    (( failed++ ))
  fi
done
[[ $failed -eq 0 ]] || { error "$failed job(s) falharam"; exit 1; }

# Com xargs paralelo (mais simples para comandos simples)
printf '%s\n' "${hosts[@]}" | xargs -P4 -I{} deploy_to {}
#                                      ↑ máximo 4 em paralelo

# GNU parallel (se disponível)
parallel -j4 deploy_to ::: "${hosts[@]}"
```

### Diretório do Próprio Script

```bash
# Obter diretório do script atual (funciona com links simbólicos)
SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)

# Incluir arquivo irmão
source "$SCRIPT_DIR/lib/helpers.sh"
```

### Verificar Comandos Necessários

```bash
require_commands() {
  local missing=()
  for cmd in "$@"; do
    command -v "$cmd" &>/dev/null || missing+=("$cmd")
  done
  if [[ ${#missing[@]} -gt 0 ]]; then
    error "Comandos necessários não encontrados: ${missing[*]}"
    exit 1
  fi
}

require_commands docker docker-compose jq curl
```

### Carregador de Arquivo de Configuração

```bash
# Carrega config chave=valor, ignora comentários e linhas em branco
load_config() {
  local file="$1"
  [[ -f "$file" ]] || return 0
  while IFS='=' read -r key value; do
    [[ "$key" =~ ^[[:space:]]*# ]] && continue  # ignora comentários
    [[ -z "$key" ]] && continue                  # ignora linhas em branco
    key="${key// /}"    # remove espaços
    value="${value// /}"
    export "$key=$value"
  done < "$file"
}

load_config .env
```

---

## Referência Rápida

```bash
# Verificar código de saída do último comando
echo $?

# Redirecionar stderr para stdout
command 2>&1

# Descartar toda a saída
command &>/dev/null

# Rodar em segundo plano
command &
echo "PID: $!"

# Aguardar job em segundo plano
wait $pid

# Source vs executar
source script.sh    # roda no shell atual (variáveis de ambiente persistem)
./script.sh         # roda em subshell

# Ler um único char sem Enter
read -r -n1 -p "Continuar? [s/N] " choice

# Timeout em um comando
timeout 30 long_running_command

# Rodar como outro usuário
sudo -u www-data command

# Heredoc para arquivo
cat > /path/to/file <<'EOF'
conteúdo aqui
EOF
```
