# General Utilities

API reference for the `utilities/utils.py` module. This module provides core utility functions and classes used throughout the MTV API test suite for provider management, cluster interaction, network mapping, and VM creation.

## Table of Contents

- [load\_source\_providers](#load_source_providers)
- [create\_source\_provider](#create_source_provider)
- [gen\_network\_map\_list](#gen_network_map_list)
- [get\_cluster\_client](#get_cluster_client)
- [get\_cluster\_version](#get_cluster_version)
- [populate\_vm\_ids](#populate_vm_ids)
- [VirtualMachineFromInstanceType](#virtualmachinefrominstancetype)

---

## `load_source_providers`

```python
def load_source_providers() -> dict[str, dict[str, Any]]
```

Loads source provider configurations from the `.providers.json` file at the repository root. Returns an empty dictionary if the file does not exist or is empty.

### Returns

| Type | Description |
|------|-------------|
| `dict[str, dict[str, Any]]` | Provider configurations keyed by provider name |

### Provider File Format

The `.providers.json` file maps provider names to their configuration objects. Each provider entry contains connection details, credentials, and type information:

```json
{
  "vsphere-8.0": {
    "type": "vsphere",
    "version": "8.0",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "secret",
    "vddk_init_image": "quay.io/org/vddk:latest"
  },
  "rhv-4.4": {
    "type": "ovirt",
    "version": "4.4",
    "fqdn": "rhvm.example.com",
    "api_url": "https://rhvm.example.com/ovirt-engine/api",
    "username": "admin@internal",
    "password": "secret"
  }
}
```

### Usage

The function is used in two contexts:

**1. Session-scoped fixture** (`conftest.py`) — provides provider data to the test session:

```python
@pytest.fixture(scope="session")
def source_providers() -> dict[str, dict[str, Any]]:
    return load_source_providers()
```

**2. Module-level provider type detection** (`tests/test_mtv_warm_migration.py`) — determines skip conditions at import time:

```python
_SOURCE_PROVIDER_TYPE = load_source_providers().get(
    py_config.get("source_provider", ""), {}
).get("type")
```

> **Note:** The provider name used at runtime is determined by the `source_provider` key in `py_config`, which must match a key in `.providers.json`. A `ValueError` is raised if the requested provider is not found.

---

## `create_source_provider`

```python
@contextmanager
def create_source_provider(
    source_provider_data: dict[str, Any],
    namespace: str,
    admin_client: DynamicClient,
    session_uuid: str,
    fixture_store: dict[str, Any],
    ocp_admin_client: DynamicClient,
    destination_ocp_secret: Secret,
    insecure: bool,
    tmp_dir: pytest.TempPathFactory | None = None,
) -> Generator[BaseProvider, None, None]
```

Context manager that creates a fully configured source provider instance with its associated OpenShift Secret and Provider custom resources. Supports all five provider types: VMware (vSphere), RHV (oVirt), OpenStack, OVA, and OpenShift.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `source_provider_data` | `dict[str, Any]` | Provider configuration from `.providers.json` |
| `namespace` | `str` | Target namespace for creating the Secret and Provider CRs |
| `admin_client` | `DynamicClient` | OpenShift client for resource creation |
| `session_uuid` | `str` | Unique session identifier for naming |
| `fixture_store` | `dict[str, Any]` | Fixture store for resource tracking and teardown |
| `ocp_admin_client` | `DynamicClient` | Admin client for OpenShift operations |
| `destination_ocp_secret` | `Secret` | Secret used by the OpenShift destination provider |
| `insecure` | `bool` | Whether to skip SSL verification |
| `tmp_dir` | `pytest.TempPathFactory \| None` | Temp directory factory for CA certificate files |

### Yields

| Type | Description |
|------|-------------|
| `BaseProvider` | A connected, tested provider instance (VMWareProvider, OvirtProvider, OpenStackProvider, OVAProvider, or OCPProvider) |

### Raises

| Exception | Condition |
|-----------|-----------|
| `ValueError` | Provider type is unrecognized or provider secret creation fails |
| `pytest.fail` | Provider connectivity test fails |

### Provider-Specific Behavior

| Provider | Secret Data | CA Certificate | Additional Config |
|----------|------------|----------------|-------------------|
| **VMware** | `user`, `password`, `url` | Fetched when `insecure=False` | `vddk_init_image`, copy-offload annotation |
| **RHV** | `user`, `password`, `url` | Always fetched (needed for imageio) | `ca_file` in secure mode |
| **OpenStack** | `username`, `password`, `regionName`, `projectName`, `domainName` | Fetched when `insecure=False` | `project_name`, `user_domain_name`, etc. |
| **OVA** | `url` | Not applicable | Minimal configuration |
| **OpenShift** | Reuses `destination_ocp_secret` | Not applicable | Uses local cluster host |

### Usage

Used as the session-scoped `source_provider` fixture in `conftest.py`:

```python
@pytest.fixture(scope="session")
def source_provider(
    fixture_store,
    session_uuid,
    source_provider_data,
    target_namespace,
    ocp_admin_client,
    tmp_path_factory,
    destination_ocp_secret,
):
    with create_source_provider(
        fixture_store=fixture_store,
        session_uuid=session_uuid,
        source_provider_data=source_provider_data,
        namespace=target_namespace,
        admin_client=ocp_admin_client,
        tmp_dir=tmp_path_factory,
        ocp_admin_client=ocp_admin_client,
        destination_ocp_secret=destination_ocp_secret,
        insecure=get_value_from_py_config(value="source_provider_insecure_skip_verify"),
    ) as _source_provider:
        yield _source_provider

    _source_provider.disconnect()
```

### Lifecycle

1. Deep copies provider data to avoid mutations
2. Builds provider-specific secret data and connection arguments
3. Fetches CA certificates when needed (via `openssl s_client`)
4. Creates the Secret CR via `create_and_store_resource()`
5. Creates the Provider CR via `create_and_store_resource()`
6. Waits for the Provider to reach `READY` status (timeout: 600s, stops on `ConnectionFailed`)
7. Opens a provider connection and verifies connectivity
8. Yields the connected provider instance
9. On exit, the caller is responsible for calling `disconnect()`

> **Warning:** The `tmp_dir` parameter is **required** for VMware (non-insecure), RHV (always), and OpenStack (non-insecure) providers. A `ValueError` is raised if it is not provided when needed.

---

## `gen_network_map_list`

```python
def gen_network_map_list(
    source_provider_inventory: ForkliftInventory,
    target_namespace: str,
    vms: list[str],
    multus_network_name: dict[str, str],
    pod_only: bool = False,
) -> list[dict[str, dict[str, str]]]
```

Generates a network mapping list that maps source VM networks to destination networks. This mapping is used when creating a `NetworkMap` custom resource for MTV migrations.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `source_provider_inventory` | `ForkliftInventory` | Forklift inventory instance to query VM network information |
| `target_namespace` | `str` | Target namespace on the destination cluster |
| `vms` | `list[str]` | List of VM names to generate mappings for |
| `multus_network_name` | `dict[str, str]` | Dictionary with `name` and `namespace` keys for the multus NetworkAttachmentDefinition |
| `pod_only` | `bool` | If `True`, all networks map to pod networking (default: `False`) |

### Returns

| Type | Description |
|------|-------------|
| `list[dict[str, dict[str, str]]]` | List of source-to-destination network mapping entries |

### Network Mapping Logic

The function iterates over the source networks discovered from the inventory and applies the following rules:

| Network Index | `pod_only=False` (default) | `pod_only=True` |
|---------------|---------------------------|-----------------|
| First (0) | Pod network (`{"type": "pod"}`) | Pod network |
| Second (1+) | Multus network (`{base_name}-1`) | Pod network |
| Third (2+) | Multus network (`{base_name}-2`) | Pod network |

Each mapping entry has this structure:

```python
{
    "destination": {"type": "pod"},           # or multus destination
    "source": {"id": "network-123", ...},     # from inventory
}
```

For multus destinations:

```python
{
    "destination": {
        "name": "cnv-bridge-abc12345-1",
        "namespace": "target-ns",
        "type": "multus",
    },
    "source": {"id": "network-456", ...},
}
```

### Usage

Called internally by `get_network_migration_map()` in `utilities/mtv_migration.py`:

```python
network_map_list = gen_network_map_list(
    target_namespace=target_namespace,
    source_provider_inventory=source_provider_inventory,
    multus_network_name=multus_network_name,
    vms=vms,
)
```

> **Tip:** The `multus_network_name` dictionary is generated by the `multus_network_name` fixture in `conftest.py`, which creates NADs with class-unique naming using SHA-256 hash prefixes (e.g., `cb-a1b2c3`) to prevent conflicts during parallel test execution.

---

## `get_cluster_client`

```python
def get_cluster_client() -> DynamicClient
```

Creates and returns a Kubernetes `DynamicClient` connected to the OpenShift cluster. Reads connection parameters from `py_config`.

### Returns

| Type | Description |
|------|-------------|
| `DynamicClient` | An authenticated client for OpenShift API interactions |

### Raises

| Exception | Condition |
|-----------|-----------|
| `ValueError` | Client creation fails or does not return a `DynamicClient` |

### Configuration

The function reads the following keys from `py_config`:

| Config Key | Description |
|------------|-------------|
| `cluster_host` | OpenShift API server URL |
| `cluster_username` | Authentication username |
| `cluster_password` | Authentication password |
| `insecure_verify_skip` | Skip TLS certificate verification (`True`/`False`) |

### Usage

**Primary usage** — the session-scoped `ocp_admin_client` fixture:

```python
@pytest.fixture(scope="session")
def ocp_admin_client():
    """OCP client"""
    LOGGER.info(msg="Creating OCP admin Client")
    _client = get_cluster_client()

    if remote_cluster_name := get_value_from_py_config("remote_ocp_cluster"):
        if remote_cluster_name not in _client.configuration.host:
            raise RemoteClusterAndLocalCluterNamesError(
                "Remote cluster must be the same as local cluster."
            )

    yield _client
```

**Teardown usage** — `session_teardown()` in `utilities/pytest_utils.py` creates a fresh client for cleanup:

```python
def session_teardown(session_store: dict[str, Any]) -> None:
    LOGGER.info("Running teardown to delete all created resources")
    ocp_client = get_cluster_client()
    # ... cleanup logic
```

> **Note:** This function wraps `ocp_resources.resource.get_client()` from the `openshift-python-wrapper` library. Direct use of `kubernetes.client` or `kubernetes.dynamic.DynamicClient` is forbidden at runtime per project conventions.

---

## `get_cluster_version`

```python
def get_cluster_version(client: DynamicClient) -> Version
```

Retrieves the OpenShift cluster version and returns it as a PEP 440 `Version` object. Handles non-standard version strings (e.g., engineering candidate builds like `4.22.0-ec.2`) by extracting the base semantic version.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `client` | `DynamicClient` | Authenticated OpenShift client |

### Returns

| Type | Description |
|------|-------------|
| `packaging.version.Version` | Parsed cluster version (e.g., `Version("4.20.1")`) |

### Raises

| Exception | Condition |
|-----------|-----------|
| `ValueError` | Version cannot be determined from the cluster or cannot be parsed |

### Version Parsing Behavior

| Cluster Reports | Returned Version | Notes |
|-----------------|-----------------|-------|
| `4.20.1` | `Version("4.20.1")` | Standard PEP 440 version |
| `4.22.0-ec.2` | `Version("4.22.0")` | Falls back to base version with a logged warning |
| `invalid` | raises `ValueError` | No parseable version found |

### Related Function

`get_cluster_version_str()` returns the raw version string without PEP 440 normalization. Use this when the exact string matters (e.g., for cache directory naming):

```python
def get_cluster_version_str(client: DynamicClient) -> str
```

Used by the `virtctl_binary` fixture to create versioned cache directories:

```python
cluster_version_str = get_cluster_version_str(ocp_admin_client)
shared_dir = Path(tempfile.gettempdir()) / "pytest-shared-virtctl" / cluster_version_str.replace(".", "-")
```

### Usage

Used in `utilities/post_migration.py` for version-dependent post-migration validation:

```python
from utilities.utils import get_cluster_version

cluster_version = get_cluster_version(client=ocp_admin_client)
if cluster_version >= Version("4.20"):
    # version-specific validation logic
```

> **Note:** Both functions query the `ClusterVersion` resource named `"version"` via the `ocp_resources.cluster_version.ClusterVersion` class.

---

## `populate_vm_ids`

```python
def populate_vm_ids(plan: dict[str, Any], inventory: ForkliftInventory) -> None
```

Populates VM IDs from the Forklift inventory into a migration plan's `virtual_machines` list. This function modifies the plan dictionary in place, adding an `"id"` key to each VM entry by querying the inventory API.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `plan` | `dict[str, Any]` | Migration plan containing a `virtual_machines` list |
| `inventory` | `ForkliftInventory` | Forklift inventory instance to query for VM IDs |

### Raises

| Exception | Condition |
|-----------|-----------|
| `ValueError` | Plan is not a dict or missing `virtual_machines` list |
| `ValueError` | VM name not found in the Forklift inventory (raised by `inventory.get_vm()`) |

### Plan Structure

Before calling `populate_vm_ids`:

```python
{
    "virtual_machines": [
        {"name": "rhel8-vm", "source_vm_power": "on", "guest_agent": True},
        {"name": "win2022-vm", "source_vm_power": "on"},
    ],
    "warm_migration": False,
}
```

After calling `populate_vm_ids`:

```python
{
    "virtual_machines": [
        {"name": "rhel8-vm", "source_vm_power": "on", "guest_agent": True, "id": "vm-1234"},
        {"name": "win2022-vm", "source_vm_power": "on", "id": "vm-5678"},
    ],
    "warm_migration": False,
}
```

### Usage

Called in every test class's `test_create_plan` method, immediately before creating the Plan CR:

```python
def test_create_plan(
    self,
    prepared_plan,
    fixture_store,
    ocp_admin_client,
    source_provider,
    destination_provider,
    target_namespace,
    source_provider_inventory,
):
    """Create MTV Plan CR resource."""
    populate_vm_ids(prepared_plan, source_provider_inventory)

    self.__class__.plan_resource = create_plan_resource(
        ocp_admin_client=ocp_admin_client,
        fixture_store=fixture_store,
        source_provider=source_provider,
        destination_provider=destination_provider,
        storage_map=self.storage_map,
        network_map=self.network_map,
        virtual_machines_list=prepared_plan["virtual_machines"],
        target_namespace=target_namespace,
        warm_migration=prepared_plan.get("warm_migration", False),
    )
    assert self.plan_resource, "Plan creation failed"
```

> **Tip:** The `prepared_plan` fixture (class-scoped) provides a deep copy of the test configuration with cloned VMs and updated names. Calling `populate_vm_ids` adds the inventory IDs needed by the Plan CR.

---

## `VirtualMachineFromInstanceType`

```python
class VirtualMachineFromInstanceType(VirtualMachine)
```

A custom `VirtualMachine` subclass that simplifies VM creation using KubeVirt instancetype and preference resources. Instead of defining a full VM specification manually, this class automatically builds the complete configuration from a small set of parameters.

Used exclusively for creating source VMs when the source provider is OpenShift (CNV-to-CNV migration testing).

### Constructor

```python
def __init__(
    self,
    instancetype_name: str,
    preference_name: str,
    datasource_name: str | None = None,
    datasource_namespace: str = "openshift-virtualization-os-images",
    storage_size: str = "30Gi",
    additional_networks: list[str] | None = None,
    cloud_init_user_data: str | None = None,
    run_strategy: str = VirtualMachine.RunStrategy.MANUAL,
    labels: dict[str, str] | None = None,
    annotations: dict[str, str] | None = None,
    **kwargs: Any,
)
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `instancetype_name` | `str` | *required* | Cluster instancetype name (e.g., `"u1.small"`) |
| `preference_name` | `str` | *required* | Cluster preference name (e.g., `"rhel.9"`) |
| `datasource_name` | `str \| None` | `None` | DataSource name for the root disk |
| `datasource_namespace` | `str` | `"openshift-virtualization-os-images"` | Namespace of the DataSource |
| `storage_size` | `str` | `"30Gi"` | Root disk size |
| `additional_networks` | `list[str] \| None` | `None` | NAD names for additional multus networks |
| `cloud_init_user_data` | `str \| None` | `None` | Cloud-init user data string |
| `run_strategy` | `str` | `VirtualMachine.RunStrategy.MANUAL` | VM run strategy |
| `labels` | `dict[str, str] \| None` | `None` | Labels for the VM template metadata |
| `annotations` | `dict[str, str] \| None` | `None` | Annotations for the VM |
| `**kwargs` | `Any` | — | Additional arguments passed to the base `VirtualMachine` (e.g., `name`, `namespace`, `client`) |

### Generated VM Specification

The class generates a complete VM spec including:

- **Instancetype reference** — `VirtualMachineClusterInstancetype` resource
- **Preference reference** — `VirtualMachineClusterPreference` resource
- **DataVolumeTemplate** — cloned from the specified `DataSource`
- **Networks** — default pod network + any additional multus networks
- **Interfaces** — masquerade for pod, bridge for multus (all `virtio`)
- **Cloud-init** — `cloudInitNoCloud` volume if user data is provided

### Usage

Used by `create_source_cnv_vms()` to create test VMs for OpenShift-to-OpenShift migration scenarios:

```python
create_and_store_resource(
    resource=VirtualMachineFromInstanceType,
    fixture_store=fixture_store,
    name=f"{vm_dict['name']}{vm_name_suffix}",
    namespace=namespace,
    client=client,
    instancetype_name="u1.small",
    preference_name="rhel.9",
    datasource_name="rhel9",
    storage_size="30Gi",
    additional_networks=[network_name],
    cloud_init_user_data="""#cloud-config
chpasswd:
  expire: false
  password: 123456
  user: rhel
""",
    run_strategy=VirtualMachine.RunStrategy.MANUAL,
)
```

After creation, VMs are started and waited on:

```python
for vm in vms_to_create:
    if not vm.ready:
        vm.start()

for vm in vms_to_create:
    vm.wait_for_ready_status(status=True)
```

### Generated YAML Structure

The `to_dict()` method produces a VM manifest equivalent to:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel8-vm-suffix
  namespace: target-ns
spec:
  runStrategy: Manual
  instancetype:
    kind: VirtualMachineClusterInstancetype
    name: u1.small
  preference:
    kind: VirtualMachineClusterPreference
    name: rhel.9
  dataVolumeTemplates:
    - metadata:
        name: rhel8-vm-suffix
      spec:
        sourceRef:
          kind: DataSource
          name: rhel9
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
              model: virtio
            - name: net1
              bridge: {}
              model: virtio
        resources: {}
      networks:
        - name: default
          pod: {}
        - name: net1
          multus:
            networkName: target-ns/cnv-bridge-abc123
      volumes:
        - name: rootdisk
          dataVolume:
            name: rhel8-vm-suffix
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              chpasswd:
                expire: false
                password: 123456
                user: rhel
```

> **Note:** This class follows the `create_and_store_resource()` pattern — all VMs created through it are tracked in `fixture_store["teardown"]` for automatic cleanup.

---

## Helper Functions

The module also contains several supporting functions used internally by the functions above:

| Function | Description |
|----------|-------------|
| `vmware_provider(provider_data)` | Returns `True` if provider type is vSphere |
| `rhv_provider(provider_data)` | Returns `True` if provider type is RHV/oVirt |
| `openstack_provider(provider_data)` | Returns `True` if provider type is OpenStack |
| `ova_provider(provider_data)` | Returns `True` if provider type is OVA |
| `ocp_provider(provider_data)` | Returns `True` if provider type is OpenShift |
| `get_value_from_py_config(value)` | Reads a value from `py_config` with string-to-boolean conversion |
| `generate_ca_cert_file(provider_fqdn, cert_file)` | Fetches a CA certificate from a provider via `openssl s_client` |
| `generate_class_hash_prefix(nodeid, length)` | Generates a FIPS-compliant SHA-256 hash prefix for unique resource naming |
| `get_cluster_version_str(client)` | Returns the raw OCP version string without PEP 440 normalization |
| `extract_vm_from_plan(prepared_plan, vm_index, fixture_name)` | Extracts a single VM from a multi-VM plan for parallel migration tests |
| `get_mtv_version(client)` | Returns the MTV operator version from ClusterServiceVersion |
| `has_mtv_minimum_version(min_version, client)` | Checks if installed MTV version meets a minimum requirement |
