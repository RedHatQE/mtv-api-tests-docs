# Cold Migration

Cold migration is the standard migration workflow in Migration Toolkit for Virtualization (MTV). During a cold migration, the source VM is powered off for the duration of the data transfer, ensuring a consistent disk state. This approach is simpler than warm migration, involves no incremental data copies, and is supported across all source provider types.

## Overview

In a cold migration:

1. The source VM is powered off (or is already off) before data transfer begins
2. All disk data is copied in a single pass from the source provider to the OpenShift destination
3. The migrated VM is created on the destination cluster
4. The VM is optionally powered on after migration completes

Cold migration is identified in the test configuration by setting `"warm_migration": False`. Unlike warm migration, cold migration does not use cut-over scheduling or precopy intervals.

> **Note:** Cold migration is the default migration type. There is no `@pytest.mark.cold` marker -- cold migration tests are identified by the absence of the `@pytest.mark.warm` marker.

## Migration Workflow

Every cold migration test follows a 5-step incremental pattern. Each step is a separate test method executed in order, with `@pytest.mark.incremental` ensuring that if one step fails, subsequent steps are skipped.

```
StorageMap → NetworkMap → Plan → Migrate → Check VMs
```

### Step 1: Create StorageMap

Maps source provider datastores to destination OpenShift storage classes:

```python
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

The `get_storage_migration_map()` function queries the source provider inventory to discover which datastores the VMs use, then creates a `StorageMap` CR mapping each source datastore to the configured `storage_class`.

### Step 2: Create NetworkMap

Maps source provider networks to destination OpenShift networks (pod network or Multus):

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
assert self.network_map, "NetworkMap creation failed"
```

### Step 3: Create Plan

Creates the MTV Plan CR that defines the migration configuration:

```python
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

The Plan CR references the source and destination providers, the storage and network maps, and the list of VMs to migrate. After creation, it waits up to 360 seconds for the Plan to reach `Ready` status.

### Step 4: Execute Migration

Creates a `Migration` CR that triggers the actual VM migration, then polls until completion:

```python
execute_migration(
    ocp_admin_client=ocp_admin_client,
    fixture_store=fixture_store,
    plan=self.plan_resource,
    target_namespace=target_namespace,
)
```

> **Note:** For cold migration, the `cut_over` parameter is not passed (it defaults to `None`). The `cut_over` datetime is only used for warm migrations to schedule the final switchover.

The migration status is polled via `wait_for_migration_complate()`, which monitors the Plan's conditions until it reaches `Succeeded` or `Failed`. The default timeout is controlled by the `plan_wait_timeout` config value (default: 3600 seconds).

### Step 5: Validate Migrated VMs

Runs post-migration checks on each migrated VM:

```python
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

The `check_vms()` function validates:

- **Power state** -- the destination VM matches the expected power state
- **Network configuration** -- NICs are mapped correctly per the NetworkMap
- **Storage** -- disks and PVCs are created correctly per the StorageMap
- **SSL configuration** -- provider SSL settings match the global configuration
- **SSH connectivity** -- the migrated VM is reachable (when guest agent is available)
- **Static IP preservation** -- IP addresses are retained (when configured)
- **Node placement** -- VM is scheduled on the correct node (when `target_node_selector` is configured)
- **VM labels** -- custom labels are applied to the migrated VM (when configured)
- **VM affinity** -- pod affinity/anti-affinity rules are applied (when configured)

## Test Scenarios

### Sanity Cold Migration

The basic smoke test for cold migration. It migrates a single RHEL8 VM with the guest agent installed.

**Test class:** `TestSanityColdMtvMigration` in `tests/test_mtv_cold_migration.py`

**Configuration:**

```python
"test_sanity_cold_mtv_migration": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "guest_agent": True},
    ],
    "warm_migration": False,
},
```

**Test class definition:**

```python
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
```

**Markers:**
- `tier0` -- core smoke test
- `incremental` -- tests run sequentially with dependency
- `usefixtures("cleanup_migrated_vms")` -- automatic VM cleanup after all tests in the class complete

### Comprehensive Cold Migration

Tests multiple advanced features in a single cold migration run, including static IP preservation, PVC naming templates, node selectors, VM labels, and affinity rules.

**Test class:** `TestColdMigrationComprehensive` in `tests/test_cold_migration_comprehensive.py`

**Configuration:**

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
        "mtv-comprehensive-node": None,  # None = auto-generate with session_uuid
    },
    "target_labels": {
        "mtv-comprehensive-label": None,  # None = auto-generate with session_uuid
        "test-type": "comprehensive",     # Static value
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

> **Tip:** Label and node selector values set to `None` are auto-generated with the `session_uuid` at runtime, ensuring uniqueness across parallel test executions.

**Plan creation with all comprehensive features:**

```python
self.__class__.plan_resource = create_plan_resource(
    fixture_store=fixture_store,
    source_provider=source_provider,
    destination_provider=destination_provider,
    storage_map=self.storage_map,
    network_map=self.network_map,
    ocp_admin_client=ocp_admin_client,
    target_namespace=target_namespace,
    virtual_machines_list=prepared_plan["virtual_machines"],
    target_power_state=prepared_plan["target_power_state"],
    warm_migration=prepared_plan["warm_migration"],
    preserve_static_ips=prepared_plan["preserve_static_ips"],
    pvc_name_template=prepared_plan["pvc_name_template"],
    pvc_name_template_use_generate_name=prepared_plan["pvc_name_template_use_generate_name"],
    target_node_selector={labeled_worker_node["label_key"]: labeled_worker_node["label_value"]},
    target_labels=target_vm_labels["vm_labels"],
    target_affinity=prepared_plan["target_affinity"],
    vm_target_namespace=prepared_plan["vm_target_namespace"],
)
```

#### Features Validated

| Feature | Config Key | Description |
|---------|-----------|-------------|
| Static IP preservation | `preserve_static_ips` | Retains VM IP addresses after migration |
| PVC name template | `pvc_name_template` | Controls PVC naming with Go templates (e.g., `{{.VmName}}-disk-{{.DiskIndex}}`) |
| PVC generateName | `pvc_name_template_use_generate_name` | Uses Kubernetes `generateName` for PVC naming |
| Node selector | `target_node_selector` | Schedules migrated VM to a specific labeled node |
| VM labels | `target_labels` | Applies custom labels to the migrated VM |
| VM affinity | `target_affinity` | Configures pod affinity/anti-affinity rules |
| Custom target namespace | `vm_target_namespace` | Places migrated VMs in a separate namespace |
| Target power state | `target_power_state` | Sets the VM power state after migration (`"on"` or `"off"`) |

### Remote Cluster Cold Migration

Tests cold migration to a remote OpenShift cluster (not the local cluster running the MTV operator). This validates cross-cluster migration scenarios.

**Test class:** `TestColdRemoteOcp` in `tests/test_mtv_cold_migration.py`

**Configuration:**

```python
"test_cold_remote_ocp": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8"},
        {"name": "mtv-win2019-79"},
    ],
    "warm_migration": False,
},
```

This test migrates two VMs -- a Linux (RHEL8) and a Windows (2019) VM -- to validate both OS types in a remote migration scenario.

**Key differences from local cold migration:**

```python
@pytest.mark.remote
@pytest.mark.incremental
@pytest.mark.skipif(
    not get_value_from_py_config("remote_ocp_cluster"),
    reason="No remote OCP cluster provided"
)
@pytest.mark.parametrize(
    "class_plan_config",
    [
        pytest.param(
            py_config["tests_params"]["test_cold_remote_ocp"],
        )
    ],
    indirect=True,
    ids=["MTV-79"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestColdRemoteOcp:
    """Cold migration test to remote OCP cluster."""
```

- Uses the `@pytest.mark.remote` marker
- Conditionally skipped when `remote_ocp_cluster` is not configured
- Uses `destination_ocp_provider` fixture instead of `destination_provider`

```python
# Remote migration uses destination_ocp_provider
self.__class__.storage_map = get_storage_migration_map(
    fixture_store=fixture_store,
    source_provider=source_provider,
    destination_provider=destination_ocp_provider,  # Remote cluster provider
    source_provider_inventory=source_provider_inventory,
    ocp_admin_client=ocp_admin_client,
    target_namespace=target_namespace,
    vms=vms,
)
```

> **Warning:** The `remote_ocp_cluster` configuration must be set in `py_config` for remote migration tests to run. Without it, the entire `TestColdRemoteOcp` class is skipped.

### Post-Hook Failure with VM Retention

Tests the scenario where a cold migration succeeds but the post-migration hook fails, validating that migrated VMs are retained despite the overall migration failure.

**Test class:** `TestPostHookRetainFailedVm` in `tests/test_post_hook_retain_failed_vm.py`

**Configuration:**

```python
"test_post_hook_retain_failed_vm": {
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8",
            "source_vm_power": "on",
            "guest_agent": True,
        },
    ],
    "warm_migration": False,
    "target_power_state": "off",
    "pre_hook": {"expected_result": "succeed"},
    "post_hook": {"expected_result": "fail"},
    "expected_migration_result": "fail",
},
```

This test uses two hooks:
- **Pre-hook** -- runs a successful Ansible playbook before migration
- **Post-hook** -- runs a deliberately failing Ansible playbook after migration

The migration is expected to fail overall, but since the failure occurs in the post-hook (after VM data transfer), the migrated VMs should still exist on the destination.

**Migration execution with expected failure:**

```python
def test_migrate_vms(self, prepared_plan, fixture_store, ocp_admin_client, target_namespace):
    expected_result = prepared_plan["expected_migration_result"]

    if expected_result == "fail":
        with pytest.raises(MigrationPlanExecError):
            execute_migration(
                ocp_admin_client=ocp_admin_client,
                fixture_store=fixture_store,
                plan=self.plan_resource,
                target_namespace=target_namespace,
            )
        self.__class__.should_check_vms = validate_hook_failure_and_check_vms(
            self.plan_resource, prepared_plan
        )
```

The `validate_hook_failure_and_check_vms()` function:
1. Retrieves the failed pipeline step for each VM from the Migration CR status
2. Validates all VMs failed at the same step
3. Confirms the failure matches the expected hook (`PostHook`)
4. Returns `True` if VMs should be validated (PostHook failure means data was transferred) or `False` (PreHook failure means data was never transferred)

## Configuration Reference

### VM Configuration Options

| Option | Required | Type | Values | Description |
|--------|----------|------|--------|-------------|
| `name` | Yes | `str` | VM name in source | Source VM identifier |
| `source_vm_power` | No | `str` | `"on"`, `"off"` | VM power state before migration |
| `guest_agent` | No | `bool` | `True`, `False` | Whether guest agent is installed |
| `clone` | No | `bool` | `True`, `False` | Clone VM before migration |
| `disk_type` | No | `str` | `"thin"`, `"thick-lazy"`, `"thick-eager"` | Disk provisioning type |

### Plan Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `warm_migration` | `bool` | -- | Must be `False` for cold migration |
| `target_power_state` | `str` | Source state | Power state after migration |
| `preserve_static_ips` | `bool` | `False` | Preserve IP addresses |
| `pvc_name_template` | `str` | `None` | Go template for PVC naming |
| `pvc_name_template_use_generate_name` | `bool` | `None` | Use Kubernetes `generateName` |
| `target_node_selector` | `dict` | `None` | Node scheduling labels |
| `target_labels` | `dict` | `None` | Labels for migrated VMs |
| `target_affinity` | `dict` | `None` | Pod affinity/anti-affinity rules |
| `vm_target_namespace` | `str` | `None` | Custom namespace for migrated VMs |
| `pre_hook` | `dict` | `None` | Pre-migration hook config |
| `post_hook` | `dict` | `None` | Post-migration hook config |
| `expected_migration_result` | `str` | `None` | Expected result: `"succeed"` or `"fail"` |

## Cold vs Warm Migration

| Aspect | Cold Migration | Warm Migration |
|--------|---------------|----------------|
| Marker | None (default) | `@pytest.mark.warm` |
| `warm_migration` config | `False` | `True` |
| `cut_over` parameter | Not used (`None`) | `datetime` object |
| Precopy interval | Not applicable | Set via `precopy_interval_forkliftcontroller` fixture |
| VM power during transfer | Off | On (running) |
| Data transfer | Single full copy | Incremental precopy phases + final cutover |
| Supported providers | All (VMware, RHV, OpenStack, OVA, OpenShift) | VMware, RHV |

## VM Name Suffix Convention

Migrated VMs receive a suffix that encodes the migration context:

```python
def get_vm_suffix(warm_migration: bool) -> str:
    migration_type = "warm" if warm_migration else "cold"
    storage_class = py_config.get("storage_class", "")
    storage_class_name = "-".join(storage_class.split("-")[-2:])
    ocp_version = py_config.get("target_ocp_version", "").replace(".", "-")
    vm_suffix = f"-{storage_class_name}-{ocp_version}-{migration_type}"
    return vm_suffix
```

For example, a cold migration with NFS storage on OCP 4.14 produces:
`-nfs-4-14-cold`

## Resource Cleanup

Cold migration tests use the `cleanup_migrated_vms` fixture for automatic resource teardown:

```python
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestSanityColdMtvMigration:
    ...
```

This class-scoped fixture:
- Runs after all tests in the class complete (teardown phase only)
- Deletes migrated VMs from the destination cluster using `vm_obj.clean_up()`
- Respects the `--skip-teardown` CLI flag for debugging
- Session-level teardown catches any leftover VMs not cleaned by class fixtures

All OpenShift resources (StorageMap, NetworkMap, Plan, Migration) are tracked via `create_and_store_resource()` in the `fixture_store` and cleaned up automatically during session teardown.

## Running Cold Migration Tests

> **Warning:** Tests require live clusters, provider connections, and credentials. They cannot be run in isolation.

**Run all tier0 cold migration tests:**

```bash
pytest -m "tier0 and not warm" tests/
```

**Run only the sanity cold migration test:**

```bash
pytest tests/test_mtv_cold_migration.py::TestSanityColdMtvMigration
```

**Run the comprehensive cold migration test:**

```bash
pytest tests/test_cold_migration_comprehensive.py::TestColdMigrationComprehensive
```

**Run remote cluster cold migration tests:**

```bash
pytest -m remote tests/test_mtv_cold_migration.py::TestColdRemoteOcp
```

**Run post-hook retention test:**

```bash
pytest tests/test_post_hook_retain_failed_vm.py::TestPostHookRetainFailedVm
```
