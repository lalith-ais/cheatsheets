# Sed (Stream Editor) Cheatsheet

## 📜 Core Concept
`sed` processes text line-by-line from a file or stdin.
**Syntax:** `sed [options] 'command' file`

**Important:** By default, `sed` prints every line to stdout (standard output). It **does not** change the original file unless you use the `-i` flag.

---

## 🚀 Basic Usage & Options

| Option | Description | Example |
| :--- | :--- | :--- |
| `-n` | Suppress automatic printing (quiet mode). Use with `p` flag. | `sed -n '/error/p' log.txt` |
| `-i` | **Edit file in-place** (Dangerous but useful!) | `sed -i 's/foo/bar/g' file.txt` |
| `-i.bak` | Edit in-place but create backup with `.bak` extension first. | `sed -i.bak 's/foo/bar/g' file.txt` |
| `-e` | Execute multiple commands. | `sed -e 's/a/A/' -e 's/b/B/'` |
| `-E` / `-r` | Extended Regular Expressions (no need to escape `+`, `?`, `()`, `{}`). | `sed -E 's/[0-9]+/NUM/g'` |

---

## 🔁 The Substitute Command (`s`)

This is the most used command in `sed`. Format: `s/pattern/replacement/flags`

### Basic Substitution
| Command | Description |
| :--- | :--- |
| `sed 's/old/new/'` | Replace **first** occurrence of "old" with "new" on each line. |
| `sed 's/old/new/g'` | Replace **all** occurrences (global). |
| `sed 's/old/new/2'` | Replace **second** occurrence only. |
| `sed 's/old/new/i'` | Case-**i**nsensitive matching. |
| `sed 's/old/new/w changes.txt'` | **W**rite changed lines to a file. |

### Delimiters
You can use any character instead of `/` if your pattern contains slashes (e.g., file paths).
| Command | Description |
| :--- | :--- |
| `sed 's#/usr/bin#/opt/bin#'` | Uses `#` as delimiter. |
| `sed 's|http://|https://|'` | Uses `\|` as delimiter. |

---

## 📍 Line Addressing (Where to Act)

You can limit commands to specific line numbers or patterns.

| Address | Example | Description |
| :--- | :--- | :--- |
| **Number** | `sed '5 s/foo/bar/'` | Only on line 5. |
| **Range** | `sed '10,20 s/foo/bar/'` | Lines 10 through 20. |
| **Pattern** | `sed '/start/,/stop/ s/foo/bar/'` | From line matching "start" to line matching "stop". |
| **Negation** | `sed '/success/! s/foo/bar/'` | Lines **NOT** containing "success". |
| **Last Line** | `sed '$ s/foo/bar/'` | `$` means last line. |

---

## ❌ Deletion (`d`)

| Command | Description |
| :--- | :--- |
| `sed '5d' file` | Delete line 5. |
| `sed '/^$/d' file` | Delete all **blank** lines. |
| `sed '/^#/d' file` | Delete lines starting with `#` (comments). |
| `sed '10,$d' file` | Delete from line 10 to end of file. |

---

## 🖨️ Printing (`p`)

*Remember: Use `-n` to avoid duplicate output.*

| Command | Description |
| :--- | :--- |
| `sed -n '5p' file` | Print **only** line 5. |
| `sed -n '/ERROR/p' log.txt` | Print lines containing ERROR (like `grep`). |
| `sed -n '10,20p' file` | Print lines 10 through 20. |

---

## ✍️ Insert, Append, Change (Multi-line editing)

| Command | Syntax | Description |
| :--- | :--- | :--- |
| **Insert** | `sed '2 i\New line'` | Insert text **before** line 2. |
| **Append** | `sed '2 a\New line'` | Append text **after** line 2. |
| **Change** | `sed '2 c\New line'` | Replace entire line 2 with text. |

---

## 🧠 Advanced Patterns (Regex Grouping)

### Capture Groups
Use `\(` and `\)` to capture text, then `\1`, `\2` to paste it back.
*(Note: Use `-E` to avoid backslashes for parens!)*

| Command (Basic Regex) | Command (Extended `-E`) | Result |
| :--- | :--- | :--- |
| `sed 's/\(.*\):\(.*\)/\2:\1/'` | `sed -E 's/(.*):(.*)/\2:\1/'` | Swaps text around colon (`foo:bar` -> `bar:foo`) |
| `sed 's/\([0-9]\{3\}\)/Area:\1/'` | `sed -E 's/([0-9]{3})/Area:\1/'` | Finds 3 digits and prefixes with "Area:" |

### Hold Space (Advanced Buffering)
*For complex multi-line parsing (like joining lines or wrapping XML).*

| Command | Mnemonic | Action |
| :--- | :--- | :--- |
| `h` | **H**old | Copy pattern space to hold space (overwrite) |
| `H` | **H**old | Append pattern space to hold space |
| `g` | **G**et | Copy hold space to pattern space (overwrite) |
| `G` | **G**et | Append hold space to pattern space |
| `x` | e**X**change | Swap hold space and pattern space |

*Example: Double-space a file:* `sed G file.txt`

---

## 🧹 Common "One-Liner" Recipes

| Task | Command |
| :--- | :--- |
| **Remove trailing spaces** | `sed -i 's/[[:space:]]*$//' file.txt` |
| **Convert DOS to Unix (CRLF->LF)** | `sed -i 's/\r$//' file.txt` |
| **Remove HTML tags** | `sed -E 's/<[^>]+>//g' file.html` |
| **Print lines 50-60** | `sed -n '50,60p' file.txt` |
| **Delete everything after # comment** | `sed 's/#.*$//'` |
| **Extract IP address (rough)** | `sed -n 's/.*\([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\).*/\1/p'` |

---

## ⚠️ Platform Differences (Mac vs Linux)

| Feature | Linux (GNU sed) | macOS (BSD sed) |
| :--- | :--- | :--- |
| **In-place edit** | `sed -i 's/foo/bar/' file` | `sed -i '' 's/foo/bar/' file` *(requires backup extension)* |
| **Extended Regex** | `-r` | `-E` |
| **Special Chars** | Handles `\n` in replacement well | Requires `$'...'` or literal newline for `\n` |

*Tip for Mac users: Install GNU sed via `brew install gnu-sed` for compatibility.*