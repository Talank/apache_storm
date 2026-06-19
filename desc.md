# Add Windows and macOS support to the CI workflow

## Purpose

The CI workflow was Linux-only. This PR extends it to run on **Windows** and **macOS** so the
project's build and test suite is validated cross-platform on every commit.

---

## Expected result

All three platforms — `ubuntu-latest`, `windows-latest`, `macos-latest` — build and test
successfully in the same workflow run.

---

## Actual result (before this fix)

### Error 1 — `/bin/bash` not found on Windows

```
Line |
   2 |  /bin/bash ./dev-tools/gitact/gitact-install.sh `pwd`
     |  ~~~~~~~~~
     | The term '/bin/bash' is not recognized as a name of a cmdlet,
     | function, script file, or executable program.
Error: Process completed with exit code 1.
```

The workflow steps hard-coded `/bin/bash` as the executable and used backtick `` `pwd` `` for
command substitution. On Windows, GitHub Actions defaults to PowerShell, which recognises
neither `/bin/bash` nor backtick substitution.

### Error 2 — `save-logs.py` cannot find `mvn` on Windows

```
FileNotFoundError: [WinError 2] The system cannot find the file specified
  File "save-logs.py", line 24, in main
    process = subprocess.Popen(cmd, ...)
```

After switching the shell to Git Bash (which works), `save-logs.py` calls
`subprocess.Popen(['mvn', 'clean', 'install', ...])`. Python's `subprocess` on Windows
requires the **full executable name** including the extension. Maven on Windows is installed
as `mvn.cmd`, so a bare `mvn` raises `WinError 2`.

### Error 3 — `-Pnative` build profile skips Windows

`gitact-install.sh` only checked for macOS (`uname == Darwin`) to skip the `-Pnative`
profile. On Windows (where `uname` returns `MINGW64_NT-...`), the script fell through to the
Linux branch and attempted to build native C libraries that cannot be compiled on a standard
Windows CI runner.

---

## Root causes

| # | Location | Cause |
|---|----------|-------|
| 1 | `maven.yaml` steps | Hard-coded `/bin/bash` and `` `pwd` `` — PowerShell-incompatible |
| 2 | `dev-tools/gitact/save-logs.py` | `subprocess.Popen(['mvn', ...])` — Windows needs `mvn.cmd`, not bare `mvn` |
| 3 | `dev-tools/gitact/gitact-install.sh` | `uname` OS check has no Windows branch; `-Pnative` profile fails on Windows |

---

## Fix

### 1. `maven.yaml` — replace `/bin/bash` + backtick with `bash` + Actions expression

`bash` is available on all three platforms via Git for Windows (Windows), the preinstalled
shell (Linux), and the system shell (macOS). `${{ github.workspace }}` is expanded by the
Actions runner before any shell sees it, making it safe in PowerShell too.

```yaml
# Before
- name: Build project (compile + install, skip tests)
  run: /bin/bash ./dev-tools/gitact/gitact-install.sh `pwd`

# After
- name: Build project (compile + install, skip tests)
  run: bash ./dev-tools/gitact/gitact-install.sh "${{ github.workspace }}"
```

Environment variables moved from shell `export` statements to the step `env:` block so they
are set before the shell starts, regardless of what shell is in use:

```yaml
# Before
- name: Run tests
  run: |
    export JDK_VERSION=${{ matrix.java }}
    export USER=github
    /bin/bash ./dev-tools/gitact/gitact-test.sh `pwd` ${{ matrix.module }};

# After
- name: Run tests
  env:
    JDK_VERSION: ${{ matrix.java }}
    USER: github
  run: bash ./dev-tools/gitact/gitact-test.sh "${{ github.workspace }}" ${{ matrix.module }}
```

The `rm -f` cleanup before the RAT check was also Linux-specific; the step now just calls
`mvn` directly (the files are build artifacts whose presence doesn't affect the RAT result).

### 2. `dev-tools/gitact/save-logs.py` — resolve executable via `shutil.which`

`shutil.which` on Windows respects `PATHEXT` (`.COM`, `.EXE`, `.BAT`, `.CMD`, …), so
`shutil.which('mvn')` returns the full path to `mvn.cmd`. This is a two-line, targeted change
with zero effect on Linux/macOS (where `which('mvn')` returns the same unextended path).

```python
# Added at the top of main():
if platform.system() == "Windows" and cmd:
    resolved = shutil.which(cmd[0])
    if resolved:
        cmd = [resolved] + list(cmd[1:])
```

### 3. `dev-tools/gitact/gitact-install.sh` — add Windows case to skip `-Pnative`

Extended the `if/else` into a `case` that matches all three platforms:

```bash
case "$(uname)" in
  Darwin*)      # macOS — no native build
  MINGW*|MSYS*|CYGWIN*)  # Windows Git Bash — no native build
  *)            # Linux — full build including native
esac
```

---

## Files changed

| File | Change |
|------|--------|
| `.github/workflows/maven.yaml` | Replace `/bin/bash` + backtick with `bash` + workspace var; move exports to `env:`; remove Linux-only `rm -f` from RAT step |
| `dev-tools/gitact/save-logs.py` | Resolve executable via `shutil.which` before `subprocess.Popen` on Windows |
| `dev-tools/gitact/gitact-install.sh` | Add Windows (`MINGW*`/`MSYS*`) `case` branch to skip `-Pnative` build profile |
