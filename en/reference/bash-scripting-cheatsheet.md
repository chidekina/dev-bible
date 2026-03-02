# Bash Scripting Cheatsheet

## Shebang and Safety Options

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e  exit on error (any command returns non-zero)
# -u  exit on unset variable reference
# -o pipefail  exit if any command in a pipe fails (not just last)

# Debug mode — print each command before executing
set -x

# Trace to file
exec 2> /tmp/debug.log
set -x

# Recommended header for production scripts
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'   # safer word splitting (no space splitting)
```

---

## Variables

```bash
# Assignment (no spaces around =)
name="Alice"
count=42
path="/usr/local/bin"

# Access
echo "$name"          # always quote variable references
echo "${name}"        # explicit braces (required before text)
echo "${name}Script"  # → "AliceScript"

# Default values
echo "${var:-default}"         # use default if var unset or empty
echo "${var:=default}"         # set and use default if unset or empty
echo "${var:?error message}"   # exit with error if unset or empty
echo "${var:+replacement}"     # use replacement if var is set

# Substrings
str="Hello World"
echo "${str:6}"       # → "World" (from index 6)
echo "${str:0:5}"     # → "Hello" (5 chars from index 0)

# String manipulation
echo "${str,,}"       # → "hello world" (lowercase)
echo "${str^^}"       # → "HELLO WORLD" (uppercase)
echo "${str/World/Bash}"  # → "Hello Bash" (replace first)
echo "${str//l/L}"    # → "HeLLo WorLd" (replace all)
echo "${#str}"        # → 11 (length)

# Strip prefix/suffix
file="path/to/script.sh"
echo "${file##*/}"    # → "script.sh"  (remove longest prefix matching */)
echo "${file%.*}"     # → "path/to/script" (remove shortest suffix matching .*)
echo "${file%%/*}"    # → "path" (remove longest suffix matching /*)
echo "${file#*/}"     # → "to/script.sh" (remove shortest prefix matching */)
```

---

## Arrays

```bash
# Indexed arrays
fruits=("apple" "banana" "cherry")
fruits[3]="date"

echo "${fruits[0]}"     # apple
echo "${fruits[@]}"     # all elements (word-split safe with quotes)
echo "${#fruits[@]}"    # length: 4
echo "${!fruits[@]}"    # indices: 0 1 2 3

# Append
fruits+=("elderberry")

# Slice
echo "${fruits[@]:1:2}"   # banana cherry (2 elements from index 1)

# Loop
for fruit in "${fruits[@]}"; do
  echo "$fruit"
done

# Associative arrays (Bash 4+)
declare -A colors
colors["red"]="#ff0000"
colors["green"]="#00ff00"
colors[blue]="#0000ff"

echo "${colors[red]}"
echo "${!colors[@]}"   # keys
echo "${colors[@]}"    # values

# Read command output into array
mapfile -t lines < file.txt
readarray -t lines < file.txt   # same as mapfile

# Split string into array
IFS=',' read -ra parts <<< "a,b,c"
```

---

## Functions

```bash
# Definition
greet() {
  local name="${1:?Usage: greet <name>}"  # required param
  local greeting="${2:-Hello}"             # optional with default
  echo "$greeting, $name!"
}

# Call
greet "Alice"           # Hello, Alice!
greet "Bob" "Hi"        # Hi, Bob!

# Return value (exit code only — 0=success, 1-255=error)
is_even() {
  [[ $(( $1 % 2 )) -eq 0 ]]  # implicit return based on test result
}
is_even 4 && echo "even"

# Return string via stdout
get_timestamp() {
  date +%Y%m%d_%H%M%S
}
ts=$(get_timestamp)

# Local variables (always use local inside functions)
calculate() {
  local x=$1
  local y=$2
  local result=$(( x + y ))
  echo "$result"
}

# Function with nameref (Bash 4.3+)
return_multiple() {
  local -n _result=$1   # nameref — modifies caller's variable
  _result=("value1" "value2")
}
declare -a output
return_multiple output
echo "${output[@]}"
```

---

## Conditionals

```bash
# if / elif / else
if [[ condition ]]; then
  ...
elif [[ other ]]; then
  ...
else
  ...
fi

# String tests
[[ "$a" == "$b" ]]      # equal
[[ "$a" != "$b" ]]      # not equal
[[ "$a" < "$b" ]]       # lexicographic less than
[[ -z "$a" ]]           # empty string
[[ -n "$a" ]]           # non-empty string
[[ "$a" =~ ^[0-9]+$ ]] # regex match (capture in BASH_REMATCH)

# Numeric tests
[[ $n -eq 0 ]]   # equal
[[ $n -ne 0 ]]   # not equal
[[ $n -lt 10 ]]  # less than
[[ $n -le 10 ]]  # less or equal
[[ $n -gt 10 ]]  # greater than
[[ $n -ge 10 ]]  # greater or equal
(( n == 0 ))     # arithmetic test (preferred for numbers)
(( n > 5 && n < 10 ))

# File tests
[[ -e "$path" ]]   # exists
[[ -f "$path" ]]   # regular file
[[ -d "$path" ]]   # directory
[[ -r "$path" ]]   # readable
[[ -w "$path" ]]   # writable
[[ -x "$path" ]]   # executable
[[ -s "$path" ]]   # non-empty file
[[ -L "$path" ]]   # symbolic link
[[ -z "$(ls -A "$dir")" ]]  # directory is empty

# Logical operators
[[ $a && $b ]]   # AND
[[ $a || $b ]]   # OR
[[ ! $a ]]       # NOT

# Short-circuit
command1 && command2   # run 2 only if 1 succeeds
command1 || command2   # run 2 only if 1 fails
command1 || { echo "error"; exit 1; }  # note: braces for compound commands
```

---

## Loops

```bash
# For loop over list
for item in a b c d; do
  echo "$item"
done

# For loop over array
for item in "${array[@]}"; do echo "$item"; done

# For loop over files
for file in /path/to/*.log; do
  [[ -f "$file" ]] || continue   # skip if glob didn't match
  process "$file"
done

# C-style for loop
for (( i = 0; i < 10; i++ )); do
  echo "$i"
done

# While loop
while [[ condition ]]; do
  ...
done

# Read lines from file
while IFS= read -r line; do
  echo "$line"
done < file.txt

# Read lines from command output
while IFS= read -r line; do
  echo "$line"
done < <(some_command)  # process substitution — runs in current shell

# Until loop
until [[ $count -ge 10 ]]; do
  (( count++ ))
done

# Loop control
continue   # skip to next iteration
break      # exit loop
break 2    # exit 2 levels of nested loops

# seq for numeric ranges
for i in $(seq 1 5); do echo "$i"; done
for i in $(seq 0 2 10); do echo "$i"; done   # 0 2 4 6 8 10
```

---

## String Manipulation

```bash
# Trim leading/trailing whitespace
trim() {
  local str="$1"
  str="${str#"${str%%[![:space:]]*}"}"   # leading
  str="${str%"${str##*[![:space:]]}"}"   # trailing
  echo "$str"
}

# Check if string contains substring
if [[ "$haystack" == *"$needle"* ]]; then echo "found"; fi

# Split string
IFS='/' read -ra parts <<< "/usr/local/bin"
# parts = ("" "usr" "local" "bin")

# Join array
function join_by { local IFS="$1"; shift; echo "$*"; }
join_by , "${array[@]}"

# Repeat string
printf '%0.s-' {1..40}   # 40 dashes

# URL encode (requires python or jq)
python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$str"
```

---

## Arithmetic

```bash
# Arithmetic expansion
result=$(( 5 + 3 ))
result=$(( a * b + c ))
result=$(( 10 / 3 ))     # integer division → 3
result=$(( 10 % 3 ))     # modulo → 1
result=$(( 2 ** 8 ))     # power → 256

# Increment/decrement
(( count++ ))
(( count-- ))
(( count += 5 ))

# Floating point (requires bc or awk)
result=$(echo "scale=2; 10 / 3" | bc)      # → 3.33
result=$(awk 'BEGIN { printf "%.2f", 10/3 }')
```

---

## File Tests and Operations

```bash
# Create directories safely
mkdir -p /path/to/nested/dir

# Temp file/dir
tmpfile=$(mktemp)
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpfile" "$tmpdir"' EXIT   # cleanup on exit

# Read file into variable
content=$(<file.txt)

# Check if command exists
if command -v docker &>/dev/null; then
  echo "docker is installed"
fi

# Absolute path
realpath ./relative/path
# Or bash-only:
abs=$(cd "$(dirname "$0")"; pwd)/$(basename "$0")
```

---

## Process Substitution

```bash
# Use command output as file input
diff <(sort file1.txt) <(sort file2.txt)

# Compare two commands
comm <(ls dir1) <(ls dir2)

# Avoid subshell (while read with pipe loses variable changes)
while IFS= read -r line; do
  count=$((count + 1))
done < <(grep "pattern" file.txt)
echo "Found $count"   # count is accessible (process substitution, not pipe)
```

---

## Here-Documents and Here-Strings

```bash
# Here-doc (multi-line string)
cat <<EOF
Line 1
Line 2 with $variable expansion
EOF

# Here-doc without expansion (quoted delimiter)
cat <<'EOF'
Line 1 with $literal dollar signs
No expansion here
EOF

# Here-doc with indentation (dash removes leading tabs, not spaces)
cat <<-EOF
	Indented content — tabs stripped
	EOF

# Here-string (single line)
grep "pattern" <<< "$variable"
read -r first rest <<< "one two three"
```

---

## Traps and Signal Handling

```bash
# Cleanup on exit (always)
cleanup() {
  local exit_code=$?
  rm -f "$tmpfile"
  [[ $exit_code -ne 0 ]] && echo "Script failed with code $exit_code" >&2
}
trap cleanup EXIT

# Trap specific signals
trap 'echo "Interrupted"; exit 1' INT TERM

# Ignore a signal
trap '' HUP

# Reset to default
trap - INT

# Common signals
# INT  — Ctrl+C
# TERM — kill / system shutdown
# HUP  — terminal closed
# EXIT — any exit (pseudo-signal)
# ERR  — any command error (with set -e)
```

---

## Common Patterns

### Argument Parsing

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $(basename "$0") [OPTIONS] <input>

Options:
  -o, --output FILE   Output file (default: output.txt)
  -v, --verbose       Enable verbose output
  -h, --help          Show this help
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
    -*) echo "Unknown option: $1" >&2; usage >&2; exit 1 ;;
    *) break ;;
  esac
done

[[ $# -lt 1 ]] && { echo "Error: input required" >&2; usage >&2; exit 1; }
input="$1"
```

### Logging

```bash
# Logging functions
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

info "Starting deployment"
warn "Config file not found, using defaults"
error "Database connection failed"
```

### Retry Loop

```bash
retry() {
  local max_attempts=${MAX_ATTEMPTS:-3}
  local delay=${RETRY_DELAY:-5}
  local attempt=1

  while (( attempt <= max_attempts )); do
    if "$@"; then
      return 0
    fi
    warn "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..."
    sleep "$delay"
    (( attempt++ ))
    delay=$(( delay * 2 ))  # exponential backoff
  done

  error "Command failed after $max_attempts attempts: $*"
  return 1
}

retry curl -f "https://api.example.com/health"
```

### Parallel Execution

```bash
# Run jobs in parallel, collect PIDs
pids=()
for host in "${hosts[@]}"; do
  deploy_to "$host" &
  pids+=($!)
done

# Wait for all and check exit codes
failed=0
for pid in "${pids[@]}"; do
  if ! wait "$pid"; then
    error "Job $pid failed"
    (( failed++ ))
  fi
done
[[ $failed -eq 0 ]] || { error "$failed job(s) failed"; exit 1; }

# With xargs parallel (simpler for simple commands)
printf '%s\n' "${hosts[@]}" | xargs -P4 -I{} deploy_to {}
#                                      ↑ max 4 parallel

# GNU parallel (if available)
parallel -j4 deploy_to ::: "${hosts[@]}"
```

### Script Self-Directory

```bash
# Get directory of current script (works with symlinks)
SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)

# Source sibling file
source "$SCRIPT_DIR/lib/helpers.sh"
```

### Require Commands

```bash
require_commands() {
  local missing=()
  for cmd in "$@"; do
    command -v "$cmd" &>/dev/null || missing+=("$cmd")
  done
  if [[ ${#missing[@]} -gt 0 ]]; then
    error "Required commands not found: ${missing[*]}"
    exit 1
  fi
}

require_commands docker docker-compose jq curl
```

### Config File Loader

```bash
# Load key=value config, skip comments and blank lines
load_config() {
  local file="$1"
  [[ -f "$file" ]] || return 0
  while IFS='=' read -r key value; do
    [[ "$key" =~ ^[[:space:]]*# ]] && continue  # skip comments
    [[ -z "$key" ]] && continue                  # skip blank lines
    key="${key// /}"    # trim spaces
    value="${value// /}"
    export "$key=$value"
  done < "$file"
}

load_config .env
```

---

## Quick Reference

```bash
# Check exit code of last command
echo $?

# Redirect stderr to stdout
command 2>&1

# Discard all output
command &>/dev/null

# Run in background
command &
echo "PID: $!"

# Wait for background job
wait $pid

# Source vs execute
source script.sh    # runs in current shell (env vars persist)
./script.sh         # runs in subshell

# Read single char without Enter
read -r -n1 -p "Continue? [y/N] " choice

# Timeout a command
timeout 30 long_running_command

# Run as different user
sudo -u www-data command

# Heredoc to file
cat > /path/to/file <<'EOF'
content here
EOF
```
