# Copy-Offload Test Scenarios

## Overview

Copy-offload (XCOPY) is an MTV migration acceleration feature that leverages the vSphere VAAI (vStorage APIs for Array Integration) to perform direct block-level copies between vSphere datastores and OpenShift persistent volumes through the shared storage array. Instead of transferring VM disk data through an intermediary network path (as VDDK-based migration does), XCOPY instructs the storage array to copy data internally, significantly reducing migration time and network load.

The test suite validates this functionality using the `vsphere-xcopy-volume-populator` across a wide range of scenarios: different disk provisioning types, multi-disk and multi-datastore configurations, RDM virtual disks, snapshots, warm migration, concurrent execution, fallback behavior, scale testing, and edge cases like non-conforming VM names.

> **Note:** Copy-offload tests are exclusive to **vSphere** source providers. The `copyoffload_config` fixture validates this prerequisite before any test runs.

## Supported Storage Vendors

The test framework supports nine storage vendors, defined in `utilities/copyoffload_constants.py`:

```python
SUPPORTED_VENDORS = (
    "ontap",           # NetApp ONTAP
    "vantara",         # Hitachi Vantara
    "primera3par",     # HPE Primera/3PAR
    "pureFlashArray",  # Pure Storage FlashArray
    "powerflex",       # Dell PowerFlex
    "powermax",        # Dell PowerMax
    "powerstore",      # Dell PowerStore
    "infinibox",       # Infinidat InfiniBox
    "flashsystem",     # IBM FlashSystem
)
```

Each vendor may require vendor-specific credentials in addition to the base storage credentials. See [Storage Credential Configuration](#storage-credential-configuration) for details.

## Test Architecture

### Standard Test Flow

All copy-offload tests follow the project's incremental 5-step migration pattern, marked with `@pytest.mark.copyoffload` and `@pytest.mark.incremental`:

1. **`test_create_storagemap`** -- Creates a `StorageMap` with the `offloadPlugin` configuration pointing to the copy-offload storage secret
2. **`test_create_networkmap`** -- Creates a `NetworkMap` for VM network mappings
3. **`test_create_plan`** -- Creates an MTV `Plan` CR with `copyoffload=True`
4. **`test_migrate_vms`** -- Executes the migration by creating a `Migration` CR
5. **`test_check_vms`** -- Validates migrated VMs and verifies disk counts post-migration

### StorageMap Offload Plugin Configuration

The key differentiator for copy-offload tests is the `offloadPlugin` configuration applied to the StorageMap. Every copy-offload test constructs this configuration:

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

> **Note:** Copy-offload requires `access_mode="ReadWriteOnce"` and `volume_mode="Block"`. NFS datastores are not supported.

### Common Fixtures

All copy-offload test classes use these fixtures via `@pytest.mark.usefixtures(...)`:

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `copyoffload_config` | Session | Validates vSphere provider type, copyoffload section exists, and storage credentials are available |
| `copyoffload_storage_secret` | Session | Creates an OpenShift `Secret` with storage credentials for the XCOPY volume populator |
| `setup_copyoffload_ssh` | Session | Sets up SSH key on ESXi host (only when `esxi_clone_method: "ssh"`) |
| `mixed_datastore_config` | Class | Validates `non_xcopy_datastore_id` is configured (used only by mixed/fallback tests) |
| `cleanup_migrated_vms` | Class | Cleans up migrated VMs after each test class completes |

---

## Test Scenarios

### Thin and Thick Provisioning

#### TestCopyoffloadThinMigration

**Test ID:** `MTV-559:copyoffload-thin`

Validates copy-offload migration of a single thin-provisioned VM disk. This is the baseline copy-offload test.

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

**What it validates:**
- Cold migration with copy-offload for thin-provisioned disks
- StorageMap creation with `offloadPlugin` configuration
- Plan creation with `copyoffload=True`
- Post-migration VM health checks

#### TestCopyoffloadThickLazyMigration

**Test ID:** `MTV-580:copyoffload-thick-lazy`

Validates copy-offload migration of a single thick-lazy (lazy-zeroed) provisioned VM disk.

```python
"test_copyoffload_thick_lazy_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thick-lazy",
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

**What it validates:**
- Thick-lazy disk provisioning is correctly handled by the XCOPY volume populator
- No data corruption during thick-lazy disk copy-offload

---

### Multi-Disk Scenarios

#### TestCopyoffloadMultiDiskMigration

**Test ID:** `MTV-561:copyoffload-multi-disk`

Migrates a VM with the template disk plus one additional 30GB thick-lazy disk.

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

**What it validates:**
- Multiple disks are migrated via copy-offload
- Post-migration disk count matches source (`verify_vm_disk_count()`)

#### TestCopyoffloadMultiDiskDifferentPathMigration

**Test ID:** `MTV-563:copyoffload-multi-disk-different-path`

Migrates a VM where the additional disk is stored on a different datastore path (`shared_disks`).

```python
"test_copyoffload_multi_disk_different_path_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "add_disks": [
                {
                    "size_gb": 30,
                    "disk_mode": "persistent",
                    "provision_type": "thick-lazy",
                    "datastore_path": "shared_disks",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

#### TestCopyoffload10MixedDisksMigration

**Test ID:** `MTV-573:copyoffload-10-mixed-disks`

Stress-tests copy-offload with 10 additional disks alternating between thin and thick-lazy provisioning, each 10GB.

```python
"test_copyoffload_10_mixed_disks_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "add_disks": [
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thin"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thick-lazy"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thin"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thick-lazy"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thin"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thick-lazy"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thin"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thick-lazy"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thin"},
                {"size_gb": 10, "disk_mode": "persistent", "provision_type": "thick-lazy"},
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

**What it validates:**
- Copy-offload handles a high number of disks per VM
- Mixed provisioning types (thin/thick-lazy) within a single VM
- Correct disk count after migration (11 total: 1 template + 10 added)

---

### Multi-Datastore Scenarios

#### TestCopyoffloadMultiDatastoreMigration

**Test ID:** `MTV-564:copyoffload-multi-datastore`

Migrates a VM with disks spanning two XCOPY-capable datastores. The test passes both `datastore_id` and `secondary_datastore_id` to the StorageMap:

```python
"test_copyoffload_multi_datastore_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "add_disks": [
                {
                    "size_gb": 30,
                    "disk_mode": "persistent",
                    "provision_type": "thin",
                    "datastore_id": "secondary_datastore_id",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

The test creates the StorageMap with both datastores:

```python
self.__class__.storage_map = get_storage_migration_map(
    # ... standard parameters ...
    datastore_id=datastore_id,
    secondary_datastore_id=secondary_datastore_id,
    offload_plugin_config=offload_plugin_config,
    access_mode="ReadWriteOnce",
    volume_mode="Block",
)
```

> **Warning:** Both datastores must reside on the same storage array and support XCOPY/VAAI primitives for copy-offload to function.

#### TestCopyoffloadMixedDatastoreMigration

**Test ID:** `MTV-565:copyoffload-mixed-datastore`

Validates hybrid migration behavior when a VM has disks on both XCOPY-capable and non-XCOPY datastores. One disk uses XCOPY acceleration while the other falls back to standard migration.

```python
"test_copyoffload_mixed_datastore_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "add_disks": [
                {
                    "size_gb": 30,
                    "provision_type": "thin",
                    "datastore_id": "non_xcopy_datastore_id",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

This test requires the `mixed_datastore_config` fixture, which validates that `non_xcopy_datastore_id` is configured:

```python
@pytest.fixture(scope="class")
def mixed_datastore_config(source_provider_data: dict[str, Any]) -> None:
    copyoffload_config_data: dict[str, Any] = source_provider_data.get("copyoffload", {})
    non_xcopy_datastore_id: str | None = copyoffload_config_data.get("non_xcopy_datastore_id")

    if not non_xcopy_datastore_id:
        raise ValueError(
            "Mixed datastore test requires 'non_xcopy_datastore_id' to be configured in copyoffload section. "
            "This should be a datastore that does NOT support XCOPY."
        )
```

**What it validates:**
- Copy-offload correctly handles VMs with mixed disk types
- XCOPY-capable disks use accelerated transfer
- Non-XCOPY disks fall back to standard migration
- All disks are present post-migration

---

### RDM Virtual Disk Migration

#### TestCopyoffloadRdmVirtualDiskMigration

**Test ID:** `MTV-562:copyoffload-rdm-virtual`

Tests copy-offload migration of a VM with a Raw Device Mapping (RDM) virtual disk.

```python
"test_copyoffload_rdm_virtual_disk_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "add_disks": [
                {"rdm_type": "virtual"},  # LUN UUID from copyoffload.rdm_lun_uuid
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

The test explicitly validates the RDM LUN UUID before proceeding:

```python
def test_create_storagemap(self, ...):
    copyoffload_config_data = source_provider_data["copyoffload"]

    # Validate RDM LUN is configured
    if "rdm_lun_uuid" not in copyoffload_config_data or not copyoffload_config_data["rdm_lun_uuid"]:
        pytest.fail("rdm_lun_uuid is required in copyoffload configuration for RDM disk tests")
```

> **Warning:** RDM copy-offload has specific constraints:
> - Only supported with **Pure Storage** (`pureFlashArray`)
> - Requires a **VMFS** datastore (not vSAN or NFS)
> - The `rdm_lun_uuid` must be configured in the copyoffload section of `.providers.json`

---

### Snapshot Migration

The snapshot tests use a shared base class `CopyoffloadSnapshotBase` that extends the standard test flow. Before creating the StorageMap, the base class powers on the VM, creates the specified number of snapshots, records them for post-migration verification, then powers off the VM for cold migration.

```python
class CopyoffloadSnapshotBase:
    """Base class for copy-offload migration tests with snapshots."""

    def test_create_storagemap(self, ...):
        """Create StorageMap with copy-offload configuration after creating snapshots."""
        vm_cfg = prepared_plan["virtual_machines"][0]
        provider_vm_api = prepared_plan["source_vms_data"][vm_cfg["name"]]["provider_vm_api"]

        # Ensure VM is powered on for snapshot creation
        source_provider.start_vm(provider_vm_api)
        source_provider.wait_for_vmware_guest_info(provider_vm_api, timeout=60)

        snapshots_to_create = int(vm_cfg["snapshots"])
        snapshot_prefix = f"{vm_cfg['name']}-{fixture_store['session_uuid']}-snapshot"

        for idx in range(1, snapshots_to_create + 1):
            source_provider.create_snapshot(
                vm=provider_vm_api,
                name=f"{snapshot_prefix}-{idx}",
                description="mtv-api-tests copy-offload snapshots migration test",
                memory=False,
                quiesce=False,
                wait_timeout=60 * 10,
            )

        # Record snapshots for post-migration verification
        vm_cfg["snapshots_before_migration"] = source_provider.vm_dict(
            provider_vm_api=provider_vm_api
        )["snapshots_data"]

        # Cold migration expects VM powered off
        source_provider.stop_vm(provider_vm_api)

        # ... proceed with StorageMap creation ...
```

> **Note:** Snapshot tests are gated with `@pytest.mark.skipif` to run only with vSphere source providers.

#### TestCopyoffloadThinSnapshotsMigration

**Test ID:** `copyoffload-thin-snapshots`

Migrates a thin-provisioned VM with 2 snapshots.

```python
"test_copyoffload_thin_snapshots_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "snapshots": 2,
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

#### TestCopyoffload2TbVmSnapshotsMigration

**Test ID:** `MTV-575:copyoffload-2tb-vm-snapshots`

Migrates a large VM (~2TB with a 2048GB added disk) with 2 snapshots. Combines large VM migration with snapshot handling.

```python
"test_copyoffload_2tb_vm_snapshots_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "add_disks": [
                {
                    "size_gb": 2048,
                    "disk_mode": "persistent",
                    "provision_type": "thin",
                },
            ],
            "snapshots": 2,
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

---

### Disk Mode Scenarios

#### TestCopyoffloadIndependentPersistentDiskMigration

**Test ID:** `MTV-567:copyoffload-independent-persistent`

Tests copy-offload with a disk configured in `independent_persistent` mode, which is not affected by snapshots.

```python
"test_copyoffload_independent_persistent_disk_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "add_disks": [
                {
                    "size_gb": 30,
                    "disk_mode": "independent_persistent",
                    "provision_type": "thin",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

#### TestCopyoffloadIndependentNonpersistentDiskMigration

**Test ID:** `MTV-568:copyoffload-independent-nonpersistent`

Tests copy-offload with a disk configured in `independent_nonpersistent` mode, which discards changes on VM power off.

```python
"test_copyoffload_independent_nonpersistent_disk_migration": {
    "virtual_machines": [
        {
            "add_disks": [
                {
                    "size_gb": 30,
                    "disk_mode": "independent_nonpersistent",
                    "provision_type": "thin",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

---

### Warm Migration with Copy-Offload

#### TestCopyoffloadWarmMigration

**Test ID:** `MTV-577:copyoffload-warm`

Combines warm migration (live VM, incremental pre-copy) with copy-offload acceleration. The VM remains powered on during migration, and the test includes a cutover step.

```python
"test_copyoffload_warm_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "on",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
        },
    ],
    "warm_migration": True,
    "copyoffload": True,
},
```

This test class uses the additional fixture `precopy_interval_forkliftcontroller` and executes migration with a cutover value:

```python
@pytest.mark.copyoffload
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.usefixtures(
    "multus_network_name", "precopy_interval_forkliftcontroller",
    "copyoffload_config", "cleanup_migrated_vms"
)
class TestCopyoffloadWarmMigration:
    # ...
    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        """Execute warm migration with cutover."""
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
            cut_over=get_cutover_value(),
        )
```

> **Tip:** The warm+copy-offload combination is identified by both `@pytest.mark.copyoffload` and `@pytest.mark.warm` markers, making it selectable via either `-m copyoffload` or `-m warm`.

---

### Large VM Migrations

#### TestCopyoffloadLargeVmMigration

**Test ID:** `MTV-600:copyoffload-large-vm`

Tests copy-offload performance and reliability with a 1TB VM disk.

```python
"test_copyoffload_large_vm_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "add_disks": [
                {
                    "size_gb": 1024,
                    "disk_mode": "persistent",
                    "provision_type": "thin",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

---

### Fallback to Non-XCOPY

#### TestCopyoffloadFallbackLargeMigration

**Test ID:** `MTV-614:copyoffload-fallback-large`

Validates the XCOPY fallback mechanism when VM disks are located on a datastore that does **not** support VAAI XCOPY. The VM is configured with both its template disk and a 100GB added disk on the `non_xcopy_datastore_id`. The migration should complete using the XCOPY populator's fallback method.

```python
"test_copyoffload_fallback_large_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "target_datastore_id": "non_xcopy_datastore_id",
            "disk_type": "thin",
            "add_disks": [
                {
                    "size_gb": 100,
                    "provision_type": "thin",
                    "datastore_id": "non_xcopy_datastore_id",
                },
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

The test class docstring describes the expected behavior:

```python
class TestCopyoffloadFallbackLargeMigration:
    """Copy-offload migration test - large VM with disks on non-XCOPY datastore.

    This test validates copy-offload fallback functionality when VM disks are
    located on a datastore that does NOT support VAAI XCOPY acceleration.
    The VM is configured with:
    - One disk from the template (relocated to non_xcopy_datastore)
    - One additional 100GB disk (also on non_xcopy_datastore)

    The migration should complete successfully using XCOPY's fallback method
    for datastores without VAAI support.
    """
```

> **Note:** This test requires the `mixed_datastore_config` fixture to validate that `non_xcopy_datastore_id` is configured.

---

### Scale Testing

#### TestCopyoffloadScaleMigration

**Test ID:** `MTV-572:copyoffload-scale`

Tests simultaneous migration of 5 VMs, each with a 30GB additional thick-lazy disk. This validates copy-offload under concurrent load.

```python
"test_copyoffload_scale_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thick-lazy",
            "add_disks": [{"size_gb": 30, "provision_type": "thick-lazy", "disk_mode": "persistent"}],
        },
        # ... repeated 5 times ...
    ],
    "warm_migration": False,
    "copyoffload": True,
    "guest_agent_timeout": 600,
},
```

> **Tip:** The `guest_agent_timeout` is set to 600 seconds (10 minutes) instead of the default to accommodate the longer wait times with 5 VMs booting simultaneously.

---

### Concurrent Migration Scenarios

#### TestSimultaneousCopyoffloadMigrations

**Test ID:** `MTV-574:simultaneous-copyoffload`

Validates that two independent copy-offload migration plans can execute simultaneously. This test departs from the standard 5-step pattern by creating two complete sets of resources (StorageMap, NetworkMap, Plan) and then launching both migrations concurrently.

The test class manages two parallel plans:

```python
class TestSimultaneousCopyoffloadMigrations:
    """Test simultaneous execution of two copyoffload migration plans."""

    storage_map_1: StorageMap
    network_map_1: NetworkMap
    plan_resource_1: Plan

    storage_map_2: StorageMap
    network_map_2: NetworkMap
    plan_resource_2: Plan
```

The migration step creates both `Migration` CRs and then validates simultaneous execution:

```python
def test_migrate_vms_simultaneously(self, fixture_store, ocp_admin_client, target_namespace):
    # Create Migration CR for first plan
    migration_1 = create_and_store_resource(
        client=ocp_admin_client,
        fixture_store=fixture_store,
        resource=Migration,
        namespace=target_namespace,
        plan_name=self.plan_resource_1.name,
        plan_namespace=self.plan_resource_1.namespace,
        test_name="simultaneous-copyoffload-migration1",
    )

    # Create Migration CR for second plan
    migration_2 = create_and_store_resource(
        client=ocp_admin_client,
        fixture_store=fixture_store,
        resource=Migration,
        namespace=target_namespace,
        plan_name=self.plan_resource_2.name,
        plan_namespace=self.plan_resource_2.namespace,
        test_name="simultaneous-copyoffload-migration2",
    )

    # Validate both migrations are executing simultaneously
    wait_for_concurrent_migration_execution([self.plan_resource_1, self.plan_resource_2])

    # Wait for both migrations to complete
    wait_for_migration_complate(plan=self.plan_resource_1)
    wait_for_migration_complate(plan=self.plan_resource_2)
```

#### TestConcurrentXcopyVddkMigration

**Test ID:** `MTV-569:concurrent-xcopy-vddk`

Validates that an XCOPY-based migration and a VDDK-based migration can execute concurrently without interference. Plan 1 uses `copyoffload=True`, while Plan 2 uses `copyoffload=False`.

```python
class TestConcurrentXcopyVddkMigration:
    """Test simultaneous execution of XCOPY and VDDK migration plans.

    Plan 1: XCOPY based (copyoffload=True)
    Plan 2: VDDK based (copyoffload=False)
    """

    storage_map_xcopy: StorageMap
    network_map_xcopy: NetworkMap
    plan_xcopy: Plan

    storage_map_vddk: StorageMap
    network_map_vddk: NetworkMap
    plan_vddk: Plan
```

The test explicitly verifies the StorageMap configuration difference:

```python
# XCOPY StorageMap must have offloadPlugin
assert self.storage_map_xcopy.instance.spec.map[0].get("offloadPlugin"), (
    "XCOPY StorageMap missing offloadPlugin configuration"
)

# VDDK StorageMap must NOT have offloadPlugin
assert not self.storage_map_vddk.instance.spec.map[0].get("offloadPlugin"), (
    "VDDK StorageMap should NOT have offloadPlugin configuration"
)
```

---

### Non-Conforming VM Name Handling

#### TestCopyoffloadNonconformingNameMigration

**Test ID:** `MTV-579:copyoffload-nonconforming-name`

Tests that MTV properly sanitizes VM names that don't conform to Kubernetes naming conventions. The source VM is cloned with the name `XCopy_Test_VM_CAPS` (containing capitals and underscores), and the test verifies the destination VM is created with a valid Kubernetes name.

```python
"test_copyoffload_nonconforming_name_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "clone_name": "XCopy_Test_VM_CAPS",
            "preserve_name_format": True,
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

The `test_check_vms` method includes additional name sanitization verification:

```python
def test_check_vms(self, ...):
    # ... standard VM checks ...

    # Verify destination VM name sanitization
    provider_vm_api = prepared_plan["source_vms_data"][vm_cfg["name"]]["provider_vm_api"]
    actual_source_vm_name = provider_vm_api.name
    expected_destination_name = sanitize_kubernetes_name(actual_source_vm_name)

    destination_vm = destination_provider.vm_dict(
        name=expected_destination_name,
        namespace=target_namespace,
    )

    actual_destination_name = destination_vm["name"]
    assert actual_destination_name == expected_destination_name, (
        f"Destination VM name mismatch!\n"
        f"  Source VM name: '{actual_source_vm_name}'\n"
        f"  Expected sanitized name: '{expected_destination_name}'\n"
        f"  Actual destination name: '{actual_destination_name}'"
    )
```

---

## Storage Credential Configuration

### Provider Configuration Template

Copy-offload credentials are configured in the `copyoffload` section of `.providers.json`:

```json
{
  "vsphere-copy-offload": {
    "type": "vsphere",
    "copyoffload": {
      "storage_vendor_product": "ontap",
      "datastore_id": "datastore-12345",
      "secondary_datastore_id": "datastore-67890",
      "non_xcopy_datastore_id": "datastore-99999",
      "default_vm_name": "rhel9-template",
      "storage_hostname": "storage.example.com",
      "storage_username": "admin",
      "storage_password": "your-password-here",
      "ontap_svm": "vserver-name",
      "esxi_clone_method": "ssh",
      "esxi_host": "esxi.example.com",
      "esxi_user": "root",
      "esxi_password": "esxi-password",
      "rdm_lun_uuid": "naa.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  }
}
```

### Environment Variable Overrides

All copy-offload credentials support environment variable overrides with the `COPYOFFLOAD_` prefix. Environment variables take precedence over config file values.

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
    env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
    return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

| Config Key | Environment Variable |
|------------|---------------------|
| `storage_hostname` | `COPYOFFLOAD_STORAGE_HOSTNAME` |
| `storage_username` | `COPYOFFLOAD_STORAGE_USERNAME` |
| `storage_password` | `COPYOFFLOAD_STORAGE_PASSWORD` |
| `ontap_svm` | `COPYOFFLOAD_ONTAP_SVM` |
| `esxi_host` | `COPYOFFLOAD_ESXI_HOST` |
| `esxi_user` | `COPYOFFLOAD_ESXI_USER` |
| `esxi_password` | `COPYOFFLOAD_ESXI_PASSWORD` |

### Vendor-Specific Required Fields

The `copyoffload_storage_secret` fixture creates an OpenShift Secret with base and vendor-specific credentials:

| Vendor | Required Fields | Secret Keys |
|--------|----------------|-------------|
| `ontap` | `ontap_svm` | `ONTAP_SVM` |
| `vantara` | `vantara_storage_id`, `vantara_storage_port`, `vantara_hostgroup_id_list` | `STORAGE_ID`, `STORAGE_PORT`, `HOSTGROUP_ID_LIST` |
| `pureFlashArray` | `pure_cluster_prefix` | `PURE_CLUSTER_PREFIX` |
| `powerflex` | `powerflex_system_id` | `SYSTEM_ID` |
| `powermax` | `powermax_symmetrix_id` | `SYMMETRIX_ID` |
| `primera3par` | _(none)_ | _(base credentials only)_ |
| `powerstore` | _(none)_ | _(base credentials only)_ |
| `infinibox` | _(none)_ | _(base credentials only)_ |
| `flashsystem` | _(none)_ | _(base credentials only)_ |

---

## ESXi Clone Method Configuration

Copy-offload supports two methods for cloning VMs on the ESXi host:

| Method | Description | Additional Config Required |
|--------|-------------|---------------------------|
| `vib` (default) | Uses VIB-based cloning | None |
| `ssh` | Uses SSH key-based cloning | `esxi_host`, `esxi_user`, `esxi_password` |

When `esxi_clone_method` is set to `"ssh"`, the `setup_copyoffload_ssh` fixture automatically:
1. Retrieves the SSH public key from the source provider
2. Installs the key on the ESXi host
3. Removes the key on teardown

```python
@pytest.fixture(scope="session")
def setup_copyoffload_ssh(source_provider, source_provider_data, copyoffload_config):
    copyoffload_cfg = source_provider_data["copyoffload"]
    if copyoffload_cfg.get("esxi_clone_method") != "ssh":
        yield
        return

    public_key = source_provider.get_ssh_public_key()
    datastore_name = source_provider.get_datastore_name_by_id(
        copyoffload_cfg.get("datastore_id")
    )

    esxi_host = get_copyoffload_credential("esxi_host", copyoffload_cfg)
    esxi_user = get_copyoffload_credential("esxi_user", copyoffload_cfg)
    esxi_password = get_copyoffload_credential("esxi_password", copyoffload_cfg)

    install_ssh_key_on_esxi(
        host=esxi_host, username=esxi_user, password=esxi_password,
        public_key=public_key, datastore_name=datastore_name,
    )
    yield
    remove_ssh_key_from_esxi(
        host=esxi_host, username=esxi_user, password=esxi_password,
        public_key=public_key,
    )
```

---

## Test Matrix Summary

| Test Class | Test ID | Disk Type | Disks | Datastore | Snapshots | Warm | Scale | Special |
|------------|---------|-----------|-------|-----------|-----------|------|-------|---------|
| `TestCopyoffloadThinMigration` | MTV-559 | thin | 1 | primary | -- | -- | -- | Baseline |
| `TestCopyoffloadThickLazyMigration` | MTV-580 | thick-lazy | 1 | primary | -- | -- | -- | -- |
| `TestCopyoffloadMultiDiskMigration` | MTV-561 | mixed | 2 | primary | -- | -- | -- | -- |
| `TestCopyoffloadMultiDiskDifferentPathMigration` | MTV-563 | mixed | 2 | different path | -- | -- | -- | `shared_disks` |
| `TestCopyoffload10MixedDisksMigration` | MTV-573 | thin+thick | 11 | primary | -- | -- | -- | Alternating types |
| `TestCopyoffloadMultiDatastoreMigration` | MTV-564 | thin | 2 | primary+secondary | -- | -- | -- | Both XCOPY |
| `TestCopyoffloadMixedDatastoreMigration` | MTV-565 | thin | 2 | XCOPY+non-XCOPY | -- | -- | -- | Hybrid migration |
| `TestCopyoffloadRdmVirtualDiskMigration` | MTV-562 | RDM | 2 | primary (VMFS) | -- | -- | -- | Pure Storage only |
| `TestCopyoffloadThinSnapshotsMigration` | -- | thin | 1 | primary | 2 | -- | -- | -- |
| `TestCopyoffload2TbVmSnapshotsMigration` | MTV-575 | thin | 2 (2TB) | primary | 2 | -- | -- | Large VM |
| `TestCopyoffloadIndependentPersistentDiskMigration` | MTV-567 | thin | 2 | primary | -- | -- | -- | `independent_persistent` |
| `TestCopyoffloadIndependentNonpersistentDiskMigration` | MTV-568 | thin | 2 | primary | -- | -- | -- | `independent_nonpersistent` |
| `TestCopyoffloadWarmMigration` | MTV-577 | thin | 1 | primary | -- | yes | -- | VM powered on |
| `TestCopyoffloadLargeVmMigration` | MTV-600 | thin | 2 (1TB) | primary | -- | -- | -- | Performance |
| `TestCopyoffloadFallbackLargeMigration` | MTV-614 | thin | 2 (100GB) | non-XCOPY | -- | -- | -- | Fallback method |
| `TestCopyoffloadScaleMigration` | MTV-572 | thick-lazy | 2 per VM | primary | -- | -- | 5 VMs | 10min timeout |
| `TestSimultaneousCopyoffloadMigrations` | MTV-574 | thin | 1 per VM | primary | -- | -- | 2 plans | Concurrent XCOPY |
| `TestConcurrentXcopyVddkMigration` | MTV-569 | thick-lazy | 1 per VM | primary | -- | -- | 2 plans | XCOPY + VDDK |
| `TestCopyoffloadNonconformingNameMigration` | MTV-579 | thin | 1 | primary | -- | -- | -- | Name sanitization |

---

## Running Copy-Offload Tests

Select copy-offload tests using the `copyoffload` pytest marker:

```bash
# Run all copy-offload tests
uv run pytest -m copyoffload -v \
  --tc=source_provider:vsphere-8.0.3.00400 \
  --tc=storage_class:my-block-storageclass

# Run a specific test by keyword
uv run pytest -m copyoffload -k test_copyoffload_thin_migration -v

# Run warm + copy-offload tests
uv run pytest -m "copyoffload and warm" -v

# Run all copy-offload tests except scale
uv run pytest -m copyoffload -k "not scale" -v
```

> **Warning:** Tests require a live vSphere environment with shared storage, configured provider credentials, and an OpenShift cluster with MTV/Forklift installed. They cannot be run in isolation or CI without these prerequisites.

---

## Utility Functions

### `get_copyoffload_credential()`

Located in `utilities/copyoffload_migration.py`. Retrieves credentials with environment variable override support:

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
    env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
    return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

### `wait_for_plan_secret()`

Located in `utilities/copyoffload_migration.py`. Polls for the Forklift-created plan-specific secret after Plan creation:

```python
def wait_for_plan_secret(
    ocp_admin_client: DynamicClient,
    namespace: str,
    plan_name: str,
) -> None:
    try:
        for _ in TimeoutSampler(
            wait_timeout=60,
            sleep=2,
            func=lambda: any(
                s.name.startswith(f"{plan_name}-")
                for s in Secret.get(client=ocp_admin_client, namespace=namespace)
            ),
        ):
            break
    except TimeoutExpiredError:
        LOGGER.warning(f"Timeout waiting for plan secret '{plan_name}-*' - continuing anyway")
```

> **Note:** This function times out after 60 seconds but continues execution. If the secret is truly missing, the migration itself will fail with a clearer error message.
