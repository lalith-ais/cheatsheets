# Regular Expressions Cheatsheet

A complete reference for regular expression syntax, patterns, and techniques.

---

## Anchors

| Pattern | Description | Example |
|---------|-------------|---------|
| `^` | Start of string (or line with `m` flag) | `^Hello` matches "Hello world" |
| `$` | End of string (or line with `m` flag) | `world$` matches "Hello world" |
| `\b` | Word boundary | `\bcat\b` matches "cat" but not "concatenate" |
| `\B` | Not a word boundary | `\Bcat\B` matches "concatenate" but not "cat" |
| `\A` | Start of string (no multiline) | `\AHello` |
| `\Z` | End of string (no multiline) | `world\Z` |

---

## Character Classes

| Pattern | Description | Example |
|---------|-------------|---------|
| `.` | Any character except newline | `a.c` matches "abc", "aXc" |
| `\d` | Digit `[0-9]` | `\d+` matches "123" |
| `\D` | Non-digit `[^0-9]` | `\D+` matches "abc" |
| `\w` | Word character `[a-zA-Z0-9_]` | `\w+` matches "hello_123" |
| `\W` | Non-word character | `\W+` matches "!@#" |
| `\s` | Whitespace (space, tab, newline, etc.) | `\s+` matches "   " |
| `\S` | Non-whitespace | `\S+` matches "hello" |
| `\n` | Newline | — |
| `\t` | Tab | — |
| `\r` | Carriage return | — |
| `\0` | Null character | — |

---

## Custom Character Classes `[ ]`

| Pattern | Description | Example |
|---------|-------------|---------|
| `[abc]` | Any one of a, b, or c | `[aeiou]` matches any vowel |
| `[^abc]` | Any character except a, b, or c | `[^0-9]` matches non-digits |
| `[a-z]` | Any character in range a–z | `[a-z]+` matches lowercase words |
| `[A-Z]` | Any character in range A–Z | — |
| `[0-9]` | Any digit | — |
| `[a-zA-Z]` | Any letter (upper or lower) | — |
| `[a-zA-Z0-9_]` | Equivalent to `\w` | — |
| `[\u0041-\u005A]` | Unicode range (A–Z) | — |

> **Tip:** Inside `[ ]`, most special characters lose their meaning. `.` inside `[.]` is a literal dot.

---

## Quantifiers

| Pattern | Description | Example |
|---------|-------------|---------|
| `*` | 0 or more (greedy) | `ab*` matches "a", "ab", "abbb" |
| `+` | 1 or more (greedy) | `ab+` matches "ab", "abbb" |
| `?` | 0 or 1 (optional) | `colou?r` matches "color" and "colour" |
| `{n}` | Exactly n times | `\d{4}` matches "2024" |
| `{n,}` | At least n times | `\d{2,}` matches "12", "123" |
| `{n,m}` | Between n and m times | `\d{2,4}` matches "12", "123", "1234" |

### Greedy vs Lazy

| Pattern | Description | Example |
|---------|-------------|---------|
| `*` | Greedy — matches as much as possible | `<.+>` on `<a><b>` matches `<a><b>` |
| `*?` | Lazy — matches as little as possible | `<.+?>` on `<a><b>` matches `<a>` |
| `+?` | Lazy one or more | — |
| `??` | Lazy zero or one | — |
| `{n,m}?` | Lazy range | — |

---

## Groups & References

| Pattern | Description | Example |
|---------|-------------|---------|
| `(abc)` | Capturing group | `(foo)bar` captures "foo" |
| `(?:abc)` | Non-capturing group | `(?:foo)bar` groups without capturing |
| `(?<name>abc)` | Named capturing group | `(?<year>\d{4})` captures as "year" |
| `\1`, `\2` | Backreference to captured group | `(\w+) \1` matches "hello hello" |
| `\k<name>` | Named backreference | `\k<year>` references named group "year" |
| `(?=abc)` | Positive lookahead | `foo(?=bar)` matches "foo" in "foobar" |
| `(?!abc)` | Negative lookahead | `foo(?!bar)` matches "foo" not followed by "bar" |
| `(?<=abc)` | Positive lookbehind | `(?<=foo)bar` matches "bar" preceded by "foo" |
| `(?<!abc)` | Negative lookbehind | `(?<!foo)bar` matches "bar" not preceded by "foo" |
| `(?>abc)` | Atomic group (no backtracking) | Prevents catastrophic backtracking |

---

## Alternation

| Pattern | Description | Example |
|---------|-------------|---------|
| `a\|b` | Match a or b | `cat\|dog` matches "cat" or "dog" |
| `(a\|b)c` | Group alternation | `(cat\|dog)s` matches "cats" or "dogs" |

---

## Flags / Modifiers

| Flag | Description |
|------|-------------|
| `g` | **Global** — find all matches, not just the first |
| `i` | **Case insensitive** — `A` matches `a` |
| `m` | **Multiline** — `^` and `$` match start/end of each line |
| `s` | **Dotall** — `.` matches newline characters too |
| `u` | **Unicode** — enables full Unicode support |
| `y` | **Sticky** — match only from `lastIndex` position |
| `x` | **Verbose** — allow whitespace and comments in pattern (some engines) |

**Usage examples (JavaScript):**
```js
/pattern/gi          // global + case insensitive
/pattern/gm          // global + multiline
/pattern/gimsuy      // all flags
```

---

## Common Patterns

### Validation

```regex
# Email address
^[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}$

# URL
https?://[\w/:%#\$&\?\(\)~\.=\+\-]+

# IPv4 address
^((25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(25[0-5]|2[0-4]\d|[01]?\d\d?)$

# UK postcode
^[A-Z]{1,2}\d[A-Z\d]?\s?\d[A-Z]{2}$

# Phone number (international)
^\+?[\d\s\-\(\)]{7,15}$

# Date (YYYY-MM-DD)
^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$

# Strong password (min 8 chars, upper, lower, digit, special)
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[\W_]).{8,}$

# Credit card (Visa/Mastercard/Amex)
^(?:4\d{12}(?:\d{3})?|5[1-5]\d{14}|3[47]\d{13})$

# Hex colour
^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$

# Slug (URL-safe string)
^[a-z0-9]+(?:-[a-z0-9]+)*$
```

### Extraction

```regex
# HTML tag contents
<([a-zA-Z]+)[^>]*>(.*?)<\/\1>

# Text inside quotes
["'](.*?)["']

# Numbers (incl. decimals and negatives)
-?\d+(\.\d+)?

# Hashtags
#\w+

# @mentions
@\w+
```

### Transformation

```regex
# Match repeated words
\b(\w+)\s+\1\b

# Match lines with specific content
^.*keyword.*$

# Match blank lines
^\s*$

# Trim whitespace (match leading/trailing)
^\s+|\s+$
```

---

## Lookahead & Lookbehind Examples

```regex
# Price: digits followed by £ (positive lookahead)
\d+(?= £)

# Digits NOT followed by px (negative lookahead)
\d+(?!px)

# Text after "Name: " (positive lookbehind)
(?<=Name: )\w+

# Numbers not preceded by # (negative lookbehind)
(?<!#)\d+
```

---

## POSIX Classes (used in some engines)

| Class | Equivalent | Description |
|-------|-----------|-------------|
| `[:alpha:]` | `[a-zA-Z]` | Letters |
| `[:digit:]` | `[0-9]` | Digits |
| `[:alnum:]` | `[a-zA-Z0-9]` | Letters and digits |
| `[:space:]` | `[\t\n\r\f\v ]` | Whitespace |
| `[:upper:]` | `[A-Z]` | Uppercase letters |
| `[:lower:]` | `[a-z]` | Lowercase letters |
| `[:punct:]` | Punctuation | Punctuation characters |
| `[:xdigit:]` | `[0-9a-fA-F]` | Hexadecimal digits |

> Use inside `[ ]` like: `[[:alpha:]]` — available in POSIX, Python `re`, and some others.

---

## Language Quick Reference

### JavaScript
```js
const re = /pattern/flags;
re.test(str)            // true/false
str.match(re)           // array of matches
str.matchAll(re)        // iterator of all matches
str.replace(re, val)    // replace first (or all with /g)
str.split(re)           // split on pattern
re.exec(str)            // detailed match info
```

### Python
```python
import re
re.search(r'pattern', str)    # first match anywhere
re.match(r'pattern', str)     # match at start
re.fullmatch(r'pattern', str) # match entire string
re.findall(r'pattern', str)   # list of all matches
re.finditer(r'pattern', str)  # iterator of match objects
re.sub(r'pattern', repl, str) # replace matches
re.split(r'pattern', str)     # split on pattern
re.compile(r'pattern', flags) # compile for reuse
```

### PHP
```php
preg_match('/pattern/', $str, $matches);
preg_match_all('/pattern/', $str, $matches);
preg_replace('/pattern/', $replacement, $str);
preg_split('/pattern/', $str);
```

### Java
```java
Pattern p = Pattern.compile("pattern", Pattern.CASE_INSENSITIVE);
Matcher m = p.matcher(str);
m.matches()    // entire string matches
m.find()       // find next match
m.group()      // get matched text
str.replaceAll("pattern", "replacement");
```

---

## Special Characters to Escape

These characters have special meaning and must be escaped with `\` when used literally:

```
. * + ? ^ $ { } [ ] ( ) | \ / -
```

**Examples:**
```regex
\.        # literal dot
\$        # literal dollar sign
\(text\)  # literal parentheses
\d\.\d    # matches "3.7" (not "3X7")
```

---

## Tips & Best Practices

1. **Use raw strings** — in Python, always use `r'pattern'` to avoid double-escaping backslashes.
2. **Compile patterns** — if reusing a pattern many times, compile it once for better performance.
3. **Anchor your patterns** — use `^` and `$` for validation to ensure the whole string matches.
4. **Prefer lazy quantifiers** — `+?` instead of `+` when you want the shortest match.
5. **Use non-capturing groups** — `(?:...)` instead of `(...)` when you don't need the capture.
6. **Avoid catastrophic backtracking** — be careful with nested quantifiers like `(a+)+`.
7. **Test your patterns** — use tools like [regex101.com](https://regex101.com) or [regexr.com](https://regexr.com) to build and test interactively.
8. **Comment complex patterns** — use verbose mode (`x` flag) or inline comments where supported.
9. **Keep it readable** — a clear, readable pattern in multiple steps beats a single unreadable mega-expression.

---

*Syntax availability varies by engine. Always check your language's documentation for engine-specific behaviour.*
