# Installation

This guide walks you through setting up the MTV API Tests project for local development. By the end, you will have a working environment ready for writing and reviewing tests.

## Prerequisites

Before you begin, ensure the following tools are available on your system.

### Python 3.12

The project requires Python 3.12.x (defined in `pyproject.toml`):

```toml
requires-python = ">=3.12, <3.14"
```

> **Tip:** You do not need to install Python manually. The `uv` package manager can download and manage the correct Python version for you automatically.

### uv

[uv](https://docs.astral.sh/uv/) is the required package manager for this project. All dependency management, virtual environment creation, and script execution use `uv`.

Install uv using the official installer:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Or via Homebrew on macOS:

```bash
brew install uv
```

Verify the installation:

```bash
uv --version
```

### Podman (for container-based execution)

[Podman](https://podman.io/) is used to build and run the test suite in a container. This is optional for local development but required for CI/CD workflows.

Install on Fedora/RHEL:

```bash
sudo dnf install podman
```

Install on Ubuntu/Debian:

```bash
sudo apt install podman
```

Install on macOS:

```bash
brew install podman
```

### System Libraries

Several Python dependencies require native C libraries for compilation. Install them before running `uv sync`.

**Fedora / RHEL / CentOS:**

```bash
sudo dnf install gcc clang python3-devel libxml2-devel libcurl-devel openssl-devel
```

**Ubuntu / Debian:**

```bash
sudo apt install gcc clang python3-dev libxml2-dev libcurl4-openssl-dev libssl-dev
```

**macOS:**

```bash
brew install libxml2 curl openssl
```

> **Note:** On macOS, the Xcode Command Line Tools (`xcode-select --install`) provide `gcc` and `clang`.

---

## Clone the Repository

```bash
git clone https://github.com/RedHatQE/mtv-api-tests.git
cd mtv-api-tests
```

The project has no git submodules — a standard clone is all that is needed.

---

## Install Dependencies

Run `uv sync` from the project root to create a virtual environment and install all dependencies from the lockfile:

```bash
uv sync
```

This command:

1. Creates a `.venv` virtual environment in the project directory (if one does not exist)
2. Installs Python 3.12 automatically if it is not found on your system
3. Installs all runtime and development dependencies as defined in `pyproject.toml` and locked in `uv.lock`

The development dependency group includes additional tools for interactive debugging:

```toml
[dependency-groups]
dev = ["ipdb>=0.13.13", "ipython>=8.12.3", "python-jenkins>=1.8.2"]
```

> **Warning:** Do not use `pip install` to manage dependencies. The project exclusively uses `uv` for package management. The `uv.lock` lockfile ensures reproducible installs across all environments.

---

## Install Pre-commit Hooks

The project uses [pre-commit](https://pre-commit.com/) to enforce code quality on every commit. The hooks are defined in `.pre-commit-config.yaml` and include:

| Hook | Purpose |
|------|---------|
| pre-commit-hooks (v6.0.0) | Trailing whitespace, end-of-file, AST checks, merge conflicts |
| flake8 (7.3.0) | Mutable default argument detection (`M511`) |
| detect-secrets (v1.5.0) | Prevents accidental secret commits |
| ruff (v0.15.4) | Code formatting and linting |
| gitleaks (v8.30.0) | Secret scanning |
| mypy (v1.19.1) | Static type checking |
| markdownlint-cli2 (v0.21.0) | Markdown linting |

Install the hooks into your local git repository:

```bash
uv run pre-commit install
```

---

## Verify the Setup

### 1. Run pre-commit checks

Run all pre-commit hooks against the entire codebase to confirm the environment is properly configured:

```bash
pre-commit run --all-files
```

A successful run produces output ending with `Passed` for each hook. This validates that all linting tools, formatters, and type checkers are functioning correctly.

### 2. Verify test collection

Confirm that pytest can discover and parse all test files without errors:

```bash
uv run pytest --collect-only
```

This performs a dry-run that imports all test modules, resolves parametrization, and lists discovered tests — without executing anything or requiring a live cluster.

### 3. Verify fixture resolution

Check that all pytest fixtures can be resolved correctly:

```bash
uv run pytest --setup-plan
```

> **Note:** Both `--collect-only` and `--setup-plan` are also available as tox environments defined in `tox.toml`:
>
> ```toml
> [env.pytest-check]
> commands = [
>   ["uv", "run", "pytest", "--setup-plan"],
>   ["uv", "run", "pytest", "--collect-only"],
> ]
> ```

> **Warning:** Never run `pytest` directly (without `--collect-only` or `--setup-plan`) in a development environment. The test suite requires live OpenShift clusters, source provider connections, and credentials to execute. Test execution must only happen against properly configured infrastructure.

---

## Configuration Files

Before running tests against a live environment, two configuration files are required. These are not needed for the installation verification steps above.

### Provider Configuration (`.providers.json`)

This file defines connection details for source virtualization providers (vSphere, RHV, OpenStack, OVA). A template is provided in the repository:

```bash
cp .providers.json.example .providers.json
chmod 600 .providers.json
```

Edit `.providers.json` with your provider credentials. The file supports multiple provider types:

```json
{
  "vsphere": {
    "type": "vsphere",
    "version": "<SERVER VERSION>",
    "fqdn": "SERVER FQDN/IP",
    "api_url": "<SERVER FQDN/IP>/sdk",
    "username": "USERNAME",
    "password": "PASSWORD",
    "vddk_init_image": "<PATH TO VDDK INIT IMAGE>"
  },
  "ovirt": {
    "type": "ovirt",
    "version": "<SERVER VERSION>",
    "fqdn": "SERVER FQDN/IP",
    "api_url": "<SERVER FQDN/IP>/ovirt-engine/api",
    "username": "USERNAME",
    "password": "PASSWORD"
  }
}
```

See `.providers.json.example` for the full list of supported providers and their fields.

> **Warning:** The `.providers.json` file contains sensitive credentials and is excluded from version control via `.gitignore`. Never commit this file.

### JIRA Configuration (`jira.cfg`, optional)

If your test runs integrate with JIRA for test case tracking, copy the example and fill in your details:

```bash
cp jira.cfg.example jira.cfg
```

```ini
[DEFAULT]
url = <Jira URL>
token = <User Token>
```

---

## Container-Based Setup

As an alternative to local installation, you can build and use the container image. The `Dockerfile` uses Fedora 41 as the base image and bundles all dependencies.

### Build the container image

```bash
podman build -f Dockerfile -t mtv-api-tests
```

The build process installs system libraries, copies the project source, and runs `uv sync --locked` to install Python dependencies from the lockfile.

### Build with custom dependency overrides

To test against specific commits of `openshift-python-wrapper` or `openshift-python-utilities`, pass build arguments:

```bash
podman build -f Dockerfile -t mtv-api-tests \
  --build-arg OPENSHIFT_PYTHON_WRAPPER_COMMIT=abc123 \
  --build-arg OPENSHIFT_PYTHON_UTILITIES_COMMIT=def456
```

### Verify the container

The default container command runs `pytest --collect-only`:

```bash
podman run --rm mtv-api-tests
```

> **Tip:** A pre-built image is available at `ghcr.io/redhatqe/mtv-api-tests:latest` if you do not need a custom build.

---

## Project Structure Overview

After installation, the project directory is organized as follows:

```
mtv-api-tests/
├── pyproject.toml              # Project metadata, dependencies, tool config
├── uv.lock                     # Locked dependency versions
├── pytest.ini                  # pytest configuration and markers
├── conftest.py                 # Root pytest fixtures and hooks
├── Dockerfile                  # Container build definition
├── .pre-commit-config.yaml     # Pre-commit hook configuration
├── .providers.json.example     # Provider credentials template
├── tests/
│   ├── test_*.py               # Test modules
│   └── tests_config/
│       └── config.py           # Test parametrization configuration
├── utilities/                  # Shared utility functions
├── libs/                       # Provider abstraction layer
└── exceptions/                 # Custom exception classes
```

---

## Troubleshooting

### `uv sync` fails with compilation errors

Ensure all system libraries are installed (see [System Libraries](#system-libraries)). The `pyvmomi`, `lxml`, and `pycurl` packages require native C headers for compilation.

### Pre-commit hooks fail on first run

Some hooks download their own tooling on first execution. If `mypy` fails, it may need to download type stubs. Re-run:

```bash
pre-commit run --all-files
```

### `pytest --collect-only` reports import errors

This typically indicates a missing dependency. Run `uv sync` again to ensure the virtual environment is up to date:

```bash
uv sync
```

If the error references `openshift-python-wrapper` or `openshift-python-utilities`, ensure you have network access to PyPI — these are external packages.

### Python version mismatch

If `uv` selects the wrong Python version, you can explicitly request 3.12:

```bash
uv sync --python 3.12
```
