# Introduction

## What is MTV API Tests?

MTV API Tests is an end-to-end test suite for **Migration Toolkit for Virtualization (MTV)**, a Kubernetes-native operator that migrates virtual machines from legacy virtualization platforms to Red Hat OpenShift. The test suite exercises the complete migration lifecycle through the MTV API -- from provider registration and resource mapping to migration execution and post-migration validation.

Rather than testing through a UI, MTV API Tests interacts directly with Kubernetes Custom Resources (CRs) using the [openshift-python-wrapper](https://github.com/RedHatQE/openshift-python-wrapper) library. This enables precise, repeatable verification of MTV's core functionality: creating `StorageMap`, `NetworkMap`, and `Plan` resources, executing migrations, and validating that migrated VMs match their source configuration.

```
┌─────────────────────┐         ┌──────────────────────┐
│   Source Provider    │         │  OpenShift Cluster    │
│                      │  MTV    │                       │
│  VMware vSphere      │────────▶│  KubeVirt VMs         │
│  RHV/oVirt           │  Plan   │                       │
│  OpenStack           │────────▶│  StorageMap           │
│  OVA Files           │         │  NetworkMap           │
│  OpenShift           │         │  Migration CR         │
└─────────────────────┘         └──────────────────────┘
```

The project is built with **Python 3.12+**, uses **pytest** as its test framework, and is managed with **uv** for dependency resolution.

## What MTV Does

MTV (Migration Toolkit for Virtualization) is deployed as an operator in the `openshift-mtv` namespace and consists of several key concepts:

| Concept | Description |
|---------|-------------|
| **Provider** | A connection definition to a source virtualization platform or destination OpenShift cluster |
| **StorageMap** | Maps source storage domains/datastores to target OpenShift storage classes |
| **NetworkMap** | Maps source networks to target OpenShift networks (pod network or Multus) |
| **Plan** | Orchestrates the migration of a set of VMs, referencing a StorageMap, NetworkMap, and provider pair |
| **Migration** | The execution instance of a Plan -- triggers the actual data transfer |

The test suite validates that these resources are created correctly, that migrations complete successfully, and that the resulting KubeVirt VMs preserve the properties of their source counterparts.

## Supported Source Providers

MTV API Tests supports five source provider types, each implemented as a subclass of `BaseProvider`:

### VMware vSphere

The most feature-rich provider, supporting cold migration, warm migration, and copy-offload (XCOPY) transfers. Uses the `pyVmomi` SDK for vSphere API interaction.

```python
# libs/providers/vmware.py
from pyVim.connect import Disconnect, SmartConnect
from pyVmomi import vim

from libs.base_provider import BaseProvider

class VMWareProvider(BaseProvider):
    # Supports disk types: thin, thick-lazy, thick-eager
    # Supports copy-offload via VAAI XCOPY
    # Supports snapshot handling and datastore management
    ...
```

### Red Hat Virtualization (RHV)

Connects to oVirt/RHV environments using the `ovirt-sdk4`. Supports template-based VM cloning and datacenter management.

```python
# libs/providers/rhv.py
import ovirtsdk4
from ovirtsdk4 import types

class OvirtProvider(BaseProvider):
    ...
```

### OpenStack

Uses the `openstacksdk` to manage instances and volumes on OpenStack platforms.

```python
# libs/providers/openstack.py
from openstack.connection import Connection

class OpenStackProvider(BaseProvider):
    ...
```

### OVA

Imports virtual machines from OVA files stored on NFS paths. A lightweight provider that requires no live connection to a hypervisor.

```python
# libs/providers/ova.py
class OVAProvider(BaseProvider):
    ...
```

### OpenShift (Virtual-to-Virtual)

Migrates VMs between OpenShift clusters (or within the same cluster). Uses `ocp_resources` for VirtualMachine resource management.

```python
# libs/providers/openshift.py
from ocp_resources.virtual_machine import VirtualMachine

class OCPProvider(BaseProvider):
    ...
```

> **Note:** OpenShift always serves as the **destination** provider. When used as a source, it enables virtual-to-virtual migration between OpenShift environments.

### Provider Abstraction

All providers inherit from `BaseProvider`, which defines the contract every provider must implement:

```python
# libs/base_provider.py
class BaseProvider(abc.ABC):
    VIRTUAL_MACHINE_TEMPLATE: dict[str, Any] = {
        "id": "",
        "name": "",
        "provider_type": "",
        "network_interfaces": [],
        "disks": [],
        "cpu": {},
        "memory_in_mb": 0,
        "snapshots_data": [],
        "power_state": "",
    }

    @abc.abstractmethod
    def connect(self) -> Any: ...

    @abc.abstractmethod
    def disconnect(self) -> Any: ...

    @abc.abstractmethod
    def vm_dict(self, **kwargs: Any) -> dict[str, Any]: ...

    @abc.abstractmethod
    def clone_vm(self, source_vm_name: str, clone_vm_name: str, session_uuid: str, **kwargs: Any) -> Any: ...

    @abc.abstractmethod
    def delete_vm(self, vm_name: str) -> Any: ...

    @abc.abstractmethod
    def get_vm_or_template_networks(self, names: list[str], inventory: ForkliftInventory) -> list[dict[str, str]]: ...
```

The unified `VIRTUAL_MACHINE_TEMPLATE` ensures VM metadata is represented consistently across all providers, enabling the same validation logic in `check_vms()` regardless of the source platform.

## Migration Types

### Cold Migration

The source VM is powered off during the entire data transfer. This is the simplest and most reliable migration strategy.

```python
# tests/tests_config/config.py
"test_sanity_cold_mtv_migration": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "guest_agent": True},
    ],
    "warm_migration": False,
},
```

- **Supported providers:** All (VMware, RHV, OpenStack, OVA, OpenShift)
- **Test marker:** `@pytest.mark.tier0`
- **Test file:** `tests/test_mtv_cold_migration.py`

### Warm Migration

The source VM remains running during the initial data copy. MTV performs incremental snapshots at configurable intervals, then executes a final cutover that briefly stops the VM to transfer the last delta.

```python
# tests/tests_config/config.py
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

- **Supported providers:** VMware vSphere, RHV
- **Test marker:** `@pytest.mark.warm`
- **Test file:** `tests/test_mtv_warm_migration.py`
- **Key parameters:** `snapshots_interval` (default: 2 minutes), `mins_before_cutover` (default: 5 minutes)

> **Note:** Warm migration is not supported for OpenStack, OVA, or OpenShift source providers. The test file automatically skips these:
>
> ```python
> pytestmark = [
>     pytest.mark.skipif(
>         _SOURCE_PROVIDER_TYPE
>         in (Provider.ProviderType.OPENSTACK, Provider.ProviderType.OPENSHIFT, Provider.ProviderType.OVA),
>         reason=f"{_SOURCE_PROVIDER_TYPE} warm migration is not supported.",
>     ),
> ]
> ```

### Copy-Offload (XCOPY)

A high-performance migration method exclusive to VMware vSphere. Instead of transferring data through the MTV controller pod (VDDK), copy-offload leverages VAAI XCOPY primitives on shared storage arrays. The storage array copies data directly between vSphere datastores and OpenShift persistent volumes, dramatically reducing migration time and network load.

```python
# tests/tests_config/config.py
"test_copyoffload_thin_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

- **Supported providers:** VMware vSphere only (requires shared storage)
- **Test marker:** `@pytest.mark.copyoffload`
- **Test file:** `tests/test_copyoffload_migration.py`
- **Disk types tested:** thin, thick-lazy, RDM virtual, independent persistent/nonpersistent
- **Scenarios covered:** multi-disk, multi-datastore, snapshots, large VMs (1TB+), simultaneous migrations, warm+copy-offload, scale (5 VMs)

## The 5-Step Test Pattern

Every migration test in the suite follows a standardized 5-step sequential pattern. Each step is a separate test method within an `@pytest.mark.incremental` class -- if any step fails, all subsequent steps are automatically marked as expected failures (xfail).

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  1. StorageMap   │────▶│  2. NetworkMap    │────▶│  3. Plan         │
│  Map source      │     │  Map source       │     │  Create          │
│  storage to      │     │  networks to      │     │  migration       │
│  target classes  │     │  target networks  │     │  orchestration   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                          │
                              ┌──────────────────┐        │
                              │  5. Check VMs    │        ▼
                              │  Validate CPU,   │  ┌──────────────────┐
                              │  memory, disks,  │◀─│  4. Migrate      │
                              │  networks, power │  │  Execute plan    │
                              └──────────────────┘  │  and wait for    │
                                                    │  completion      │
                                                    └──────────────────┘
```

Here is the complete pattern from an actual test class (`tests/test_mtv_cold_migration.py`):

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

    def test_create_storagemap(self, prepared_plan, fixture_store, ocp_admin_client,
                                source_provider, destination_provider,
                                source_provider_inventory, target_namespace):
        """Step 1: Create StorageMap resource for migration."""
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

    def test_create_networkmap(self, prepared_plan, fixture_store, ocp_admin_client,
                                source_provider, destination_provider,
                                source_provider_inventory, target_namespace,
                                multus_network_name):
        """Step 2: Create NetworkMap resource for migration."""
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

    def test_create_plan(self, prepared_plan, fixture_store, ocp_admin_client,
                          source_provider, destination_provider, target_namespace,
                          source_provider_inventory):
        """Step 3: Create MTV Plan CR resource."""
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

    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        """Step 4: Execute migration."""
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
        )

    def test_check_vms(self, prepared_plan, source_provider, destination_provider,
                        source_provider_data, target_namespace,
                        source_vms_namespace, source_provider_inventory,
                        vm_ssh_connections):
        """Step 5: Validate migrated VMs."""
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

### What Each Step Does

| Step | Method | Purpose |
|------|--------|---------|
| **1** | `test_create_storagemap` | Creates a `StorageMap` CR that maps source storage (datastores, storage domains) to target OpenShift storage classes |
| **2** | `test_create_networkmap` | Creates a `NetworkMap` CR that maps source networks to target networks (pod network or Multus NADs) |
| **3** | `test_create_plan` | Creates a `Plan` CR that references the StorageMap, NetworkMap, source/destination providers, and the list of VMs to migrate |
| **4** | `test_migrate_vms` | Creates a `Migration` CR to trigger execution of the Plan, then waits for completion |
| **5** | `test_check_vms` | Validates that each migrated VM matches its source: power state, CPU, memory, disks, networks, and optionally SSH connectivity |

### Key Pattern Features

- **`@pytest.mark.incremental`** -- If Step 1 fails, Steps 2-5 are automatically xfailed. This prevents cascading errors from obscuring the root cause.
- **`self.__class__.attribute`** -- Resources created in earlier steps are stored on the class so later steps can reference them (e.g., `self.storage_map` in `test_create_plan`).
- **`indirect=True` parametrization** -- Test configuration from `tests_config/config.py` is passed through the `class_plan_config` fixture, which processes it into a `prepared_plan` with cloned VMs and unique names.
- **`@pytest.mark.usefixtures("cleanup_migrated_vms")`** -- A teardown fixture automatically removes migrated VMs from the target cluster after the test class completes.

## Post-Migration Validation

The `check_vms()` function in `utilities/post_migration.py` performs comprehensive validation of each migrated VM. It compares the destination VM's properties against the source, checking:

- **Power state** -- Matches source power or explicit `target_power_state`
- **CPU** -- Core count and topology preserved
- **Memory** -- Allocation matches source
- **Networks** -- Interface count and network mappings match the NetworkMap
- **Disks** -- Count and capacity match the StorageMap
- **SSL configuration** -- Provider connection security settings are consistent
- **SSH connectivity** -- When guest agent is present, validates the VM is reachable
- **Static IPs** -- For warm migrations with `preserve_static_ips: True`

Failures are collected per-VM rather than failing on the first error, providing a complete picture of all validation mismatches in a single test run.

## Test Markers

Tests are organized using pytest markers that map to migration scenarios:

```ini
# pytest.ini
markers =
    tier0: Core functionality tests (smoke tests)
    remote: Remote cluster migration tests
    warm: Warm migration tests
    copyoffload: Copy-offload (XCOPY) tests
    incremental: marks tests as incremental (xfail on previous failure)
    min_mtv_version: mark test to require minimum MTV version
```

Run specific test categories:

```bash
pytest -m tier0          # Smoke tests only
pytest -m warm           # Warm migration tests
pytest -m copyoffload    # Copy-offload tests
pytest -m "tier0 or warm"  # Multiple categories
```

## Project Structure

```
mtv-api-tests/
├── conftest.py                    # Pytest fixtures and hooks (~1500 lines)
├── pyproject.toml                 # Dependencies and project metadata
├── pytest.ini                     # Test runner configuration
├── Dockerfile                     # Container image for test execution
│
├── libs/                          # Provider abstraction layer
│   ├── base_provider.py           # Abstract base class for all providers
│   ├── forklift_inventory.py      # Forklift inventory API client
│   └── providers/
│       ├── vmware.py              # VMware vSphere provider
│       ├── rhv.py                 # RHV/oVirt provider
│       ├── openstack.py           # OpenStack provider
│       ├── ova.py                 # OVA file provider
│       └── openshift.py           # OpenShift provider
│
├── utilities/                     # Core utilities
│   ├── mtv_migration.py           # Migration orchestration (StorageMap, NetworkMap, Plan, Migration)
│   ├── post_migration.py          # Post-migration validation (check_vms)
│   ├── resources.py               # OpenShift resource creation (create_and_store_resource)
│   ├── utils.py                   # General utilities (provider loading, client creation)
│   ├── copyoffload_migration.py   # Copy-offload specific utilities
│   ├── hooks.py                   # Pre/post migration hooks
│   ├── ssh_utils.py               # SSH connection management
│   └── ...
│
├── tests/                         # Test files
│   ├── test_mtv_cold_migration.py
│   ├── test_mtv_warm_migration.py
│   ├── test_copyoffload_migration.py
│   ├── test_cold_migration_comprehensive.py
│   ├── test_warm_migration_comprehensive.py
│   ├── test_post_hook_retain_failed_vm.py
│   └── tests_config/
│       └── config.py              # Test parameters and VM definitions
│
└── exceptions/
    └── exceptions.py              # Domain-specific exception classes
```

## Resource Lifecycle

Every OpenShift resource created during testing is managed through a single function: `create_and_store_resource()`. This ensures consistent naming, automatic cleanup, and conflict handling:

```python
# utilities/resources.py
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    # 1. Auto-generates unique names using session UUID
    # 2. Truncates to 63 chars (Kubernetes name limit)
    # 3. Deploys resource with wait=True
    # 4. Handles ConflictError (safe re-runs)
    # 5. Stores in fixture_store["teardown"] for automatic cleanup
    ...
```

> **Warning:** Direct resource creation (e.g., `Namespace(name=...).deploy()`) bypasses the tracking system and will not be cleaned up automatically. Always use `create_and_store_resource()`.

## Requirements

- **Python:** 3.12 or 3.13
- **OpenShift cluster** with MTV operator installed in the `openshift-mtv` namespace
- **Source provider** with pre-configured VMs matching the test configuration
- **Provider credentials** defined in a `.providers.json` file
- **Storage class** available on the target OpenShift cluster

> **Tip:** Tests require live infrastructure and cannot be run in isolation. The test suite connects to real clusters and source providers to perform actual VM migrations.
