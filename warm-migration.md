# Warm Migration

Warm migration enables virtual machine migration with minimal downtime by performing incremental data synchronization while the source VM remains running. Unlike cold migration where the VM is shut down for the entire data transfer, warm migration copies data in the background and only requires a brief cutover window for final synchronization.

## Overview

Warm migration operates in three phases:

1. **Precopy Phase** -- The source VM continues running while MTV takes periodic snapshots and incrementally syncs changed disk blocks to the destination. Change Block Tracking (CBT) on the source hypervisor identifies which blocks have been modified between snapshots.
2. **Cutover Phase** -- At a scheduled cutover time, the source VM is stopped, a final incremental sync transfers the remaining dirty blocks, and the destination VM is started.
3. **Validation Phase** -- Post-migration checks verify the migrated VM is running correctly on the destination cluster.

```
Source VM (running)          Destination (OCP)
      │                            │
      ├── Snapshot 1 ────────────► Initial data copy
      │   (VM stays on)            │
      ├── Snapshot 2 ────────────► Incremental sync (changed blocks only)
      │   (VM stays on)            │
      ├── Snapshot N ────────────► Incremental sync
      │                            │
      ├── CUTOVER ─────────────► Final sync + VM start
      │   (VM powered off)         │
      └──────────────────────────► VM running on OCP
```

### Supported Source Providers

Warm migration is supported for **VMware (vSphere)** and **RHV** source providers only. OpenStack, OpenShift, and OVA providers are not supported.

From `tests/test_mtv_warm_migration.py`:

```python
pytestmark = [
    pytest.mark.skipif(
        _SOURCE_PROVIDER_TYPE
        in (Provider.ProviderType.OPENSTACK, Provider.ProviderType.OPENSHIFT, Provider.ProviderType.OVA),
        reason=f"{_SOURCE_PROVIDER_TYPE} warm migration is not supported.",
    ),
]
```

> **Note:** RHV warm migration has a known issue tracked in Jira as MTV-2846. Tests are conditionally skipped when this issue is unresolved.

## Configuration

### Global Parameters

Three global parameters in `tests/tests_config/config.py` control warm migration behavior:

```python
snapshots_interval: int = 2       # Precopy snapshot interval in seconds
mins_before_cutover: int = 5      # Minutes before automatic cutover
plan_wait_timeout: int = 3600     # Migration timeout (1 hour)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `snapshots_interval` | `2` | Frequency (in seconds) at which the ForkliftController takes precopy snapshots during the incremental sync phase |
| `mins_before_cutover` | `5` | Number of minutes in the future to schedule the cutover when `get_cutover_value()` is called |
| `plan_wait_timeout` | `3600` | Maximum time (in seconds) to wait for a migration plan to complete |

### Test Configuration

Each warm migration test is configured in `tests/tests_config/config.py` under `tests_params`. The `warm_migration: True` flag is the key differentiator from cold migration tests.

**Basic warm migration test:**

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

> **Warning:** Source VMs for warm migration **must** have `source_vm_power: "on"`. The VM must be running for Change Block Tracking to function and for incremental snapshots to capture changed blocks.

**Multi-disk/multi-NIC warm migration:**

```python
"test_mtv_migration_warm_2disks2nics": {
    "virtual_machines": [
        {
            "name": "mtv-rhel8-warm-2disks2nics",
            "source_vm_power": "on",
            "guest_agent": True,
        },
    ],
    "warm_migration": True,
},
```

**Warm migration to a remote OCP cluster:**

```python
"test_warm_remote_ocp": {
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

### Warm vs. Cold Configuration Comparison

| Config Key | Cold Migration | Warm Migration |
|------------|---------------|----------------|
| `warm_migration` | `False` | `True` |
| `source_vm_power` | Can be `"off"` or omitted | Must be `"on"` |
| Cutover parameter | Not used | `cut_over=get_cutover_value()` |
| Precopy fixture | Not required | `precopy_interval_forkliftcontroller` |
| Change Block Tracking | Not enabled | Enabled automatically (`enable_ctk`) |
| Resource name suffix | `-cold` | `-warm` |

## Incremental Data Sync

### Change Block Tracking (CBT)

For VMware source providers, Change Block Tracking is automatically enabled on cloned VMs during the `prepared_plan` fixture setup. This allows the hypervisor to track which disk blocks change between snapshots, enabling efficient incremental transfers.

From `conftest.py`:

```python
warm_migration = plan.get("warm_migration", False)

# ...

for vm in virtual_machines:
    # Add enable_ctk flag for warm migrations
    clone_options = {**vm, "enable_ctk": warm_migration}
    provider_vm_api = source_provider.get_vm_by_name(
        query=vm["name"],
        vm_name_suffix=vm_name_suffix,
        clone_vm=True,
        session_uuid=fixture_store["session_uuid"],
        clone_options=clone_options,
    )
```

The VMware provider implementation enables CBT at both the VM level and per-disk level:

- **VM-level CBT:** Sets `ctkEnabled = "true"` in the VM's `extraConfig`
- **Per-disk CBT:** Sets `scsi{bus}:{unit}.ctkEnabled = "true"` for each virtual disk, including newly added disks

### Precopy Interval Configuration

The `precopy_interval_forkliftcontroller` fixture configures the MTV ForkliftController to perform incremental snapshots at a defined interval. This is a **session-scoped** fixture that patches the ForkliftController CR.

From `conftest.py`:

```python
@pytest.fixture(scope="session")
def precopy_interval_forkliftcontroller(ocp_admin_client, mtv_namespace):
    """Set the snapshots interval in the forklift-controller ForkliftController"""
    forklift_controller = ForkliftController(
        client=ocp_admin_client,
        name="forklift-controller",
        namespace=mtv_namespace,
        ensure_exists=True,
    )

    snapshots_interval = py_config["snapshots_interval"]
    forklift_controller.wait_for_condition(
        status=forklift_controller.Condition.Status.TRUE,
        condition=forklift_controller.Condition.Type.RUNNING,
        timeout=300,
    )

    LOGGER.info(
        f"Updating forklift-controller ForkliftController CR with "
        f"snapshots interval={snapshots_interval} seconds"
    )

    with ResourceEditor(
        patches={
            forklift_controller: {
                "spec": {
                    "controller_precopy_interval": str(snapshots_interval),
                }
            }
        }
    ):
        forklift_controller.wait_for_condition(
            status=forklift_controller.Condition.Status.TRUE,
            condition=forklift_controller.Condition.Type.SUCCESSFUL,
            timeout=300,
        )

        yield
```

> **Tip:** The `ResourceEditor` context manager ensures the ForkliftController configuration is restored to its original state after the test session completes.

## Cutover Scheduling

The cutover time determines when the source VM is powered off and the final data sync begins. The `get_cutover_value()` function in `utilities/migration_utils.py` calculates this timestamp:

```python
def get_cutover_value(current_cutover: bool = False) -> datetime:
    datetime_utc = datetime.now(pytz.utc)
    if current_cutover:
        return datetime_utc

    return datetime_utc + timedelta(minutes=int(py_config["mins_before_cutover"]))
```

| Parameter | Behavior |
|-----------|----------|
| `current_cutover=False` (default) | Returns `now + mins_before_cutover` (scheduled cutover) |
| `current_cutover=True` | Returns current UTC time (immediate cutover) |

The cutover value is passed to `execute_migration()` which creates the Migration CR with the cutover timestamp:

```python
def execute_migration(
    ocp_admin_client: DynamicClient,
    fixture_store: dict[str, Any],
    plan: Plan,
    target_namespace: str,
    cut_over: datetime | None = None,
) -> None:
    """Create Migration CR and wait for completion."""
    create_and_store_resource(
        client=ocp_admin_client,
        fixture_store=fixture_store,
        resource=Migration,
        namespace=target_namespace,
        plan_name=plan.name,
        plan_namespace=plan.namespace,
        cut_over=cut_over,
    )

    wait_for_migration_complate(plan=plan)
```

> **Note:** For cold migrations, `cut_over` is `None` (not passed). For warm migrations, it is always set via `get_cutover_value()`.

## Resource Naming

Plan and Migration resources automatically receive a `-warm` or `-cold` suffix to distinguish migration types. This is handled in `utilities/resources.py`:

```python
if resource.kind in (Migration.kind, Plan.kind):
    _resource_name = f"{_resource_name}-{'warm' if kwargs.get('warm_migration') else 'cold'}"
```

VM names also include the migration type in their suffix via `get_vm_suffix()` in `utilities/mtv_migration.py`:

```python
def get_vm_suffix(warm_migration: bool) -> str:
    migration_type = "warm" if warm_migration else "cold"
    storage_class = py_config.get("storage_class", "")
    storage_class_name = "-".join(storage_class.split("-")[-2:])
    ocp_version = py_config.get("target_ocp_version", "").replace(".", "-")
    vm_suffix = f"-{storage_class_name}-{ocp_version}-{migration_type}"

    if len(vm_suffix) > 63:
        LOGGER.warning(f"VM suffix '{vm_suffix}' is too long ({len(vm_suffix)} > 63). Truncating.")
        vm_suffix = vm_suffix[-63:]

    return vm_suffix
```

This produces suffixes like `-ceph-rbd-4-16-warm` appended to cloned VM names.

## Warm Migration Tests

### Test Structure

All warm migration tests follow the standard 5-step incremental test pattern with two key additions:

1. The `precopy_interval_forkliftcontroller` fixture is requested via `@pytest.mark.usefixtures()`
2. The `test_migrate_vms` step passes `cut_over=get_cutover_value()` to `execute_migration()`

### TestSanityWarmMtvMigration

Basic warm migration smoke test. Migrates a single RHEL8 VM with the guest agent installed.

**File:** `tests/test_mtv_warm_migration.py`
**Markers:** `@pytest.mark.tier0`, `@pytest.mark.warm`, `@pytest.mark.incremental`

```python
@pytest.mark.tier0
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [
        pytest.param(
            py_config["tests_params"]["test_sanity_warm_mtv_migration"],
        )
    ],
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

The critical difference in `test_create_plan` is passing the `warm_migration` flag:

```python
def test_create_plan(self, prepared_plan, fixture_store, ocp_admin_client,
                     source_provider, destination_provider, target_namespace,
                     source_provider_inventory):
    populate_vm_ids(plan=prepared_plan, inventory=source_provider_inventory)

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

And in `test_migrate_vms`, the cutover is scheduled:

```python
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

### TestMtvMigrationWarm2disks2nics

Tests warm migration for a VM with multiple disks and multiple network interfaces, validating that CBT and incremental sync work correctly across all disk devices.

**File:** `tests/test_mtv_warm_migration.py`
**Markers:** `@pytest.mark.warm`, `@pytest.mark.incremental`
**Config key:** `test_mtv_migration_warm_2disks2nics`

### TestWarmRemoteOcp

Tests warm migration to a remote OCP cluster. This test is conditionally skipped when no remote cluster is configured.

**File:** `tests/test_mtv_warm_migration.py`
**Markers:** `@pytest.mark.remote`, `@pytest.mark.incremental`
**Config key:** `test_warm_remote_ocp`

```python
@pytest.mark.skipif(
    not get_value_from_py_config("remote_ocp_cluster"),
    reason="No remote OCP cluster provided"
)
```

> **Note:** The remote warm migration test uses `destination_ocp_provider` instead of `destination_provider` in its fixture parameters, as the migration targets a separate OCP cluster.

## Comprehensive Warm Migration Test

The `TestWarmMigrationComprehensive` class validates multiple advanced MTV 2.10.0+ features in a single warm migration. This test exercises the full breadth of Plan CR options alongside warm migration.

**File:** `tests/test_warm_migration_comprehensive.py`
**Markers:** `@pytest.mark.tier0`, `@pytest.mark.warm`, `@pytest.mark.incremental`

### Features Tested

| Feature | Config Key | Description |
|---------|-----------|-------------|
| Static IP preservation | `preserve_static_ips` | Retains VM IP addresses after migration |
| Custom VM namespace | `vm_target_namespace` | Deploys VMs to a namespace different from the Plan namespace |
| PVC naming template | `pvc_name_template` | Custom Go template for PVC names |
| PVC generateName | `pvc_name_template_use_generate_name` | Uses Kubernetes `generateName` for PVC creation |
| Target VM labels | `target_labels` | Applies custom metadata labels to migrated VMs |
| Target VM affinity | `target_affinity` | Sets pod affinity/anti-affinity scheduling rules |
| Target power state | `target_power_state` | Controls VM power state after migration |

### Configuration

```python
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
    "multus_namespace": "default",
    "pvc_name_template": '{{ .FileName | trimSuffix ".vmdk" | replace "_" "-" }}-{{.DiskIndex}}',
    "pvc_name_template_use_generate_name": True,
    "target_labels": {
        "mtv-comprehensive-test": None,      # None = auto-generate with session_uuid
        "static-label": "static-value",
    },
    "target_affinity": {
        "podAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [
                {
                    "podAffinityTerm": {
                        "labelSelector": {"matchLabels": {"app": "comprehensive-test"}},
                        "topologyKey": "kubernetes.io/hostname",
                    },
                    "weight": 75,
                }
            ]
        }
    },
},
```

### Static IP Preservation

When `preserve_static_ips: True` is set, the Plan CR instructs MTV to preserve static IP addresses during migration. This is particularly important for Windows VMs with statically configured network interfaces.

```python
self.__class__.plan_resource = create_plan_resource(
    # ...
    preserve_static_ips=prepared_plan["preserve_static_ips"],
    # ...
)
```

### Custom VM Target Namespace

The `vm_target_namespace` parameter deploys migrated VMs to a different namespace than where the Plan CR resides. The `prepared_plan` fixture automatically creates this namespace if it does not exist:

```python
vm_target_namespace = plan.get("vm_target_namespace")
if vm_target_namespace:
    LOGGER.info(f"Using custom VM target namespace: {vm_target_namespace}")
    get_or_create_namespace(
        fixture_store=fixture_store,
        ocp_admin_client=ocp_admin_client,
        namespace_name=vm_target_namespace,
    )
    plan["_vm_target_namespace"] = vm_target_namespace
else:
    plan["_vm_target_namespace"] = target_namespace
```

### PVC Naming Templates

Two parameters control PVC naming:

- **`pvc_name_template`** -- A Go template string that defines how PVC names are generated. Available template variables include `.FileName`, `.DiskIndex`, and `.VmName`.
- **`pvc_name_template_use_generate_name`** -- When `True`, Kubernetes `generateName` is used instead of `name`, appending a random suffix to avoid collisions.

```python
"pvc_name_template": '{{ .FileName | trimSuffix ".vmdk" | replace "_" "-" }}-{{.DiskIndex}}',
"pvc_name_template_use_generate_name": True,
```

### Target VM Labels

The `target_labels` configuration supports both static values and auto-generated values using the session UUID. Labels with a `None` value are replaced with the `session_uuid` at runtime by the `target_vm_labels` fixture:

```python
@pytest.fixture(scope="class")
def target_vm_labels(prepared_plan: dict[str, Any], fixture_store: dict[str, Any]) -> dict[str, Any]:
    target_labels = prepared_plan["target_labels"]
    session_uuid = fixture_store["session_uuid"]

    vm_labels = {}
    for label_key, config_value in target_labels.items():
        # None means auto-generate using session_uuid, otherwise use provided value
        label_value = session_uuid if config_value is None else config_value
        vm_labels[label_key] = label_value

    LOGGER.info(f"Generated VM labels: {vm_labels}")
    return {"vm_labels": vm_labels}
```

Given the config `{"mtv-comprehensive-test": None, "static-label": "static-value"}`, the generated labels would be:

```python
{
    "mtv-comprehensive-test": "auto-abc123",   # session_uuid
    "static-label": "static-value",            # unchanged
}
```

### Target VM Affinity Rules

The `target_affinity` configuration defines Kubernetes pod affinity/anti-affinity rules applied to migrated VMs. This uses standard Kubernetes scheduling constructs:

```python
"target_affinity": {
    "podAffinity": {
        "preferredDuringSchedulingIgnoredDuringExecution": [
            {
                "podAffinityTerm": {
                    "labelSelector": {"matchLabels": {"app": "comprehensive-test"}},
                    "topologyKey": "kubernetes.io/hostname",
                },
                "weight": 75,
            }
        ]
    }
},
```

### Plan Creation with All Features

The comprehensive test passes all parameters to `create_plan_resource()`:

```python
def test_create_plan(self, prepared_plan, fixture_store, ocp_admin_client,
                     source_provider, destination_provider, target_namespace,
                     source_provider_inventory, target_vm_labels):
    populate_vm_ids(plan=prepared_plan, inventory=source_provider_inventory)

    self.__class__.plan_resource = create_plan_resource(
        ocp_admin_client=ocp_admin_client,
        fixture_store=fixture_store,
        source_provider=source_provider,
        destination_provider=destination_provider,
        storage_map=self.storage_map,
        network_map=self.network_map,
        virtual_machines_list=prepared_plan["virtual_machines"],
        target_power_state=prepared_plan["target_power_state"],
        target_namespace=target_namespace,
        warm_migration=prepared_plan["warm_migration"],
        preserve_static_ips=prepared_plan["preserve_static_ips"],
        vm_target_namespace=prepared_plan["vm_target_namespace"],
        pvc_name_template=prepared_plan["pvc_name_template"],
        pvc_name_template_use_generate_name=prepared_plan["pvc_name_template_use_generate_name"],
        target_labels=target_vm_labels["vm_labels"],
        target_affinity=prepared_plan["target_affinity"],
    )
    assert self.plan_resource, "Plan creation failed"
```

## Running Warm Migration Tests

### Using pytest Markers

Warm migration tests are tagged with the `@pytest.mark.warm` marker defined in `pytest.ini`:

```ini
markers =
    tier0: Core functionality tests (smoke tests)
    remote: Remote cluster migration tests
    warm: Warm migration tests
    copyoffload: Copy-offload (XCOPY) tests
    incremental: marks tests as incremental (xfail on previous failure)
```

Run all warm migration tests:

```bash
podman run -v /path/to/config:/config:Z \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m warm -v --tc=source_provider:vsphere-8.0.1
```

Run only tier0 warm migration tests:

```bash
podman run -v /path/to/config:/config:Z \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m "tier0 and warm" -v --tc=source_provider:vsphere-8.0.1
```

Run warm migration tests for a remote cluster:

```bash
podman run -v /path/to/config:/config:Z \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m remote -v --tc=source_provider:vsphere-8.0.1
```

> **Warning:** Tests require a live cluster with configured providers. They cannot be executed in a standalone environment. See the project README for full environment setup.

## Fixture Dependencies

The following diagram shows how warm migration fixtures connect:

```
precopy_interval_forkliftcontroller (session)
  └── Patches ForkliftController CR with controller_precopy_interval

prepared_plan (class)
  ├── Reads warm_migration flag from class_plan_config
  ├── Calls get_vm_suffix(warm_migration=True) → "-...-warm" suffix
  ├── Clones VMs with enable_ctk=True (enables Change Block Tracking)
  ├── Starts source VMs (source_vm_power="on")
  ├── Creates custom VM namespace if vm_target_namespace is set
  └── Captures snapshots_before_migration

target_vm_labels (class)
  └── Replaces None values in target_labels with session_uuid

cleanup_migrated_vms (class)
  └── Cleans up migrated VMs after test class completes
```

## Post-Migration Validation

The `test_check_vms` step invokes `check_vms()` from `utilities/post_migration.py`, which runs several warm migration-relevant validations:

- **Snapshot preservation** -- For VMware, verifies that source VM snapshots (id, name, state, create_time) are preserved after migration
- **SSH connectivity** -- Validates SSH access to migrated VMs that are powered on
- **Static IP preservation** -- For Windows VMs from VMware, verifies that static IP addresses were preserved
- **Serial number preservation** -- Validates VMware UUID mapping to OpenShift serial numbers
- **Standard checks** -- CPU, memory, network, and storage validation shared with cold migration

The comprehensive test also passes `target_vm_labels` to `check_vms()` for label validation:

```python
def test_check_vms(self, prepared_plan, source_provider, destination_provider,
                   source_provider_data, source_vms_namespace,
                   source_provider_inventory, vm_ssh_connections,
                   target_vm_labels):
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
        target_vm_labels=target_vm_labels,
    )
```

## Writing New Warm Migration Tests

To add a new warm migration test:

1. **Add configuration** in `tests/tests_config/config.py`:

```python
"test_my_warm_feature": {
    "virtual_machines": [
        {
            "name": "my-vm-name",
            "source_vm_power": "on",      # Required for warm
            "guest_agent": True,
        },
    ],
    "warm_migration": True,               # Required for warm
},
```

2. **Create the test class** with required markers and fixtures:

```python
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_my_warm_feature"])],
    indirect=True,
    ids=["my-warm-test"],
)
@pytest.mark.usefixtures("precopy_interval_forkliftcontroller", "cleanup_migrated_vms")
class TestMyWarmFeature:
    """Description of what this warm migration test covers."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    def test_create_storagemap(self, ...): ...
    def test_create_networkmap(self, ...): ...
    def test_create_plan(self, ...):
        self.__class__.plan_resource = create_plan_resource(
            ...,
            warm_migration=prepared_plan.get("warm_migration", False),
        )
    def test_migrate_vms(self, ...):
        execute_migration(
            ...,
            cut_over=get_cutover_value(),    # Warm migration cutover
        )
    def test_check_vms(self, ...): ...
```

Key requirements for warm migration tests:

- Include `"precopy_interval_forkliftcontroller"` in `@pytest.mark.usefixtures()`
- Pass `warm_migration=True` to `create_plan_resource()`
- Pass `cut_over=get_cutover_value()` to `execute_migration()`
- Set `source_vm_power: "on"` in VM configuration
- Add the `@pytest.mark.warm` marker for test selection
