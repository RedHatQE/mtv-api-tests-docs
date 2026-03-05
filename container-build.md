# Container Build and Registry

This page covers building the mtv-api-tests container image, explains the Dockerfile internals, and describes how to push images to `ghcr.io/redhatqe/mtv-api-tests`.

---

## Pre-built Image

A public pre-built image is available and recommended for most users:

```
ghcr.io/redhatqe/mtv-api-tests:latest
```

You only need to build a custom image if you are modifying tests, updating dependencies, or pinning specific versions of OpenShift libraries.

---

## Building the Image

### Basic Build

Build with either Podman (preferred) or Docker:

```bash
# Using Podman
podman build -f Dockerfile -t mtv-api-tests .

# Using Docker
docker build -f Dockerfile -t mtv-api-tests .
```

### Build with Custom OpenShift Dependencies

The Dockerfile supports two build arguments for pinning specific commits of OpenShift Python libraries. This is useful for testing against unreleased fixes or development branches:

```bash
podman build \
  --build-arg OPENSHIFT_PYTHON_WRAPPER_COMMIT=abc123def456 \
  --build-arg OPENSHIFT_PYTHON_UTILITIES_COMMIT=789xyz \
  -f Dockerfile -t mtv-api-tests .
```

| Build Argument | Default | Description |
|---|---|---|
| `OPENSHIFT_PYTHON_WRAPPER_COMMIT` | `''` (empty) | Git commit hash from [openshift-python-wrapper](https://github.com/RedHatQE/openshift-python-wrapper) |
| `OPENSHIFT_PYTHON_UTILITIES_COMMIT` | `''` (empty) | Git commit hash from [openshift-python-utilities](https://github.com/RedHatQE/openshift-python-utilities) |

When these arguments are empty (the default), the locked versions from `uv.lock` are used. When a commit hash is provided, `uv pip install` overrides the locked version with the specified commit from GitHub.

### Tagging for a Registry

Tag the image for your target registry:

```bash
# For GitHub Container Registry
podman build -f Dockerfile -t ghcr.io/redhatqe/mtv-api-tests:latest .

# For Quay.io
podman build -t quay.io/youruser/mtv-api-tests:latest .

# For Docker Hub
podman build -t docker.io/youruser/mtv-api-tests:latest .
```

---

## Dockerfile Internals

The Dockerfile uses a multi-stage copy pattern on a Fedora 41 base with uv as the Python package manager.

### Base Image

```dockerfile
FROM quay.io/fedora/fedora:41
```

Fedora 41 provides an up-to-date Linux userland with dnf package management. It serves as a stable foundation for the compilation toolchain required by several Python dependencies.

### Environment Variables

```dockerfile
ENV JUNITFILE=${APP_DIR}/output/
ENV UV_PYTHON=python3.12
ENV UV_COMPILE_BYTECODE=1
ENV UV_NO_SYNC=1
ENV UV_CACHE_DIR=${APP_DIR}/.cache
```

| Variable | Value | Purpose |
|---|---|---|
| `JUNITFILE` | `/app/output/` | Directory for JUnit XML test output |
| `UV_PYTHON` | `python3.12` | Forces uv to use Python 3.12 |
| `UV_COMPILE_BYTECODE` | `1` | Pre-compiles `.py` files to `.pyc` bytecode for faster container startup |
| `UV_NO_SYNC` | `1` | Prevents automatic `uv sync` on every `uv run`, since dependencies are already installed at build time |
| `UV_CACHE_DIR` | `/app/.cache` | Isolates the uv cache for clean removal after dependency installation |

### System Dependencies

```dockerfile
RUN dnf -y install \
  libxml2-devel \
  libcurl-devel \
  openssl \
  openssl-devel \
  libcurl-devel \
  gcc \
  clang \
  python3-devel \
  && dnf clean all \
  && rm -rf /var/cache/dnf \
  && rm -rf /var/lib/dnf \
  && truncate -s0 /var/log/*.log && rm -rf /var/cache/yum
```

These packages fall into two categories:

**Compilation toolchain** — required to build Python C extensions from source:
- `gcc` — GNU C compiler
- `clang` — LLVM C/C++ compiler
- `python3-devel` — Python headers and development files

**Library development headers** — required by specific Python packages:
- `libxml2-devel` — XML parsing (used by lxml, needed by OpenShift client libraries)
- `libcurl-devel` — HTTP client (used by pycurl)
- `openssl` / `openssl-devel` — TLS/SSL support (used by cryptography and other security libraries)

The final commands in this layer clean up dnf caches, package metadata, and log files to reduce image size.

### Source Code Copy

```dockerfile
WORKDIR ${APP_DIR}

RUN mkdir -p ${APP_DIR}/output

COPY utilities utilities
COPY tests tests
COPY libs libs
COPY exceptions exceptions
COPY README.md pyproject.toml uv.lock conftest.py pytest.ini ./
```

The image includes only the files necessary for test execution:

| Path | Contents |
|---|---|
| `utilities/` | Helper modules for migration operations, post-migration checks, and resource management |
| `tests/` | Test files, test configuration (`tests/tests_config/config.py`), and conftest files |
| `libs/` | Provider abstraction layer (`BaseProvider` and implementations) |
| `exceptions/` | Custom exception classes |
| `pyproject.toml` | Project metadata and dependency declarations |
| `uv.lock` | Locked dependency versions with integrity hashes |
| `conftest.py` | Root-level pytest fixtures |
| `pytest.ini` | Pytest configuration (markers, default options, JUnit output) |

> **Note:** Development tooling (`pre-commit`, `.github/`, `tox.toml`, `tools/`, `docs/`) is intentionally excluded from the container to keep the image focused and small.

### uv Package Manager Installation

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/
```

Rather than installing uv through pip or curl, the Dockerfile uses a multi-stage `COPY --from` to pull the pre-built `uv` and `uvx` binaries directly from the [official uv container image](https://github.com/astral-sh/uv). This approach is fast, reproducible, and avoids adding installation tooling to the image.

### Dependency Installation

```dockerfile
RUN uv sync --locked\
  && if [ -n "${OPENSHIFT_PYTHON_WRAPPER_COMMIT}" ]; then uv pip install git+https://github.com/RedHatQE/openshift-python-wrapper.git@$OPENSHIFT_PYTHON_WRAPPER_COMMIT; fi \
  && if [ -n "${OPENSHIFT_PYTHON_UTILITIES_COMMIT}" ]; then uv pip install git+https://github.com/RedHatQE/openshift-python-utilities.git@$OPENSHIFT_PYTHON_UTILITIES_COMMIT; fi \
  && find ${APP_DIR}/ -type d -name "__pycache__" -print0 | xargs -0 rm -rfv \
  && rm -rf ${APP_DIR}/.cache
```

This layer performs four operations in a single `RUN` to minimize image layers:

1. **`uv sync --locked`** — Installs all dependencies exactly as specified in `uv.lock`, ensuring reproducible builds
2. **Conditional overrides** — If build arguments are provided, overrides specific OpenShift packages with pinned Git commits
3. **Cache cleanup** — Removes `__pycache__` directories (bytecode was already compiled via `UV_COMPILE_BYTECODE=1`)
4. **uv cache removal** — Deletes the uv download cache (`/app/.cache`) since packages are already installed

### Default Command

```dockerfile
CMD ["uv", "run", "pytest", "--collect-only"]
```

The default command runs pytest in `--collect-only` mode, which lists all discovered tests without executing them. This serves as a safe default that verifies the container is functional. Override it with your actual test command at runtime:

```bash
podman run --rm ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v --tc=source_provider:vsphere-8.0.1 ...
```

### .dockerignore

The `.dockerignore` file excludes Python bytecode from the build context:

```
*.pyc
*.pyo
*__pycache__/
```

This prevents stale compiled bytecode from the host from being copied into the image, since the container builds its own via `UV_COMPILE_BYTECODE=1`.

---

## Image Size Optimizations

The Dockerfile employs several strategies to minimize the final image size:

1. **Single-layer cleanup** — dnf cache removal, log truncation, and yum cache deletion in the same `RUN` as installation
2. **Multi-stage binary copy** — Copies only the `uv`/`uvx` binaries from the astral image instead of installing from the internet
3. **Build cache removal** — Deletes the uv download cache (`/app/.cache`) after dependency installation
4. **Bytecode cleanup** — Removes `__pycache__` directories (the compiled `.pyc` files from `UV_COMPILE_BYTECODE` are embedded elsewhere)
5. **Selective COPY** — Only the directories required for test execution are copied; dev tools and documentation are excluded
6. **`.dockerignore`** — Prevents host-side `.pyc`/`.pyo` files from inflating the build context

---

## Pushing to ghcr.io

### Authentication

Authenticate with GitHub Container Registry before pushing:

```bash
# Using a GitHub Personal Access Token (PAT) with write:packages scope
echo $GITHUB_TOKEN | podman login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

> **Warning:** Your PAT must have the `write:packages` scope. Classic tokens also need the `read:packages` scope. Fine-grained tokens should have "Packages: Read and write" permission.

### Push the Image

```bash
# Tag and push
podman build -f Dockerfile -t ghcr.io/redhatqe/mtv-api-tests:latest .
podman push ghcr.io/redhatqe/mtv-api-tests:latest
```

To push a versioned tag alongside `latest`:

```bash
podman tag ghcr.io/redhatqe/mtv-api-tests:latest ghcr.io/redhatqe/mtv-api-tests:2.8.3
podman push ghcr.io/redhatqe/mtv-api-tests:2.8.3
podman push ghcr.io/redhatqe/mtv-api-tests:latest
```

> **Tip:** The project version is defined in `pyproject.toml` (`version = "2.8.3"`). Use this as a reference when tagging release images.

### Verify the Push

```bash
# List tags for the package
podman pull ghcr.io/redhatqe/mtv-api-tests:latest
podman run --rm ghcr.io/redhatqe/mtv-api-tests:latest
```

The default `CMD` runs `pytest --collect-only`, so a successful pull and run displays all discovered tests without executing them.

---

## Dependency Management with uv

The container uses [uv](https://github.com/astral-sh/uv) as its Python package manager. Dependencies are managed through two files:

### pyproject.toml

Declares project metadata and dependency version ranges. Key dependencies include:

- **Test framework**: `pytest`, `pytest-xdist` (parallel execution), `pytest-testconfig`, `pytest-harvest`
- **OpenShift interaction**: `openshift-python-wrapper`, `openshift-python-utilities`
- **Provider SDKs**: `pyvmomi` (VMware), `ovirt-python-sdk` (RHV), `openstacksdk` / `python-glanceclient` (OpenStack)
- **Utilities**: `pandas`, `rich`, `typer`, `requests`, `marshmallow`

The full Python version requirement is `>=3.12, <3.14`.

### uv.lock

A lock file that pins every transitive dependency to an exact version with SHA-256 integrity hashes. The `uv sync --locked` command in the Dockerfile enforces these exact versions, ensuring deterministic builds regardless of when the image is built.

> **Tip:** To update locked dependencies locally, run `uv lock --upgrade` followed by `uv sync` and verify tests pass before rebuilding the container.

---

## Verifying the Container

After building, verify the image works correctly:

```bash
# List all discovered tests (default CMD)
podman run --rm mtv-api-tests

# List tests quietly
podman run --rm mtv-api-tests uv run pytest --collect-only -q

# Check Python version
podman run --rm mtv-api-tests uv run python --version

# Inspect installed packages
podman run --rm mtv-api-tests uv pip list

# Interactive shell for debugging
podman run --rm -it mtv-api-tests /bin/bash
```

> **Note:** Test execution requires a live OpenShift cluster, provider connections, and credentials. The commands above only verify the container build, not test functionality.
