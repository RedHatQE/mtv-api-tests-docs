# Writing New Tests

This guide walks through the complete process of adding a new migration test to the mtv-api-tests suite. Every test follows a standardized 5-step class-based pattern that creates OpenShift resources, executes the migration, and validates the results.

## Overview

Adding a new test involves three steps:

1. **Add test configuration** to `tests/tests_config/config.py`
2. **Create the test file** in `tests/`
3. **Implement the 5 test methods** following the established pattern

Each test class represents a single end-to-end migration scenario. The framework handles VM cloning, resource cleanup, unique naming, and parallel execution safety automatically through fixtures.

## Step 1: Add Test Configuration

Every test is driven by a configuration entry in `tests/tests_config/config.py`. This file defines a `tests_params` dictionary where each key is a test name and each value describes the migration scenario.

### Minimal Configuration

At minimum, a test configuration requires the VMs to migrate and whether it's a warm migration:

```python
tests_params: dict = {
    # ... existing tests ...
    "test_my_new_migration": {
        "virtual_machines": [
            {
                "name": "mtv-tests-rhel8",
                "guest_agent": True,
            },
        ],
        "warm_migration": False,
    },
}
```

### VM Configuration Options

Each entry in the `virtual_machines` list supports these fields:

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | `str` | VM name in the source provider |
| `source_vm_power` | No | `"on"` or `"off"` | Desired power state before migration. If omitted, the VM power state is left unchanged |
| `guest_agent` | No | `bool` | Whether guest agent is installed on the VM |
| `clone` | No | `bool` | Clone the VM before migration (used for copy-offload tests) |
| `disk_type` | No | `str` | `"thin"`, `"thick-lazy"`, or `"thick-eager"` |
| `clone_name` | No | `str` | Custom name for the cloned VM |
| `preserve_name_format` | No | `bool` | Keep original name format without sanitization |
| `snapshots` | No | `int` | Number of snapshots to create on the source VM before migration |
| `add_disks` | No | `list[dict]` | Additional disks to attach before migration |

### Plan-Level Configuration Options

Beyond VMs, the plan configuration supports these optional fields:

| Field | Type | Description |
|-------|------|-------------|
| `warm_migration` | `bool` | `True` for warm (incremental) migration, `False` for cold |
| `copyoffload` | `bool` | Enable copy-offload (XCOPY) mode |
| `target_power_state` | `"on"` or `"off"` | Desired VM power state after migration |
| `preserve_static_ips` | `bool` | Preserve static IP addresses during migration |
| `vm_target_namespace` | `str` | Custom namespace for migrated VMs (instead of the default target namespace) |
| `multus_namespace` | `str` | Namespace for cross-namespace NAD access |
| `pvc_name_template` | `str` | Go template for PVC naming (e.g., `"{{.VmName}}-disk-{{.DiskIndex}}"`) |
| `pvc_name_template_use_generate_name` | `bool` | Use Kubernetes `generateName` for PVCs |
| `target_labels` | `dict` | Labels to apply to migrated VMs. Use `None` as a value for auto-generation with `session_uuid` |
| `target_affinity` | `dict` | Pod affinity/anti-affinity rules for migrated VMs |
| `target_node_selector` | `dict` | Node selector constraints. Use `None` as a value for auto-generation |
| `pre_hook` / `post_hook` | `dict` | Hook configuration with `{"expected_result": "succeed" | "fail"}` |
| `expected_migration_result` | `str` | Set to `"fail"` when migration is expected to fail (e.g., hook failure tests) |
| `guest_agent_timeout` | `int` | Custom timeout in seconds for guest agent readiness |

### Configuration Examples

**Cold migration with a powered-off VM:**

```python
"test_cold_powered_off_vm": {
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8",
            "source_vm_power": "off",
            "guest_agent": True,
        },
    ],
    "warm_migration": False,
},
```

**Warm migration with multiple VMs:**

```python
"test_sanity_warm_mtv_migration": {
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8",
            "source_vm_power": "on",
            "guest_agent": True,
        },
    ],
    "warm_migration": True,
},
```

**Comprehensive test with advanced features (from `test_cold_migration_comprehensive`):**

```python
"test_cold_migration_comprehensive": {
    "virtual_machines": [
        {
            "name": "mtv-win2019-3disks",
            "source_vm_power": "off",
            "guest_agent": True,
        },
    ],
    "warm_migration": False,
    "target_power_state": "on",
    "preserve_static_ips": True,
    "pvc_name_template": "{{.VmName}}-disk-{{.DiskIndex}}",
    "pvc_name_template_use_generate_name": False,
    "target_node_selector": {
        "mtv-comprehensive-node": None,
    },
    "target_labels": {
        "mtv-comprehensive-label": None,
        "test-type": "comprehensive",
    },
    "target_affinity": {
        "podAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [
                {
                    "podAffinityTerm": {
                        "labelSelector": {"matchLabels": {"app": "test"}},
                        "topologyKey": "kubernetes.io/hostname",
                    },
                    "weight": 50,
                }
            ]
        }
    },
    "vm_target_namespace": "mtv-comprehensive-vms",
    "multus_namespace": "default",
},
```

**Copy-offload test with cloned VM and extra disks:**

```python
"test_copyoffload_multi_disk_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "add_disks": [
                {"size_gb": 30, "disk_mode": "persistent", "provision_type": "thick-lazy"},
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

> **Warning:** Never use default values for configuration keys you control. Access `tests_params` values with direct key access (`plan["warm_migration"]`), not `.get()` with defaults. See the "Deterministic Tests" rule in CLAUDE.md.

> **Tip:** Set label and node selector values to `None` for auto-generation with the session UUID, which ensures uniqueness across parallel test runs. Use a string value for static labels.

## Step 2: Create the Test File

Create a new file in the `tests/` directory. The naming convention is `test_<feature>_migration.py`.

### Required Imports

```python
from typing import TYPE_CHECKING, Any

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
from utilities.utils import populate_vm_ids

if TYPE_CHECKING:
    from kubernetes.dynamic import DynamicClient
    from libs.base_provider import BaseProvider
    from libs.forklift_inventory import ForkliftInventory
    from utilities.ssh_utils import SSHConnectionManager
```

> **Note:** The `kubernetes` package import inside `TYPE_CHECKING` is allowed for type annotations only. Direct runtime usage of the `kubernetes` package is forbidden — all cluster interactions must go through `openshift-python-wrapper`.

### Additional Imports for Specific Test Types

For warm migrations, add the cutover utility:

```python
from utilities.migration_utils import get_cutover_value
```

For tests expecting migration failure:

```python
from exceptions.exceptions import MigrationPlanExecError
```

For tests using comprehensive features (node selectors, labels):

```python
from libs.providers.openshift import OCPProvider  # Inside TYPE_CHECKING block
```

### Module-Level Markers

If an entire file should be skipped for certain providers, use `pytestmark`:

```python
from ocp_resources.provider import Provider
from utilities.utils import load_source_providers
from pytest_testconfig import py_config

_SOURCE_PROVIDER_TYPE = load_source_providers().get(
    py_config.get("source_provider", ""), {}
).get("type")

pytestmark = [
    pytest.mark.skipif(
        _SOURCE_PROVIDER_TYPE
        in (Provider.ProviderType.OPENSTACK, Provider.ProviderType.OPENSHIFT, Provider.ProviderType.OVA),
        reason=f"{_SOURCE_PROVIDER_TYPE} warm migration is not supported.",
    ),
]
```

This pattern is used in `test_mtv_warm_migration.py` to skip the entire file for providers that don't support warm migration.

## Step 3: Apply Class Decorators

Every test class requires a specific set of decorators. The order matters for readability but not for functionality.

### Standard Cold Migration Class

```python
@pytest.mark.tier0
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_my_new_migration"])],
    indirect=True,
    ids=["descriptive-test-id"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestMyNewMigration:
    """Description of what this migration test validates."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan
```

### Decorator Breakdown

| Decorator | Required | Purpose |
|-----------|----------|---------|
| `@pytest.mark.incremental` | Yes | Marks subsequent tests as `xfail` if a prior test in the class fails |
| `@pytest.mark.parametrize(...)` | Yes | Links the class to its `tests_params` configuration |
| `@pytest.mark.usefixtures("cleanup_migrated_vms")` | Yes | Registers the teardown fixture for VM cleanup after all tests |
| `@pytest.mark.tier0` | No | Marks as core smoke test |
| `@pytest.mark.warm` | No | Marks as warm migration test |
| `@pytest.mark.remote` | No | Marks as remote cluster test |
| `@pytest.mark.copyoffload` | No | Marks as copy-offload (XCOPY) test |
| `@pytest.mark.min_mtv_version("2.10.0")` | No | Skip test if MTV version is below threshold |
| `@pytest.mark.skipif(...)` | No | Conditional skip based on configuration |

### Parametrize Details

The `@pytest.mark.parametrize` decorator is the critical link between the test class and its configuration:

```python
@pytest.mark.parametrize(
    "class_plan_config",                                          # Fixture name - must be exactly this
    [pytest.param(py_config["tests_params"]["test_name_here"])],  # Config entry
    indirect=True,                                                # CRITICAL: passes via request.param
    ids=["human-readable-id"],                                    # Shows in test output
)
```

- **`"class_plan_config"`**: Must match the fixture name exactly. This fixture is defined in `conftest.py` and simply returns `request.param`.
- **`indirect=True`**: This is critical. Without it, the config dict would be passed directly to the test method instead of going through the fixture system.
- **`ids`**: A human-readable identifier shown in test output and reports. Use descriptive names like `"rhel8"`, `"MTV-200 rhel"`, or `"comprehensive-cold"`.

### Class-Level Type Annotations

Every test class declares three class-level attributes to store resources shared across the 5 test methods:

```python
class TestMyNewMigration:
    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan
```

These are set during the first three test methods using `self.__class__.attribute = value` and read in later methods via `self.attribute`.

### Warm Migration Class

Warm migration tests add the `warm` marker and the `precopy_interval_forkliftcontroller` fixture:

```python
@pytest.mark.tier0
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_sanity_warm_mtv_migration"])],
    indirect=True,
    ids=["rhel8"],
)
@pytest.mark.usefixtures("precopy_interval_forkliftcontroller", "cleanup_migrated_vms")
class TestSanityWarmMtvMigration:
    """Warm migration sanity test."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan
```

> **Note:** The `precopy_interval_forkliftcontroller` fixture configures the snapshot interval on the Forklift controller. It is only needed for warm migration tests.

### Remote Cluster Class

Remote cluster tests use `destination_ocp_provider` instead of `destination_provider` and add a `skipif` guard:

```python
@pytest.mark.remote
@pytest.mark.incremental
@pytest.mark.skipif(
    not get_value_from_py_config("remote_ocp_cluster"),
    reason="No remote OCP cluster provided",
)
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_cold_remote_ocp"])],
    indirect=True,
    ids=["MTV-79"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestColdRemoteOcp:
    """Cold migration test to remote OCP cluster."""
```

## Step 4: Implement the 5 Test Methods

Every migration test class implements exactly 5 methods in this order. The `@pytest.mark.incremental` marker ensures that if any step fails, subsequent steps are automatically marked as `xfail`.

### Method 1: test_create_storagemap

Creates the `StorageMap` custom resource that maps source storage to destination storage classes.

```python
def test_create_storagemap(
    self,
    prepared_plan: dict[str, Any],
    fixture_store: dict[str, Any],
    source_provider: "BaseProvider",
    destination_provider: "BaseProvider",
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
    source_provider_inventory: "ForkliftInventory",
) -> None:
    """Create StorageMap resource.

    Args:
        prepared_plan (dict[str, Any]): Test plan configuration with VM details
        fixture_store (dict[str, Any]): Resource tracking dictionary
        source_provider (BaseProvider): Source provider connection
        destination_provider (BaseProvider): Destination provider connection
        ocp_admin_client (DynamicClient): OpenShift admin client
        target_namespace (str): Target namespace for migration resources
        source_provider_inventory (ForkliftInventory): Source provider inventory

    Raises:
        AssertionError: If StorageMap creation fails
    """
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

**Key points:**
- Extract VM names from `prepared_plan["virtual_machines"]` — these are the cloned names, not the original config names.
- Store the result on the **class** using `self.__class__.storage_map`, not on the instance. This makes it accessible to later test methods.
- The `get_storage_migration_map` function queries the Forklift inventory for VM storage details and creates the `StorageMap` CR.

### Method 2: test_create_networkmap

Creates the `NetworkMap` custom resource that maps source networks to destination networks.

```python
def test_create_networkmap(
    self,
    prepared_plan: dict[str, Any],
    fixture_store: dict[str, Any],
    source_provider: "BaseProvider",
    destination_provider: "BaseProvider",
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
    source_provider_inventory: "ForkliftInventory",
    multus_network_name: dict[str, str],
) -> None:
    """Create NetworkMap resource.

    Args:
        prepared_plan (dict[str, Any]): Test plan configuration with VM details
        fixture_store (dict[str, Any]): Resource tracking dictionary
        source_provider (BaseProvider): Source provider connection
        destination_provider (BaseProvider): Destination provider connection
        ocp_admin_client (DynamicClient): OpenShift admin client
        target_namespace (str): Target namespace for migration resources
        source_provider_inventory (ForkliftInventory): Source provider inventory
        multus_network_name (dict[str, str]): Multus network name for network mapping

    Raises:
        AssertionError: If NetworkMap creation fails
    """
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
```

**Key points:**
- The `multus_network_name` fixture returns a dict with `{"name": "...", "namespace": "..."}` containing the Network Attachment Definition details.
- The first network interface maps to `"pod"` (default pod network), and additional interfaces map to Multus NADs.

### Method 3: test_create_plan

Creates the MTV `Plan` custom resource that defines the full migration plan.

```python
def test_create_plan(
    self,
    prepared_plan: dict[str, Any],
    fixture_store: dict[str, Any],
    source_provider: "BaseProvider",
    destination_provider: "BaseProvider",
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
    source_provider_inventory: "ForkliftInventory",
) -> None:
    """Create MTV Plan CR resource.

    Args:
        prepared_plan (dict[str, Any]): Test plan configuration with VM details
        fixture_store (dict[str, Any]): Resource tracking dictionary
        source_provider (BaseProvider): Source provider connection
        destination_provider (BaseProvider): Destination provider connection
        ocp_admin_client (DynamicClient): OpenShift admin client
        target_namespace (str): Target namespace for migration resources
        source_provider_inventory (ForkliftInventory): Source provider inventory

    Raises:
        AssertionError: If Plan creation fails
    """
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

**Key points:**
- Always call `populate_vm_ids()` before `create_plan_resource()`. This resolves VM names to provider-specific IDs in the Forklift inventory.
- The `create_plan_resource` function creates the Plan CR and waits for it to reach `Ready` status (360s timeout).
- Reference `self.storage_map` and `self.network_map` from the previous steps.

**For comprehensive tests with advanced features,** pass additional parameters:

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
    warm_migration=prepared_plan["warm_migration"],
    # Advanced features
    target_power_state=prepared_plan["target_power_state"],
    preserve_static_ips=prepared_plan["preserve_static_ips"],
    pvc_name_template=prepared_plan["pvc_name_template"],
    pvc_name_template_use_generate_name=prepared_plan["pvc_name_template_use_generate_name"],
    target_node_selector={labeled_worker_node["label_key"]: labeled_worker_node["label_value"]},
    target_labels=target_vm_labels["vm_labels"],
    target_affinity=prepared_plan["target_affinity"],
    vm_target_namespace=prepared_plan["vm_target_namespace"],
)
```

When using advanced features, add the corresponding fixtures to the method signature (e.g., `labeled_worker_node`, `target_vm_labels`).

### Method 4: test_migrate_vms

Executes the migration and waits for completion.

**Cold migration:**

```python
def test_migrate_vms(
    self,
    fixture_store: dict[str, Any],
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
) -> None:
    """Execute migration.

    Args:
        fixture_store (dict[str, Any]): Resource tracking dictionary
        ocp_admin_client (DynamicClient): OpenShift admin client
        target_namespace (str): Target namespace for migration resources
    """
    execute_migration(
        ocp_admin_client=ocp_admin_client,
        fixture_store=fixture_store,
        plan=self.plan_resource,
        target_namespace=target_namespace,
    )
```

**Warm migration:**

```python
def test_migrate_vms(
    self,
    fixture_store: dict[str, Any],
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
) -> None:
    """Execute warm migration with cutover.

    Args:
        fixture_store (dict[str, Any]): Resource tracking dictionary
        ocp_admin_client (DynamicClient): OpenShift admin client
        target_namespace (str): Target namespace for migration resources
    """
    execute_migration(
        ocp_admin_client=ocp_admin_client,
        fixture_store=fixture_store,
        plan=self.plan_resource,
        target_namespace=target_namespace,
        cut_over=get_cutover_value(),
    )
```

**Key points:**
- This is the simplest test method — it only needs `fixture_store`, `ocp_admin_client`, and `target_namespace`.
- For warm migrations, pass `cut_over=get_cutover_value()` which calculates the cutover time.
- The function creates a `Migration` CR and waits for it to reach `Succeeded` status (default timeout from `plan_wait_timeout` in config, typically 3600s).

### Method 5: test_check_vms

Validates the migrated VMs match the expected state.

```python
def test_check_vms(
    self,
    prepared_plan: dict[str, Any],
    source_provider: "BaseProvider",
    destination_provider: "BaseProvider",
    source_provider_data: dict[str, Any],
    target_namespace: str,
    source_vms_namespace: str,
    source_provider_inventory: "ForkliftInventory",
    vm_ssh_connections: "SSHConnectionManager | None",
) -> None:
    """Validate migrated VMs.

    Args:
        prepared_plan (dict[str, Any]): Test plan configuration with VM details
        source_provider (BaseProvider): Source provider connection
        destination_provider (BaseProvider): Destination provider connection
        source_provider_data (dict[str, Any]): Source provider configuration
        target_namespace (str): Target namespace for migration resources
        source_vms_namespace (str): Source VMs namespace
        source_provider_inventory (ForkliftInventory): Source provider inventory
        vm_ssh_connections (SSHConnectionManager | None): SSH connection manager

    Raises:
        AssertionError: If any VM validation checks fail
    """
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

**Key points:**
- The `check_vms` function performs comprehensive validation: CPU, memory, network interfaces, storage class mappings, guest agent, SSH connectivity, and more.
- It reads source VM details from `prepared_plan["source_vms_data"]` (populated by the `prepared_plan` fixture).
- For comprehensive tests with node selectors and labels, pass additional parameters:

```python
check_vms(
    plan=prepared_plan,
    source_provider=source_provider,
    destination_provider=destination_provider,
    source_provider_data=source_provider_data,
    network_map_resource=self.network_map,
    storage_map_resource=self.storage_map,
    source_vms_namespace=source_vms_namespace,
    source_provider_inventory=source_provider_inventory,
    labeled_worker_node=labeled_worker_node,
    target_vm_labels=target_vm_labels,
    vm_ssh_connections=vm_ssh_connections,
)
```

## Understanding `prepared_plan`

The `prepared_plan` fixture is the backbone of the test class. It is a class-scoped fixture that transforms your raw `tests_params` configuration into a fully prepared migration plan.

### What `prepared_plan` Does

1. **Deep copies** the `class_plan_config` to prevent mutation
2. **Initializes** `source_vms_data` — a dictionary for storing complete source VM details
3. **Handles custom VM target namespace** — creates it if configured via `vm_target_namespace`
4. **Clones VMs** with unique names incorporating session UUID, storage class, and migration type
5. **Controls power state** — starts or stops VMs based on `source_vm_power`
6. **Collects VM details** — calls the provider's `vm_dict()` to capture disks, NICs, CPU, memory, and snapshots
7. **Waits for inventory sync** — ensures cloned VMs appear in the Forklift inventory
8. **Creates hooks** — sets up pre/post migration hooks if configured

### Output Structure

After preparation, `prepared_plan` contains:

```python
{
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8-ceph-rbd-4-16-cold-abc123",  # Updated clone name
            "guest_agent": True,
            "source_vm_power": "on",
            "snapshots_before_migration": [...],
        }
    ],
    "warm_migration": False,
    "source_vms_data": {
        "mtv-tests-rhel8-ceph-rbd-4-16-cold-abc123": {
            "name": "...",
            "provider_vm_api": "<provider VM object>",
            "disks": [...],
            "network_interfaces": [...],
            "cpu": {"cores": 2, "sockets": 1},
            "memory_in_mb": 8192,
            "snapshots_data": [...],
            "power_state": "on",
        }
    },
    "_vm_target_namespace": "auto-abc123",
    # ... all other fields from config preserved
}
```

> **Note:** The `virtual_machines` list contains the cloned/updated VM names, while `source_vms_data` holds the complete source VM details indexed by the updated name. Test methods should read VM names from `prepared_plan["virtual_machines"]` and detailed VM data from `prepared_plan["source_vms_data"]`.

## Complete Example: Cold Migration Test

Here is a complete, minimal test file for a cold migration, taken from `tests/test_mtv_cold_migration.py`:

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
from utilities.utils import populate_vm_ids


@pytest.mark.tier0
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [
        pytest.param(
            py_config["tests_params"]["test_sanity_cold_mtv_migration"],
        )
    ],
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
        self,
        prepared_plan,
        fixture_store,
        ocp_admin_client,
        source_provider,
        destination_provider,
        source_provider_inventory,
        target_namespace,
    ):
        """Create StorageMap resource for migration."""
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
        self,
        prepared_plan,
        fixture_store,
        ocp_admin_client,
        source_provider,
        destination_provider,
        source_provider_inventory,
        target_namespace,
        multus_network_name,
    ):
        """Create NetworkMap resource for migration."""
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

    def test_migrate_vms(
        self,
        fixture_store,
        ocp_admin_client,
        target_namespace,
    ):
        """Execute migration."""
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
        )

    def test_check_vms(
        self,
        prepared_plan,
        source_provider,
        destination_provider,
        source_provider_data,
        target_namespace,
        source_vms_namespace,
        source_provider_inventory,
        vm_ssh_connections,
    ):
        """Validate migrated VMs."""
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

## Fixture Reference

### Fixtures Used in Test Methods

| Fixture | Scope | Description |
|---------|-------|-------------|
| `prepared_plan` | Class | Prepared plan with cloned VMs, updated names, and `source_vms_data` |
| `fixture_store` | Session | Dict for resource tracking and teardown |
| `ocp_admin_client` | Session | OpenShift `DynamicClient` for cluster operations |
| `source_provider` | Session | Source provider instance (VMware, RHV, OpenStack, OVA, OpenShift) |
| `destination_provider` | Session | Destination provider instance (local OCP cluster) |
| `destination_ocp_provider` | Session | Remote OCP destination provider (for remote tests only) |
| `source_provider_inventory` | Session | Forklift inventory with VMs, networks, and storage for the source provider |
| `target_namespace` | Session | Target namespace for migration resources (auto-created with session UUID) |
| `multus_network_name` | Class | Multus NAD name and namespace for network mapping |
| `source_provider_data` | Session | Raw provider configuration from `.providers.json` |
| `source_vms_namespace` | Session | Namespace for OpenShift source VMs |
| `vm_ssh_connections` | Class | SSH connection manager for validating migrated VM connectivity |
| `cleanup_migrated_vms` | Class | Teardown fixture — deletes migrated VMs after test class completes |
| `precopy_interval_forkliftcontroller` | Session | Configures snapshot intervals (warm migration only) |
| `labeled_worker_node` | Class | Labels a worker node for node selector tests |
| `target_vm_labels` | Class | Generates VM labels with session UUID substitution |

### Fixture Request Patterns

Choose the right pattern for requesting fixtures:

- **Method parameter**: When the test method needs the fixture's return value

  ```python
  def test_create_plan(self, prepared_plan, fixture_store, ...):
      # Uses prepared_plan and fixture_store values directly
  ```

- **`@pytest.mark.usefixtures()`**: When a fixture performs setup/teardown but its return value isn't needed

  ```python
  @pytest.mark.usefixtures("cleanup_migrated_vms")
  class TestMyMigration:
      # cleanup_migrated_vms runs teardown after all tests, no return value needed
  ```

> **Warning:** Never request a fixture via both `@pytest.mark.usefixtures()` and a method parameter. Choose one pattern.

## Resource Cleanup

All OpenShift resources created during tests are cleaned up automatically through two mechanisms:

1. **`cleanup_migrated_vms` fixture**: Deletes migrated VMs after each test class completes. Applied via `@pytest.mark.usefixtures("cleanup_migrated_vms")`.

2. **`fixture_store["teardown"]`**: The `create_and_store_resource()` function registers every resource it creates. Session-level teardown iterates through this dict and deletes all tracked resources.

> **Warning:** Every OpenShift resource must be created via `create_and_store_resource()`. Never instantiate and deploy resources directly — this bypasses tracking and will leak resources.

## Parallel Execution Safety

Tests are safe for parallel execution with `pytest-xdist` because:

- Each worker session gets a unique `session_uuid`, which is incorporated into all resource names
- Namespaces are unique per session (e.g., `auto-abc123`)
- `create_and_store_resource()` generates names incorporating the session UUID
- Each worker has an isolated `fixture_store`
- VM clones include the session UUID in their names

> **Tip:** Never hardcode namespace names, resource names, or VM names. Always use fixtures and let the framework generate unique names.

## Checklist

Before submitting a new test:

- [ ] Configuration added to `tests_params` in `tests/tests_config/config.py`
- [ ] Test file created in `tests/` with naming convention `test_<feature>_migration.py`
- [ ] `@pytest.mark.incremental` applied at class level
- [ ] `@pytest.mark.parametrize` with `indirect=True` links to the correct `tests_params` key
- [ ] `@pytest.mark.usefixtures("cleanup_migrated_vms")` applied at class level
- [ ] Appropriate tier/feature markers applied (`tier0`, `warm`, `remote`, `copyoffload`)
- [ ] All 5 test methods implemented in order: storagemap, networkmap, plan, migrate, check
- [ ] Resources stored on class via `self.__class__.<attribute>`
- [ ] `populate_vm_ids()` called before `create_plan_resource()`
- [ ] Each test method requests only the fixtures it needs
- [ ] Type annotations on all function signatures
- [ ] Docstrings with Args, Returns, and Raises sections
- [ ] Pre-commit passes: `pre-commit run --all-files`
