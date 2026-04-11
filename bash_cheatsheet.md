# Bash Cheatsheet

## Table of Contents
1. [Shebang & Script Basics](#shebang--script-basics)
2. [Variables](#variables)
3. [Special Variables](#special-variables)
4. [String Operations](#string-operations)
5. [Arrays](#arrays)
6. [Arithmetic](#arithmetic)
7. [Conditionals](#conditionals)
8. [Loops](#loops)
9. [Functions](#functions)
10. [Input & Output](#input--output)
11. [Redirection & Pipes](#redirection--pipes)
12. [Process Substitution](#process-substitution)
13. [Command Substitution](#command-substitution)
14. [File Tests](#file-tests)
15. [String Tests](#string-tests)
16. [Numeric Tests](#numeric-tests)
17. [Glob & Pattern Matching](#glob--pattern-matching)
18. [Here Documents & Here Strings](#here-documents--here-strings)
19. [Error Handling](#error-handling)
20. [Debugging](#debugging)
21. [Job Control](#job-control)
22. [Signals & Traps](#signals--traps)
23. [Useful Built-ins](#useful-built-ins)
24. [Quick Reference Card](#quick-reference-card)

---

## Shebang & Script Basics

```bash
#!/usr/bin/env bash          # Preferred shebang (portable)
#!/bin/bash                  # Direct path shebang

# Make a script executable
chmod +x script.sh

# Run a script
./script.sh
bash script.sh

# Run with options
bash -x script.sh            # Debug (trace execution)
bash -n script.sh            # Check syntax without running
bash -e script.sh            # Exit on first error
```

---

## Variables

```bash
# Assignment (no spaces around =)
name="Alice"
count=42
pi=3.14

# Access a variable
echo $name
echo ${name}                 # Preferred — unambiguous

# Unset a variable
unset name

# Read-only variable
readonly MAX=100

# Export to child processes
export PATH="$PATH:/custom/bin"

# Default value if unset or empty
echo ${var:-"default"}       # Use default, don't assign
echo ${var:="default"}       # Use default AND assign to var
echo ${var:?"error msg"}     # Exit with error if unset
echo ${var:+"other"}         # Use "other" only if var IS set

# Indirect reference
ref="name"
echo ${!ref}                 # Prints value of $name
```

---

## Special Variables

| Variable | Description |
|----------|-------------|
| `$0` | Name of the script |
| `$1` … `$9` | Positional arguments 1–9 |
| `${10}` … | Positional arguments 10+ |
| `$#` | Number of arguments |
| `$@` | All arguments as separate words |
| `$*` | All arguments as a single word |
| `$?` | Exit code of last command |
| `$$` | PID of current shell |
| `$!` | PID of last background process |
| `$-` | Current shell option flags |
| `$_` | Last argument of previous command |
| `$IFS` | Internal Field Separator |
| `$HOME` | Home directory |
| `$PWD` | Current directory |
| `$OLDPWD` | Previous directory |
| `$RANDOM` | Random integer (0–32767) |
| `$LINENO` | Current line number in script |
| `$BASH_VERSION` | Bash version string |
| `$FUNCNAME` | Current function name |

---

## String Operations

```bash
str="Hello, World!"

# Length
echo ${#str}                 # 13

# Substring: ${var:offset:length}
echo ${str:7:5}              # World

# Uppercase / Lowercase (Bash 4+)
echo ${str^^}                # HELLO, WORLD!
echo ${str,,}                # hello, world!
echo ${str^}                 # Capitalise first char
echo ${str,}                 # Lowercase first char

# Remove prefix (shortest match)
echo ${str#Hello, }          # World!

# Remove prefix (longest/greedy match)
echo ${str##*,}              # (space)World!

# Remove suffix (shortest match)
echo ${str%World!}           # Hello, 

# Remove suffix (longest/greedy match)
echo ${str%%,*}              # Hello

# Find & replace (first occurrence)
echo ${str/World/Bash}       # Hello, Bash!

# Find & replace (all occurrences)
echo ${str//l/L}             # HeLLo, WorLd!

# Replace prefix
echo ${str/#Hello/Hi}        # Hi, World!

# Replace suffix
echo ${str/%!/...}           # Hello, World...

# Check if string contains substring
if [[ "$str" == *"World"* ]]; then
  echo "Found"
fi
```

---

## Arrays

```bash
# Declare
fruits=("apple" "banana" "cherry")
declare -a nums=(1 2 3 4 5)

# Access element
echo ${fruits[0]}            # apple

# All elements
echo ${fruits[@]}            # apple banana cherry

# All indices
echo ${!fruits[@]}           # 0 1 2

# Length
echo ${#fruits[@]}           # 3

# Append
fruits+=("date")

# Modify element
fruits[1]="blueberry"

# Delete element (leaves gap)
unset fruits[2]

# Slice: ${arr[@]:offset:length}
echo ${fruits[@]:1:2}        # blueberry date

# Iterate
for fruit in "${fruits[@]}"; do
  echo "$fruit"
done

# Associative arrays (Bash 4+)
declare -A person
person[name]="Alice"
person[age]=30
echo ${person[name]}
echo ${!person[@]}           # All keys
echo ${person[@]}            # All values
```

---

## Arithmetic

```bash
# $(( )) — integer arithmetic
echo $(( 3 + 4 ))            # 7
echo $(( 10 % 3 ))           # 1
echo $(( 2 ** 8 ))           # 256

# let
let x=5+3
let x++
let x*=2

# Increment / Decrement
(( x++ ))
(( x-- ))
(( x += 5 ))

# Assign result
result=$(( x * 2 ))

# Ternary
echo $(( x > 5 ? 1 : 0 ))

# Floating point (requires bc or awk)
echo "scale=2; 10 / 3" | bc          # 3.33
awk 'BEGIN { printf "%.4f\n", 10/3 }' # 3.3333
```

---

## Conditionals

### if / elif / else

```bash
if [[ condition ]]; then
  # ...
elif [[ other ]]; then
  # ...
else
  # ...
fi
```

### case

```bash
case "$var" in
  "start")
    echo "Starting"
    ;;
  "stop"|"quit")
    echo "Stopping"
    ;;
  [0-9]*)
    echo "Starts with digit"
    ;;
  *)
    echo "Unknown"
    ;;
esac
```

### Short-circuit Operators

```bash
command1 && command2         # Run command2 only if command1 succeeds
command1 || command2         # Run command2 only if command1 fails
command1 ; command2          # Run both regardless
```

### Single vs Double Brackets

| Feature | `[ ]` (POSIX) | `[[ ]]` (Bash) |
|---------|---------------|----------------|
| Pattern matching | ✗ | ✓ (`==`) |
| Regex matching | ✗ | ✓ (`=~`) |
| `&&` / `||` inside | ✗ | ✓ |
| Word splitting | Yes (quote carefully) | No |
| Preferred for Bash | — | ✓ |

```bash
# Regex match
if [[ "$email" =~ ^[a-z]+@[a-z]+\.[a-z]+$ ]]; then
  echo "Valid email format"
fi
```

---

## Loops

### for

```bash
# Range
for i in {1..5}; do echo $i; done

# Range with step
for i in {0..20..5}; do echo $i; done

# C-style
for (( i=0; i<5; i++ )); do echo $i; done

# Iterate array
for item in "${arr[@]}"; do echo "$item"; done

# Iterate files
for file in *.txt; do echo "$file"; done

# Iterate command output
for user in $(cut -d: -f1 /etc/passwd); do echo "$user"; done
```

### while

```bash
# Basic
while [[ condition ]]; do
  # ...
done

# Read lines from a file
while IFS= read -r line; do
  echo "$line"
done < file.txt

# Read lines from a command
while IFS= read -r line; do
  echo "$line"
done < <(command)

# Infinite loop
while true; do
  sleep 1
done
```

### until

```bash
until [[ condition ]]; do
  # Runs while condition is FALSE
done
```

### Loop Control

```bash
break        # Exit loop
break 2      # Exit 2 levels of nested loops
continue     # Skip to next iteration
```

---

## Functions

```bash
# Define (two styles)
greet() {
  echo "Hello, $1!"
}

function greet {
  echo "Hello, $1!"
}

# Call
greet "Alice"

# Return value (0–255 only, use for exit status)
is_even() {
  (( $1 % 2 == 0 )) && return 0 || return 1
}

# Return strings via echo + command substitution
get_date() {
  echo "$(date +%Y-%m-%d)"
}
today=$(get_date)

# Local variables
my_func() {
  local x=10
  echo $x
}

# Default parameter
greet() {
  local name="${1:-World}"
  echo "Hello, $name!"
}

# Variadic function
sum_all() {
  local total=0
  for n in "$@"; do (( total += n )); done
  echo $total
}
```

---

## Input & Output

```bash
# Read user input
read name
read -p "Enter name: " name
read -sp "Password: " pass     # Silent (no echo)
read -t 5 -p "Timeout in 5s: " ans   # Timeout
read -n 1 key                  # Read single character
read -a arr                    # Read into array

# Print
echo "Hello"
echo -n "No newline"
echo -e "Tab:\there"           # Enable escape sequences
printf "%-10s %5d\n" "item" 42  # Formatted output

# printf format specifiers
# %s  string    %d  integer    %f  float
# %x  hex       %o  octal      %-  left-align
# %05d  zero-padded
```

---

## Redirection & Pipes

```bash
# Redirect stdout
command > file.txt             # Overwrite
command >> file.txt            # Append

# Redirect stderr
command 2> errors.txt

# Redirect both stdout and stderr
command > out.txt 2>&1
command &> out.txt             # Bash shorthand

# Append both
command &>> out.txt

# Redirect stdin
command < input.txt

# Discard output
command > /dev/null 2>&1

# Pipes
command1 | command2            # Pipe stdout to stdin
command1 |& command2           # Pipe stdout + stderr

# tee — write to file AND stdout
command | tee file.txt
command | tee -a file.txt      # Append mode
```

---

## Process Substitution

```bash
# Treat command output as a file
diff <(sort file1.txt) <(sort file2.txt)

# Read from multiple sources
while read line; do
  echo "$line"
done < <(grep "pattern" *.log)

# Write to a command as if it were a file
command > >(tee output.log)
```

---

## Command Substitution

```bash
# $() — preferred
files=$(ls -1)
now=$(date +%s)

# Backticks — legacy, avoid nesting
files=`ls -1`

# Nest substitutions
lines=$(wc -l < $(find . -name "*.log" | head -1))
```

---

## File Tests

| Operator | True if… |
|----------|----------|
| `-e file` | File exists |
| `-f file` | Regular file exists |
| `-d file` | Directory exists |
| `-L file` | Symlink exists |
| `-r file` | File is readable |
| `-w file` | File is writable |
| `-x file` | File is executable |
| `-s file` | File exists and is non-empty |
| `-z file` | File exists and is empty |
| `f1 -nt f2` | f1 is newer than f2 |
| `f1 -ot f2` | f1 is older than f2 |
| `f1 -ef f2` | f1 and f2 are the same file |

```bash
if [[ -f "/etc/passwd" ]]; then echo "exists"; fi
if [[ ! -d "/tmp/mydir" ]]; then mkdir /tmp/mydir; fi
```

---

## String Tests

| Operator | True if… |
|----------|----------|
| `-z "$str"` | String is empty |
| `-n "$str"` | String is non-empty |
| `"$a" == "$b"` | Strings are equal |
| `"$a" != "$b"` | Strings are not equal |
| `"$a" < "$b"` | a sorts before b (lexicographic) |
| `"$a" > "$b"` | a sorts after b |
| `"$a" =~ regex` | String matches regex (`[[ ]]` only) |

---

## Numeric Tests

| Operator | Meaning |
|----------|---------|
| `-eq` | Equal |
| `-ne` | Not equal |
| `-lt` | Less than |
| `-le` | Less than or equal |
| `-gt` | Greater than |
| `-ge` | Greater than or equal |

```bash
if [[ $x -gt 10 ]]; then echo "big"; fi
# Or with (( ))
if (( x > 10 )); then echo "big"; fi
```

---

## Glob & Pattern Matching

```bash
*          # Match any string (including empty)
?          # Match any single character
[abc]      # Match a, b, or c
[a-z]      # Match any lowercase letter
[^abc]     # Match any char NOT a, b, or c

# Extended globs (enable with: shopt -s extglob)
?(pat)     # Zero or one occurrence
*(pat)     # Zero or more occurrences
+(pat)     # One or more occurrences
@(pat)     # Exactly one occurrence
!(pat)     # Anything NOT matching pat

# Examples
ls *.{jpg,png}              # Multiple extensions
ls file[0-9][0-9].txt       # file00.txt … file99.txt
shopt -s extglob
ls !(*.log)                 # All files except .log
ls +(foo|bar)*              # Files starting with foo or bar

# nullglob — expand to nothing if no match
shopt -s nullglob

# globstar — ** matches across directories (Bash 4+)
shopt -s globstar
ls **/*.sh                  # All .sh files recursively
```

---

## Here Documents & Here Strings

```bash
# Here document — multiline stdin
cat <<EOF
Line 1
Line 2
$variable is expanded
EOF

# Indented here doc (strip leading tabs)
cat <<-EOF
	Indented with tab
	$variable still expands
	EOF

# Quoted delimiter — no variable expansion
cat <<'EOF'
Literal $variable — not expanded
EOF

# Here string — single-line stdin
grep "pattern" <<< "some string to search"
base64 <<< "encode this"
```

---

## Error Handling

```bash
# Exit on error
set -e                       # Exit if any command fails
set -u                       # Treat unset variables as errors
set -o pipefail              # Catch errors in pipelines
set -euo pipefail            # All three together (recommended)

# Disable temporarily
set +e
risky_command
set -e

# Custom error handler
trap 'echo "Error on line $LINENO"; exit 1' ERR

# Check exit code manually
if ! command; then
  echo "Command failed"
  exit 1
fi

command || { echo "Failed"; exit 1; }

# Exit with a message
die() {
  echo "ERROR: $*" >&2
  exit 1
}
[[ -f "$file" ]] || die "File not found: $file"
```

---

## Debugging

```bash
# Trace execution (print each command before running)
set -x
# Disable trace
set +x

# Trace a section only
set -x
some_command
set +x

# Run entire script in debug mode
bash -x script.sh

# Check syntax only (no execution)
bash -n script.sh

# Verbose mode (print each line as read)
bash -v script.sh

# Print debug to stderr from within a script
>&2 echo "DEBUG: value=$var"

# Check where a command is found
type -a command
which command
```

---

## Job Control

```bash
command &            # Run in background
jobs                 # List background jobs
fg                   # Bring last job to foreground
fg %2                # Bring job 2 to foreground
bg                   # Resume stopped job in background
bg %2                # Resume job 2 in background
Ctrl+Z               # Suspend foreground job
Ctrl+C               # Kill foreground job
kill %1              # Kill job 1
wait                 # Wait for all background jobs
wait $!              # Wait for last background process
wait $pid            # Wait for specific PID
disown %1            # Detach job from shell (survives logout)
nohup command &      # Run immune to hangup signals
```

---

## Signals & Traps

```bash
# Trap a signal
trap 'echo "Caught SIGINT"' INT
trap 'cleanup_function' EXIT     # Runs on any exit
trap 'rm -f /tmp/tmpfile' EXIT TERM INT

# Ignore a signal
trap '' SIGPIPE

# Reset to default
trap - INT

# Common signals
# INT   (2)  — Ctrl+C
# TERM  (15) — Default kill signal
# HUP   (1)  — Hangup / reload config
# KILL  (9)  — Forceful kill (cannot be trapped)
# EXIT  (0)  — Script exit (any cause)
# ERR        — Any command fails (with set -e)
# DEBUG      — Before every command

# Cleanup pattern
tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT
```

---

## Useful Built-ins

```bash
# source / . — run script in current shell
source ~/.bashrc
. ~/.bashrc

# eval — execute a string as a command
eval "echo hello"

# exec — replace current shell with command
exec bash                    # Replace shell

# printf — formatted output (prefer over echo)
printf "%s has %d items\n" "list" 5

# declare — set variable attributes
declare -i num=5             # Integer
declare -r CONST="fixed"     # Readonly
declare -a arr               # Array
declare -A map               # Associative array
declare -l lower             # Auto lowercase
declare -u upper             # Auto uppercase
declare -x exported          # Export to env

# typeset — alias for declare
# local — limit scope to function
# readonly — make variable constant

# shift — shift positional parameters
shift                        # Discard $1, $2→$1, etc.
shift 2                      # Discard $1 and $2

# getopts — parse short options
while getopts ":a:b:h" opt; do
  case $opt in
    a) aval="$OPTARG" ;;
    b) bval="$OPTARG" ;;
    h) usage; exit 0 ;;
    :) echo "Option -$OPTARG requires argument" ;;
    ?) echo "Unknown option: -$OPTARG" ;;
  esac
done
shift $(( OPTIND - 1 ))      # Remove parsed options

# mapfile / readarray — read lines into array (Bash 4+)
mapfile -t lines < file.txt
readarray -t lines < file.txt

# select — interactive menu
select choice in "Option A" "Option B" "Quit"; do
  case $choice in
    "Option A") echo "A chosen"; break ;;
    "Option B") echo "B chosen"; break ;;
    "Quit")     exit 0 ;;
  esac
done
```

---

## Quick Reference Card

```
VARIABLES
  $var / ${var}               Access variable
  ${var:-default}             Fallback if unset
  ${#var}                     String length
  ${var:2:5}                  Substring (offset:length)
  ${var/old/new}              Replace first
  ${var//old/new}             Replace all
  ${var#prefix}               Strip prefix (short)
  ${var%suffix}               Strip suffix (short)

TESTS
  [[ -f $f ]]                 File exists
  [[ -d $d ]]                 Directory exists
  [[ -z $s ]]                 String empty
  [[ -n $s ]]                 String non-empty
  [[ $a == $b ]]              String equal
  [[ $n -eq $m ]]             Numeric equal
  (( n > m ))                 Arithmetic test

LOOPS
  for i in {1..10}; do        Range loop
  for f in *.txt; do          Glob loop
  while IFS= read -r l; do    Line-by-line read
  done < file                 (close while loop)

REDIRECTS
  >                           Stdout overwrite
  >>                          Stdout append
  2>                          Stderr
  &>                          Stdout + stderr
  < file                      Stdin from file
  |                           Pipe
  <()                         Process substitution (read)

SCRIPT SAFETY
  set -euo pipefail           Exit on error, unset var, pipe fail
  trap 'cleanup' EXIT         Always run cleanup
  local var=value             Scope to function
  "$var"                      Always quote variables

USEFUL PATTERNS
  cmd || { echo "fail"; exit 1; }    Die on failure
  val=$(cmd)                         Capture output
  [[ $x =~ ^[0-9]+$ ]]              Regex test
  printf "%s\n" "${arr[@]}"          Safe array print
  mapfile -t arr < file.txt          File to array
```

---

*See `man bash` or https://www.gnu.org/software/bash/manual/ for complete documentation.*
