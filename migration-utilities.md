# Migration Utilities

API reference for `utilities/mtv_migration.py` — the core module that orchestrates VM migration workflows in the MTV API test suite. This module provides functions to create migration resources (StorageMap, NetworkMap, Plan), execute migrations, and monitor migration status.

## Overview

The migration utilities module sits at the center of the MTV test framework's 5-step migration pattern:

```
StorageMap → NetworkMap → Plan → Migration → Validation
```

Each function creates Kubernetes Custom Resources (CRs) through the `openshift-python-wrapper` library and tracks them in the `fixture_store` for automatic cleanup.

**Source file:** `utilities/mtv_migration.py`

**Imports:**

```python
from utilities.mtv_migration import (
    create_plan_resource,
    execute_migration,
    get_storage_migration_map,
    get_network_migration_map,
    get_plan_migration_status,
    wait_for_migration_complate,
    wait_for_concurrent_migration_execution,
    verify_vm_disk_count,
    get_vm_suffix,
)
```

---

## Resource Creation Functions

### `get_storage_migration_map()`

Creates a `StorageMap` Custom Resource that defines how storage from the source provider maps to OpenShift storage classes. Supports both standard inventory-based mapping and copy-offload (XCOPY) mapping.

```python
def get_storage_migration_map(
    fixture_store: dict[str, Any],
    target_namespace: str,
    source_provider: BaseProvider,
    destination_provider: BaseProvider,
    ocp_admin_client: DynamicClient,
    source_provider_inventory: ForkliftInventory,
    vms: list[str],
    storage_class: str | None = None,
    # Copy-offload specific parameters
    datastore_id: str | None = None,
    secondary_datastore_id: str | None = None,
    non_xcopy_datastore_id: str | None = None,
    offload_plugin_config: dict[str, Any] | None = None,
    access_mode: str | None = None,
    volume_mode: str | None = None,
) -> StorageMap:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fixture_store` | `dict[str, Any]` | Yes | Pytest fixture store for resource tracking and cleanup |
| `target_namespace` | `str` | Yes | Namespace where the StorageMap CR is created |
| `source_provider` | `BaseProvider` | Yes | Source provider instance (VMware, RHV, OpenStack, etc.) |
| `destination_provider` | `BaseProvider` | Yes | Destination OpenShift provider instance |
| `ocp_admin_client` | `DynamicClient` | Yes | OpenShift admin client for API interactions |
| `source_provider_inventory` | `ForkliftInventory` | Yes | Forklift inventory for querying VM storage mappings |
| `vms` | `list[str]` | Yes | List of VM names to map storage for |
| `storage_class` | `str \| None` | No | Target storage class. Defaults to `py_config["storage_class"]` |
| `datastore_id` | `str \| None` | No | Primary datastore ID — triggers copy-offload mode when set |
| `secondary_datastore_id` | `str \| None` | No | Secondary datastore ID for multi-datastore copy-offload |
| `non_xcopy_datastore_id` | `str \| None` | No | Non-XCOPY datastore ID for mixed migration scenarios |
| `offload_plugin_config` | `dict[str, Any] \| None` | No | Copy-offload plugin config — required if `datastore_id` is set |
| `access_mode` | `str \| None` | No | PVC access mode for copy-offload (e.g., `"ReadWriteOnce"`) |
| `volume_mode` | `str \| None` | No | PVC volume mode for copy-offload (e.g., `"Block"`) |

**Returns:** `StorageMap` — The created StorageMap CR resource.

**Raises:**
- `ValueError` — If `source_provider.ocp_resource` or `destination_provider.ocp_resource` is not set, or if copy-offload parameter combinations are invalid.

#### Standard Migration Example

From `tests/test_mtv_cold_migration.py`:

```python
def test_create_storagemap(
    self, prepared_plan, fixture_store, ocp_admin_client,
    source_provider, destination_provider,
    source_provider_inventory, target_namespace,
):
    vms = [vm["name"] for vm in prepared_plan["virtual_machines"]]
    self.__class__.storage_map = get_storage_migration_map(
        fixture_store=fixture_store,
        source_provider=source_provider,
        destination_provider=destination_provider,
        source_provider_inventory=source_provider_inventory,
        ocp_admin_client=ocp_admin_client,
        target_namespace=target_namespace,
        vms=vms,
    )
    assert self.storage_map, "StorageMap creation failed"
```

In standard mode, the function queries the Forklift inventory to discover which datastores/storage domains the VMs use, then creates a mapping entry for each one pointing to the configured `storage_class`.

#### Copy-Offload Migration Example

From `tests/test_copyoffload_migration.py`:

```python
offload_plugin_config = {
    "vsphereXcopyConfig": {
        "secretRef": copyoffload_storage_secret.name,
        "storageVendorProduct": storage_vendor_product,
    }
}

self.__class__.storage_map = get_storage_migration_map(
    fixture_store=fixture_store,
    target_namespace=target_namespace,
    source_provider=source_provider,
    destination_provider=destination_provider,
    ocp_admin_client=ocp_admin_client,
    source_provider_inventory=source_provider_inventory,
    vms=vms_names,
    storage_class=storage_class,
    datastore_id=datastore_id,
    offload_plugin_config=offload_plugin_config,
    access_mode="ReadWriteOnce",
    volume_mode="Block",
)
```

> **Note:** When `datastore_id` is provided, the function switches to copy-offload mode. The `offload_plugin_config` parameter becomes required, and the inventory is not queried — datastore IDs are mapped directly.

#### Copy-Offload Parameter Validation

The function enforces the following rules for copy-offload parameters:

| Scenario | Result |
|----------|--------|
| `datastore_id` without `offload_plugin_config` | `ValueError` |
| `secondary_datastore_id` without `datastore_id` | `ValueError` |
| `non_xcopy_datastore_id` without `datastore_id` | `ValueError` |
| `datastore_id` + `offload_plugin_config` | Copy-offload mode |
| None of the above | Standard inventory-based mode |

---

### `get_network_migration_map()`

Creates a `NetworkMap` Custom Resource that defines how source provider networks map to OpenShift networks (pod network or Multus network attachment definitions).

```python
def get_network_migration_map(
    fixture_store: dict[str, Any],
    source_provider: BaseProvider,
    destination_provider: BaseProvider,
    multus_network_name: dict[str, str],
    ocp_admin_client: DynamicClient,
    target_namespace: str,
    source_provider_inventory: ForkliftInventory,
    vms: list[str],
) -> NetworkMap:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fixture_store` | `dict[str, Any]` | Yes | Pytest fixture store for resource tracking |
| `source_provider` | `BaseProvider` | Yes | Source provider instance |
| `destination_provider` | `BaseProvider` | Yes | Destination provider instance |
| `multus_network_name` | `dict[str, str]` | Yes | Multus network config with `"name"` and `"namespace"` keys |
| `ocp_admin_client` | `DynamicClient` | Yes | OpenShift admin client |
| `target_namespace` | `str` | Yes | Target namespace for the NetworkMap CR |
| `source_provider_inventory` | `ForkliftInventory` | Yes | Forklift inventory for querying VM network mappings |
| `vms` | `list[str]` | Yes | List of VM names to map networks for |

**Returns:** `NetworkMap` — The created NetworkMap CR resource.

**Raises:**
- `ValueError` — If `source_provider.ocp_resource` or `destination_provider.ocp_resource` is not set.

#### Network Mapping Logic

The function delegates to `gen_network_map_list()` in `utilities/utils.py`, which maps VM networks according to this scheme:

- **First network interface** (index 0) always maps to the **pod network** (`{"type": "pod"}`)
- **Additional network interfaces** map to **Multus NADs** with auto-generated names (`{base_name}-1`, `{base_name}-2`, etc.)

#### Usage Example

From `tests/test_mtv_cold_migration.py`:

```python
vms = [vm["name"] for vm in prepared_plan["virtual_machines"]]
self.__class__.network_map = get_network_migration_map(
    fixture_store=fixture_store,
    source_provider=source_provider,
    destination_provider=destination_provider,
    source_provider_inventory=source_provider_inventory,
    ocp_admin_client=ocp_admin_client,
    target_namespace=target_namespace,
    multus_network_name=multus_network_name,
    vms=vms,
)
```

---

### `create_plan_resource()`

Creates an MTV `Plan` Custom Resource that defines the full migration configuration: source/destination providers, storage/network mappings, VM list, and optional features like warm migration, hooks, and copy-offload.

```python
def create_plan_resource(
    ocp_admin_client: DynamicClient,
    fixture_store: dict[str, Any],
    source_provider: BaseProvider,
    destination_provider: OCPProvider,
    storage_map: StorageMap,
    network_map: NetworkMap,
    virtual_machines_list: list[dict[str, Any]],
    target_namespace: str,
    warm_migration: bool = False,
    pre_hook_name: str | None = None,
    pre_hook_namespace: str | None = None,
    after_hook_name: str | None = None,
    after_hook_namespace: str | None = None,
    test_name: str | None = None,
    copyoffload: bool = False,
    preserve_static_ips: bool = False,
    pvc_name_template: str | None = None,
    pvc_name_template_use_generate_name: bool | None = None,
    target_node_selector: dict[str, str] | None = None,
    target_labels: dict[str, str] | None = None,
    target_affinity: dict[str, Any] | None = None,
    vm_target_namespace: str | None = None,
    target_power_state: str | None = None,
) -> Plan:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `ocp_admin_client` | `DynamicClient` | Yes | OpenShift admin client |
| `fixture_store` | `dict[str, Any]` | Yes | Fixture store for resource tracking |
| `source_provider` | `BaseProvider` | Yes | Source provider with `ocp_resource` set |
| `destination_provider` | `OCPProvider` | Yes | Destination OpenShift provider with `ocp_resource` set |
| `storage_map` | `StorageMap` | Yes | StorageMap CR created by `get_storage_migration_map()` |
| `network_map` | `NetworkMap` | Yes | NetworkMap CR created by `get_network_migration_map()` |
| `virtual_machines_list` | `list[dict[str, Any]]` | Yes | List of VM configurations from the test plan |
| `target_namespace` | `str` | Yes | Target namespace for the Plan CR |
| `warm_migration` | `bool` | No | Enable warm migration mode. Default: `False` |
| `pre_hook_name` | `str \| None` | No | Name of a pre-migration Hook CR |
| `pre_hook_namespace` | `str \| None` | No | Namespace of the pre-migration Hook CR |
| `after_hook_name` | `str \| None` | No | Name of a post-migration Hook CR |
| `after_hook_namespace` | `str \| None` | No | Namespace of the post-migration Hook CR |
| `test_name` | `str \| None` | No | Test name for resource naming |
| `copyoffload` | `bool` | No | Enable copy-offload PVC template settings. Default: `False` |
| `preserve_static_ips` | `bool` | No | Preserve static IP addresses during migration. Default: `False` |
| `pvc_name_template` | `str \| None` | No | Go template for PVC naming |
| `pvc_name_template_use_generate_name` | `bool \| None` | No | Use Kubernetes `generateName` for PVCs |
| `target_node_selector` | `dict[str, str] \| None` | No | Node selector labels for scheduling migrated VMs |
| `target_labels` | `dict[str, str] \| None` | No | Custom labels to apply to migrated VMs |
| `target_affinity` | `dict[str, Any] \| None` | No | Pod affinity/anti-affinity rules for migrated VMs |
| `vm_target_namespace` | `str \| None` | No | Custom namespace for VMs (overrides `target_namespace`) |
| `target_power_state` | `str \| None` | No | Desired power state after migration (`"on"` or `"off"`) |

**Returns:** `Plan` — The created Plan CR resource after it reaches `Ready` status.

**Raises:**
- `ValueError` — If `source_provider.ocp_resource` or `destination_provider.ocp_resource` is not set.
- `TimeoutExpiredError` — If the Plan fails to reach `Ready` status within 360 seconds.

#### Behavior Details

1. **Resource naming**: The Plan CR is automatically named via `create_and_store_resource()`, which generates a unique name with UUID and appends `-warm` or `-cold` based on `warm_migration`. Names exceeding 63 characters are truncated (Kubernetes limit).

2. **Ready condition wait**: After creation, the function waits up to 360 seconds for the Plan condition `READY=True`. On timeout, it logs the full Plan instance state plus provider details before raising.

3. **Copy-offload mode**: When `copyoffload=True`, the function automatically sets `pvc_name_template="pvc"` to enable the volume populator framework's consistent PVC naming.

4. **Plan secret wait**: For copy-offload plans, the function additionally waits up to 60 seconds for Forklift to create a plan-specific Secret (`{plan_name}-*`).

#### Cold Migration Example

From `tests/test_mtv_cold_migration.py`:

```python
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
```

#### Warm Migration with Advanced Options

From `tests/test_warm_migration_comprehensive.py` test configuration in `tests/tests_config/config.py`:

```python
# Test config enabling advanced Plan options
"test_warm_migration_comprehensive": {
    "virtual_machines": [
        {
            "name": "mtv-win2022-ip-3disks",
            "source_vm_power": "on",
            "guest_agent": True,
        },
    ],
    "warm_migration": True,
    "target_power_state": "on",
    "preserve_static_ips": True,
    "vm_target_namespace": "custom-vm-namespace",
    "pvc_name_template": '{{ .FileName | trimSuffix ".vmdk" | replace "_" "-" }}-{{.DiskIndex}}',
    "pvc_name_template_use_generate_name": True,
    "target_labels": {
        "mtv-comprehensive-test": None,
        "static-label": "static-value",
    },
    "target_affinity": {
        "podAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [...]
        }
    },
},
```

---

## Migration Execution Functions

### `execute_migration()`

Creates a `Migration` Custom Resource to trigger the actual VM migration and waits for it to complete (succeed or fail).

```python
def execute_migration(
    ocp_admin_client: DynamicClient,
    fixture_store: dict[str, Any],
    plan: Plan,
    target_namespace: str,
    cut_over: datetime | None = None,
) -> None:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `ocp_admin_client` | `DynamicClient` | Yes | OpenShift admin client |
| `fixture_store` | `dict[str, Any]` | Yes | Fixture store for resource tracking |
| `plan` | `Plan` | Yes | The Plan CR that defines the migration |
| `target_namespace` | `str` | Yes | Namespace for the Migration CR |
| `cut_over` | `datetime \| None` | No | Cut-over time for warm migration. Default: `None` (cold migration) |

**Returns:** `None`

**Raises:**
- `MigrationPlanExecError` — If the migration fails or times out.

#### Cold Migration Example

```python
execute_migration(
    ocp_admin_client=ocp_admin_client,
    fixture_store=fixture_store,
    plan=self.plan_resource,
    target_namespace=target_namespace,
)
```

#### Warm Migration Example

For warm migrations, pass a `cut_over` datetime. The `get_cutover_value()` helper from `utilities/migration_utils.py` computes the cutover time:

```python
from utilities.migration_utils import get_cutover_value

execute_migration(
    ocp_admin_client=ocp_admin_client,
    fixture_store=fixture_store,
    plan=self.plan_resource,
    target_namespace=target_namespace,
    cut_over=get_cutover_value(),  # Now + mins_before_cutover from config
)
```

> **Note:** `get_cutover_value()` returns `datetime.now(UTC) + timedelta(minutes=py_config["mins_before_cutover"])`. Pass `current_cutover=True` for an immediate cutover.

#### Internal Flow

1. Creates a `Migration` CR via `create_and_store_resource()` referencing the Plan name/namespace
2. Calls `wait_for_migration_complate()` to poll until the Plan reaches `Succeeded` or `Failed`
3. On failure or timeout, raises `MigrationPlanExecError` with the full Plan instance state

---

## Migration Status Functions

### `get_plan_migration_status()`

Reads the current migration status from a Plan's conditions and associated Migration CR.

```python
def get_plan_migration_status(plan: Plan) -> str:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `plan` | `Plan` | Yes | The Plan resource to check |

**Returns:** `str` — One of the following status values:

| Return Value | Meaning |
|-------------|---------|
| `Plan.Status.SUCCEEDED` | Migration completed successfully |
| `Plan.Status.FAILED` | Migration failed |
| `Plan.Status.EXECUTING` | Migration is currently running |
| `""` (empty string) | No migration started or status unknown |

**Status Resolution Logic:**

1. Checks Plan conditions for an `Advisory` category condition with `status=True`
2. If condition type is `Succeeded` or `Failed`, returns that type
3. Otherwise, looks for a Migration CR owned by the Plan — if found, returns `Executing`
4. If no Migration CR exists, returns an empty string

---

### `wait_for_migration_complate()`

Polls the Plan migration status until it reaches `Succeeded` or `Failed`.

```python
def wait_for_migration_complate(plan: Plan) -> None:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `plan` | `Plan` | Yes | The Plan resource to monitor |

**Returns:** `None` — Returns when migration succeeds.

**Raises:**
- `MigrationPlanExecError` — If the migration fails or if the timeout expires.

**Timeout:** Configured via `py_config["plan_wait_timeout"]` (default: 600 seconds, typically set to 3600 for production runs).

**Polling interval:** 1 second.

The function logs status transitions as they occur, so you'll see messages like:

```
Plan 'auto-abc123-cold' migration status: 'Executing'
Plan 'auto-abc123-cold' migration status: 'Succeeded'
```

---

### `wait_for_concurrent_migration_execution()`

Validates that multiple migration plans reach the `Executing` state simultaneously. Used for testing concurrent migration scenarios.

```python
def wait_for_concurrent_migration_execution(
    plan_list: list[Plan],
    timeout: int = 120,
) -> None:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `plan_list` | `list[Plan]` | Yes | List of Plan resources to monitor |
| `timeout` | `int` | No | Timeout in seconds. Default: 120 |

**Returns:** `None` — Returns when all plans are executing simultaneously.

**Raises:**
- `AssertionError` — If any plan completes (`Succeeded`/`Failed`) before all plans are executing, or if the timeout expires.

#### Usage Example

From `tests/test_copyoffload_migration.py`:

```python
wait_for_concurrent_migration_execution(
    plan_list=[self.plan_resource_1, self.plan_resource_2]
)
```

---

## Diagnostic Functions

### `verify_vm_disk_count()`

Verifies that each migrated VM has the expected number of disks based on the plan configuration.

```python
def verify_vm_disk_count(
    destination_provider,
    plan,
    target_namespace,
) -> None:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `destination_provider` | `OCPProvider` | Yes | Destination OpenShift provider |
| `plan` | `dict` | Yes | Test plan dictionary containing VM configuration |
| `target_namespace` | `str` | Yes | Namespace where VMs were migrated |

**Returns:** `None`

**Raises:**
- `AssertionError` — If any VM's disk count doesn't match expectations.

**Disk Calculation:** `expected_disks = 1 (base disk) + len(vm_config.get("add_disks", []))`

---

### `get_vm_suffix()`

Generates a suffix string for VM names based on migration type, storage class, and OCP version. Used by fixtures to create unique, descriptive VM clone names.

```python
def get_vm_suffix(warm_migration: bool) -> str:
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `warm_migration` | `bool` | Yes | Whether this is a warm migration |

**Returns:** `str` — A suffix like `-csi-cephfs-4-16-cold` (truncated to 63 chars if needed).

---

## Internal Functions

These private functions support the public API and are not typically called directly by tests.

### `_find_migration_for_plan()`

Finds the `Migration` CR owned by a given Plan by inspecting `ownerReferences`.

```python
def _find_migration_for_plan(plan: Plan) -> Migration:
```

**Raises:** `MigrationNotFoundError` — If no Migration CR with a matching owner reference is found.

### `_get_failed_migration_step()`

Examines a Migration CR's status to determine which pipeline step a specific VM failed at.

```python
def _get_failed_migration_step(plan: Plan, vm_name: str) -> str:
```

Returns the step name (e.g., `"PreHook"`, `"DiskTransfer"`, `"PostHook"`) and logs the error message.

**Raises:**
- `MigrationNotFoundError` — If the Migration CR cannot be found
- `MigrationStatusError` — If the Migration has no status
- `VmPipelineError` — If the VM has no pipeline or no failed step
- `VmNotFoundError` — If the VM is not found in Migration status

### `_get_all_vms_failed_steps()`

Maps all VM names to their failed pipeline step. Catches exceptions per-VM so one lookup failure doesn't prevent others.

```python
def _get_all_vms_failed_steps(
    plan_resource: Plan,
    vm_names: list[str],
) -> dict[str, str | None]:
```

Returns a dictionary like `{"vm-1": "DiskTransfer", "vm-2": None}`.

---

## Custom Exceptions

All migration-related exceptions are defined in `exceptions/exceptions.py`:

| Exception | Description |
|-----------|-------------|
| `MigrationNotFoundError` | Migration CR not found for a Plan |
| `MigrationStatusError` | Migration CR has no status or incomplete status |
| `MigrationPlanExecError` | Migration plan execution failed or timed out |
| `VmNotFoundError` | VM not found in Migration status |
| `VmPipelineError` | VM pipeline is missing or has no failed step |
| `VmMigrationStepMismatchError` | VMs in the same plan failed at different steps |

---

## Resource Lifecycle

All resources created by migration utility functions are managed through `create_and_store_resource()` from `utilities/resources.py`. This provides:

- **Automatic naming**: Generates unique names using the session UUID (e.g., `auto-abc123-vsphere-8-0-cold`)
- **Kubernetes name compliance**: Truncates names exceeding 63 characters
- **Conflict handling**: Reuses existing resources on `ConflictError`
- **Cleanup tracking**: Stores metadata in `fixture_store["teardown"]` for session teardown

```python
# Internal resource tracking structure
fixture_store["teardown"] = {
    "StorageMap": [{"name": "auto-abc123-...", "namespace": "mtv-ns", "module": "..."}],
    "NetworkMap": [{"name": "auto-abc123-...", "namespace": "mtv-ns", "module": "..."}],
    "Plan": [{"name": "auto-abc123-...-cold", "namespace": "mtv-ns", "module": "..."}],
    "Migration": [{"name": "auto-abc123-...-cold", "namespace": "mtv-ns", "module": "..."}],
}
```

> **Warning:** Never create migration resources (StorageMap, NetworkMap, Plan, Migration) directly. Always use the utility functions to ensure proper tracking and cleanup.

---

## Complete Test Example

A full migration test class showing all utility functions working together:

```python
import pytest
from ocp_resources.network_map import NetworkMap
from ocp_resources.plan import Plan
from ocp_resources.storage_map import StorageMap
from pytest_testconfig import config as py_config

from utilities.mtv_migration import (
    create_plan_resource,
    execute_migration,
    get_network_migration_map,
    get_storage_migration_map,
)
from utilities.post_migration import check_vms


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

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    def test_create_storagemap(
        self, prepared_plan, fixture_store, ocp_admin_client,
        source_provider, destination_provider,
        source_provider_inventory, target_namespace,
    ):
        vms = [vm["name"] for vm in prepared_plan["virtual_machines"]]
        self.__class__.storage_map = get_storage_migration_map(
            fixture_store=fixture_store,
            source_provider=source_provider,
            destination_provider=destination_provider,
            source_provider_inventory=source_provider_inventory,
            ocp_admin_client=ocp_admin_client,
            target_namespace=target_namespace,
            vms=vms,
        )
        assert self.storage_map, "StorageMap creation failed"

    def test_create_networkmap(
        self, prepared_plan, fixture_store, ocp_admin_client,
        source_provider, destination_provider,
        source_provider_inventory, target_namespace, multus_network_name,
    ):
        vms = [vm["name"] for vm in prepared_plan["virtual_machines"]]
        self.__class__.network_map = get_network_migration_map(
            fixture_store=fixture_store,
            source_provider=source_provider,
            destination_provider=destination_provider,
            source_provider_inventory=source_provider_inventory,
            ocp_admin_client=ocp_admin_client,
            target_namespace=target_namespace,
            multus_network_name=multus_network_name,
            vms=vms,
        )
        assert self.network_map, "NetworkMap creation failed"

    def test_create_plan(
        self, prepared_plan, fixture_store, ocp_admin_client,
        source_provider, destination_provider,
        target_namespace, source_provider_inventory,
    ):
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

    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
        )

    def test_check_vms(
        self, prepared_plan, source_provider, destination_provider,
        source_provider_data, target_namespace,
        source_vms_namespace, source_provider_inventory, vm_ssh_connections,
    ):
        check_vms(
            plan=prepared_plan,
            source_provider=source_provider,
            destination_provider=destination_provider,
            network_map_resource=self.network_map,
            storage_map_resource=self.storage_map,
            source_provider_data=source_provider_data,
            source_vms_namespace=source_vms_namespace,
            source_provider_inventory=source_provider_inventory,
            vm_ssh_connections=vm_ssh_connections,
        )
```

---

## Configuration Reference

Key configuration values from `tests/tests_config/config.py` that affect migration behavior:

| Config Key | Default | Description |
|------------|---------|-------------|
| `storage_class` | — | Target OpenShift storage class (required) |
| `plan_wait_timeout` | `3600` | Seconds to wait for migration completion |
| `mins_before_cutover` | `5` | Minutes to add for warm migration cutover time |
| `target_ocp_version` | — | OCP version string used in VM name suffixes |
| `snapshots_interval` | `2` | Interval in minutes between warm migration snapshots |

> **Tip:** The `plan_wait_timeout` default of 3600 seconds (1 hour) applies to the full migration cycle. For large VMs or slow storage, consider increasing this value in your test configuration.
