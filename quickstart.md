# Quick Start

Running your first test collection with podman, understanding the container-based execution model, and interpreting output.

---

## Before You Begin

Make sure you have the following ready:

- **Podman** (or Docker) installed on your local machine
- **An OpenShift cluster** with the MTV (Migration Toolkit for Virtualization) operator installed
- **A source virtualization provider** (VMware vSphere, RHV, OpenStack, or OVA) with at least one VM or template available for migration testing

```bash
# Verify podman is installed
podman --version
```

> **Note:** You can substitute `docker` for `podman` in all commands throughout this guide.

---

## How the Container-Based Execution Model Works

MTV API Tests are designed to run inside a container. This means you do **not** need Python, pytest, or any dependencies installed on your local machine. The container image packages everything needed to execute tests against your cluster.

### What's Inside the Container

The test image is built from a `Dockerfile` based on Fedora 41:

```dockerfile
FROM quay.io/fedora/fedora:41

# Python 3.12 managed by uv (an ultra-fast Python package manager)
ENV UV_PYTHON=python3.12

# All test code is copied into /app
WORKDIR /app
COPY utilities tests libs exceptions ./
COPY pyproject.toml uv.lock conftest.py pytest.ini ./

# Dependencies are installed from the lock file
RUN uv sync --locked

# Default command: list available tests (does not execute them)
CMD ["uv", "run", "pytest", "--collect-only"]
```

The container holds the full test framework — pytest with plugins, provider client libraries (pyvmomi, ovirt-sdk, openstacksdk), the OpenShift Python wrapper, and all utilities. When you run the container, you override the default `CMD` with a `uv run pytest` command that targets your cluster.

### Execution Flow

```
┌──────────────────────────────────────────────────────┐
│  Your Machine                                        │
│                                                      │
│  podman run                                          │
│    -v .providers.json:/app/.providers.json   ◄── credentials mount
│    -e CLUSTER_PASSWORD                       ◄── env variable
│    ghcr.io/redhatqe/mtv-api-tests:latest     ◄── test image
│    uv run pytest -m tier0 --tc=...           ◄── test command
│                                                      │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│  Container (/app)                                    │
│                                                      │
│  1. pytest session starts                            │
│  2. Validates required config (storage_class,        │
│     source_provider)                                 │
│  3. Connects to OpenShift cluster via API            │
│  4. Connects to source provider (.providers.json)    │
│  5. Creates namespace, maps, plans                   │
│  6. Executes VM migrations                           │
│  7. Validates migrated VMs                           │
│  8. Tears down created resources                     │
│  9. Writes junit-report.xml                          │
│                                                      │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│  OpenShift Cluster                                   │
│                                                      │
│  MTV operator creates:                               │
│  - Provider CRs                                      │
│  - StorageMap / NetworkMap                            │
│  - Plan CR                                           │
│  - Migration CR → migrates VMs to OpenShift          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

Key points about this model:

- **No local Python required** — everything runs inside the container
- **Credentials stay local** — `.providers.json` is bind-mounted read-only into the container
- **Cluster access via API** — the container connects to your OpenShift API endpoint over the network
- **Automatic cleanup** — resources created during tests are torn down at session end (unless `--skip-teardown` is used)

---

## Step 1: Get the Test Image

You have two options:

**Option A: Use the pre-built public image** (recommended for first run):

```bash
# The image is ready to use — no build needed
# ghcr.io/redhatqe/mtv-api-tests:latest
```

**Option B: Build your own image**:

```bash
git clone https://github.com/RedHatQE/mtv-api-tests.git
cd mtv-api-tests
podman build -f Dockerfile -t mtv-api-tests .
```

---

## Step 2: Create Your Provider Configuration

The `.providers.json` file tells the test suite how to connect to your source virtualization platform. Create this file in your current working directory.

**VMware vSphere example:**

```json
{
  "vsphere-8.0.1": {
    "type": "vsphere",
    "version": "8.0.1",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "your-password",
    "guest_vm_linux_user": "root",
    "guest_vm_linux_password": "your-vm-password"
  }
}
```

The configuration key (e.g., `vsphere-8.0.1`) follows the pattern `{type}-{version}` and is what you pass as `--tc=source_provider:vsphere-8.0.1` when running tests.

> **Warning:** This file contains sensitive credentials. Set restrictive permissions (`chmod 600 .providers.json`), never commit it to git, and delete it when no longer needed.

For RHV, OpenStack, OVA, or copy-offload setups, see the `.providers.json.example` file in the repository for complete templates of all supported providers.

---

## Step 3: Find Your Storage Class

Tests need to know which OpenShift storage class to use for migrated VM disks. Check what's available on your cluster:

```bash
oc get storageclass
```

Choose a storage class that supports block storage (e.g., `ocs-storagecluster-ceph-rbd`, `ontap-san-block`). You'll use this value in the next step.

---

## Step 4: Run Your First Test Collection

The `tier0` marker selects the smoke tests — the smallest, fastest collection that validates core migration functionality. This is the best starting point.

```bash
# Set cluster password as environment variable (avoids shell history exposure)
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
    --tc=storage_class:ocs-storagecluster-ceph-rbd
```

**Replace these values:**

| Placeholder | Replace With |
|---|---|
| `https://api.your-cluster.com:6443` | Your OpenShift API URL |
| `kubeadmin` | Your cluster username |
| `your-cluster-password` | Your cluster password |
| `vsphere-8.0.1` | The key from your `.providers.json` |
| `ocs-storagecluster-ceph-rbd` | Your storage class from Step 3 |

> **Note:** On RHEL/Fedora with SELinux, add `,z` to the volume mount: `-v $(pwd)/.providers.json:/app/.providers.json:ro,z`

> **Tip:** To see what tests would run without actually executing them, replace the pytest command with the container's default: `podman run --rm ghcr.io/redhatqe/mtv-api-tests:latest` — this runs `pytest --collect-only` and lists all available tests.

### Understanding the `--tc` Parameters

Test configuration is passed via `--tc` flags (from the `pytest-testconfig` plugin). These map to values in the test configuration module at `tests/tests_config/config.py`:

| Parameter | Required | Description |
|---|---|---|
| `cluster_host` | Yes | OpenShift API server URL |
| `cluster_username` | Yes | Cluster login username |
| `cluster_password` | Yes | Cluster login password |
| `source_provider` | Yes | Key from `.providers.json` (e.g., `vsphere-8.0.1`) |
| `storage_class` | Yes | OpenShift storage class name |

The test framework validates that `source_provider` and `storage_class` are present at session start. If either is missing, pytest exits immediately with a clear error:

```python
# From conftest.py — session-start validation
required_config = ("storage_class", "source_provider")
if missing_configs:
    pytest.exit(reason=f"Some required config is missing {required_config=} - {missing_configs=}", returncode=1)
```

---

## Step 5: Understand What the Tests Do

Each test class follows a 5-step incremental pattern. Here's what `TestSanityColdMtvMigration` (the tier0 smoke test) does:

```python
@pytest.mark.tier0
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_sanity_cold_mtv_migration"])],
    indirect=True,
    ids=["rhel8"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestSanityColdMtvMigration:
    """Cold migration test - sanity check."""

    def test_create_storagemap(self, ...):    # Step 1: Map source → target storage
    def test_create_networkmap(self, ...):    # Step 2: Map source → target networks
    def test_create_plan(self, ...):          # Step 3: Create the migration plan
    def test_migrate_vms(self, ...):          # Step 4: Execute the migration
    def test_check_vms(self, ...):            # Step 5: Validate migrated VMs
```

The `@pytest.mark.incremental` marker means each step depends on the previous one. If `test_create_storagemap` fails, subsequent steps are automatically marked as `xfail` (expected failure) and skipped.

The test configuration for this smoke test is defined in `tests/tests_config/config.py`:

```python
"test_sanity_cold_mtv_migration": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "guest_agent": True},
    ],
    "warm_migration": False,
},
```

This tells the framework to migrate a single RHEL 8 VM with the QEMU guest agent installed, using cold migration (VM powered off during transfer).

---

## Interpreting the Output

### Live Output During Execution

The test suite streams output to your terminal in real time. Here's what to expect:

**Session startup** — validates configuration and connects to the cluster:

```
------------ SESSION START ------------
Executing session fixture: ocp_admin_client
Executing session fixture: source_provider
Executing session fixture: target_namespace
```

**Test execution** — each test method logs its phase:

```
------------ test_create_storagemap ------------
------------ SETUP ------------
Executing class fixture: prepared_plan
------------ CALL ------------

TEST: test_create_storagemap STATUS: PASSED

------------ test_create_networkmap ------------
------------ SETUP ------------
------------ CALL ------------

TEST: test_create_networkmap STATUS: PASSED
```

**Status indicators** — color-coded results:

| Output | Meaning |
|---|---|
| `STATUS: PASSED` (green) | Test step completed successfully |
| `STATUS: FAILED` (red) | Test step failed — check the error traceback |
| `STATUS: SKIPPED` (yellow) | Test was skipped (e.g., missing prerequisite) |
| `STATUS: ERROR` (red, with `[setup]` or `[teardown]`) | Failure in fixture setup or teardown, not the test itself |

**Session finish** — cleanup and summary:

```
------------ SESSION FINISH ------------
short test summary info
========= 5 passed in 342.18s (0:05:42) =========
```

### JUnit XML Report

Tests automatically produce a JUnit XML report at `/app/junit-report.xml` inside the container. To persist it, mount a volume:

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
    --tc=storage_class:ocs-storagecluster-ceph-rbd
```

The report is saved to `./results/junit-report.xml` on your host machine and can be consumed by CI/CD dashboards (Jenkins, GitLab CI, GitHub Actions) that parse JUnit XML.

### Data Collector

When a test fails, the framework automatically collects debug data into `.data-collector/` inside the container. This includes:

- **`resources.json`** — a manifest of all OpenShift resources created during the session
- **Per-test directories** — `must-gather` output for failed tests

To access this data, mount the directory:

```bash
podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -v $(pwd)/.data-collector:/app/.data-collector \
  -e CLUSTER_PASSWORD \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v \
    --tc=cluster_host:https://api.your-cluster.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:ocs-storagecluster-ceph-rbd
```

> **Tip:** If you don't need resource tracking (e.g., quick iteration), pass `--skip-data-collector` to skip this overhead.

---

## Choosing a Test Collection

The `tier0` smoke tests are a good starting point, but the suite includes several marker-based collections:

| Marker | Description | Command |
|---|---|---|
| `tier0` | Core smoke tests — fastest, validates critical paths | `-m tier0` |
| `warm` | Warm migration — VMs stay running during transfer | `-m warm` |
| `copyoffload` | Accelerated migration via shared storage (XCOPY) | `-m copyoffload` |
| `remote` | Migration to a remote OpenShift cluster | `-m remote` |

Combine markers with standard pytest expressions:

```bash
# Run both tier0 and warm tests
uv run pytest -m "tier0 or warm" -v --tc=...

# Run only a specific test by name
uv run pytest -k test_sanity_cold_mtv_migration -v --tc=...
```

To list all available tests without executing them:

```bash
podman run --rm ghcr.io/redhatqe/mtv-api-tests:latest uv run pytest --collect-only -q
```

---

## Debugging a Failed Run

When something goes wrong, use these flags to get more information:

```bash
podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -v $(pwd)/.data-collector:/app/.data-collector \
  -e CLUSTER_PASSWORD \
  -e OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL=DEBUG \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -s -vv --skip-teardown \
    --tc=cluster_host:https://api.your-cluster.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:ocs-storagecluster-ceph-rbd
```

| Flag | Effect |
|---|---|
| `-s -vv` | Maximum verbosity with full output capture disabled |
| `--skip-teardown` | Keep all created resources (VMs, plans, maps) after tests finish for manual inspection |
| `--skip-data-collector` | Disable resource tracking for faster iteration |
| `-e OPENSHIFT_PYTHON_WRAPPER_LOG_LEVEL=DEBUG` | Log every OpenShift API call made by the test framework |

> **Warning:** When using `--skip-teardown`, resources remain on your cluster after the test run. Clean them up manually or use the included cleanup tool: `uv run tools/clean_cluster.py .data-collector/resources.json`

---

## Common First-Run Issues

**"Some required config is missing"**

You forgot a required `--tc` parameter. Both `source_provider` and `storage_class` are mandatory:

```bash
# Both of these are required
--tc=source_provider:vsphere-8.0.1
--tc=storage_class:ocs-storagecluster-ceph-rbd
```

**"pytest: command not found"**

Always use `uv run pytest`, not bare `pytest`:

```bash
# Correct
podman run ... uv run pytest -m tier0 ...

# Wrong — pytest is not on PATH
podman run ... pytest -m tier0 ...
```

**Provider connection failures**

Verify that your cluster can reach the source provider's API endpoint and that the credentials in `.providers.json` are correct:

```bash
# Check connectivity from within the cluster
oc run test-curl --rm -it --image=curlimages/curl -- curl -k https://vcenter.example.com

# Verify your provider key
cat .providers.json | jq 'keys'
```

**Authentication failures**

Ensure the user you're authenticating as has sufficient permissions to create MTV resources:

```bash
oc whoami
oc auth can-i create virtualmachines
```

For test/development clusters, the simplest (though least secure) option is cluster-admin:

```bash
oc adm policy add-cluster-role-to-user cluster-admin $(oc whoami)
```

> **Warning:** Granting cluster-admin is only appropriate for isolated test/development clusters. For production or shared environments, create a dedicated role scoped to MTV resources.

---

## Next Steps

- **Run as an OpenShift Job** — for long-running suites or CI/CD integration, deploy tests as a Kubernetes Job with credentials stored in OpenShift Secrets
- **Copy-offload testing** — if your environment has shared storage between vSphere and OpenShift, explore the `copyoffload` test suite for accelerated migration validation
- **AI-powered failure analysis** — use the `--analyze-with-ai` flag with a Jenkins Job Insight server to get automated root-cause analysis for test failures
