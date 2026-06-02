# grep Command Reference Book

A practical reference for searching text with **grep** (GNU/BSD) and **ripgrep** (`rg`).  
Use this as a quick lookup while working on Linux servers, Git repos, and log files.

---

## Table of Contents

1. [What grep does](#1-what-grep-does)
2. [Basic syntax](#2-basic-syntax)
3. [Essential options](#3-essential-options)
4. [Regular expressions](#4-regular-expressions)
5. [Searching files and directories](#5-searching-files-and-directories)
6. [Context and output control](#6-context-and-output-control)
7. [Real-world recipes](#7-real-world-recipes)
8. [grep + pipes and other tools](#8-grep--pipes-and-other-tools)
9. [ripgrep (rg) cheat sheet](#9-ripgrep-rg-cheat-sheet)
10. [grep vs other tools](#10-grep-vs-other-tools)
11. [Common mistakes](#11-common-mistakes)
12. [Quick option index](#12-quick-option-index)

---

## 1. What grep does

**grep** = **G**lobally search a **R**egular **E**xpression and **P**rint matching lines.

| Use case | Example idea |
|----------|----------------|
| Find errors in logs | `grep ERROR app.log` |
| Find function definitions | `grep -n "def foo" *.py` |
| Search a codebase | `grep -r "TODO" src/` |
| Filter command output | `ps aux \| grep nginx` |
| Invert match (exclude lines) | `grep -v DEBUG app.log` |

---

## 2. Basic syntax

```bash
grep [OPTIONS] PATTERN [FILE...]
grep [OPTIONS] -e PATTERN [FILE...]
grep [OPTIONS] -f PATTERN_FILE [FILE...]
```

| Form | Meaning |
|------|---------|
| `grep foo file.txt` | Lines containing `foo` in one file |
| `grep foo a.txt b.txt` | Search multiple files |
| `grep foo *.log` | Shell expands glob; grep searches each file |
| `grep -r foo .` | Recursive search from current directory |
| `cat file \| grep foo` | Search stdin (pipe) |

**Exit codes (useful in scripts):**

| Code | Meaning |
|------|---------|
| `0` | At least one match |
| `1` | No match |
| `2` | Error (e.g. file not found) |

```bash
if grep -q "error" app.log; then
  echo "Errors found"
fi
```

---

## 3. Essential options

### Matching behavior

| Option | Long form | Description |
|--------|-----------|-------------|
| `-i` | `--ignore-case` | Case-insensitive search |
| `-v` | `--invert-match` | Show lines that do **not** match |
| `-w` | `--word-regexp` | Match whole words only |
| `-x` | `--line-regexp` | Match whole lines only |
| `-F` | `--fixed-strings` | Treat pattern as literal (no regex) |
| `-E` | `--extended-regexp` | Extended regex (ERE); `+`, `?`, `\|`, `()` |
| `-G` | `--basic-regexp` | Basic regex (default on many systems) |
| `-P` | `--perl-regexp` | Perl regex (GNU grep only; not on macOS BSD grep) |
| `-e` | `--regexp` | Pattern (use multiple times for OR) |
| `-f` | `--file` | Read patterns from a file |

```bash
grep -i "error" app.log              # ERROR, Error, error
grep -v "DEBUG" app.log              # Hide debug lines
grep -w "log" app.log                # Matches "log" not "logging"
grep -F "*.txt" filelist             # Literal *.txt, not regex
grep -E "error|warn|fatal" app.log   # OR with extended regex
grep -e "foo" -e "bar" file.txt      # Lines with foo OR bar
```

### File selection

| Option | Description |
|--------|-------------|
| `-r` / `-R` | Recursive (follows symlinks with `-R` on GNU) |
| `--include=GLOB` | Only search files matching glob |
| `--exclude=GLOB` | Skip files matching glob |
| `--exclude-dir=DIR` | Skip directories (e.g. `.git`, `node_modules`) |
| `-d skip` | Skip directories (default for without `-r`) |
| `-d read` | Read directories as files (rare) |

```bash
grep -r "password" --include="*.env" .
grep -r "TODO" --exclude-dir={.git,node_modules,dist} src/
grep -r "import" --include="*.py" --include="*.js" project/
```

### Output format

| Option | Description |
|--------|-------------|
| `-n` | Line numbers |
| `-H` | Print filename (default with multiple files) |
| `-h` | Suppress filenames |
| `-l` | Only filenames with matches |
| `-L` | Only filenames with **no** matches |
| `-c` | Count matching lines per file |
| `-o` | Print only the matched part |
| `--color=auto` | Highlight matches (often default) |
| `-q` | Quiet: no output; exit status only |
| `-s` | Suppress errors for missing files |

```bash
grep -rn "function" src/             # file:line:match
grep -rl "API_KEY" .                 # List files containing API_KEY
grep -c "ERROR" *.log                # Count per file
grep -oE "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" access.log
```

### Context lines

| Option | Description |
|--------|-------------|
| `-A NUM` | After: show NUM lines after match |
| `-B NUM` | Before: show NUM lines before match |
| `-C NUM` | Context: NUM lines before and after |

```bash
grep -A 5 "Exception" app.log        # Stack trace after exception
grep -B 2 -A 2 "error" app.log       # Surrounding context
grep -C 3 "failed" deploy.log
```

### Line endings and binary

| Option | Description |
|--------|-------------|
| `-a` / `--text` | Treat binary files as text |
| `-I` | Skip binary files without message |
| `-Z` | Null-terminated output (for `xargs -0`) |
| `-u` | Unix line endings (legacy) |

---

## 4. Regular expressions

### Basic regex (default `-G`)

| Pattern | Meaning |
|---------|---------|
| `.` | Any single character |
| `*` | Zero or more of preceding |
| `^` | Start of line |
| `$` | End of line |
| `[]` | Character class |
| `[^]` | Negated class |
| `\|` | Alternation (may need `-E`) |

### Extended regex (`-E`)

| Pattern | Meaning |
|---------|---------|
| `+` | One or more |
| `?` | Zero or one |
| `{n,m}` | Repeat n to m times |
| `(a\|b)` | Grouping and alternation |
| `(?:...)` | Non-capturing (GNU `-P` / some `-E`) |

### Common patterns

```bash
# Email-ish
grep -E "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}" contacts.txt

# IPv4
grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log

# HTTP status 4xx or 5xx
grep -E "HTTP/1\.[01]\" [45][0-9]{2}" access.log

# Lines starting with #
grep "^#" config.ini

# Empty lines
grep "^$" file.txt

# Lines with only whitespace
grep -E "^[[:space:]]*$" file.txt

# Word boundary (GNU)
grep -E "\berror\b" app.log
```

### Character classes (POSIX)

| Class | Meaning |
|-------|---------|
| `[[:digit:]]` | Digits |
| `[[:alpha:]]` | Letters |
| `[[:alnum:]]` | Letters and digits |
| `[[:space:]]` | Whitespace |
| `[[:upper:]]` / `[[:lower:]]` | Case |

```bash
grep -E "^[[:digit:]]+:" /etc/passwd
```

### Escaping special characters

In basic regex, escape: `. * [ ] ^ $ \`

```bash
grep -F "price is $9.99" sales.txt     # Literal dollar and dot
grep "func\(\)" code.c                 # Literal parentheses
grep -E "version [0-9]+\.[0-9]+" package.json
```

---

## 5. Searching files and directories

```bash
# Current directory, all text files
grep -r "pattern" .

# Specific path
grep -r "import os" /home/user/project/src

# Only certain extensions
grep -r --include="*.{java,xml}" "javax" .

# Skip heavy dirs
grep -r "TODO" . \
  --exclude-dir=.git \
  --exclude-dir=node_modules \
  --exclude-dir=target \
  --exclude-dir=build

# Search compressed logs (zgrep / zcat)
zgrep "ERROR" /var/log/app.log.1.gz
zcat app.log.gz | grep "timeout"
```

### Search hidden files

```bash
grep -r "config" . --include=".*"      # dotfiles if named in include
grep -r "alias" ~/.bashrc ~/.profile
```

### Multiple patterns from file

```bash
# patterns.txt:
# error
# warn
# fatal

grep -f patterns.txt app.log
grep -Ef patterns.txt app.log         # If patterns use ERE syntax
```

---

## 6. Context and output control

### Grouping and separators (GNU)

```bash
grep -r --group=separate "error" logs/    # Blank line between file groups
grep -r --no-grouping "error" logs/       # Continuous output
```

### Only matching part with color off

```bash
grep -o "user=[^ ]*" auth.log
```

### Count total matches (all files)

```bash
grep -r "ERROR" logs/ | wc -l
grep -rc "ERROR" logs/ | awk -F: '{s+=$2} END {print s}'
```

### Sort and dedupe results

```bash
grep -rh "import" src/ | sort -u
```

---

## 7. Real-world recipes

### Logs and operations

```bash
# Last 100 lines, then filter
tail -n 100 /var/log/syslog | grep -i error

# Follow log live and filter
tail -f app.log | grep --line-buffered "ERROR"

# Count errors per hour (if log has ISO timestamps)
grep "2026-06-01" app.log | grep -c ERROR

# Exclude noisy lines
grep ERROR app.log | grep -v "healthcheck" | grep -v "kube-probe"

# Find failed SSH logins
grep "Failed password" /var/log/auth.log

# Find OOM kills
grep -i "out of memory" /var/log/kern.log
```

### Code and Git repos

```bash
# Find all TODO/FIXME
grep -rn "TODO\|FIXME" --include="*.{py,js,ts,java,go}" .

# Find hardcoded secrets (careful: review results)
grep -rn "password\s*=\|api_key\|secret" --include="*.py" .

# Find function definition in Python
grep -rn "^def my_function" .

# Find where a symbol is used
grep -rn "MyClass" src/

# Search only tracked files (git)
git grep "pattern"
git grep -n "pattern" -- "*.go"
```

### Config and system admin

```bash
grep -r "PermitRootLogin" /etc/ssh/
grep "^[^#]" /etc/nginx/nginx.conf      # Active (non-comment) lines
grep -E "^[[:space:]]*[^#[:space:]]" file.conf

# Check if a package is installed (Debian)
dpkg -l | grep nginx

# Running process
ps aux | grep "[n]ginx"                 # [n] trick avoids grep in ps output
pgrep -a nginx
```

### Data files

```bash
# CSV: lines where column contains value (simple)
grep ",active," users.csv

# JSON lines (one JSON object per line)
grep '"status":"failed"' events.jsonl

# Extract URLs
grep -oE 'https?://[^ ]+' page.html
```

---

## 8. grep + pipes and other tools

```bash
# Chain filters
cat access.log | grep "POST" | grep -v "/health" | grep " 500 "

# With awk
grep "ERROR" app.log | awk '{print $1, $2, $NF}'

# With sed (replace after grep)
grep -l "oldname" *.py | xargs sed -i 's/oldname/newname/g'

# With find + grep
find . -name "*.log" -exec grep -l "ERROR" {} \;
find . -name "*.java" -print0 | xargs -0 grep -l "SQLException"

# With sort/uniq
grep "user_id" access.log | sort | uniq -c | sort -rn | head

# Parallel (GNU)
grep -r "pattern" large_project/ | parallel -j4 process_line {}
```

### `git grep` (preferred inside repos)

```bash
git grep "main("
git grep -n "TODO" HEAD
git grep "fix" $(git rev-list --all)
git grep -i "password" -- '*.env*'
```

---

## 9. ripgrep (rg) cheat sheet

**ripgrep** (`rg`) is faster, respects `.gitignore` by default, and is used by tools like Cursor. Install: `apt install ripgrep` / `brew install ripgrep` / Windows: `winget install BurntSushi.ripgrep.MSVC`.

| Task | grep | ripgrep (rg) |
|------|------|----------------|
| Recursive search | `grep -r pat .` | `rg pat` |
| Case insensitive | `grep -ri pat .` | `rg -i pat` |
| Line numbers | `grep -rn pat .` | `rg -n pat` |
| Literal string | `grep -rF pat .` | `rg -F pat` |
| File type filter | `--include=*.py` | `rg -t py pat` |
| Only filenames | `grep -rl pat .` | `rg -l pat` |
| Count | `grep -rc pat .` | `rg -c pat` |
| Context | `grep -C 3 pat f` | `rg -C 3 pat f` |
| Invert | `grep -rv pat .` | `rg -v pat` |
| PCRE2 regex | `grep -P` (GNU) | `rg -P` |

### Useful `rg` options

```bash
rg "error"                              # Search cwd, skip gitignored
rg -i "todo" src/
rg -n "fn main"                         # Line numbers
rg -l "password"                        # Files with matches only
rg -c "import"                          # Count per file
rg -w "null"                            # Whole word
rg -F "C:\Users"                        # Fixed string (Windows paths)
rg -t py "def authenticate"             # Python files only
rg -t js -t ts "useState"
rg -g "*.yaml" -g "!*test*" "image:"    # Glob include/exclude
rg -uu "secret"                         # Search ignored files too
rg --hidden "dotfile"                   # Include hidden files
rg -C 2 "panic"                         # Context
rg --json "pattern"                     # JSON output (tooling)
rg --replace '$1' -o 'user=(\w+)' log   # Show capture group 1 only
```

### Type list

```bash
rg --type-list                         # Show all built-in types
rg -t rust "unwrap"
rg -t java "SQLException"
```

---

## 10. grep vs other tools

| Tool | When to use |
|------|-------------|
| **grep** | Universal, on every Unix system, scripts, logs |
| **rg** | Large codebases, respects `.gitignore`, speed |
| **git grep** | Only tracked files in a Git repo |
| **ag / ack** | Legacy code search (less common now) |
| **sed -n '/pat/p'** | Print lines when already in sed script |
| **awk '/pat/'** | When you need column processing too |
| **find + grep** | Complex file metadata filters |

### PowerShell (Windows) equivalent

```powershell
Select-String -Pattern "error" -Path *.log
Select-String -Pattern "todo" -Path .\src -Recurse
Get-Content app.log | Select-String "ERROR"
```

### BSD grep (macOS) notes

- No `-P` (Perl regex) by default — use `-E` or install GNU grep as `ggrep`
- Some options differ; check `man grep` on the host

```bash
# macOS: install GNU grep
brew install grep
ggrep -P "\d+" file.txt
```

---

## 11. Common mistakes

| Mistake | Fix |
|---------|-----|
| Pattern starts with `-` | `grep -e "-option" file` or `grep -- "-option" file` |
| Unquoted `*` in shell | Quote pattern: `grep "a*b" file` |
| grep shows itself in `ps` | Use `pgrep` or `grep "[p]attern"` |
| Slow search in `node_modules` | `--exclude-dir=node_modules` or use `rg` |
| Binary garbage on screen | `grep -I` or `grep -a` intentionally |
| Regex special chars in URLs | Use `-F` or escape: `grep -F "http://example.com"` |
| Expecting grep to search filenames | Use `find` + `grep` or `rg --files \| rg pattern` |

---

## 12. Quick option index

| Option | Short description |
|--------|-------------------|
| `-a` | Binary as text |
| `-A n` | Lines after match |
| `-B n` | Lines before match |
| `-C n` | Context before and after |
| `-c` | Count matches |
| `-E` | Extended regex |
| `-e` | Pattern argument |
| `-f` | Patterns from file |
| `-F` | Fixed string (literal) |
| `-h` | No filename prefix |
| `-H` | Always show filename |
| `-i` | Ignore case |
| `-I` | Skip binary |
| `-l` | Files with matches |
| `-L` | Files without matches |
| `-n` | Line numbers |
| `-o` | Only matched text |
| `-P` | Perl regex (GNU) |
| `-q` | Quiet |
| `-r` | Recursive |
| `-s` | Silent errors |
| `-v` | Invert match |
| `-w` | Word match |
| `-x` | Whole line match |
| `-Z` | Null-terminated |

---

## One-page command summary

```bash
# Basics
grep "pattern" file
grep -i "pattern" file
grep -v "pattern" file
grep -n "pattern" file
grep -c "pattern" file

# Recursive
grep -rn "pattern" path/
grep -rl "pattern" path/

# Regex
grep -E "pat1|pat2" file
grep -F "literal.string" file

# Context
grep -C 3 "pattern" file

# Logs
tail -f app.log | grep --line-buffered ERROR

# Repo (prefer)
git grep -n "pattern"

# Fast modern search
rg -n "pattern"
rg -t py "pattern" src/
```

---

## Further reading

- `man grep` — full option list on your system
- `man rg` or https://github.com/BurntSushi/ripgrep — ripgrep documentation
- `info grep` — GNU grep manual (if installed)

---

*Last updated: 2026-06-01 · Personal notebook reference*
