# Pre-Commit and Linting

This project enforces code quality through a comprehensive suite of pre-commit hooks. Every commit is automatically checked for formatting, linting, type errors, security issues, and markdown consistency. Pre-commit must pass before any commit — bypassing hooks with `--no-verify` is forbidden.

## Setup

Install the pre-commit framework and register the Git hooks:

```bash
pip install pre-commit
pre-commit install
```

The project pins Python 3.13 as the default language version for all hooks:

```yaml
default_language_version:
  python: python3.13
```

## Running Pre-Commit

To run all hooks against every file in the repository:

```bash
pre-commit run --all-files
```

To run hooks only against staged files (this happens automatically on `git commit`):

```bash
pre-commit run
```

To run a specific hook in isolation:

```bash
pre-commit run ruff --all-files
pre-commit run mypy --all-files
pre-commit run flake8 --all-files
pre-commit run gitleaks --all-files
pre-commit run markdownlint-cli2 --all-files
```

> **Warning:** Never use `git commit --no-verify` to skip pre-commit hooks. This is explicitly prohibited by the project's coding standards.

## Hook Overview

The `.pre-commit-config.yaml` configures seven hook repositories, executed in order:

| Hook Repository | Version | Purpose |
|---|---|---|
| `pre-commit-hooks` | v6.0.0 | General file hygiene checks |
| `flake8` | 7.3.0 | Mutable default argument detection |
| `detect-secrets` | v1.5.0 | Secret/credential scanning |
| `ruff-pre-commit` | v0.15.4 | Python linting and formatting |
| `gitleaks` | v8.30.0 | Git history secret scanning |
| `mirrors-mypy` | v1.19.1 | Static type checking |
| `markdownlint-cli2` | v0.21.0 | Markdown file linting |

## General File Hygiene (pre-commit-hooks)

The first set of hooks performs basic file quality checks:

```yaml
- repo: https://github.com/pre-commit/pre-commit-hooks
  rev: v6.0.0
  hooks:
    - id: check-added-large-files
    - id: check-docstring-first
    - id: check-executables-have-shebangs
    - id: check-merge-conflict
    - id: check-symlinks
    - id: detect-private-key
    - id: mixed-line-ending
    - id: debug-statements
    - id: trailing-whitespace
      args: [--markdown-linebreak-ext=md]
    - id: end-of-file-fixer
    - id: check-ast
    - id: check-builtin-literals
    - id: check-toml
```

These hooks catch common issues:

- **check-added-large-files** — Prevents accidentally committing large binary files
- **check-docstring-first** — Ensures docstrings appear before code in modules
- **check-executables-have-shebangs** — Validates executable scripts have proper shebang lines
- **check-merge-conflict** — Detects leftover merge conflict markers (`<<<<<<<`, `>>>>>>>`)
- **check-symlinks** — Flags broken symlinks
- **detect-private-key** — Scans for accidentally committed private keys
- **mixed-line-ending** — Enforces consistent line endings (no mixed CRLF/LF)
- **debug-statements** — Catches leftover `breakpoint()`, `pdb.set_trace()`, and similar debug calls
- **trailing-whitespace** — Removes trailing whitespace (Markdown files excluded via `--markdown-linebreak-ext=md`)
- **end-of-file-fixer** — Ensures files end with a single newline
- **check-ast** — Validates Python files parse correctly
- **check-builtin-literals** — Flags `dict()` and `list()` calls that should use `{}` and `[]` literals
- **check-toml** — Validates TOML file syntax

## Ruff — Formatting and Linting

[Ruff](https://docs.astral.sh/ruff/) handles both Python formatting and linting via two separate hooks:

```yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.15.4
  hooks:
    - id: ruff
    - id: ruff-format
```

### Ruff Configuration

Ruff is configured in `pyproject.toml`:

```toml
[tool.ruff]
preview = true
line-length = 120
fix = true
output-format = "grouped"

[tool.ruff.format]
exclude = [".git", ".venv", ".mypy_cache", ".tox", "__pycache__"]

[tool.ruff.lint]
select = ["PLC0415"]
```

| Setting | Value | Description |
|---|---|---|
| `preview` | `true` | Enables preview/unstable rules for early adoption |
| `line-length` | `120` | Maximum line length (characters) |
| `fix` | `true` | Automatically fix violations where possible |
| `output-format` | `"grouped"` | Groups lint results by file for readability |

### Lint Rules

The `select = ["PLC0415"]` setting enables a specific Pylint convention rule:

- **PLC0415** (`import-outside-top-level`) — Flags `import` statements that appear outside the top level of a module (e.g., inside functions or conditionals)

> **Note:** Ruff's `fix = true` means many formatting and linting issues are auto-corrected when you run pre-commit. If a hook "fails" but the files have been modified, simply `git add` the changed files and commit again.

### Excluded Directories

The formatter skips these directories:

- `.git` — Git internals
- `.venv` — Virtual environment
- `.mypy_cache` — Mypy cache
- `.tox` — Tox environments
- `__pycache__` — Python bytecode cache

## Flake8 — Mutable Default Detection

Flake8 is configured with a narrow scope: it only checks for mutable default arguments using the `flake8-mutable` plugin.

```yaml
- repo: https://github.com/PyCQA/flake8
  rev: 7.3.0
  hooks:
    - id: flake8
      args: [--config=.flake8]
      additional_dependencies: [flake8-mutable]
```

### Flake8 Configuration

The `.flake8` configuration file limits checks to a single rule:

```ini
[flake8]
select=M511

exclude =
    doc,
    .tox,
    .git,
    .yml,
    Pipfile.*,
    docs/*,
    .cache/*
```

**M511** — Flags mutable default argument values in function definitions. This catches a common Python pitfall:

```python
# Wrong — mutable default is shared across calls
def process_vms(vm_list: list = []):
    vm_list.append("new_vm")
    return vm_list

# Correct — use None and create a new list
def process_vms(vm_list: list | None = None):
    if vm_list is None:
        vm_list = []
    vm_list.append("new_vm")
    return vm_list
```

## Mypy — Static Type Checking

Mypy performs static type analysis to catch type errors before runtime:

```yaml
- repo: https://github.com/pre-commit/mirrors-mypy
  rev: v1.19.1
  hooks:
    - id: mypy
      additional_dependencies:
        [
          "types-pyvmomi",
          "types-requests",
          "types-six",
          "types-pytz",
          "types-PyYAML",
          "types-paramiko",
        ]
```

### Type Stub Dependencies

Mypy requires type stubs for third-party libraries that don't ship their own type annotations. The following stub packages are installed as additional dependencies:

| Stub Package | Provides Types For |
|---|---|
| `types-pyvmomi` | VMware vSphere SDK |
| `types-requests` | HTTP requests library |
| `types-six` | Python 2/3 compatibility |
| `types-pytz` | Timezone handling |
| `types-PyYAML` | YAML parsing |
| `types-paramiko` | SSH connections |

### Mypy Configuration

Mypy is configured in `pyproject.toml`:

```toml
[tool.mypy]
disallow_any_generics = false
disallow_incomplete_defs = true
no_implicit_optional = true
show_error_codes = true
warn_unused_ignores = true

[[tool.mypy.overrides]]
module = "paramiko"
ignore_missing_imports = true
```

| Setting | Value | Effect |
|---|---|---|
| `disallow_any_generics` | `false` | Allows `list` without type parameters (e.g., `list` instead of `list[str]`) |
| `disallow_incomplete_defs` | `true` | Requires all function parameters and return types to be annotated if any are |
| `no_implicit_optional` | `true` | Requires explicit `Optional` or `X \| None` — `def f(x: str = None)` is an error |
| `show_error_codes` | `true` | Displays error codes like `[assignment]` for easier suppression |
| `warn_unused_ignores` | `true` | Flags `# type: ignore` comments that are no longer needed |

The `paramiko` module override suppresses import errors since paramiko's type stubs may not cover all submodules.

> **Tip:** When adding a `# type: ignore` comment, always include the specific error code: `# type: ignore[assignment]` rather than a bare `# type: ignore`. The `warn_unused_ignores` setting will flag stale suppressions during pre-commit.

## Secret Scanning

Two complementary tools scan for accidentally committed secrets and credentials:

### Gitleaks

[Gitleaks](https://github.com/gitleaks/gitleaks) scans the Git history and staged changes for secrets such as API keys, tokens, and passwords:

```yaml
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.30.0
  hooks:
    - id: gitleaks
```

Gitleaks uses its built-in rules to detect patterns matching common secret formats (AWS keys, GitHub tokens, generic passwords, etc.). No custom `.gitleaks.toml` configuration is present — the default ruleset is used.

### Detect-Secrets

[Detect-secrets](https://github.com/Yelp/detect-secrets) from Yelp provides an additional layer of secret detection using a different set of heuristics:

```yaml
- repo: https://github.com/Yelp/detect-secrets
  rev: v1.5.0
  hooks:
    - id: detect-secrets
```

> **Warning:** If either secret scanner flags a false positive, do not simply remove the hook. Instead, configure an appropriate allowlist or baseline file for the specific tool.

## Markdownlint — Markdown Linting

All Markdown files (`.md`) are linted using `markdownlint-cli2`, configured to auto-fix issues:

```yaml
- repo: https://github.com/DavidAnson/markdownlint-cli2
  rev: v0.21.0
  hooks:
    - id: markdownlint-cli2
      args: ["--fix"]
```

### Markdownlint Configuration

The `.markdownlint.yaml` file customizes the default rules:

```yaml
# Default state for all rules
default: true

# MD013/line-length - Line length
MD013:
  line_length: 180

# MD033/no-inline-html - Allow inline HTML for collapsible sections
MD033:
  allowed_elements:
    - details
    - summary
    - strong
```

| Rule | Setting | Description |
|---|---|---|
| `default` | `true` | All markdownlint rules are enabled by default |
| `MD013` | `line_length: 180` | Maximum line length for Markdown set to 180 characters |
| `MD033` | Allows `details`, `summary`, `strong` | Permits specific inline HTML elements used for collapsible sections |

> **Note:** The `--fix` argument means markdownlint will auto-correct fixable issues (trailing spaces, blank lines, heading formatting). Like ruff, you may need to `git add` modified files and re-commit after auto-fixes are applied.

## Troubleshooting

### Pre-commit fails after auto-fixing files

Several hooks (ruff, ruff-format, markdownlint-cli2, trailing-whitespace, end-of-file-fixer) automatically modify files. When this happens:

1. The hook reports a failure because files were changed
2. Review the changes with `git diff`
3. Stage the fixed files with `git add`
4. Commit again

### Mypy reports missing imports

If mypy fails with `Cannot find implementation or library stub`, the library likely needs a type stub. Check if a `types-*` package exists on PyPI and add it to the `additional_dependencies` list in `.pre-commit-config.yaml`.

### Updating hook versions

To update all hooks to their latest versions:

```bash
pre-commit autoupdate
```

To update a specific hook:

```bash
pre-commit autoupdate --repo https://github.com/astral-sh/ruff-pre-commit
```

### Clearing the pre-commit cache

If hooks behave unexpectedly after version changes:

```bash
pre-commit clean
pre-commit install
```
