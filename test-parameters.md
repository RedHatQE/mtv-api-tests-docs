# Test Parameters

This page covers the `tests/tests_config/config.py` file, which defines all test configurations used by the mtv-api-tests framework. Every test class references an entry from this file through `pytest.mark.parametrize` with `indirect=True`, making it the central place for controlling what each migration test does.

## Configuration File Structure

The configuration file at `tests/tests_config/config.py` contains two sections:

1. **Global settings** -- top-level variables that apply to all tests
2. **`tests_params` dictionary** -- individual test configurations keyed by test name

### Global Settings

```python
insecure_verify_skip: str = "true"                    # SSL verification for OCP API connections
source_provider_insecure_skip_verify: str = "false"   # SSL verification for source provider
number_of_vms: int = 1                                # Number of VMs to create
check_vms_signals: bool = True                        # Enable VM signal checking
target_namespace_prefix: str = "auto"                 # Namespace naming prefix
mtv_namespace: str = "openshift-mtv"                  # MTV operator namespace
vm_name_search_pattern: str = ""                      # VM name search pattern
remote_ocp_cluster: str = ""                          # Remote OCP cluster address
snapshots_interval: int = 2                           # Seconds between precopy snapshots
mins_before_cutover: int = 5                          # Minutes to wait before warm migration cutover
plan_wait_timeout: int = 3600                         # Timeout (seconds) for plan completion
```

### tests_params Dictionary

Each entry in `tests_params` maps a test name to a configuration dictionary. Test classes reference these entries via parametrization:

```python
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_sanity_cold_mtv_migration"])],
    indirect=True,
    ids=["rhel8"],
)
class TestSanityColdMtvMigration:
    ...
```

The `indirect=True` parameter causes pytest to pass the value through the `class_plan_config` fixture, which then feeds into the `prepared_plan` fixture for processing (cloning VMs, creating snapshots, setting up hooks, etc.).

## Test Entry Anatomy

A test entry is a dictionary with two required sections and several optional feature flags:

```python
"test_name": {
    "virtual_machines": [...],    # Required: list of VM configurations
    "warm_migration": False,      # Required: migration mode flag
    # ... optional feature flags
}
```

## Virtual Machine Configuration

The `virtual_machines` key holds a list of dictionaries, each describing a source VM to migrate. The simplest form requires only a `name`:

```python
"virtual_machines": [
    {"name": "mtv-tests-rhel8"},
]
```

### VM Configuration Options Reference

| Option                 | Type          | Required | Default   | Description                                       |
| ---------------------- | ------------- | -------- | --------- | ------------------------------------------------- |
| `name`                 | `str`         | Yes      | --        | Source VM or template name in the source provider  |
| `source_vm_power`      | `str`         | No       | unchanged | Power state before migration: `"on"` or `"off"`   |
| `guest_agent`          | `bool`        | No       | `False`   | Whether the VM has QEMU guest agent installed      |
| `clone`                | `bool`        | No       | `False`   | Clone the VM before migration (VMware only)        |
| `clone_name`           | `str`         | No       | auto      | Custom name for the cloned VM                      |
| `preserve_name_format` | `bool`        | No       | `False`   | Keep custom clone_name without sanitization         |
| `disk_type`            | `str`         | No       | provider  | Disk provisioning type for all disks during clone  |
| `add_disks`            | `list[dict]`  | No       | `[]`      | Additional disks to attach during cloning          |
| `snapshots`            | `int`         | No       | `0`       | Number of snapshots to create before migration     |
| `target_datastore_id`  | `str`         | No       | source    | Override target datastore for the cloned VM        |

### name

The name of the VM or template in the source provider. This is the only required field. For clone-based tests, this is the source template to clone from.

```python
{"name": "mtv-tests-rhel8"}
```

### source_vm_power

Controls the VM power state before migration begins. When omitted, the VM is left in whatever state it is currently in.

```python
# Power on before migration (required for warm migration)
{"name": "mtv-tests-rhel8", "source_vm_power": "on"}

# Power off before migration (common for copy-offload tests)
{"name": "xcopy-template-test", "source_vm_power": "off"}
```

The `prepared_plan` fixture handles power state changes in `conftest.py`:

```python
source_vm_power = vm.get("source_vm_power")
if source_vm_power == "on":
    source_provider.start_vm(provider_vm_api)
elif source_vm_power == "off":
    source_provider.stop_vm(provider_vm_api)
```

### guest_agent

Indicates whether the VM has QEMU guest agent installed. This affects post-migration validation -- when `True`, the framework verifies that the guest agent is running on the migrated VM.

```python
{"name": "mtv-tests-rhel8", "guest_agent": True}
```

### clone

When `True`, the framework clones the source VM/template before migration instead of migrating the original. This is essential for copy-offload tests and any scenario where the original VM must remain untouched.

```python
{
    "name": "xcopy-template-test",
    "clone": True,
    "source_vm_power": "off",
}
```

> **Note:** Cloning is currently implemented for VMware (vSphere) and OpenStack providers. The clone process automatically generates a unique name using the `session_uuid` and handles MAC address regeneration to prevent conflicts.

### clone_name and preserve_name_format

By default, cloned VMs receive a sanitized, auto-generated name. Use `clone_name` to specify a custom name and `preserve_name_format` to retain non-standard characters (capitals, underscores) -- useful for testing how MTV handles non-conforming VM names.

```python
{
    "name": "xcopy-template-test",
    "clone_name": "XCopy_Test_VM_CAPS",
    "preserve_name_format": True,
    "clone": True,
}
```

When `preserve_name_format` is `False` (the default), clone names are sanitized to be Kubernetes-compatible (lowercase, hyphens only). When `True`, the original format is preserved with only a UUID prefix and random suffix appended.

### disk_type

Sets the disk provisioning type for **all disks** during the clone operation. This maps directly to vSphere disk transformation options:

| Value          | vSphere Transform    | Description                |
| -------------- | -------------------- | -------------------------- |
| `"thin"`       | `sparse`             | Thin provisioned           |
| `"thick-lazy"` | `flat`               | Thick lazy-zeroed          |
| `"thick-eager"`| `eagerZeroedThick`   | Thick eager-zeroed         |

```python
{
    "name": "xcopy-template-test",
    "clone": True,
    "disk_type": "thin",
}
```

> **Note:** `disk_type` applies to the VM's existing disks during the clone operation. For additional disks added via `add_disks`, each disk specifies its own `provision_type` independently.

### add_disks

A list of dictionaries defining additional disks to attach to the VM during cloning. Each disk dictionary supports the following keys:

| Key               | Type   | Required | Description                                                |
| ----------------- | ------ | -------- | ---------------------------------------------------------- |
| `size_gb`         | `int`  | Yes*     | Disk size in gigabytes                                     |
| `disk_mode`       | `str`  | Yes*     | Disk mode (see table below)                                |
| `provision_type`  | `str`  | Yes*     | Provisioning type: `"thin"`, `"thick-lazy"`, `"thick-eager"` |
| `datastore_path`  | `str`  | No       | Custom folder on the datastore for the VMDK file           |
| `datastore_id`    | `str`  | No       | MoRef ID or keyword for a specific target datastore        |
| `rdm_type`        | `str`  | No       | Set to `"virtual"` for RDM disks                           |

\* Not required for RDM disks (`rdm_type` is used instead).

**Disk modes:**

| Mode                          | Description                                                    |
| ----------------------------- | -------------------------------------------------------------- |
| `"persistent"`                | Standard persistent disk -- changes are immediately written     |
| `"independent_persistent"`    | Independent disk -- not affected by snapshots, changes persist  |
| `"independent_nonpersistent"` | Independent disk -- changes discarded on power-off or reset     |

**Regular disk example:**

```python
"add_disks": [
    {"size_gb": 30, "disk_mode": "persistent", "provision_type": "thick-lazy"},
    {"size_gb": 50, "disk_mode": "persistent", "provision_type": "thin"},
]
```

**Disk on a custom datastore path:**

```python
"add_disks": [
    {
        "size_gb": 30,
        "disk_mode": "persistent",
        "provision_type": "thick-lazy",
        "datastore_path": "shared_disks",
    },
]
```

**Disk on a different datastore (multi-datastore test):**

```python
"add_disks": [
    {
        "size_gb": 30,
        "disk_mode": "persistent",
        "provision_type": "thin",
        "datastore_id": "secondary_datastore_id",
    },
]
```

The `datastore_id` value can be a literal vSphere MoRef ID or one of these special keywords that resolve from the provider's `copyoffload` config:
- `"secondary_datastore_id"` -- secondary XCOPY-capable datastore
- `"non_xcopy_datastore_id"` -- datastore without XCOPY support (for fallback tests)

**RDM (Raw Device Mapping) disk:**

```python
"add_disks": [
    {"rdm_type": "virtual"},
]
```

> **Note:** RDM disks do not specify `size_gb` or `provision_type`. The LUN UUID is sourced from the provider's `copyoffload.rdm_lun_uuid` configuration. RDM disks are added **after** the clone completes, unlike regular disks which are included in the clone specification.

### snapshots

Number of snapshots to create on the source VM before migration begins. Used to test migration of VMs with snapshot chains.

```python
{
    "name": "xcopy-template-test",
    "clone": True,
    "disk_type": "thin",
    "snapshots": 2,
}
```

Snapshot data is captured in the `prepared_plan` fixture under `source_vms_data` and `snapshots_before_migration`, and is validated during post-migration checks.

### target_datastore_id

Overrides the target datastore for the entire cloned VM (as opposed to `add_disks[].datastore_id` which applies to individual additional disks). Supports the same special keywords as `add_disks[].datastore_id`.

```python
{
    "name": "xcopy-template-test",
    "clone": True,
    "target_datastore_id": "non_xcopy_datastore_id",
    "disk_type": "thin",
}
```

## Feature Flags

Feature flags are top-level keys in the test entry dictionary that control migration behavior and enable specific MTV Plan CR features.

### warm_migration

**Required.** Controls whether the migration uses warm (live) or cold mode.

- `True` -- Warm migration with precopy snapshots and cutover. Requires `source_vm_power: "on"`.
- `False` -- Cold migration (standard offline transfer).

```python
# Warm migration
"test_sanity_warm_mtv_migration": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "source_vm_power": "on", "guest_agent": True},
    ],
    "warm_migration": True,
}

# Cold migration
"test_sanity_cold_mtv_migration": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "guest_agent": True},
    ],
    "warm_migration": False,
}
```

> **Tip:** Warm migration automatically enables Change Block Tracking (CBT) on VMware VMs during the clone process. The global `snapshots_interval` and `mins_before_cutover` settings control the precopy cadence and cutover timing.

### copyoffload

Enables the vSphere XCOPY copy-offload path for migration. When `True`, the Plan CR is configured with copy-offload-specific settings, including a fixed PVC naming template (`"pvc"`) and waiting for Forklift to create the plan-specific secret.

```python
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
}
```

> **Warning:** Copy-offload tests require provider-level configuration (`copyoffload` section in source provider config) with datastore IDs, credentials, and optional RDM LUN UUIDs. The test will fail if this provider configuration is missing.

### target_power_state

Sets the desired power state for VMs after migration completes. Passed directly to the Plan CR.

```python
"target_power_state": "on"   # VM powered on after migration
"target_power_state": "off"  # VM powered off after migration
```

### preserve_static_ips

When `True`, instructs MTV to preserve static IP addresses during migration. The migrated VM retains its original network configuration.

```python
"preserve_static_ips": True
```

### vm_target_namespace

Deploys migrated VMs into a custom namespace instead of the default test namespace. The `prepared_plan` fixture automatically creates this namespace if it does not exist.

```python
"vm_target_namespace": "custom-vm-namespace"
```

### multus_namespace

Creates Network Attachment Definitions (NADs) in a different namespace than the VMs, enabling cross-namespace network access testing.

```python
"multus_namespace": "default"
```

### pvc_name_template and pvc_name_template_use_generate_name

Control PVC naming for migrated VM disks using Go template syntax. These are passed directly to the Plan CR's `pvcNameTemplate` field.

Available template variables:
- `{{.VmName}}` -- the VM name
- `{{.DiskIndex}}` -- zero-based disk index
- `{{.FileName}}` -- source disk filename

```python
# Template with VM name and disk index
"pvc_name_template": "{{.VmName}}-disk-{{.DiskIndex}}"
"pvc_name_template_use_generate_name": False

# Template with filename processing and random suffix
"pvc_name_template": '{{ .FileName | trimSuffix ".vmdk" | replace "_" "-" }}-{{.DiskIndex}}'
"pvc_name_template_use_generate_name": True
```

When `pvc_name_template_use_generate_name` is `True`, Kubernetes appends a random suffix to the PVC name (using `generateName` instead of `name`).

> **Note:** When `copyoffload` is `True`, the PVC name template is automatically overridden to `"pvc"` regardless of any configured value.

### target_labels

Custom labels to apply to migrated VM metadata. Values set to `None` are auto-generated using the test session's `session_uuid`, ensuring uniqueness across parallel test runs.

```python
"target_labels": {
    "mtv-comprehensive-test": None,        # Becomes the session_uuid value
    "static-label": "static-value",        # Used as-is
}
```

The `target_vm_labels` fixture in `conftest.py` resolves `None` values:

```python
for label_key, config_value in target_labels.items():
    label_value = session_uuid if config_value is None else config_value
    vm_labels[label_key] = label_value
```

### target_node_selector

Kubernetes node selector labels for scheduling migrated VMs to specific nodes. Like `target_labels`, `None` values are replaced with the `session_uuid`.

```python
"target_node_selector": {
    "mtv-comprehensive-node": None,    # Auto-generated with session_uuid
}
```

The `labeled_worker_node` fixture uses Prometheus to select the worker node with the most available memory, then applies the label using a `ResourceEditor` context manager that automatically cleans up on teardown.

### target_affinity

Kubernetes pod affinity/anti-affinity rules for migrated VMs. Passed directly to the Plan CR as a standard Kubernetes affinity specification.

```python
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
}
```

### guest_agent_timeout

Override the default timeout (in seconds) for waiting for the guest agent to become ready on migrated VMs. Useful for scale tests where many VMs take longer to boot.

```python
"guest_agent_timeout": 600    # 10-minute timeout for scale tests
```

### Hook Configuration: pre_hook and post_hook

Configure pre-migration and post-migration hooks. Each hook is a dictionary with an `expected_result` key indicating whether the hook should succeed or fail.

```python
"pre_hook": {"expected_result": "succeed"},    # Hook must succeed
"post_hook": {"expected_result": "fail"},      # Hook expected to fail
```

The `prepared_plan` fixture calls `create_hook_if_configured()` to create the Hook CR resources and stores references as `_pre_hook_name`, `_pre_hook_namespace`, `_post_hook_name`, and `_post_hook_namespace` in the plan dictionary.

### expected_migration_result

Used in conjunction with hook configuration to indicate the overall expected outcome of the migration. When set to `"fail"`, the test wraps the migration execution in `pytest.raises(MigrationPlanExecError)`.

```python
"test_post_hook_retain_failed_vm": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "source_vm_power": "on", "guest_agent": True},
    ],
    "warm_migration": False,
    "target_power_state": "off",
    "pre_hook": {"expected_result": "succeed"},
    "post_hook": {"expected_result": "fail"},
    "expected_migration_result": "fail",
}
```

## Complete Examples

### Minimal Cold Migration

The simplest possible test configuration:

```python
"test_sanity_cold_mtv_migration": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8", "guest_agent": True},
    ],
    "warm_migration": False,
}
```

### Multi-VM Cold Migration

Migrating multiple VMs in a single plan:

```python
"test_cold_remote_ocp": {
    "virtual_machines": [
        {"name": "mtv-tests-rhel8"},
        {"name": "mtv-win2019-79"},
    ],
    "warm_migration": False,
}
```

### Copy-Offload with Mixed Disk Types

Testing XCOPY with 10 additional disks alternating between thin and thick provisioning:

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
                # ... repeated for 10 total disks
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
}
```

### Comprehensive Feature Test

Combining multiple Plan CR features in a single test:

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
}
```

### Scale Test with Extended Timeout

Five identical VMs with a longer guest agent timeout:

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
        # ... repeated 5 times
    ],
    "warm_migration": False,
    "copyoffload": True,
    "guest_agent_timeout": 600,
}
```

## Data Flow: From Config to Plan CR

Understanding how test parameters flow through the system:

```
config.py                    conftest.py                      test file
┌──────────────┐     ┌─────────────────────┐     ┌──────────────────────┐
│ tests_params │────>│ class_plan_config    │────>│ @pytest.mark.        │
│   ["test_x"] │     │   (raw config)      │     │   parametrize(       │
└──────────────┘     └─────────┬───────────┘     │   "class_plan_config"│
                               │                  │   indirect=True)     │
                               v                  └──────────────────────┘
                     ┌─────────────────────┐
                     │ prepared_plan        │
                     │  - deep copies config│
                     │  - clones VMs        │
                     │  - sets power state  │
                     │  - creates snapshots │
                     │  - creates hooks     │
                     │  - adds source_vms_  │
                     │    data {}           │
                     │  - resolves _vm_     │
                     │    target_namespace  │
                     └─────────┬───────────┘
                               │
                               v
                     ┌─────────────────────┐
                     │ test_create_plan()   │
                     │  Reads from          │
                     │  prepared_plan to    │
                     │  call create_plan_   │
                     │  resource()          │
                     └─────────────────────┘
```

The `prepared_plan` fixture enriches the raw configuration with runtime data:

- **`source_vms_data`** -- a dictionary keyed by VM name containing provider-specific VM details, snapshot data, and the provider API object
- **`_vm_target_namespace`** -- resolved target namespace (custom or default)
- **`_pre_hook_name` / `_post_hook_name`** -- hook CR names (if hooks are configured)
- **Updated `virtual_machines[].name`** -- cloned VMs have their names updated to the actual clone name

> **Warning:** Test methods should access configuration values directly from `prepared_plan` using dictionary key access (e.g., `prepared_plan["warm_migration"]`), not with `.get()` and defaults. This follows the project's "No Defaults for Our Config" rule -- missing keys should raise `KeyError` immediately rather than silently using a fallback.

## Adding a New Test Entry

1. Add your configuration to `tests/tests_config/config.py`:

```python
tests_params: dict = {
    # ... existing entries ...
    "test_my_new_migration": {
        "virtual_machines": [
            {
                "name": "my-source-vm",
                "source_vm_power": "on",
                "guest_agent": True,
            },
        ],
        "warm_migration": False,
    },
}
```

2. Reference it in your test file:

```python
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_my_new_migration"])],
    indirect=True,
    ids=["my-new-migration"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
@pytest.mark.incremental
class TestMyNewMigration:
    ...
```

> **Tip:** The `ids` parameter in `pytest.param` controls how the test appears in pytest output. Use short, descriptive kebab-case identifiers.
