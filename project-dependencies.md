# Project Dependencies

This page documents the key third-party dependencies used in the MTV API Tests project, their roles within the test framework, and how they are used in practice.

## Dependency Overview

| Package | Version | Purpose |
|---------|---------|---------|
| `openshift-python-wrapper` | >= 11.0.53 | OpenShift/Kubernetes resource management |
| `pyvmomi` | >= 8.0.3.0.1 | VMware vSphere API interaction |
| `ovirt-python-sdk` | >= 1.2.0 | Red Hat Virtualization (oVirt) API interaction |
| `openstacksdk` | >= 4.0.0 | OpenStack API interaction |
| `pytest-xdist` | >= 3.6.1 | Parallel test execution |
| `pytest-testconfig` | >= 0.2.0 | Global test configuration management |
| `python-rrmngmnt` | >= 0.2.0 | SSH connection management for remote hosts |
| `shortuuid` | >= 1.0.13 | Short unique identifier generation |
| `jc` | >= 1.25.0 | Structured parsing of CLI command output |

All dependencies are declared in `pyproject.toml` and managed with `uv`.

---

## openshift-python-wrapper

**Package:** `openshift-python-wrapper>=11.0.53`

The foundational dependency of the project. Provides Pythonic wrappers around OpenShift and Kubernetes resources, enabling the test framework to create, manage, and query cluster resources without using the `kubernetes` client library directly.

> **Warning:** Direct use of the `kubernetes` package at runtime is forbidden. All cluster interactions must go through `openshift-python-wrapper`. See the [Code Standards](./code-standards.md) for details.

### Resource Classes

The project imports resource classes from the `ocp_resources` module for every type of OpenShift/Kubernetes object it interacts with:

```python
# conftest.py
from ocp_resources.forklift_controller import ForkliftController
from ocp_resources.namespace import Namespace
from ocp_resources.network_attachment_definition import NetworkAttachmentDefinition
from ocp_resources.node import Node
from ocp_resources.pod import Pod
from ocp_resources.provider import Provider
from ocp_resources.resource import ResourceEditor
from ocp_resources.secret import Secret
from ocp_resources.storage_class import StorageClass
from ocp_resources.storage_profile import StorageProfile
from ocp_resources.virtual_machine import VirtualMachine
```

### Resource Lifecycle Management

All OpenShift resources are created through `create_and_store_resource()`, which wraps the wrapper's `deploy()` and `wait()` methods and tracks resources for teardown:

```python
# utilities/resources.py
from ocp_resources.namespace import Namespace
from ocp_resources.resource import Resource

def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    # ...
    _resource = resource(**kwargs)

    try:
        _resource.deploy(wait=True)
    except ConflictError:
        LOGGER.warning(f"{_resource.kind} {_resource_name} already exists, reusing it.")
        _resource.wait()

    fixture_store["teardown"].setdefault(_resource.kind, []).append(_resource_dict)
    return _resource
```

### Companion Package: openshift-python-utilities

The sibling package `openshift-python-utilities>=6.0.7` provides additional cluster utilities such as Prometheus monitoring support:

```python
# utilities/worker_node_selection.py (TYPE_CHECKING import)
from ocp_utilities.monitoring import Prometheus
```

---

## pyvmomi

**Package:** `pyvmomi>=8.0.3.0.1`

The VMware vSphere API Python SDK. Used exclusively within the VMware provider to manage VMs, datastores, networks, and perform clone operations against a vCenter server.

### Connection

```python
# libs/providers/vmware.py
from pyVim.connect import Disconnect, SmartConnect
from pyVmomi import vim

class VMWareProvider(BaseProvider):
    def connect(self) -> Self:
        self.api = SmartConnect(
            host=self.host,
            user=self.username,
            pwd=self.password,
            port=443,
            disableSslCertValidation=True,
        )
        return self

    def disconnect(self) -> None:
        Disconnect(si=self.api)
```

### VM Operations

The `vim` module provides type objects for all vSphere API interactions, including VM lookups, datastore queries, disk management, and task monitoring:

```python
# libs/providers/vmware.py
@property
def content(self) -> vim.ServiceInstanceContent:
    return self.api.RetrieveContent()

def get_vm_by_name(self, query: str, ...) -> vim.VirtualMachine:
    target_vm = self.get_obj(vimtype=[vim.VirtualMachine], name=target_vm_name)
    # ...

def get_datastore_name_by_id(self, datastore_id: str) -> str:
    datastore = self.get_obj([vim.Datastore], datastore_id)
    if not datastore:
        raise VmBadDatastoreError(f"Datastore with ID '{datastore_id}' not found.")
    return datastore.name
```

> **Note:** The VMware provider handles VM cloning, snapshot management, network configuration, and disk attachment operations — all through pyvmomi's `vim` object hierarchy.

---

## ovirt-python-sdk

**Package:** `ovirt-python-sdk>=1.2.0`

The oVirt/RHV Engine SDK for Python. Provides API access to Red Hat Virtualization environments for managing VMs, disks, networks, and datacenters.

### Connection

```python
# libs/providers/rhv.py
import ovirtsdk4
from ovirtsdk4 import NotFoundError, types
from ovirtsdk4.types import VmStatus

class OvirtProvider(BaseProvider):
    def connect(self) -> Self:
        self.api = ovirtsdk4.Connection(
            url=self.host,
            username=self.username,
            password=self.password,
            ca_file=self.ca_file if not self.insecure else None,
            debug=self.debug,
            log=self.log,
            insecure=self.insecure,
        )
        if not self.is_mtv_datacenter_ok:
            raise OvirtMTVDatacenterStatusError()
        return self
```

### Service Access

The provider exposes oVirt system services as properties for clean access to VMs, disks, networks, storage, and templates:

```python
# libs/providers/rhv.py
@property
def vms_services(self) -> ovirtsdk4.services.VmsService:
    return self.api.system_service().vms_service()

@property
def disks_service(self) -> ovirtsdk4.services.DisksService:
    return self.api.system_service().disks_service()

@property
def network_services(self) -> ovirtsdk4.services.NetworksService:
    # ...
```

---

## openstacksdk

**Package:** `openstacksdk>=4.0.0`

The official OpenStack SDK for Python. Used within the OpenStack provider to manage compute instances, volumes, networks, and images.

### Connection

```python
# libs/providers/openstack.py
from openstack import exceptions as os_exc
from openstack.compute.v2.server import Server as OSP_Server
from openstack.connection import Connection
from openstack.image.v2.image import Image as OSP_Image

class OpenStackProvider(BaseProvider):
    def connect(self) -> Self:
        self.api = Connection(
            auth_url=self.auth_url,
            project_name=self.project_name,
            username=self.username,
            password=self.password,
            user_domain_name=self.user_domain_name,
            region_name=self.region_name,
            user_domain_id=self.user_domain_id,
            project_domain_id=self.project_domain_id,
        )
        return self
```

### Compute and Storage Operations

```python
# libs/providers/openstack.py
def get_instance_id_by_name(self, name_filter: str) -> str:
    instance_id = ""
    for server in self.api.compute.servers(details=True):
        if server.name == name_filter:
            instance_id = server.id
            break
    return instance_id
```

> **Note:** The OpenStack provider accesses `compute`, `block_storage`, `network`, and `image` services through the unified `Connection` object.

---

## pytest-xdist

**Package:** `pytest-xdist>=3.6.1`

Enables parallel test execution across multiple worker processes. This is critical for reducing total test execution time when running large migration test suites.

### Worker Coordination

The project implements custom xdist hooks for safe cross-worker data sharing using pickle serialization and file locking:

```python
# conftest.py
RESULTS_PATH = Path("./.xdist_results/")
RESULTS_PATH.mkdir(exist_ok=True)

def pytest_harvest_xdist_init():
    """Reset the recipient folder at the start of a session."""
    if RESULTS_PATH.exists():
        rmtree(RESULTS_PATH)
    RESULTS_PATH.mkdir(exist_ok=False)
    return True

def pytest_harvest_xdist_worker_dump(worker_id, session_items, fixture_store):
    """Persist each worker's results to disk."""
    with open(RESULTS_PATH / (f"{worker_id}.pkl"), "wb") as f:
        try:
            pickle.dump((session_items, fixture_store), f)
        except Exception as exp:
            LOGGER.warning(
                f"Error while pickling worker {worker_id}'s harvested results: [{exp.__class__}] {exp}"
            )
    return True

def pytest_harvest_xdist_load():
    """Restore saved objects from the file system."""
    workers_saved_material = {}
    for pkl_file in RESULTS_PATH.glob("*.pkl"):
        wid = pkl_file.stem
        with pkl_file.open("rb") as f:
            workers_saved_material[wid] = pickle.load(f)
    return workers_saved_material
```

### Parallel Safety

Tests are parallel-safe through several mechanisms:

- **Unique namespaces** per session via `session_uuid` (generated with `shortuuid`)
- **Isolated `fixture_store`** per worker process
- **File locking** for shared resources like the `virtctl` binary download:

```python
# conftest.py — virtctl binary download uses file locking for xdist safety
# Uses file locking to handle pytest-xdist parallel execution safely.
# The binary is downloaded to a shared directory that all workers can access.
```

> **Tip:** Tests within a class marked `@pytest.mark.incremental` run sequentially on the same worker, while independent test classes are distributed across workers.

---

## pytest-testconfig

**Package:** `pytest-testconfig>=0.2.0`

Provides a global configuration object (`py_config`) that makes test configuration accessible from any module — fixtures, utilities, and test classes alike.

### Configuration Access

The `py_config` dictionary is imported and used throughout the entire codebase:

```python
from pytest_testconfig import config as py_config

# Test parametrization
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_warm_migration_comprehensive"])],
    indirect=True,
    ids=["warm-migration-comprehensive"],
)
class TestWarmMigrationComprehensive:
    ...
```

### Session Startup Validation

Required configuration keys are validated at session start to fail fast:

```python
# conftest.py
def pytest_sessionstart(session):
    required_config = ("storage_class", "source_provider")

    missing_configs: list[str] = []
    for _req in required_config:
        if not py_config.get(_req):
            missing_configs.append(_req)

    if missing_configs:
        pytest.exit(
            reason=f"Some required config is missing {required_config=} - {missing_configs=}",
            returncode=1,
        )
```

### Common Configuration Patterns

```python
# Direct key access (required config — fail if missing)
storage_class = py_config["storage_class"]
mtv_namespace = py_config["mtv_namespace"]
target_prefix = py_config["target_namespace_prefix"]

# Optional feature flags
if plan.get("warm_migration", False):
    setup_warm_migration()

# Provider-qualified test naming
item.name = f"{item.name}-{py_config.get('source_provider')}-{py_config.get('storage_class')}"
```

> **Note:** `pytest-testconfig` supports loading configuration from Python files passed via the `--tc-file` CLI option at runtime.

---

## python-rrmngmnt

**Package:** `python-rrmngmnt>=0.2.0`

Remote Resource Management library. Provides SSH connectivity and command execution on remote hosts. Used in mtv-api-tests to connect to migrated VMs running on OpenShift/KubeVirt via `virtctl port-forward`.

### SSH Connection Wrapper

The project wraps `rrmngmnt` in a `VMSSHConnection` class that combines `virtctl port-forward` with SSH:

```python
# utilities/ssh_utils.py
from rrmngmnt import Host, RootUser, User, UserWithPKey

class VMSSHConnection:
    def connect(self) -> Host:
        """Establish SSH connection using virtctl port-forward and rrmngmnt."""
        self.rrmngmnt_host = Host("localhost")

        # Configure user authentication
        if self.private_key_path:
            user = UserWithPKey(self.username, self.private_key_path)
        elif self.username == "root":
            user = RootUser(self.password)
        else:
            user = User(self.username, self.password)

        self.rrmngmnt_host.users.append(user)

        # Test connectivity
        executor = self.rrmngmnt_host.executor(user=user)
        executor.port = self.local_port
        rc, out, err = executor.run_cmd(["echo", "test"])
        if rc != 0:
            raise RuntimeError(f"Connection test failed: {err}")

        return self.rrmngmnt_host
```

### Context Manager Support

Connections support the context manager protocol for safe cleanup:

```python
# Usage pattern
with VMSSHConnection(vm=my_vm, username="root", password="pass") as ssh:
    executor = ssh.rrmngmnt_host.executor(user=ssh.rrmngmnt_user)
    executor.port = ssh.local_port
    rc, out, err = executor.run_cmd(["whoami"])
```

### Authentication Methods

| Class | Use Case |
|-------|----------|
| `User(username, password)` | Standard username/password authentication |
| `RootUser(password)` | Root user with password |
| `UserWithPKey(username, key_path)` | SSH key-based authentication |

---

## shortuuid

**Package:** `shortuuid>=1.0.13`

Generates compact, URL-safe UUIDs. Used to create unique, DNS-compliant resource names that avoid collisions across parallel test sessions and workers.

### Name Generation

The primary usage is in `utilities/naming.py`, which generates Kubernetes-safe names with a random suffix:

```python
# utilities/naming.py
import shortuuid

def generate_name_with_uuid(name: str) -> str:
    _name = f"{name}-{shortuuid.ShortUUID().random(length=4).lower()}"
    _name = _name.replace("_", "-").replace(".", "-").lower()
    return _name
```

### Session-Level Isolation

Every test session gets a unique identifier that prefixes all created resources, ensuring no collisions between parallel runs:

```python
# conftest.py
@pytest.fixture(scope="session")
def session_uuid(fixture_store):
    _session_uuid = generate_name_with_uuid(name="auto")
    fixture_store["session_uuid"] = _session_uuid
    return _session_uuid
    # Example output: "auto-k7xm"
```

This `session_uuid` flows into namespace names, resource names, and VM clone names:

```python
# conftest.py — namespace creation
unique_namespace_name = f"{session_uuid}{_target_namespace}"[:63]
```

### VM Cloning

The VMware provider also uses `shortuuid` directly when generating clone names that need to preserve non-standard casing:

```python
# libs/providers/vmware.py
random_suffix = shortuuid.ShortUUID().random(length=4).lower()
prefix = f"{session_uuid}-"
suffix = f"-{random_suffix}"
```

---

## jc

**Package:** `jc>=1.25.0`

A CLI output parser that converts the text output of common system commands into structured Python data. Used in post-migration validation to parse Windows network configuration.

### Windows Network Validation

After migrating Windows VMs, the test framework validates that network configuration (IP addresses, subnet masks, MAC addresses, gateways) was preserved. The `jc` library parses `ipconfig /all` output into structured dictionaries:

```python
# utilities/post_migration.py
import jc

def _parse_windows_network_config(ipconfig_output: str) -> dict[str, dict[str, Any]]:
    """Parse Windows ipconfig /all output to extract network interface information.
    Uses the jc library for robust parsing.

    Args:
        ipconfig_output: Output from 'ipconfig /all' command

    Returns:
        Dictionary mapping interface names to their configuration
    """
    parsed = jc.parse("ipconfig", ipconfig_output)

    interfaces: dict[str, dict[str, Any]] = {}
    for adapter in parsed.get("adapters", []):
        interface_name = adapter.get("name", "unknown")

        ip_addresses: list[dict[str, Any]] = []
        subnet_mask = ""

        for ipv4 in adapter.get("ipv4_addresses", []):
            ip_addresses.append({
                "ip_address": ipv4.get("address", ""),
                "status": ipv4.get("status", ""),
            })
            if not subnet_mask and ipv4.get("subnet_mask"):
                subnet_mask = ipv4.get("subnet_mask", "")

        interface_config: dict[str, Any] = {
            "name": interface_name,
            "ip_addresses": ip_addresses,
            "subnet_mask": subnet_mask,
        }

        if adapter.get("physical_address"):
            interface_config["macAddress"] = adapter["physical_address"]

        gateways = adapter.get("default_gateways", [])
        if gateways:
            interface_config["gateway"] = gateways[0]
```

> **Tip:** The `jc` library supports over 300 command parsers. In this project it is used specifically for `ipconfig` parsing, but could be extended to parse other command outputs during post-migration checks (e.g., `ifconfig`, `route`, `ss`).

---

## Installing Dependencies

All dependencies are managed with `uv`. To install:

```bash
uv sync
```

This installs both runtime and development dependencies as declared in `pyproject.toml`.
