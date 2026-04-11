# AWK Cheatsheet

> A comprehensive reference for `awk` — the powerful text-processing tool available on all Unix/Linux systems.

---

## Table of Contents

- [Basic Syntax](#basic-syntax)
- [Running AWK](#running-awk)
- [Built-in Variables](#built-in-variables)
- [Patterns](#patterns)
- [Actions](#actions)
- [Operators](#operators)
- [String Functions](#string-functions)
- [Math Functions](#math-functions)
- [Input/Output](#inputoutput)
- [Arrays](#arrays)
- [Control Flow](#control-flow)
- [User-Defined Functions](#user-defined-functions)
- [Field Separators](#field-separators)
- [Multiple File Processing](#multiple-file-processing)
- [Practical Examples](#practical-examples)
- [AWK vs GAWK](#awk-vs-gawk)

---

## Basic Syntax

```
awk 'pattern { action }' file
awk 'BEGIN { setup } pattern { action } END { teardown }' file
```

AWK processes input **line by line**. For each line (record), it tests every `pattern` and runs the matching `action`.

### Structure of an AWK Program

```
BEGIN   { runs before any input is read     }
/regex/ { runs for lines matching regex     }
expr    { runs when expression is true      }
END     { runs after all input is processed }
```

---

## Running AWK

| Command | Description |
|---|---|
| `awk '{ print }' file` | Print every line |
| `awk '{ print $1 }' file` | Print first field of every line |
| `awk -f script.awk file` | Run AWK program from a file |
| `awk -v var=value '{ ... }' file` | Pass a shell variable into AWK |
| `command \| awk '{ ... }'` | Pipe output into AWK |
| `awk '{ ... }' file1 file2` | Process multiple files |
| `awk 'NR==5'` | Print only line 5 |

---

## Built-in Variables

| Variable | Description |
|---|---|
| `$0` | Entire current record (line) |
| `$1, $2, ...` | Field 1, field 2, etc. |
| `NF` | Number of fields in current record |
| `NR` | Total number of records read so far |
| `FNR` | Record number within the current file |
| `FS` | Input field separator (default: space/tab) |
| `OFS` | Output field separator (default: space) |
| `RS` | Input record separator (default: newline) |
| `ORS` | Output record separator (default: newline) |
| `FILENAME` | Name of the current input file |
| `ARGC` | Number of command-line arguments |
| `ARGV` | Array of command-line arguments |
| `ENVIRON` | Array of environment variables |
| `SUBSEP` | Separator for multi-dimensional arrays (`\034`) |

### Examples

```bash
# Print the number of fields on each line
awk '{ print NF }' file

# Print last field of each line
awk '{ print $NF }' file

# Print second-to-last field
awk '{ print $(NF-1) }' file

# Print line number alongside each line
awk '{ print NR": "$0 }' file

# Use a custom output field separator
awk 'BEGIN { OFS="," } { print $1, $2, $3 }' file
```

---

## Patterns

Patterns control which lines trigger an action.

| Pattern | Description |
|---|---|
| `BEGIN` | Before any input is read |
| `END` | After all input is read |
| `/regex/` | Lines matching the regular expression |
| `!/regex/` | Lines **not** matching the regex |
| `expr` | Lines where expression is true |
| `pat1, pat2` | Range: from line matching `pat1` to `pat2` |

### Examples

```bash
# Lines containing "error"
awk '/error/ { print }' file

# Lines NOT containing "error"
awk '!/error/ { print }' file

# Lines where field 3 is greater than 100
awk '$3 > 100' file

# Lines 5 through 10 (range pattern)
awk 'NR==5, NR==10' file

# Lines between two patterns (inclusive)
awk '/START/, /END/' file

# Empty pattern matches all lines
awk '{ print $1 }' file
```

---

## Actions

Actions are enclosed in `{ }`. Multiple statements are separated by `;` or newlines.

```bash
# Multiple actions on one line
awk '{ x = $1; y = $2; print x + y }' file

# No action — default is print $0
awk '/pattern/' file

# No pattern — runs on every line
awk '{ print $2 }' file
```

---

## Operators

### Arithmetic

| Operator | Description |
|---|---|
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Division |
| `%` | Modulo |
| `^` or `**` | Exponentiation |
| `++` / `--` | Increment / Decrement |
| `+=`, `-=`, `*=`, `/=`, `%=`, `^=` | Assignment operators |

### Comparison

| Operator | Description |
|---|---|
| `==` | Equal to |
| `!=` | Not equal to |
| `<` | Less than |
| `<=` | Less than or equal |
| `>` | Greater than |
| `>=` | Greater than or equal |

### Logical

| Operator | Description |
|---|---|
| `&&` | Logical AND |
| `\|\|` | Logical OR |
| `!` | Logical NOT |

### String

| Operator | Description |
|---|---|
| `" " " "` (space) | String concatenation |
| `~` | Matches regex |
| `!~` | Does not match regex |

```bash
# Regex match operator
awk '$2 ~ /^[0-9]+$/ { print "numeric:", $2 }' file

# String concatenation
awk '{ full = $1 " " $2; print full }' file
```

### Ternary

```bash
awk '{ status = ($3 > 50) ? "pass" : "fail"; print $1, status }' file
```

---

## String Functions

| Function | Description |
|---|---|
| `length(s)` | Length of string `s` (or `$0` if omitted) |
| `substr(s, i)` | Substring from position `i` to end |
| `substr(s, i, n)` | Substring from position `i`, length `n` |
| `index(s, t)` | Position of `t` in `s` (0 if not found) |
| `split(s, a, sep)` | Split `s` into array `a` using `sep`; returns count |
| `sub(r, s, t)` | Replace **first** match of regex `r` with `s` in `t` |
| `gsub(r, s, t)` | Replace **all** matches of regex `r` with `s` in `t` |
| `match(s, r)` | Position of regex `r` in `s`; sets `RSTART`, `RLENGTH` |
| `sprintf(fmt, ...)` | Format string (like `printf`, returns string) |
| `tolower(s)` | Convert to lowercase |
| `toupper(s)` | Convert to uppercase |

### Examples

```bash
# Get length of field 1
awk '{ print length($1) }' file

# Extract characters 2–5
awk '{ print substr($0, 2, 4) }' file

# Replace first occurrence
awk '{ sub(/foo/, "bar"); print }' file

# Replace all occurrences
awk '{ gsub(/foo/, "bar"); print }' file

# Split a field by a delimiter
awk '{ n = split($1, parts, ":"); print parts[1], parts[2] }' file

# Convert to uppercase
awk '{ print toupper($0) }' file

# Find position of a substring
awk '{ print index($0, "error") }' file
```

---

## Math Functions

| Function | Description |
|---|---|
| `sin(x)` | Sine (radians) |
| `cos(x)` | Cosine (radians) |
| `atan2(y, x)` | Arctangent of y/x |
| `exp(x)` | e raised to power x |
| `log(x)` | Natural logarithm |
| `sqrt(x)` | Square root |
| `int(x)` | Truncate to integer |
| `rand()` | Random float: `0 <= n < 1` |
| `srand(seed)` | Set random seed (uses time if no arg) |

```bash
# Round a number
awk '{ print int($1 + 0.5) }' file

# Random integer 1–100
awk 'BEGIN { srand(); print int(rand() * 100) + 1 }'
```

---

## Input/Output

### print vs printf

```bash
# print adds ORS (newline) automatically
awk '{ print $1, $2 }' file

# printf gives full format control (no automatic newline)
awk '{ printf "%-10s %5d\n", $1, $2 }' file
```

### printf Format Specifiers

| Specifier | Description |
|---|---|
| `%s` | String |
| `%d` | Integer |
| `%f` | Float |
| `%e` | Scientific notation |
| `%g` | Shorter of `%f` or `%e` |
| `%o` | Octal |
| `%x` | Hexadecimal |
| `%%` | Literal `%` |
| `%-10s` | Left-align, width 10 |
| `%05d` | Zero-pad, width 5 |
| `%.2f` | 2 decimal places |

### Redirecting Output

```bash
# Write to a file (overwrites)
awk '{ print $0 > "output.txt" }' file

# Append to a file
awk '{ print $0 >> "output.txt" }' file

# Pipe to another command
awk '{ print $1 | "sort -u" }' file

# Print to stderr
awk '{ print "Error: "$0 > "/dev/stderr" }' file
```

### Reading Input

```bash
# Read from another file inside AWK
awk '{ while ((getline line < "other.txt") > 0) print line }' file

# Read output of a command
awk 'BEGIN { while ("date" | getline line) print line }'

# getline return values: 1=success, 0=EOF, -1=error
```

---

## Arrays

AWK arrays are **associative** (key-value pairs). Keys can be strings or numbers.

```bash
# Assign to an array
awk '{ count[$1]++ } END { for (k in count) print k, count[k] }' file

# Check if key exists
awk '{ if ("key" in myarray) print "found" }' file

# Delete a key
awk '{ delete myarray["key"] }' file

# Delete entire array
awk '{ delete myarray }' file

# Multi-dimensional arrays (simulated with SUBSEP)
awk '{ matrix[$1][$2] = $3 }' file
# Equivalent:
awk '{ matrix[$1, $2] = $3 }' file
```

### Looping Over Arrays

```bash
# Unordered iteration
awk 'END { for (key in arr) print key, arr[key] }' file

# Sorted keys (gawk only)
awk 'END { PROCINFO["sorted_in"] = "@ind_str_asc"; for (k in arr) print k }' file
```

---

## Control Flow

### if / else

```bash
awk '{
  if ($1 > 100) {
    print "high"
  } else if ($1 > 50) {
    print "medium"
  } else {
    print "low"
  }
}' file
```

### while

```bash
awk '{
  i = 1
  while (i <= NF) {
    print i, $i
    i++
  }
}' file
```

### do-while

```bash
awk '{
  i = 1
  do {
    print $i
    i++
  } while (i <= NF)
}' file
```

### for

```bash
# C-style for loop
awk '{
  for (i = 1; i <= NF; i++) {
    print i, $i
  }
}' file

# for-in (array iteration)
awk '{
  for (key in arr) {
    print key, arr[key]
  }
}' file
```

### break / continue / next / exit

| Statement | Description |
|---|---|
| `break` | Exit current loop |
| `continue` | Skip to next iteration |
| `next` | Skip to next input record |
| `nextfile` | Skip to next input file (gawk) |
| `exit [code]` | Stop processing, run END block |

```bash
# Skip blank lines
awk 'NF == 0 { next } { print }' file

# Stop after first match
awk '/pattern/ { print; exit }' file
```

---

## User-Defined Functions

```awk
function name(param1, param2,    localvar1, localvar2) {
  # local variables are declared as extra parameters (by convention, with extra spaces)
  statements
  return value
}
```

```bash
awk '
function max(a, b) {
  return (a > b) ? a : b
}
function min(a, b) {
  return (a < b) ? a : b
}
{
  print "max:", max($1, $2), "min:", min($1, $2)
}
' file
```

> **Tip:** AWK passes scalars by value and arrays by reference.

---

## Field Separators

### Input Field Separator (FS)

```bash
# Use comma as separator
awk -F',' '{ print $1 }' file

# Use colon as separator
awk -F':' '{ print $1 }' /etc/passwd

# Use regex as separator
awk -F'[,;:]' '{ print $1 }' file

# Set inside BEGIN
awk 'BEGIN { FS=":" } { print $1 }' /etc/passwd

# Multiple characters as literal separator
awk -F'::' '{ print $1 }' file
```

### Output Field Separator (OFS)

```bash
# Change field and reassign to trigger OFS
awk 'BEGIN { OFS="," } { $1=$1; print }' file

# Note: simply printing with commas uses OFS
awk 'BEGIN { OFS="\t" } { print $1, $2, $3 }' file
```

### Record Separators

```bash
# Use blank lines as record separators (paragraph mode)
awk 'BEGIN { RS="" } { print NR, $0 }' file

# Use a custom record separator
awk 'BEGIN { RS="---" } { print NR": "$0 }' file
```

---

## Multiple File Processing

```bash
# FNR resets per file; NR does not
awk '{ print FILENAME, FNR, NR, $0 }' file1 file2

# Process only the first file
awk 'FNR==NR { store[$0]=1; next } $0 in store' file1 file2

# Print lines in file2 that are NOT in file1
awk 'FNR==NR { seen[$0]=1; next } !($0 in seen)' file1 file2
```

---

## Practical Examples

### System Administration

```bash
# List all users from /etc/passwd
awk -F':' '{ print $1 }' /etc/passwd

# Find users with UID >= 1000
awk -F':' '$3 >= 1000 { print $1, $3 }' /etc/passwd

# Show disk usage over 1GB
df -h | awk 'NR>1 && $5+0 > 80 { print $6, $5 }'

# Count processes per user
ps aux | awk 'NR>1 { count[$1]++ } END { for (u in count) print u, count[u] }'

# Parse /var/log/syslog for errors
awk '/ERROR/ { print NR": "$0 }' /var/log/syslog
```

### Text Processing

```bash
# Count word frequency
awk '{ for (i=1; i<=NF; i++) freq[$i]++ }
     END { for (w in freq) print freq[w], w }' file | sort -rn

# Print lines between two patterns (exclusive)
awk '/START/ { found=1; next } /END/ { found=0 } found' file

# Remove duplicate lines (preserving order)
awk '!seen[$0]++' file

# Print only unique lines
awk '{ count[$0]++ } END { for (l in count) if (count[l]==1) print l }' file

# Double-space a file
awk '{ print; print "" }' file

# Number all lines
awk '{ printf "%4d  %s\n", NR, $0 }' file

# Strip leading/trailing whitespace
awk '{ gsub(/^[ \t]+|[ \t]+$/, ""); print }' file
```

### CSV Processing

```bash
# Print specific columns from CSV
awk -F',' '{ print $1, $3 }' data.csv

# Sum a column
awk -F',' 'NR>1 { sum += $2 } END { print "Total:", sum }' data.csv

# Filter rows where column 3 > 100
awk -F',' 'NR==1 || $3 > 100' data.csv

# Convert CSV to TSV
awk 'BEGIN { FS=","; OFS="\t" } { $1=$1; print }' data.csv

# Count rows (excluding header)
awk -F',' 'END { print NR-1, "records" }' data.csv
```

### Log Analysis

```bash
# Count HTTP status codes in access log
awk '{ count[$9]++ } END { for (s in count) print s, count[s] }' access.log

# Find top 10 IPs by request count
awk '{ count[$1]++ } END { for (ip in count) print count[ip], ip }' access.log \
  | sort -rn | head -10

# Calculate average response time
awk '{ sum += $NF; count++ } END { print "Avg:", sum/count "ms" }' access.log

# Extract requests in a time range
awk '$4 >= "[01/Jan/2025" && $4 <= "[31/Jan/2025"' access.log
```

### Arithmetic & Statistics

```bash
# Sum column 1
awk '{ sum += $1 } END { print sum }' file

# Average of column 1
awk '{ sum += $1; n++ } END { print sum/n }' file

# Min and max
awk 'NR==1 { min=max=$1 }
     { if ($1<min) min=$1; if ($1>max) max=$1 }
     END { print "min:", min, "max:", max }' file

# Running total
awk '{ total += $1; print $0, total }' file
```

---

## AWK vs GAWK

`gawk` (GNU AWK) is the most common AWK implementation on Linux and includes many extensions.

| Feature | AWK | GAWK |
|---|---|---|
| `nextfile` | ❌ | ✅ |
| `PROCINFO` array | ❌ | ✅ |
| Sorted array iteration | ❌ | ✅ |
| TCP/UDP networking | ❌ | ✅ |
| `strftime()` / `systime()` | ❌ | ✅ |
| `gensub()` | ❌ | ✅ |
| `patsplit()` | ❌ | ✅ |
| `@include` | ❌ | ✅ |
| Regex intervals `{n,m}` | Varies | ✅ |

### Useful GAWK Extensions

```bash
# gensub — replace with back-references (gawk only)
gawk '{ print gensub(/([0-9]+)/, "[\\1]", "g") }' file

# strftime — format timestamps (gawk only)
gawk 'BEGIN { print strftime("%Y-%m-%d", systime()) }'

# Read a whole file into a variable (gawk only)
gawk 'BEGIN { RS="^$"; getline content < "file.txt"; print length(content) }'
```

---

## Quick Reference Card

```
awk 'BEGIN{} /pattern/{action} END{}' file

$0          entire line          NR    record number
$1..$NF     fields               NF    number of fields
FS/OFS      field sep in/out     RS    record sep in
FILENAME    current file         FNR   file record number

print       output with ORS      printf  formatted output
length()    string length        substr()  substring
gsub()      global replace       sub()   first replace
split()     split string         match() regex match
tolower()   lowercase            toupper() uppercase
int()       truncate             rand()  random float
sprintf()   format string        system() run command

if/else  while  do-while  for  break  continue  next  exit
```

---

> **Further Reading:**
> - `man awk` or `man gawk`
> - [GNU AWK User's Guide](https://www.gnu.org/software/gawk/manual/)
> - `gawk --version` to check your version
