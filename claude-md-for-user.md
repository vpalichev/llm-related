## Environment — MANDATORY RULES

This is Windows 10/11 with Git Bash (MSYS2) and PowerShell available.
The VS Code terminal is set to Git Bash (to my knowledge, don't fail to recheck that).

## Clojure MCP Light notice
- We frequently use a tool named Clojure MCP Light, it's usage is usually described in project's CLAUDE.md file, in case of Clojure(script) projects.

### Shell context
- The active shell is Git Bash. Use bash syntax directly.
- This is NOT native Linux. It is MSYS2 on Windows.
- Do not use `wsl`, `wsl.exe`, or reference `/mnt/c/` paths.
- Do not suggest installing WSL as a solution to any problem.

### Path handling — critical
- Inside Git Bash: use `/c/Users/...` style paths.
- Backslashes are escape characters in bash. Never use `C:\path` in bash commands.
- When calling native Windows executables (docker, dotnet, az, terraform, etc.),
  prefix with `MSYS_NO_PATHCONV=1` if any argument starts with `/`.
- Do not reference `/tmp/`. Use `$TEMP` or `$TMP`.
- Git Bash home is `/c/Users/<username>`, not `/home/<username>`.
- Quote all paths containing spaces.

### Pre-flight check — EVERY command
- Bash paths (cd, cat, cp, rg, source, etc.): always forward slashes, /c/Users/... style
- PowerShell invoked via `powershell -Command "..."`: Windows backslash paths INSIDE the quotes are correct
- Native .exe arguments: use backslashes OR prefix with MSYS_NO_PATHCONV=1 if forward-slash args get mangled
- Clojure MCP tool calls: paths need either escaped backslashes (C:\\Users\\...) or forward slashes (C:/Users/...). Single backslashes WILL be eaten as escape characters.
- If unsure, use `cygpath -w` or `cygpath -u` to convert explicitly.

### Clojure MCP path gotcha
❌ C:\Users\me\project\src\core.clj     → backslashes eaten, path mangled
✅ C:\\Users\\me\\project\\src\\core.clj → escaped, works
✅ C:/Users/me/project/src/core.clj      → forward slashes, works

### Search operations — use ripgrep (`rg`), not `grep`
`rg` is a native Windows binary. No MSYS2 path translation issues, no codepage
problems. It searches recursively by default, respects `.gitignore`, skips
hidden files and binary files automatically. Regex is the default pattern mode.

**Basic usage:**
```
rg 'pattern'                        # recursive search from current dir
rg 'pattern' path/to/dir            # search specific directory
rg 'pattern' file.txt               # search specific file
rg -i 'pattern'                     # case-insensitive
rg -w 'pattern'                     # whole-word match
rg -F 'literal.string'              # fixed string, no regex
rg -S 'pattern'                     # smart case: insensitive unless pattern has uppercase
```

**Output control:**
```
rg -n 'pattern'                     # show line numbers (default in TTY)
rg -N 'pattern'                     # suppress line numbers
rg -l 'pattern'                     # list only filenames with matches
rg -c 'pattern'                     # count matching lines per file
rg --count-matches 'pattern'        # count all matches, not just lines
rg -o 'pattern'                     # print only matching parts
rg -q 'pattern'                     # quiet, exit code only
```

**Context lines (like grep -A/-B/-C):**
```
rg -A 3 'pattern'                   # 3 lines after match
rg -B 3 'pattern'                   # 3 lines before match
rg -C 3 'pattern'                   # 3 lines before and after
```

**File type filtering:**
```
rg -t py 'pattern'                  # search only Python files
rg -T js 'pattern'                  # exclude JavaScript files
rg --type-list                      # show all known file types
rg -g '*.json' 'pattern'            # glob filter
rg -g '!*.min.js' 'pattern'         # glob exclusion (prefix with !)
rg -g '!dist/' 'pattern'            # exclude directory by glob
```

**Filter bypass (`-u` stacks up to 3 times):**
```
rg -. 'pattern'                     # include hidden files (--hidden)
rg -u 'pattern'                     # skip .gitignore rules
rg -uu 'pattern'                    # also search hidden files
rg -uuu 'pattern'                   # also search binary files (≈ grep -r)
```

**Multi-pattern, invert, and replace:**
```
rg -e 'pat1' -e 'pat2'              # OR multiple patterns
rg -v 'pattern'                     # invert match
rg 'pattern' -r 'replacement'       # replace in output only — does NOT modify files
rg 'foo(\d+)' -r 'bar$1'            # capture groups in replacement
```

**Advanced:**
```
rg -P 'look(?=ahead)'               # PCRE2 for lookahead/behind (if built with PCRE2)
rg -m 5 'pattern'                   # stop after 5 matches per file
rg --max-depth 2 'pattern'          # limit directory recursion depth
rg --files                          # list files rg would search (no pattern needed)
rg --files | rg -i 'name'           # find files by name
rg --no-ignore 'pattern'            # ignore ALL ignore files
rg -x 'entire line'                 # match must span entire line
```

**Flags that DO NOT EXIST — do not use these:**
```
# There is no -R or -r for recursion (already recursive by default)
# There is no --include (use -g '*.ext' or -t type instead)
# There is no --exclude-dir (use -g '!dirname/' instead)
# There is no -E for encoding (the flag is --encoding)
# There is no --color=auto flag (color is auto by default)
# Do not transfer grep flags to rg — they are different tools
```

Also use `fd` instead of `find` if installed; otherwise GNU `find` is acceptable.

### Commands that DO NOT EXIST in this environment
- `apt`, `yum`, `dnf`, `pacman`, `brew` — no package managers
- `sudo` — no sudo; run elevated terminal if needed
- `systemctl`, `journalctl`, `service` — no systemd
- `ip`, `ss`, `ifconfig` — use `ipconfig` via PowerShell instead
- `lsof`, `strace`, `ltrace` — not available
- `readlink -f` — use `realpath` or `cygpath -w` instead
- `chmod` — has no real effect on NTFS; use `git update-index --chmod=+x` for git
- `chown` — no effect on NTFS
- `ln -s` — symlinks require Developer Mode or admin privileges; may silently create copies

### When to use PowerShell instead
- Anything involving Windows services, registry, or .NET
- When calling native Windows tools that choke on MSYS2 path translation
- Complex JSON manipulation (PowerShell handles it natively)
- Invoke like: `powershell -Command "..."` or `pwsh -Command "..."`

### Line endings and encoding
- All shell scripts (.sh) must use LF line endings, not CRLF.
- The repo uses `.gitattributes` with `* text=auto` (verify this).
- Do not create files with BOM (EF BB BF).
- Git Bash uses UTF-8. PowerShell 5.1 defaults to UTF-16LE (with BOM) for file
  output, ASCII for `$OutputEncoding`. PowerShell 7+ defaults to UTF-8 NoBOM for
  file output and `$OutputEncoding`, but `[Console]::OutputEncoding` still uses the
  OEM codepage — non-ASCII from external programs can still be mangled.
- When piping between Git Bash and PowerShell, encoding mismatches will occur.
- Prefer doing all work within one shell context, not crossing boundaries.

### Performance awareness
- Avoid deeply nested subshells and pipes — fork() is emulated and slow on MSYS2.
- Prefer `rg` over `grep | grep | grep` chains.
- For large file operations, prefer single-pass tools over shell loops.

### Interactive programs
- Python, Node.js, and other interactive programs may need `winpty` prefix
  in MinTTY. If a command hangs, retry with `winpty <command>`.


### Prompt mode prefixes

Three prefixes control which tools Claude may use when responding.
Each prefix is a distinct mode with an explicit scope of allowed and forbidden tools.

`.??` — Knowledge only
  Allowed: web_search, web_fetch
  Forbidden: read_file, list_dir, bash, clj-nrepl-eval, write_file, edit_file, create_file, delete
  Do not interact with the project in any way. Answer from training knowledge and/or web search.

`.?>` — Read-only investigation
  Allowed: web_search, web_fetch, read_file, list_dir, bash (rg, cat, fd only), clj-nrepl-eval
  Forbidden: write_file, edit_file, create_file, delete, any file mutation
  Read the codebase, search the web, evaluate REPL expressions. Report findings as explanation only.

(no prefix) — Full action
  Allowed: All tools, no restrictions
  Default behavior. Read, write, edit, execute, create, delete as needed.

#### Prefix recognition rules
Prefixes are recognized ONLY when they appear as the first non-whitespace
characters of the prompt, immediately followed by a space. Ignore `.??` and `.?>`
appearing anywhere else in the text — those are not mode switches.

  Matches:    .?? how does ratom work
  Matches:    .?> which namespaces use re-frame
  NO match:   what is this.?? and why does it fail
  NO match:   see the docs at example.com/.?> for details
  NO match:   I have a question .?? about routing
