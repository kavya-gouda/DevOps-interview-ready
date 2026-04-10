# Shell Scripting — Concepts Reference

> Everything a senior DevOps engineer must know cold.
> Concept → why it matters → syntax → gotcha.

---

## 1. The Non-Negotiable Header

Every script you write in an interview must start with this. Not optional.

```bash
#!/usr/bin/env bash
set -euo pipefail
```

| Flag | What it does | Why it matters |
|---|---|---|
| `-e` | Exit immediately on any command failure | Prevents silent failures propagating |
| `-u` | Treat unset variables as errors | Catches typos like `$NAMEPSACE` silently expanding to empty string |
| `-o pipefail` | Pipe fails if ANY command in it fails | Without this, `false \| true` exits 0 |

**Gotcha:** `set -e` doesn't trigger inside `if`, `while`, `&&`, `\|\|` conditions — those are expected to fail. It only exits on unguarded failures.

```bash
# This WON'T exit even with -e (condition context)
if grep -q "pattern" file.txt; then echo "found"; fi

# This WILL exit with -e
grep -q "pattern" file.txt   # exits if pattern not found
```

---

## 2. Variables — Declaration, Scope, Expansion

### Basic rules
```bash
# Assignment — NO spaces around =
NAME="devops"           # correct
NAME = "devops"         # WRONG — treated as command named NAME

# Convention
GLOBAL_VAR="value"      # UPPER_CASE for env vars / constants
local_var="value"       # lower_case for local script variables

# local keyword — scope to function
my_func() {
    local result="hello"   # not visible outside function
    echo "$result"
}
```

### Parameter Expansion (critical for senior interviews)
```bash
VAR="hello"

${VAR}              # basic expansion
${VAR:-default}     # use default if VAR is unset OR empty
${VAR:=default}     # assign default if VAR is unset OR empty
${VAR:?error msg}   # exit with error if VAR is unset OR empty  ← use in scripts
${VAR:+other}       # use 'other' if VAR is set and non-empty

# String operations
${VAR#pattern}      # remove shortest prefix matching pattern
${VAR##pattern}     # remove longest prefix matching pattern
${VAR%pattern}      # remove shortest suffix matching pattern
${VAR%%pattern}     # remove longest suffix matching pattern
${VAR/old/new}      # replace first occurrence
${VAR//old/new}     # replace all occurrences

# Length
${#VAR}             # length of VAR

# Substring
${VAR:2}            # from index 2 to end
${VAR:2:4}          # from index 2, length 4
```

```bash
FILE="/var/log/nginx/access.log"
echo "${FILE##*/}"     # access.log   (basename)
echo "${FILE%/*}"      # /var/log/nginx  (dirname)
echo "${FILE%.log}"    # /var/log/nginx/access  (strip extension)

VERSION="v1.2.3"
echo "${VERSION#v}"    # 1.2.3   (strip prefix)
```

### Arrays
```bash
# Indexed array
fruits=("apple" "banana" "cherry")
echo "${fruits[0]}"         # apple
echo "${fruits[@]}"         # all elements
echo "${#fruits[@]}"        # count = 3
echo "${fruits[@]:1:2}"     # slice: banana cherry

# Iterate
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Associative array (bash 4+)
declare -A person
person["name"]="Alice"
person["role"]="DevOps"
echo "${person["name"]}"
for key in "${!person[@]}"; do
    echo "$key = ${person[$key]}"
done
```

---

## 3. Quoting Rules — The Most Common Source of Bugs

```bash
# Double quotes: expand variables and command substitution
echo "Hello $NAME"       # Hello devops
echo "Today: $(date)"    # Today: Mon Apr 7 ...

# Single quotes: literal, NO expansion
echo 'Hello $NAME'       # Hello $NAME
echo 'Today: $(date)'    # Today: $(date)

# Always quote variables to prevent word splitting and glob expansion
FILE="my file.txt"
cat $FILE      # WRONG — treated as two arguments: "my" and "file.txt"
cat "$FILE"    # correct

# Arrays must use [@] with quotes
for f in "${files[@]}"; do ...   # correct — preserves spaces in filenames
for f in ${files[@]};  do ...   # WRONG — splits on spaces
```

---

## 4. Conditionals — `[[ ]]` vs `[ ]`

**Always use `[[ ]]`** in bash scripts. `[ ]` is POSIX sh — less features, more surprises.

```bash
# String tests
[[ -z "$VAR" ]]       # true if empty
[[ -n "$VAR" ]]       # true if non-empty
[[ "$A" == "$B" ]]    # string equality
[[ "$A" != "$B" ]]    # string inequality
[[ "$A" =~ ^[0-9]+$ ]] # regex match (no quotes on pattern!)

# Numeric tests
[[ "$A" -eq "$B" ]]   # equal
[[ "$A" -ne "$B" ]]   # not equal
[[ "$A" -lt "$B" ]]   # less than
[[ "$A" -gt "$B" ]]   # greater than

# File tests
[[ -f "$FILE" ]]      # regular file exists
[[ -d "$DIR" ]]       # directory exists
[[ -r "$FILE" ]]      # readable
[[ -w "$FILE" ]]      # writable
[[ -x "$FILE" ]]      # executable
[[ -s "$FILE" ]]      # file exists and is non-empty
[[ -L "$FILE" ]]      # symbolic link

# Logical operators
[[ -f "$F" && -r "$F" ]]   # AND
[[ -z "$A" || -z "$B" ]]   # OR
[[ ! -f "$F" ]]            # NOT
```

---

## 5. Loops

```bash
# C-style for loop
for (( i=0; i<10; i++ )); do
    echo "$i"
done

# Iterate over range
for i in {1..10}; do echo "$i"; done
for i in {0..100..5}; do echo "$i"; done  # step 5

# Iterate over array
for item in "${array[@]}"; do ...

# Iterate over command output
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# Process substitution (avoid subshell — variables persist)
while IFS= read -r line; do
    echo "$line"
done < <(find . -name "*.log")

# Until loop
until [[ "$STATUS" == "healthy" ]]; do
    STATUS=$(check_health)
    sleep 5
done
```

**Critical gotcha — subshell variable scope:**
```bash
# WRONG — count is modified in subshell, loses value after pipe
count=0
cat file.txt | while read line; do
    (( count++ ))
done
echo $count   # still 0!

# CORRECT — use process substitution (no subshell)
count=0
while IFS= read -r line; do
    (( count++ ))
done < <(cat file.txt)
echo $count   # correct
```

---

## 6. Functions

```bash
# Definition
check_pod_health() {
    local namespace="${1:?namespace required}"
    local pod="${2:?pod name required}"
    local timeout="${3:-30}"

    kubectl get pod "$pod" -n "$namespace" \
        --output jsonpath='{.status.phase}' 2>/dev/null
}

# Call
status=$(check_pod_health "default" "myapp-abc123" 60)

# Return values
# Bash functions return exit codes (0=success, 1-255=failure)
# To return data: use echo + command substitution

validate_input() {
    local input="$1"
    if [[ -z "$input" ]]; then
        echo "ERROR: input is empty" >&2
        return 1
    fi
    return 0
}

if ! validate_input "$USER_INPUT"; then
    exit 1
fi
```

---

## 7. Error Handling — Production-Grade Patterns

```bash
#!/usr/bin/env bash
set -euo pipefail

# Trap for cleanup on exit
TMPFILE=""

cleanup() {
    local exit_code=$?
    [[ -n "$TMPFILE" ]] && rm -f "$TMPFILE"
    if [[ $exit_code -ne 0 ]]; then
        echo "ERROR: Script failed at line $LINENO with exit code $exit_code" >&2
    fi
    exit $exit_code
}
trap cleanup EXIT

# Trap specific signals
trap 'echo "Interrupted"; exit 130' INT TERM

# Usage/argument validation pattern
usage() {
    cat >&2 <<EOF
Usage: $(basename "$0") [OPTIONS] <required_arg>

Options:
  -n NAMESPACE   Kubernetes namespace (default: default)
  -t TIMEOUT     Timeout in seconds (default: 60)
  -h             Show this help

Example:
  $(basename "$0") -n production myapp
EOF
    exit 1
}

# Parse args with getopts
NAMESPACE="default"
TIMEOUT=60
while getopts "n:t:h" opt; do
    case $opt in
        n) NAMESPACE="$OPTARG" ;;
        t) TIMEOUT="$OPTARG" ;;
        h) usage ;;
        ?) usage ;;
    esac
done
shift $(( OPTIND - 1 ))

[[ $# -lt 1 ]] && usage
REQUIRED_ARG="$1"
```

---

## 8. Text Processing — awk, sed, grep

### grep
```bash
grep "pattern" file.txt             # basic search
grep -i "pattern" file.txt          # case-insensitive
grep -v "pattern" file.txt          # invert match (exclude)
grep -c "pattern" file.txt          # count matching lines
grep -n "pattern" file.txt          # show line numbers
grep -r "pattern" /dir/             # recursive
grep -l "pattern" /dir/*.log        # only filenames
grep -E "error|warn|fatal" file     # extended regex (ERE)
grep -o "Error: [^$]*" file         # print only matching part
grep -A 3 "ERROR" file              # 3 lines AFTER match
grep -B 2 "ERROR" file              # 2 lines BEFORE match
grep -P '\d{4}-\d{2}-\d{2}' file   # Perl regex
```

### sed
```bash
sed 's/old/new/' file           # replace first occurrence per line
sed 's/old/new/g' file          # replace all occurrences per line
sed 's/old/new/gi' file         # case-insensitive replace all
sed -i 's/old/new/g' file       # in-place edit
sed -i.bak 's/old/new/g' file   # in-place with backup
sed -n '5,10p' file             # print lines 5-10
sed '1,3d' file                 # delete lines 1-3
sed '/^#/d' file                # delete comment lines
sed '/^$/d' file                # delete empty lines
sed -n '/START/,/END/p' file    # print between patterns
sed 's/^/  /' file              # indent every line
sed 's/[[:space:]]*$//' file    # strip trailing whitespace
```

### awk
```bash
# awk 'pattern { action }' file
# Built-in variables: NR=line number, NF=field count, FS=field separator

awk '{print $1}' file           # print first field
awk '{print $NF}' file          # print last field
awk -F: '{print $1}' /etc/passwd  # colon-separated
awk 'NR==5' file                # print line 5
awk 'NR>=3 && NR<=7' file       # print lines 3-7
awk '/ERROR/' file              # lines matching pattern
awk '!/DEBUG/' file             # lines NOT matching
awk '{sum += $3} END {print sum}' file   # sum column 3
awk '{count[$1]++} END {for (k in count) print k, count[k]}' file  # frequency count

# Multi-condition
awk '$3 > 100 {print $1, $3}' file

# printf formatting
awk '{printf "%-20s %5d\n", $1, $2}' file

# BEGIN/END blocks
awk 'BEGIN {print "Header"} {print} END {print "Footer"}' file

# Field separator output
awk -F: 'BEGIN{OFS=","} {print $1,$3}' /etc/passwd
```

---

## 9. Process Management and Job Control

```bash
# Background execution
long_command &
PID=$!              # capture PID of last backgrounded job
echo "Started PID: $PID"

# Wait for specific PID
wait $PID
echo "Exit code: $?"

# Wait for all background jobs
wait
echo "All done"

# Job control
jobs                # list background jobs
fg %1               # bring job 1 to foreground
bg %1               # send job 1 to background
kill %1             # kill job 1

# Parallel execution with tracking
pids=()
for host in "${hosts[@]}"; do
    check_host "$host" &
    pids+=($!)
done

# Wait and collect exit codes
failed=0
for pid in "${pids[@]}"; do
    if ! wait "$pid"; then
        (( failed++ )) || true
    fi
done
[[ $failed -gt 0 ]] && echo "WARN: $failed checks failed"
```

---

## 10. Signals and Traps

```bash
# Signal list
# SIGINT  (2)  — Ctrl+C
# SIGTERM (15) — graceful shutdown request
# SIGKILL (9)  — cannot be trapped/ignored
# SIGHUP  (1)  — terminal hangup / config reload
# EXIT        — pseudo-signal: runs on any exit

# Graceful shutdown pattern
RUNNING=true

cleanup() {
    echo "Caught signal — cleaning up..."
    RUNNING=false
    # kill child processes
    kill 0  # kills all processes in process group
}

trap cleanup INT TERM EXIT

# Main loop
while $RUNNING; do
    do_work
    sleep 5
done
```

---

## 11. String Manipulation

```bash
STR="Hello, World 2026"

# Case conversion (bash 4+)
echo "${STR,,}"     # hello, world 2026  (lowercase)
echo "${STR^^}"     # HELLO, WORLD 2026  (uppercase)
echo "${STR^}"      # Hello, World 2026  (capitalize first)

# Trim whitespace (no built-in, use parameter expansion)
trim() {
    local var="$1"
    var="${var#"${var%%[![:space:]]*}"}"   # trim leading
    var="${var%"${var##*[![:space:]]}"}"   # trim trailing
    echo "$var"
}

# Split string into array
IFS=',' read -ra parts <<< "a,b,c,d"
echo "${parts[1]}"   # b

# Join array into string
arr=("one" "two" "three")
IFS=','; joined="${arr[*]}"; IFS=' '
echo "$joined"   # one,two,three

# Check if string contains substring
if [[ "$STR" == *"World"* ]]; then
    echo "contains World"
fi

# Check if string starts/ends with
if [[ "$STR" == Hello* ]]; then echo "starts with Hello"; fi
if [[ "$STR" == *2026 ]]; then echo "ends with 2026"; fi

# String length
echo "${#STR}"   # 18
```

---

## 12. File and Directory Operations

```bash
# Read file line by line (safest pattern)
while IFS= read -r line || [[ -n "$line" ]]; do
    echo "$line"
done < "file.txt"
# Note: || [[ -n "$line" ]] handles files without trailing newline

# Check and create directory
mkdir -p "/path/to/dir"   # -p: no error if exists, creates parents

# Find files
find /var/log -name "*.log" -mtime +7        # older than 7 days
find /app -name "*.py" -not -path "*/venv/*" # exclude directory
find . -type f -size +100M                    # files > 100MB
find . -perm /u+x                             # executable by owner
find . -name "*.log" -exec gzip {} \;         # compress each found file
find . -name "*.log" -print0 | xargs -0 gzip # faster for many files

# Temporary files
TMPFILE=$(mktemp)               # /tmp/tmp.XXXXXX
TMPDIR=$(mktemp -d)             # temporary directory
trap "rm -rf $TMPFILE $TMPDIR" EXIT   # always clean up

# File checksums
md5sum file.txt
sha256sum file.txt
# Verify
sha256sum -c checksums.txt

# Atomic file write (prevents partial reads)
write_atomic() {
    local target="$1"
    local tmpfile
    tmpfile=$(mktemp "${target}.tmp.XXXXXX")
    # write to temp first
    cat > "$tmpfile"
    # atomic rename
    mv "$tmpfile" "$target"
}
echo "new content" | write_atomic "/etc/app/config.yaml"
```

---

## 13. Here Documents and Here Strings

```bash
# Here document — multi-line input
cat > /tmp/script.py << 'EOF'
import sys
print(f"Hello from Python {sys.version}")
EOF
# Single-quoted EOF: NO variable expansion inside
# Unquoted EOF: variables ARE expanded

# Indented here doc (bash 4+)
cat > /tmp/config.yaml <<- EOF
	database:
	  host: ${DB_HOST}
	  port: ${DB_PORT}
EOF
# Tab indentation stripped from each line

# Here string — single-line input
grep "pattern" <<< "$VARIABLE"
base64 --decode <<< "SGVsbG8="
```

---

## 14. Networking in Scripts

```bash
# HTTP requests
curl -sf --max-time 10 "https://api.example.com/health"
# -s: silent, -f: fail on HTTP errors (4xx/5xx return exit code 22)

# With auth and JSON body
curl -sf \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${TOKEN}" \
    -d '{"key": "value"}' \
    "https://api.example.com/endpoint"

# Download with retry
curl -fSL --retry 3 --retry-delay 2 \
    -o /tmp/file.tar.gz \
    "https://example.com/file.tar.gz"

# Check port is open
nc -z -w5 hostname 443 && echo "open" || echo "closed"

# Wait for port
wait_for_port() {
    local host="$1" port="$2" timeout="${3:-60}"
    local deadline=$(( $(date +%s) + timeout ))
    until nc -z -w1 "$host" "$port" 2>/dev/null; do
        [[ $(date +%s) -gt $deadline ]] && { echo "Timeout waiting for $host:$port" >&2; return 1; }
        sleep 2
    done
}

# DNS lookup
dig +short api.example.com
nslookup api.example.com
host api.example.com
```

---

## 15. Locking — Prevent Concurrent Execution

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/var/lock/$(basename "$0").lock"

# flock — file-based locking
exec 9>"$LOCK_FILE"
if ! flock -n 9; then
    echo "ERROR: Another instance is already running" >&2
    exit 1
fi
trap "flock -u 9; rm -f $LOCK_FILE" EXIT

# Now running exclusively
echo "Running with lock..."
```

---

## 16. Arithmetic

```bash
# Integer arithmetic
(( result = 5 + 3 ))
(( count++ ))
(( count += 10 ))
echo $(( 100 / 7 ))   # integer division = 14

# Check arithmetic condition
(( count > 10 )) && echo "over 10"
if (( count >= threshold )); then ...

# Floating point — bash doesn't support it, use bc or awk
echo "scale=2; 100/7" | bc          # 14.28
awk 'BEGIN {printf "%.2f\n", 100/7}' # 14.29

# Increment safely with set -e
(( count++ )) || true   # (( 0++ )) returns 1 (falsy) — needs || true
# Better:
count=$(( count + 1 ))
```

---

## 17. Input/Output Redirection

```bash
# Standard streams
#  0 = stdin
#  1 = stdout
#  2 = stderr

command > file.txt       # stdout to file (overwrite)
command >> file.txt      # stdout to file (append)
command 2> error.txt     # stderr to file
command 2>&1             # redirect stderr to stdout
command > out.txt 2>&1   # both to file (order matters!)
command &> file.txt      # shorthand: both to file

# Discard output
command > /dev/null 2>&1
command &> /dev/null

# Read from file
command < input.txt

# Tee — write to file AND stdout
command | tee output.txt
command | tee -a output.txt   # append

# Process substitution
diff <(command1) <(command2)  # compare outputs without temp files
```

---

## 18. xargs — Build Commands from Input

```bash
# Basic
find . -name "*.log" | xargs rm

# Handle filenames with spaces
find . -name "*.log" -print0 | xargs -0 rm

# Run N processes in parallel
cat hosts.txt | xargs -P 10 -I {} ssh {} "uptime"
# -P 10: 10 parallel processes
# -I {}: replace {} with input

# With a shell function
process() { echo "Processing: $1"; }
export -f process
cat items.txt | xargs -P 5 -I {} bash -c 'process "$@"' _ {}
```

---

## 19. jq — JSON Processing in Shell

```bash
# Parse JSON
echo '{"name":"alice","age":30}' | jq '.name'          # "alice"
echo '{"name":"alice","age":30}' | jq -r '.name'        # alice (raw, no quotes)

# Array
echo '[1,2,3]' | jq '.[]'                               # each element
echo '[1,2,3]' | jq '.[1]'                              # second element
echo '[1,2,3]' | jq 'length'                            # 3

# Filter objects in array
echo '[{"name":"a","status":"ok"},{"name":"b","status":"fail"}]' | \
    jq '.[] | select(.status == "fail") | .name'

# Construct new JSON
kubectl get pods -o json | \
    jq -r '.items[] | select(.status.phase != "Running") | 
    [.metadata.namespace, .metadata.name, .status.phase] | 
    @tsv'

# With default values
echo '{}' | jq '.name // "unknown"'

# Multiple outputs
kubectl get nodes -o json | \
    jq -r '.items[] | .metadata.name + "\t" + .status.conditions[-1].type'
```

---

## 20. Common Gotchas — Senior-Level Awareness

```bash
# 1. Glob expansion in conditionals
files=(*.txt)
if [[ "${#files[@]}" -gt 0 && -f "${files[0]}" ]]; then ...
# If no .txt files, files=("*.txt") — the literal glob string!
# Always check -f before using glob results

# 2. Unintended field splitting with IFS
OLD_IFS="$IFS"
IFS=','
read -ra parts <<< "$csv_line"
IFS="$OLD_IFS"   # always restore

# 3. set -e and subshells
set -e
result=$(false)   # THIS DOES EXIT — command substitution propagates
(false)           # THIS DOES EXIT — subshell exit propagates
{ false; }        # THIS DOES EXIT — group commands propagate

# 4. Integer check
is_integer() {
    [[ "$1" =~ ^-?[0-9]+$ ]]
}

# 5. Comparing numbers as strings
[[ "10" > "9" ]]    # WRONG — string comparison: "1" < "9"
[[ 10 -gt 9 ]]      # CORRECT — numeric comparison

# 6. || and && with set -e
# These suppress -e within the condition
[[ -f file ]] || echo "missing"   # -e won't trigger even if [[ fails

# 7. Read from command that may produce no output
output=$(command 2>/dev/null) || true   # don't fail if empty

# 8. Nested quotes in commands
# Use arrays instead of strings for commands with spaces
cmd=(kubectl get pods -n "$NAMESPACE" -l "app=$APP")
"${cmd[@]}"
```
