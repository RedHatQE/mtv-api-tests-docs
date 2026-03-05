# Environment Variables

Reference for all environment variables used in mtv-api-tests. Variables are grouped by function: cluster authentication, copy-offload credentials, AI-powered analysis, and runtime configuration.

## Cluster Authentication

Cluster credentials are passed to pytest via `--tc` (test config) flags, not read directly from environment variables. However, using environment variables to hold sensitive values avoids exposing them in shell history.

| Variable | Description |
| --- | --- |
| `CLUSTER_PASSWORD` | OpenShift cluster password |
| `CLUSTER_HOST` | OpenShift API endpoint (e.g., `https://api.your-cluster.com:6443`) |
| `CLUSTER_USERNAME` | OpenShift username (e.g., `kubeadmin`) |

These variables are expanded into `--tc` flags at the command line:

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

When running as an OpenShift Job, inject them from a Secret:

```yaml
env:
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
    uv run pytest -m tier0 -v \
      ${CLUSTER_HOST:+--tc=cluster_host:${CLUSTER_HOST}} \
      ${CLUSTER_USERNAME:+--tc=cluster_username:${CLUSTER_USERNAME}} \
      ${CLUSTER_PASSWORD:+--tc=cluster_password:${CLUSTER_PASSWORD}} \
      --tc=source_provider:vsphere-8.0.1 \
      --tc=storage_class:YOUR-STORAGE-CLASS
```

> **Note:** The `${VAR:+--tc=key:${VAR}}` syntax conditionally adds the flag only when the variable is set, allowing the values to fall back to configuration defaults.

> **Warning:** While environment variables avoid shell-history exposure, expanded values can still appear in process listings. For production CI/CD, prefer OpenShift Secrets or other secret injection mechanisms.

Internally, these values are read from pytest's test config (`py_config`) by `get_cluster_client()` in `utilities/utils.py`:

```python
def get_cluster_client() -> DynamicClient:
    host = get_value_from_py_config("cluster_host")
    username = get_value_from_py_config("cluster_username")
    password = get_value_from_py_config("cluster_password")
    insecure_verify_skip = get_value_from_py_config("insecure_verify_skip")
    _client = get_client(host=host, username=username, password=password, verify_ssl=not insecure_verify_skip)
    ...
```

---

## Copy-Offload Credentials (`COPYOFFLOAD_*`)

Copy-offload (XCOPY) tests require storage array credentials. These can be provided via environment variables or the `copyoffload` section of `.providers.json`. **Environment variables take precedence** over config file values.

Variable names are constructed dynamically from credential names using the pattern `COPYOFFLOAD_{CREDENTIAL_NAME_UPPER}`, as implemented in `utilities/copyoffload_migration.py`:

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
    env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
    return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

### Required Credentials (All Vendors)

These three variables are required for any copy-offload migration:

| Variable | Description |
| --- | --- |
| `COPYOFFLOAD_STORAGE_HOSTNAME` | Storage array hostname or IP address |
| `COPYOFFLOAD_STORAGE_USERNAME` | Storage array authentication username |
| `COPYOFFLOAD_STORAGE_PASSWORD` | Storage array authentication password |

Validation occurs in the `copyoffload_storage_secret` fixture (`conftest.py`):

```python
storage_hostname = get_copyoffload_credential("storage_hostname", copyoffload_cfg)
storage_username = get_copyoffload_credential("storage_username", copyoffload_cfg)
storage_password = get_copyoffload_credential("storage_password", copyoffload_cfg)

if not all([storage_hostname, storage_username, storage_password]):
    raise ValueError(
        "Storage credentials are required. Set COPYOFFLOAD_STORAGE_HOSTNAME, COPYOFFLOAD_STORAGE_USERNAME, "
        "and COPYOFFLOAD_STORAGE_PASSWORD environment variables or include them in .providers.json"
    )
```

### Vendor-Specific Credentials

Additional variables are required depending on the `storage_vendor_product` value in your copy-offload configuration. Supported vendors are defined in `utilities/copyoffload_constants.py`:

```python
SUPPORTED_VENDORS = (
    "ontap",
    "vantara",
    "primera3par",
    "pureFlashArray",
    "powerflex",
    "powermax",
    "powerstore",
    "infinibox",
    "flashsystem",
)
```

| Variable | Vendor | Description |
| --- | --- | --- |
| `COPYOFFLOAD_ONTAP_SVM` | `ontap` | NetApp ONTAP Storage Virtual Machine name |
| `COPYOFFLOAD_VANTARA_STORAGE_ID` | `vantara` | Vantara storage system ID |
| `COPYOFFLOAD_VANTARA_STORAGE_PORT` | `vantara` | Vantara storage port |
| `COPYOFFLOAD_VANTARA_HOSTGROUP_ID_LIST` | `vantara` | Vantara hostgroup ID list |
| `COPYOFFLOAD_PURE_CLUSTER_PREFIX` | `pureFlashArray` | Pure Storage FlashArray cluster prefix |
| `COPYOFFLOAD_POWERFLEX_SYSTEM_ID` | `powerflex` | Dell PowerFlex system ID |
| `COPYOFFLOAD_POWERMAX_SYMMETRIX_ID` | `powermax` | Dell PowerMax Symmetrix ID |

> **Note:** Vendors `primera3par`, `powerstore`, `infinibox`, and `flashsystem` require only the three base credentials (hostname, username, password) with no vendor-specific fields.

### ESXi SSH Clone Credentials

When the copy-offload configuration specifies `esxi_clone_method: ssh`, three additional credentials are required:

| Variable | Description |
| --- | --- |
| `COPYOFFLOAD_ESXI_HOST` | ESXi host address for SSH cloning |
| `COPYOFFLOAD_ESXI_USER` | ESXi SSH username |
| `COPYOFFLOAD_ESXI_PASSWORD` | ESXi SSH password |

These are used in the `setup_copyoffload_ssh` fixture (`conftest.py`):

```python
esxi_host = get_copyoffload_credential("esxi_host", copyoffload_cfg)
esxi_user = get_copyoffload_credential("esxi_user", copyoffload_cfg)
esxi_password = get_copyoffload_credential("esxi_password", copyoffload_cfg)

if not all([esxi_host, esxi_user, esxi_password]):
    pytest.fail(
        "esxi_host, esxi_user, and esxi_password are required in the 'copyoffload' "
        "section of provider config for SSH method."
    )
```

### Equivalent `.providers.json` Configuration

All `COPYOFFLOAD_*` variables can alternatively be set in `.providers.json` under the provider's `copyoffload` section:

```json
{
  "vsphere-8.0.3": {
    "type": "vsphere",
    "copyoffload": {
      "storage_vendor_product": "ontap",
      "datastore_id": "datastore-123",
      "storage_hostname": "storage.example.com",
      "storage_username": "admin",
      "storage_password": "secret",
      "ontap_svm": "svm0",
      "esxi_clone_method": "ssh",
      "esxi_host": "esxi1.example.com",
      "esxi_user": "root",
      "esxi_password": "secret"
    }
  }
}
```

> **Tip:** Use environment variables for credentials that differ between CI/CD runs (e.g., rotating passwords). Use `.providers.json` for stable configuration that doesn't change between runs.

---

## AI Analysis Settings (`JJI_*`)

These variables configure the AI-powered test failure analysis feature, powered by [Jenkins Job Insight (JJI)](https://github.com/myk-org/jenkins-job-insight). The feature enriches JUnit XML reports with AI-generated analysis of test failures.

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `JJI_SERVER_URL` | Yes | — | URL of the JJI server (e.g., `https://jji.example.com`) |
| `JJI_AI_PROVIDER` | No | `claude` | AI provider name sent to the JJI server |
| `JJI_AI_MODEL` | No | `claude-opus-4-6[1m]` | AI model identifier sent to the JJI server |
| `JJI_TIMEOUT` | No | `600` | HTTP request timeout in seconds for the analysis call |

### How It Works

AI analysis is activated by the `--analyze-with-ai` pytest flag. During session startup, `setup_ai_analysis()` in `utilities/pytest_utils.py` loads variables from the environment (including `.env` files via `python-dotenv`) and validates prerequisites:

```python
def setup_ai_analysis(session: pytest.Session) -> None:
    load_dotenv()

    LOGGER.info("Setting up AI-powered test failure analysis")

    if not os.environ.get("JJI_SERVER_URL"):
        LOGGER.warning("JJI_SERVER_URL is not set. Analyze with AI features will be disabled.")
        session.config.option.analyze_with_ai = False
    else:
        if not os.environ.get("JJI_AI_PROVIDER"):
            os.environ["JJI_AI_PROVIDER"] = "claude"

        if not os.environ.get("JJI_AI_MODEL"):
            os.environ["JJI_AI_MODEL"] = "claude-opus-4-6[1m]"
```

At session end, the JUnit XML report is sent to the JJI server for analysis. The server returns an enriched XML with AI-generated failure insights:

```python
timeout_value = int(os.environ.get("JJI_TIMEOUT", "600"))

response = requests.post(
    f"{server_url.rstrip('/')}/analyze-failures",
    json={
        "raw_xml": raw_xml,
        "ai_provider": ai_provider,
        "ai_model": ai_model,
    },
    timeout=timeout_value,
)
```

### Usage

```bash
# Via environment variables
export JJI_SERVER_URL="https://jji.example.com"
export JJI_AI_PROVIDER="claude"
export JJI_AI_MODEL="claude-opus-4-6[1m]"
export JJI_TIMEOUT="900"

uv run pytest -m tier0 -v --analyze-with-ai \
  --tc=source_provider:vsphere-8.0.1 \
  --tc=storage_class:YOUR-STORAGE-CLASS
```

```bash
# Via .env file (loaded automatically with --analyze-with-ai)
cat > .env << 'EOF'
JJI_SERVER_URL=https://jji.example.com
JJI_AI_PROVIDER=claude
JJI_AI_MODEL=claude-opus-4-6[1m]
JJI_TIMEOUT=900
EOF

uv run pytest -m tier0 -v --analyze-with-ai ...
```

> **Note:** The feature requires `--junit-xml` to be enabled, which is set by default in `pytest.ini`. AI analysis is automatically skipped during dry runs (`--collectonly`, `--setupplan`). If `JJI_SERVER_URL` is not set, the feature is silently disabled with a warning.

> **Tip:** If the enrichment request fails (network error, server error, timeout), the original JUnit XML is preserved unchanged.

---

## OpenShift Python Wrapper Logging

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL` | No | (library default) | Log level for the `openshift-python-wrapper` library |

This variable controls the verbosity of the [openshift-python-wrapper](https://github.com/RedHatQE/openshift-python-wrapper) library, which handles all OpenShift API interactions. Setting it to `DEBUG` logs every API call made to the cluster.

It can be set in two ways:

**1. Via pytest flag** (recommended):

```bash
uv run pytest --openshift-python-wrapper-log-debug ...
```

This sets the variable internally in `conftest.py`:

```python
if session.config.getoption("openshift_python_wrapper_log_debug"):
    os.environ["OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL"] = "DEBUG"
```

**2. Via environment variable directly** (useful in containers):

```bash
podman run --rm \
  -e OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL=DEBUG \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v ...
```

> **Tip:** Use `DEBUG` level when you need to see the exact API requests and responses to/from the OpenShift cluster. This is particularly useful for diagnosing migration failures or resource creation issues.

---

## Container Build-Time Variables

These variables are set in the `Dockerfile` and affect the container runtime environment. They are not typically set by users.

| Variable | Value | Description |
| --- | --- | --- |
| `JUNITFILE` | `${APP_DIR}/output/` | Directory for JUnit XML output inside the container |
| `UV_PYTHON` | `python3.12` | Python version used by `uv` |
| `UV_COMPILE_BYTECODE` | `1` | Enables bytecode compilation for faster startup |
| `UV_NO_SYNC` | `1` | Prevents automatic `uv sync` on every `uv run` |
| `UV_CACHE_DIR` | `${APP_DIR}/.cache` | Cache directory for `uv` (cleaned during build) |

```dockerfile
FROM quay.io/fedora/fedora:41

ARG APP_DIR=/app

ENV JUNITFILE=${APP_DIR}/output/
ENV UV_PYTHON=python3.12
ENV UV_COMPILE_BYTECODE=1
ENV UV_NO_SYNC=1
ENV UV_CACHE_DIR=${APP_DIR}/.cache
```

> **Note:** `OPENSHIFT_PYTHON_WRAPPER_COMMIT` and `OPENSHIFT_PYTHON_UTILITIES_COMMIT` are build-time arguments (`ARG`), not runtime environment variables. They allow pinning specific commits of upstream dependencies during image builds.

---

## `.env` File Support

The project supports a `.env` file in the repository root for convenient local development. This file is loaded by `python-dotenv` when the `--analyze-with-ai` flag is used.

The `.env` file is listed in `.gitignore` and should never be committed.

Example `.env` file:

```bash
# AI Analysis
JJI_SERVER_URL=https://jji.example.com
JJI_AI_PROVIDER=claude
JJI_AI_MODEL=claude-opus-4-6[1m]
JJI_TIMEOUT=600

# Copy-offload credentials (override .providers.json)
COPYOFFLOAD_STORAGE_HOSTNAME=storage.example.com
COPYOFFLOAD_STORAGE_USERNAME=admin
COPYOFFLOAD_STORAGE_PASSWORD=secret
COPYOFFLOAD_ONTAP_SVM=svm0
```

> **Warning:** Never commit a `.env` file containing real credentials. The `.gitignore` already excludes `.env` files, but always verify before committing.

---

## Quick Reference

| Variable | Context | Required | Default |
| --- | --- | --- | --- |
| `CLUSTER_PASSWORD` | Shell (passed via `--tc`) | Yes | — |
| `CLUSTER_HOST` | Shell (passed via `--tc`) | Yes | — |
| `CLUSTER_USERNAME` | Shell (passed via `--tc`) | Yes | — |
| `COPYOFFLOAD_STORAGE_HOSTNAME` | Copy-offload tests | Yes | — |
| `COPYOFFLOAD_STORAGE_USERNAME` | Copy-offload tests | Yes | — |
| `COPYOFFLOAD_STORAGE_PASSWORD` | Copy-offload tests | Yes | — |
| `COPYOFFLOAD_ONTAP_SVM` | `ontap` vendor | Conditional | — |
| `COPYOFFLOAD_VANTARA_STORAGE_ID` | `vantara` vendor | Conditional | — |
| `COPYOFFLOAD_VANTARA_STORAGE_PORT` | `vantara` vendor | Conditional | — |
| `COPYOFFLOAD_VANTARA_HOSTGROUP_ID_LIST` | `vantara` vendor | Conditional | — |
| `COPYOFFLOAD_PURE_CLUSTER_PREFIX` | `pureFlashArray` vendor | Conditional | — |
| `COPYOFFLOAD_POWERFLEX_SYSTEM_ID` | `powerflex` vendor | Conditional | — |
| `COPYOFFLOAD_POWERMAX_SYMMETRIX_ID` | `powermax` vendor | Conditional | — |
| `COPYOFFLOAD_ESXI_HOST` | SSH clone method | Conditional | — |
| `COPYOFFLOAD_ESXI_USER` | SSH clone method | Conditional | — |
| `COPYOFFLOAD_ESXI_PASSWORD` | SSH clone method | Conditional | — |
| `JJI_SERVER_URL` | `--analyze-with-ai` | Yes | — |
| `JJI_AI_PROVIDER` | `--analyze-with-ai` | No | `claude` |
| `JJI_AI_MODEL` | `--analyze-with-ai` | No | `claude-opus-4-6[1m]` |
| `JJI_TIMEOUT` | `--analyze-with-ai` | No | `600` |
| `OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL` | Runtime | No | (library default) |
| `JUNITFILE` | Container | No | `/app/output/` |
