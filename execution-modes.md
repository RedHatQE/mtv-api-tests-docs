# Execution Modes

This page covers the three primary ways to run mtv-api-tests: locally with `uv run pytest`, inside a container with Podman or Docker, and as an OpenShift Job for CI/CD pipelines.

---

## Local Execution

Running tests locally is suited for test developers who need a fast feedback loop. It requires manual installation of system dependencies and direct access to an OpenShift cluster with MTV installed.

### Prerequisites

**System packages** (required for Python extension compilation):

```bash
# RHEL/Fedora
sudo dnf install gcc clang libxml2-devel libcurl-devel openssl-devel

# Ubuntu/Debian
sudo apt install gcc clang libxml2-dev libcurl4-openssl-dev libssl-dev

# macOS
brew install gcc libxml2 curl openssl
```

**Required tools:**

- **uv** — Package manager (automatically manages Python 3.12)
- **oc** — OpenShift CLI
- **virtctl** — KubeVirt CLI for VM operations

> **Note:** uv automatically downloads and manages the appropriate Python version (>=3.12, <3.14). No system Python installation is needed.

### Setup

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and install dependencies
git clone https://github.com/RedHatQE/mtv-api-tests.git
cd mtv-api-tests
uv sync
```

### Running Tests

Tests are executed with `uv run pytest`. Configuration is passed via `--tc=key:value` flags using the `pytest-testconfig` plugin.

**Required configuration parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `source_provider` | Provider key from `.providers.json` | `vsphere-8.0.1` |
| `storage_class` | OpenShift storage class name | `ocs-storagecluster-ceph-rbd` |
| `cluster_host` | OpenShift API URL | `https://api.cluster.com:6443` |
| `cluster_username` | Cluster username | `kubeadmin` |
| `cluster_password` | Cluster password | (from env var) |

**Basic execution:**

```bash
export CLUSTER_PASSWORD='your-cluster-password'

uv run pytest -m tier0 -v \
  --tc=cluster_host:https://api.cluster.com:6443 \
  --tc=cluster_username:kubeadmin \
  --tc=cluster_password:${CLUSTER_PASSWORD} \
  --tc=source_provider:vsphere-8.0.1 \
  --tc=storage_class:standard-csi
```

**Run specific test categories using markers:**

```bash
# Warm migration tests
uv run pytest -m warm -v --tc=source_provider:vsphere-8.0.1 ...

# Copy-offload tests
uv run pytest -m copyoffload -v --tc=source_provider:vsphere-8.0.1 ...

# Combine markers
uv run pytest -m "tier0 or warm" -v --tc=source_provider:vsphere-8.0.1 ...
```

**Run specific tests by name:**

```bash
# Match by test name pattern
uv run pytest -k test_copyoffload_thin_migration -v ...

# Combine marker and name filter
uv run pytest -m copyoffload -k "thin or thick" -v ...
```

**List all available tests (dry run):**

```bash
uv run pytest --collect-only -q
```

### Test Markers

Markers select which test categories to run via the `-m` flag:

| Marker | Description |
|--------|-------------|
| `tier0` | Core functionality — smoke tests for critical paths |
| `warm` | Warm migration — VMs stay running during migration |
| `remote` | Remote cluster migration tests |
| `copyoffload` | Copy-offload (XCOPY) — accelerated migrations via shared storage |
| `incremental` | Sequential tests — subsequent tests xfail if a prior test fails |
| `min_mtv_version` | Version-gated tests (e.g., `@pytest.mark.min_mtv_version("2.6.0")`) |

These markers are registered in `pytest.ini` with `--strict-markers` enforced:

```ini
[pytest]
markers =
    tier0: Core functionality tests (smoke tests)
    remote: Remote cluster migration tests
    warm: Warm migration tests
    copyoffload: Copy-offload (XCOPY) tests
    incremental: marks tests as incremental (xfail on previous failure)
    min_mtv_version: mark test to require minimum MTV version
```

### Custom Command-Line Options

The test suite registers several custom pytest options (defined in `conftest.py`):

| Option | Description |
|--------|-------------|
| `--skip-teardown` | Keep all created resources after tests finish (useful for debugging) |
| `--skip-data-collector` | Disable resource tracking to `.data-collector/resources.json` |
| `--data-collector-path PATH` | Custom path for resource tracking output (default: `.data-collector`) |
| `--openshift-python-wrapper-log-debug` | Enable DEBUG-level logging for all OpenShift API calls |
| `--analyze-with-ai` | Enrich JUnit XML with AI-powered failure analysis via JJI server |

**Debugging example — verbose output with resource preservation:**

```bash
uv run pytest -s -vv -m tier0 --skip-teardown \
  --openshift-python-wrapper-log-debug \
  --tc=cluster_host:https://api.cluster.com:6443 \
  --tc=cluster_username:kubeadmin \
  --tc=cluster_password:${CLUSTER_PASSWORD} \
  --tc=source_provider:vsphere-8.0.1 \
  --tc=storage_class:standard-csi
```

### Pytest Configuration

The full default configuration lives in `pytest.ini`:

```ini
[pytest]
testpaths = tests

addopts =
  -s
  -o log_cli=true
  -p no:logging
  --tc-file=tests/tests_config/config.py
  --tc-format=python
  --junit-xml=junit-report.xml
  --basetemp=/tmp/pytest
  --show-progress
  --strict-markers
  --jira
  --dist=loadscope

junit_logging = all
```

Key defaults applied on every run:

- **`-s`** — Disable output capture (print statements visible in real time)
- **`--tc-file=tests/tests_config/config.py`** — Load test parameters from Python config file
- **`--junit-xml=junit-report.xml`** — Generate JUnit XML report
- **`--dist=loadscope`** — Parallel distribution groups tests by class/module (via `pytest-xdist`)
- **`--strict-markers`** — Unknown markers cause errors (prevents typos)
- **`--jira`** — Enable JIRA integration for linking test results to issues

### Parallel Execution

Tests run in parallel by default via `pytest-xdist` with `--dist=loadscope`. This groups all tests within the same class onto a single worker, which is critical because test classes use `@pytest.mark.incremental` and share state through class attributes.

To control the number of workers:

```bash
# Use 4 parallel workers
uv run pytest -n 4 -m tier0 ...

# Disable parallel execution (single process)
uv run pytest -n 0 -m tier0 ...

# Auto-detect based on CPU cores
uv run pytest -n auto -m tier0 ...
```

> **Note:** Parallel safety is guaranteed by unique namespaces per session (`session_uuid`), isolated `fixture_store` per worker, and unique resource names generated by `create_and_store_resource()`.

### Validation Without a Cluster

You can validate test structure and fixture wiring without a live cluster using tox:

```bash
# Verify test collection and fixture setup plans
tox -e pytest-check
```

This runs two commands from `tox.toml`:

```toml
[env.pytest-check]
commands = [
  ["uv", "run", "pytest", "--setup-plan"],
  ["uv", "run", "pytest", "--collect-only"],
]
```

### Test Output and Reports

Every test run produces:

| Output | Path | Description |
|--------|------|-------------|
| JUnit XML | `junit-report.xml` | Machine-readable test results (configurable via `--junit-xml`) |
| Test log | `pytest-tests.log` | Detailed execution log with fixture setup, API calls, and teardown |
| Resource tracker | `.data-collector/resources.json` | JSON file of all created OpenShift resources (unless `--skip-data-collector`) |

---

## Container Execution

Running tests in a container is the recommended approach for most users. It eliminates system dependency management and provides a consistent, reproducible environment.

### Container Image

A pre-built public image is available:

```
ghcr.io/redhatqe/mtv-api-tests:latest
```

The image is based on Fedora 41 with Python 3.12, all system dependencies pre-installed, and the test suite ready to run.

### Building a Custom Image

Build from the project `Dockerfile`:

```bash
# Standard build
podman build -f Dockerfile -t mtv-api-tests .

# With custom openshift-python-wrapper commit
podman build \
  --build-arg OPENSHIFT_PYTHON_WRAPPER_COMMIT=abc123def \
  -t mtv-api-tests .

# With custom openshift-python-utilities commit
podman build \
  --build-arg OPENSHIFT_PYTHON_UTILITIES_COMMIT=456789abc \
  -t mtv-api-tests .
```

> **Tip:** Use `docker` instead of `podman` if that is your container runtime. All commands are interchangeable.

**Build arguments:**

| Argument | Default | Description |
|----------|---------|-------------|
| `OPENSHIFT_PYTHON_WRAPPER_COMMIT` | (empty) | Override openshift-python-wrapper to a specific Git commit |
| `OPENSHIFT_PYTHON_UTILITIES_COMMIT` | (empty) | Override openshift-python-utilities to a specific Git commit |

The Dockerfile structure:

```dockerfile
FROM quay.io/fedora/fedora:41

ARG APP_DIR=/app
ENV UV_PYTHON=python3.12
ENV UV_COMPILE_BYTECODE=1
ENV UV_NO_SYNC=1
ENV UV_CACHE_DIR=${APP_DIR}/.cache

RUN dnf -y install \
  libxml2-devel libcurl-devel openssl openssl-devel gcc clang python3-devel \
  && dnf clean all ...

WORKDIR ${APP_DIR}
COPY utilities tests libs exceptions ./
COPY README.md pyproject.toml uv.lock conftest.py pytest.ini ./
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

RUN uv sync --locked ...

CMD ["uv", "run", "pytest", "--collect-only"]
```

> **Note:** The default `CMD` runs `--collect-only`, which lists tests without executing them. Override the command to actually run tests.

### Running Tests in a Container

**Basic execution:**

```bash
export CLUSTER_PASSWORD='your-cluster-password'

podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -e CLUSTER_PASSWORD \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v \
    --tc=cluster_host:https://api.your-cluster.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:YOUR-STORAGE-CLASS
```

**With persistent test results:**

```bash
podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -v $(pwd)/results:/app/results \
  -e CLUSTER_PASSWORD \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v \
    --junit-xml=/app/results/junit-report.xml \
    --tc=cluster_host:https://api.your-cluster.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:YOUR-STORAGE-CLASS
```

**With debug logging and resource preservation:**

```bash
podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -e OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL=DEBUG \
  -e CLUSTER_PASSWORD \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -s -vv -m tier0 --skip-teardown \
    --tc=cluster_host:https://api.your-cluster.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:YOUR-STORAGE-CLASS
```

> **Warning:** On RHEL/Fedora with SELinux, add `,z` to volume mounts: `-v $(pwd)/.providers.json:/app/.providers.json:ro,z`

> **Note:** Windows users should replace `$(pwd)` with `${PWD}` in PowerShell or use absolute paths like `C:\path\to\.providers.json:/app/.providers.json:ro`.

### Volume Mounts

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `.providers.json` | `/app/.providers.json` | Source provider configuration (required) |
| `./results/` | `/app/results/` | Persistent test results (optional) |

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `CLUSTER_PASSWORD` | Yes | Cluster password (avoids shell history exposure) |
| `OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL` | No | Set to `DEBUG` for verbose API call logging |
| `JJI_SERVER_URL` | No | Jenkins Job Insight server URL (for `--analyze-with-ai`) |
| `JJI_AI_PROVIDER` | No | AI provider (default: `claude`) |
| `JJI_AI_MODEL` | No | AI model identifier (default: `claude-opus-4-6[1m]`) |
| `JJI_TIMEOUT` | No | AI analysis request timeout in seconds (default: `600`) |

---

## OpenShift Job Execution

For long-running test suites or automated CI/CD pipelines, tests can run as OpenShift Jobs directly on the cluster. This approach keeps credentials in Kubernetes Secrets and allows monitoring through standard OpenShift tooling.

### Step 1: Create a Namespace and Secret

Store provider configuration and cluster credentials in an OpenShift Secret:

**Option A — Providers configuration only** (credentials passed in Job command):

```bash
oc create namespace mtv-tests
oc create secret generic mtv-test-config \
  --from-file=providers.json=.providers.json \
  -n mtv-tests
```

**Option B — Include cluster credentials in Secret** (recommended):

```bash
oc create namespace mtv-tests
oc create secret generic mtv-test-config \
  --from-file=providers.json=.providers.json \
  --from-literal=cluster_host=https://api.your-cluster.com:6443 \
  --from-literal=cluster_username=kubeadmin \
  --from-literal=cluster_password=your-cluster-password \
  -n mtv-tests
```

> **Warning:** Option A exposes credentials in plaintext via `oc get job -o yaml`, CI logs, and etcd backups. Always prefer Option B for production and shared environments.

### Step 2: Create and Apply the Job

Use this template, replacing the placeholders:

- `[JOB_NAME]` — Unique job name (e.g., `mtv-tier0-tests`)
- `[TEST_MARKERS]` — Pytest marker(s) (e.g., `tier0`, `warm`, `copyoffload`)
- `[SOURCE_PROVIDER]` — Key from `.providers.json` (e.g., `vsphere-8.0.3.00400`)
- `[STORAGE_CLASS]` — OpenShift storage class name

```bash
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: [JOB_NAME]
  namespace: mtv-tests
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: tests
        image: ghcr.io/redhatqe/mtv-api-tests:latest
        env:
        - name: CLUSTER_HOST
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_host
              optional: true
        - name: CLUSTER_USERNAME
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_username
              optional: true
        - name: CLUSTER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_password
              optional: true
        command:
          - /bin/bash
          - -c
          - |
            uv run pytest -m [TEST_MARKERS] \
              -v \
              ${CLUSTER_HOST:+--tc=cluster_host:${CLUSTER_HOST}} \
              ${CLUSTER_USERNAME:+--tc=cluster_username:${CLUSTER_USERNAME}} \
              ${CLUSTER_PASSWORD:+--tc=cluster_password:${CLUSTER_PASSWORD}} \
              --tc=source_provider:[SOURCE_PROVIDER] \
              --tc=storage_class:[STORAGE_CLASS]
        volumeMounts:
        - name: config
          mountPath: /app/.providers.json
          subPath: providers.json
      volumes:
      - name: config
        secret:
          secretName: mtv-test-config
EOF
```

The `${CLUSTER_HOST:+--tc=cluster_host:${CLUSTER_HOST}}` syntax conditionally includes the flag only when the environment variable is set, making the Job work with both Secret options.

> **Tip:** To run a specific test, add `-k [TEST_FILTER]` to the command. For example: `-k test_copyoffload_thin_migration`.

### Step 3: Monitor and Retrieve Results

**Follow logs in real time:**

```bash
oc logs -n mtv-tests job/[JOB_NAME] -f
```

**Check Job status:**

```bash
oc get jobs -n mtv-tests
# COMPLETIONS: 1/1 = success, 0/1 = still running or failed
```

**Retrieve the JUnit XML report:**

```bash
POD_NAME=$(oc get pods -n mtv-tests -l job-name=[JOB_NAME] \
  -o jsonpath='{.items[0].metadata.name}')
oc cp mtv-tests/$POD_NAME:/app/junit-report.xml ./junit-report.xml
```

**Clean up:**

```bash
oc delete job [JOB_NAME] -n mtv-tests
```

### Example: Run Tier0 Tests as a Job

```bash
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: mtv-tier0-tests
  namespace: mtv-tests
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: tests
        image: ghcr.io/redhatqe/mtv-api-tests:latest
        env:
        - name: CLUSTER_HOST
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_host
              optional: true
        - name: CLUSTER_USERNAME
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_username
              optional: true
        - name: CLUSTER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_password
              optional: true
        command:
          - /bin/bash
          - -c
          - |
            uv run pytest -m tier0 \
              -v \
              ${CLUSTER_HOST:+--tc=cluster_host:${CLUSTER_HOST}} \
              ${CLUSTER_USERNAME:+--tc=cluster_username:${CLUSTER_USERNAME}} \
              ${CLUSTER_PASSWORD:+--tc=cluster_password:${CLUSTER_PASSWORD}} \
              --tc=source_provider:vsphere-8.0.3.00400 \
              --tc=storage_class:rhosqe-ontap-san-block
        volumeMounts:
        - name: config
          mountPath: /app/.providers.json
          subPath: providers.json
      volumes:
      - name: config
        secret:
          secretName: mtv-test-config
EOF
```

---

## Session Lifecycle

Regardless of execution mode, every test session follows the same lifecycle managed by pytest hooks in `conftest.py`.

### Session Start (`pytest_sessionstart`)

1. **Validates required configuration** — `storage_class` and `source_provider` must be set. The session exits immediately with returncode 1 if either is missing.
2. **Initializes the fixture store** — Creates the `teardown` dictionary used to track all created resources.
3. **Prepares the data collector** — Creates the `.data-collector/` directory for resource tracking (unless `--skip-data-collector`).
4. **Configures logging** — Sets up file and console logging to `pytest-tests.log`.
5. **Initializes AI analysis** — Loads `.env` file and validates JJI server configuration (if `--analyze-with-ai`).

### Test Collection (`pytest_collection_modifyitems`)

Test names are modified to include the source provider and storage class, making results distinguishable when running against different providers:

```
test_create_storagemap → test_create_storagemap-vsphere-8.0.1-standard-csi
```

### Session Finish (`pytest_sessionfinish`)

1. **Collects created resources** — Writes `resources.json` to the data collector path.
2. **Runs teardown** — Deletes all created resources (Plans, Providers, VMs, Namespaces, etc.) unless `--skip-teardown` is set.
3. **Runs must-gather on failure** — If teardown fails, automatically runs `oc adm must-gather` to collect MTV diagnostic data.
4. **Enriches JUnit XML** — If `--analyze-with-ai` is set and tests failed, sends the report to JJI for AI-powered failure classification.

---

## Resource Cleanup

### Automatic Cleanup

By default, the test suite automatically deletes all created resources at session end. The teardown order ensures proper dependency resolution — Migrations and Plans are cancelled/archived first, then dependent resources are deleted, and Namespaces are deleted last.

### Manual Cleanup

If tests fail or `--skip-teardown` was used, clean up using the resource tracker:

```bash
uv run tools/clean_cluster.py .data-collector/resources.json
```

This reads the tracked resources from `resources.json` and calls `.clean_up()` on each one:

```python
def clean_cluster_by_resources_file(resources_file: str) -> None:
    with open(resources_file, "r") as fd:
        data: dict[str, list[dict[str, str]]] = json.load(fd)

    for _resource_kind, _resources_list in data.items():
        for _resource in _resources_list:
            _resource_module = importlib.import_module(_resource["module"])
            _resource_class = getattr(_resource_module, _resource_kind)
            _kwargs = {"name": _resource["name"]}
            if _resource.get("namespace"):
                _kwargs["namespace"] = _resource["namespace"]

            _resource_class(**_kwargs).clean_up()
```

If the data collector was disabled, fall back to manual `oc delete` commands:

```bash
oc delete vm --all -n <test-namespace>
oc delete plan --all -n openshift-mtv
oc delete provider <provider-name> -n openshift-mtv
```

---

## AI-Powered Failure Analysis

The `--analyze-with-ai` flag integrates with [Jenkins Job Insight](https://github.com/myk-org/jenkins-job-insight) (JJI) to classify test failures and suggest fixes.

### Configuration

Set environment variables directly or in a `.env` file at the project root:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JJI_SERVER_URL` | Yes | — | JJI server URL |
| `JJI_AI_PROVIDER` | No | `claude` | AI provider name |
| `JJI_AI_MODEL` | No | `claude-opus-4-6[1m]` | AI model identifier |
| `JJI_TIMEOUT` | No | `600` | Request timeout in seconds |

### Usage

```bash
# Local
uv run pytest -m tier0 -v --analyze-with-ai \
  --tc=source_provider:vsphere-8.0.1 \
  --tc=storage_class:YOUR-STORAGE-CLASS

# Container
podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -e JJI_SERVER_URL=https://jji.example.com \
  -e CLUSTER_PASSWORD \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v --analyze-with-ai \
    --tc=cluster_host:https://api.cluster.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:YOUR-STORAGE-CLASS
```

### Behavior

- Analysis runs only when tests fail (exit code != 0)
- Skipped automatically during `--collect-only` and `--setup-plan` dry runs
- If `JJI_SERVER_URL` is not set, the feature is disabled with a warning
- The original JUnit XML is preserved if enrichment fails
- Results are injected into the JUnit XML as structured properties (classification, details, suggested fixes)

---

## Quick Reference

| Task | Command |
|------|---------|
| Install dependencies | `uv sync` |
| Run tier0 tests locally | `uv run pytest -m tier0 -v --tc=source_provider:... --tc=storage_class:...` |
| Run tests in container | `podman run --rm -v .providers.json:/app/.providers.json:ro ghcr.io/redhatqe/mtv-api-tests:latest uv run pytest -m tier0 ...` |
| Build custom image | `podman build -f Dockerfile -t mtv-api-tests .` |
| List available tests | `uv run pytest --collect-only -q` |
| Validate test structure | `tox -e pytest-check` |
| Debug with verbose output | Add `-s -vv --skip-teardown` |
| Enable API debug logging | Set `OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL=DEBUG` |
| Clean up leftover resources | `uv run tools/clean_cluster.py .data-collector/resources.json` |
| Retrieve Job results | `oc cp mtv-tests/$POD_NAME:/app/junit-report.xml ./junit-report.xml` |
