# GREP Cheatsheet

> A comprehensive reference for `grep` — the essential pattern-searching tool on all Unix/Linux systems.

---

## Table of Contents

- [Basic Syntax](#basic-syntax)
- [Common Options](#common-options)
- [Output Control](#output-control)
- [Context Lines](#context-lines)
- [Pattern Types](#pattern-types)
- [Basic Regular Expressions (BRE)](#basic-regular-expressions-bre)
- [Extended Regular Expressions (ERE)](#extended-regular-expressions-ere)
- [Character Classes](#character-classes)
- [Anchors & Boundaries](#anchors--boundaries)
- [Searching Files & Directories](#searching-files--directories)
- [Practical Examples](#practical-examples)
- [grep vs egrep vs fgrep](#grep-vs-egrep-vs-fgrep)
- [Quick Reference Card](#quick-reference-card)

---

## Basic Syntax

```
grep [options] pattern [file...]
grep [options] -e pattern [file...]
grep [options] -f patternfile [file...]
```

- If no file is given, grep reads from **stdin**
- Returns exit code `0` if match found, `1` if not, `2` on error

```bash
grep "hello" file.txt
echo "hello world" | grep "hello"
grep "error" /var/log/syslog
```

---

## Common Options

### Matching Behaviour

| Option | Long Form | Description |
|---|---|---|
| `-i` | `--ignore-case` | Case-insensitive matching |
| `-v` | `--invert-match` | Print lines that do **not** match |
| `-w` | `--word-regexp` | Match whole words only |
| `-x` | `--line-regexp` | Match whole lines only |
| `-F` | `--fixed-strings` | Treat pattern as a literal string (no regex) |
| `-E` | `--extended-regexp` | Use extended regex (ERE) |
| `-P` | `--perl-regexp` | Use Perl-compatible regex (PCRE) |
| `-e pat` | `--regexp=pat` | Specify pattern (allows multiple `-e`) |
| `-f file` | `--file=file` | Read patterns from a file |

### Examples

```bash
# Case-insensitive
grep -i "error" file.txt

# Invert match — lines without "debug"
grep -v "debug" file.txt

# Whole word match (won't match "errors")
grep -w "error" file.txt

# Whole line match
grep -x "exact line content" file.txt

# Literal string — no regex interpretation
grep -F "1.2.3" file.txt

# Multiple patterns
grep -e "error" -e "warning" -e "critical" file.txt

# Patterns from a file (one per line)
grep -f patterns.txt file.txt
```

---

## Output Control

| Option | Long Form | Description |
|---|---|---|
| `-c` | `--count` | Print count of matching lines per file |
| `-l` | `--files-with-matches` | Print only filenames with matches |
| `-L` | `--files-without-match` | Print filenames with **no** matches |
| `-o` | `--only-matching` | Print only the matched part, not full line |
| `-q` | `--quiet` | Suppress output; exit code only |
| `-s` | `--no-messages` | Suppress error messages |
| `-n` | `--line-number` | Prefix each match with its line number |
| `-b` | `--byte-offset` | Print byte offset of each match |
| `-H` | `--with-filename` | Always print filename (default with multiple files) |
| `-h` | `--no-filename` | Never print filename |
| `--label=name` | | Use `name` as filename for stdin |

### Examples

```bash
# Count matching lines
grep -c "error" file.txt

# Just the filenames containing matches
grep -l "TODO" *.py

# Print only the matched text
grep -o "[0-9]\+\.[0-9]\+" file.txt

# Silent — use exit code in scripts
if grep -q "pattern" file.txt; then echo "found"; fi

# Show line numbers
grep -n "error" file.txt

# Print filename alongside matches (forced)
grep -H "error" file.txt
```

---

## Context Lines

| Option | Description |
|---|---|
| `-A n` | Print `n` lines **after** each match |
| `-B n` | Print `n` lines **before** each match |
| `-C n` | Print `n` lines **before and after** each match |
| `--group-separator=sep` | Use `sep` between match groups (default: `--`) |
| `--no-group-separator` | Suppress the `--` separator between groups |

```bash
# 3 lines after match
grep -A 3 "error" file.txt

# 2 lines before match
grep -B 2 "function" script.py

# 5 lines of context either side
grep -C 5 "panic" app.log

# Suppress the -- separator
grep -C 2 --no-group-separator "error" file.txt
```

---

## Pattern Types

grep supports four pattern engines, selected by flag:

| Flag | Type | Notes |
|---|---|---|
| *(default)* | BRE — Basic Regular Expressions | `+`, `?`, `\|` need backslash |
| `-E` | ERE — Extended Regular Expressions | `+`, `?`, `\|` work without backslash |
| `-F` | Fixed string | No regex — fastest option |
| `-P` | PCRE — Perl-Compatible Regular Expressions | Lookaheads, `\d`, `\w`, etc. |

---

## Basic Regular Expressions (BRE)

Default mode — some metacharacters require a backslash.

| Pattern | Description |
|---|---|
| `.` | Any single character (except newline) |
| `*` | Zero or more of the preceding |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Any one of: a, b, or c |
| `[^abc]` | Any character NOT in set |
| `\+` | One or more (BRE needs backslash) |
| `\?` | Zero or one (BRE needs backslash) |
| `\{n\}` | Exactly n repetitions |
| `\{n,\}` | n or more repetitions |
| `\{n,m\}` | Between n and m repetitions |
| `\(pat\)` | Grouping (BRE needs backslash) |
| `\1` | Backreference to group 1 |
| `\|` | Alternation — this OR that (BRE needs backslash) |

```bash
# Match lines starting with "Error"
grep "^Error" file.txt

# Match lines ending with a digit
grep "[0-9]$" file.txt

# Match "colour" or "color"
grep "colou\?r" file.txt

# Match 3-digit numbers
grep "\b[0-9]\{3\}\b" file.txt
```

---

## Extended Regular Expressions (ERE)

Use `grep -E` or `egrep`. Metacharacters work without backslashes.

| Pattern | Description |
|---|---|
| `.` | Any single character |
| `*` | Zero or more |
| `+` | One or more |
| `?` | Zero or one |
| `{n}` | Exactly n repetitions |
| `{n,}` | n or more repetitions |
| `{n,m}` | Between n and m repetitions |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Character class |
| `[^abc]` | Negated character class |
| `(pat)` | Grouping |
| `a\|b` | Alternation — a or b |
| `\b` | Word boundary |

```bash
# Match "color" or "colour"
grep -E "colou?r" file.txt

# Match one or more digits
grep -E "[0-9]+" file.txt

# Match "cat" or "dog" or "bird"
grep -E "cat|dog|bird" file.txt

# Match IP address pattern
grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" file.txt

# Match lines with 2 or more consecutive spaces
grep -E " {2,}" file.txt
```

---

## Character Classes

### POSIX Named Classes (use inside `[ ]`)

| Class | Equivalent | Description |
|---|---|---|
| `[:alpha:]` | `[a-zA-Z]` | Letters |
| `[:digit:]` | `[0-9]` | Digits |
| `[:alnum:]` | `[a-zA-Z0-9]` | Letters and digits |
| `[:space:]` | `[ \t\n\r]` | Whitespace |
| `[:blank:]` | `[ \t]` | Space and tab |
| `[:upper:]` | `[A-Z]` | Uppercase letters |
| `[:lower:]` | `[a-z]` | Lowercase letters |
| `[:punct:]` | | Punctuation characters |
| `[:print:]` | | Printable characters |
| `[:graph:]` | | Visible characters (no space) |
| `[:xdigit:]` | `[0-9a-fA-F]` | Hexadecimal digits |

```bash
# Match lines with at least one digit
grep "[[:digit:]]" file.txt

# Match lines with only alphabetic characters
grep -E "^[[:alpha:]]+$" file.txt

# Match hex values
grep -E "0x[[:xdigit:]]+" file.txt
```

### PCRE Shorthand Classes (`-P` only)

| Class | Description |
|---|---|
| `\d` | Digit `[0-9]` |
| `\D` | Non-digit |
| `\w` | Word character `[a-zA-Z0-9_]` |
| `\W` | Non-word character |
| `\s` | Whitespace |
| `\S` | Non-whitespace |

```bash
grep -P "\d{4}-\d{2}-\d{2}" file.txt    # ISO date format
grep -P "\w+@\w+\.\w+" file.txt          # Simple email pattern
```

---

## Anchors & Boundaries

| Anchor | Description |
|---|---|
| `^` | Start of line |
| `$` | End of line |
| `\b` | Word boundary |
| `\B` | Non-word boundary |
| `\<` | Start of word |
| `\>` | End of word |

```bash
# Lines starting with "#" (comments)
grep "^#" config.txt

# Blank lines
grep "^$" file.txt

# Non-blank lines
grep -v "^$" file.txt

# Lines ending with a semicolon
grep ";$" code.c

# Whole word "log" (not "logged", "syslog")
grep "\blog\b" file.txt
# or
grep -w "log" file.txt

# Word starting with "pre"
grep "\bpre" file.txt
```

---

## Searching Files & Directories

| Option | Long Form | Description |
|---|---|---|
| `-r` | `--recursive` | Recurse into subdirectories |
| `-R` | `--dereference-recursive` | Recurse, following symlinks |
| `--include=glob` | | Only search files matching glob |
| `--exclude=glob` | | Skip files matching glob |
| `--exclude-dir=dir` | | Skip directories matching pattern |
| `-d action` | `--directories=action` | How to handle directories: `read`, `recurse`, `skip` |
| `--binary-files=type` | | Handle binaries: `binary`, `text`, `without-match` |
| `-a` | `--text` | Treat binary files as text |
| `-I` | | Ignore binary files |
| `-Z` | `--null` | Print null byte after filename (for `xargs -0`) |

```bash
# Recurse through all files
grep -r "TODO" /home/user/project

# Only search Python files
grep -r --include="*.py" "import os" /project

# Exclude log files
grep -r --exclude="*.log" "error" /var/

# Exclude a directory
grep -r --exclude-dir=".git" "fixme" .

# Safe piping to xargs
grep -rl "TODO" . | xargs grep -l "FIXME"

# Null-delimited for filenames with spaces
grep -rlZ "error" . | xargs -0 rm
```

---

## Practical Examples

### Log Analysis

```bash
# Find all errors in syslog
grep -i "error" /var/log/syslog

# Show errors with 2 lines of context
grep -i -C 2 "error" /var/log/syslog

# Count errors per log file
grep -rc "error" /var/log/*.log

# Extract all IP addresses
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log | sort -u

# Find failed SSH logins
grep "Failed password" /var/log/auth.log

# Top 10 IPs hitting your server
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log | sort | uniq -c | sort -rn | head -10
```

### Code Searching

```bash
# Find all TODO/FIXME comments
grep -rn "TODO\|FIXME" --include="*.py" .

# Find function definitions in C
grep -n "^[a-zA-Z].*(.*)$" *.c

# Find all imports in Python files
grep -r "^import\|^from" --include="*.py" .

# Find hardcoded IP addresses in code
grep -rP "\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b" --include="*.py" .

# Find long lines (over 100 characters)
grep -n ".\{101\}" file.py
```

### System Administration

```bash
# Find users with bash shell
grep "/bin/bash$" /etc/passwd

# Check if a package is installed (Debian/Ubuntu)
dpkg -l | grep -i "nginx"

# Find listening ports
ss -tuln | grep "LISTEN"

# Search running processes
ps aux | grep "python" | grep -v grep

# Find files containing a pattern (with filenames only)
grep -rl "password" /etc/

# Check cron jobs mentioning a script
grep -r "backup.sh" /etc/cron*
```

### Text Processing

```bash
# Extract email addresses
grep -Eo "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt

# Extract URLs
grep -Eo "https?://[^ ]+" file.txt

# Extract lines between two patterns
grep -A 1000 "START" file.txt | grep -B 1000 "END"

# Remove blank lines and comments
grep -v "^$\|^#" config.conf

# Count non-empty, non-comment lines
grep -cE "^[^#]" config.conf

# Find duplicate lines
sort file.txt | grep -c "^" && sort file.txt | uniq -d
```

### Scripting & Pipelines

```bash
# Exit-code based check in shell scripts
if grep -q "^root:" /etc/passwd; then
  echo "root account exists"
fi

# Count matches
match_count=$(grep -c "pattern" file.txt)

# Get only the matching part with -o
grep -oE "[0-9]+" numbers.txt

# Combine with xargs
grep -rl "old_function" . | xargs sed -i 's/old_function/new_function/g'

# Chain grep filters
cat server.log | grep "2025" | grep "ERROR" | grep -v "timeout"

# Use with wc to count
grep -c "pattern" file.txt        # preferred
grep "pattern" file.txt | wc -l  # alternative
```

---

## grep vs egrep vs fgrep

| Command | Equivalent | Use Case |
|---|---|---|
| `grep` | `grep` (BRE) | Default; good for simple patterns |
| `egrep` | `grep -E` | Extended regex; `+`, `?`, `\|` without backslash |
| `fgrep` | `grep -F` | Fixed strings; fastest; no regex at all |
| `grep -P` | PCRE | Perl regex; `\d`, `\w`, lookaheads |

> `egrep` and `fgrep` are deprecated in favour of `grep -E` and `grep -F`, but still widely available.

```bash
# These are equivalent:
egrep "cats?|dogs?"  file.txt
grep -E "cats?|dogs?" file.txt

# These are equivalent (literal string, no regex):
fgrep "1.2.3.4" file.txt
grep -F "1.2.3.4" file.txt
```

---

## Quick Reference Card

```
grep [options] pattern [file...]

MATCH CONTROL                   OUTPUT CONTROL
  -i   ignore case                -n   line numbers
  -v   invert match               -c   count matches
  -w   whole word                 -l   filenames only
  -x   whole line                 -L   filenames without match
  -F   fixed string               -o   only matching part
  -E   extended regex             -q   quiet (exit code only)
  -P   Perl regex                 -H   always show filename
  -e   specify pattern            -h   never show filename
  -f   patterns from file

CONTEXT                         FILES
  -A n  n lines after             -r   recursive
  -B n  n lines before            --include=GLOB
  -C n  n lines either side       --exclude=GLOB
                                  --exclude-dir=DIR
                                  -I   ignore binaries

ANCHORS           QUANTIFIERS (ERE / -E)
  ^  start         *   zero or more
  $  end           +   one or more
  \b word boundary ?   zero or one
  \< word start    {n,m} n to m times
  \> word end      |   alternation
```

---

> **Further Reading:**
> - `man grep`
> - `grep --help`
> - [GNU grep manual](https://www.gnu.org/software/grep/manual/)
