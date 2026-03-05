# Post-Migration Utilities

API reference for `utilities/post_migration.py` — the validation layer that runs after MTV migration completes. This module verifies that migrated VMs match their source configuration across CPU, memory, network, storage, power state, SSH connectivity, static IPs, PVC naming, and provider-specific attributes.

---

## Overview

The post-migration utilities provide a comprehensive validation framework invoked as the final step (`test_check_vms`) of every migration test class. The central orchestrator function `check_vms()` delegates to specialized check functions, collecting all failures and reporting them at the end.

```
check_vms()
├── check_ssl_configuration()          # Provider SSL settings
├── Per-VM checks:
│   ├── check_vms_power_state()        # Power state validation
│   ├── check_cpu()                    # CPU cores/sockets
│   ├── check_memory()                 # Memory size
│   ├── check_network()               # NIC mapping
│   ├── check_storage()               # Disk/storage class
│   ├── check_pvc_names()             # PVC naming templates
│   ├── check_snapshots()             # Snapshot preservation (vSphere)
│   ├── check_serial_preservation()   # UUID serial (vSphere)
│   ├── check_guest_agent()           # QEMU guest agent
│   ├── check_ssh_connectivity()      # SSH access
│   ├── check_static_ip_preservation()# Static IP (Windows/vSphere)
│   ├── check_vm_node_placement()     # Node selector
│   ├── check_vm_labels()             # VM labels
│   ├── check_vm_affinity()           # Affinity rules
│   └── check_false_vm_power_off()    # RHV power-off event
```

---

## check_vms()

The main orchestration function that runs all post-migration validations for every VM in the migration plan.

### Signature

```python
def check_vms(
    plan: dict[str, Any],
    source_provider: BaseProvider,
    destination_provider: BaseProvider,
    network_map_resource: NetworkMap,
    storage_map_resource: StorageMap,
    source_provider_data: dict[str, Any],
    source_vms_namespace: str,
    source_provider_inventory: ForkliftInventory | None = None,
    vm_ssh_connections: SSHConnectionManager | None = None,
    labeled_worker_node: dict[str, Any] | None = None,
    target_vm_labels: dict[str, Any] | None = None,
) -> None:
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `plan` | `dict[str, Any]` | Yes | Processed plan config from `prepared_plan` fixture |
| `source_provider` | `BaseProvider` | Yes | Source provider instance (vSphere, RHV, OpenStack, OCP) |
| `destination_provider` | `BaseProvider` | Yes | Destination OpenShift provider |
| `network_map_resource` | `NetworkMap` | Yes | NetworkMap CR created during migration |
| `storage_map_resource` | `StorageMap` | Yes | StorageMap CR created during migration |
| `source_provider_data` | `dict[str, Any]` | Yes | Provider config from `.providers.json` |
| `source_vms_namespace` | `str` | Yes | Namespace of source VMs |
| `source_provider_inventory` | `ForkliftInventory \| None` | No | Forklift inventory for VM lookups |
| `vm_ssh_connections` | `SSHConnectionManager \| None` | No | SSH connection manager for connectivity checks |
| `labeled_worker_node` | `dict[str, Any] \| None` | No | Expected worker node for placement validation |
| `target_vm_labels` | `dict[str, Any] \| None` | No | Expected labels for label validation |

### Behavior

1. **OVA provider early exit**: Returns immediately for OVA providers since OVA VMs have no source stats to compare.
2. **SSL configuration check**: For VMware, RHV, and OpenStack providers, validates the provider secret's `insecureSkipVerify` setting matches the global config.
3. **Per-VM validation loop**: Iterates each VM in `plan["virtual_machines"]` and runs all applicable checks.
4. **Error collection**: Each check runs in a `try/except` block. Failures are collected per VM and reported together at the end.
5. **Conditional checks**: Several checks only execute when their configuration is present in the plan (e.g., `pvc_name_template`, `target_node_selector`, `target_affinity`).

### Usage in Tests

Every migration test class includes `check_vms()` as its final `test_check_vms` method:

```python
from utilities.post_migration import check_vms

@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_cold_migration_comprehensive"])],
    indirect=True,
    ids=["comprehensive-cold"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
@pytest.mark.incremental
@pytest.mark.tier0
class TestColdMigrationComprehensive:
    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    # ... test_create_storagemap, test_create_networkmap, test_create_plan, test_migrate_vms ...

    def test_check_vms(
        self,
        prepared_plan,
        source_provider,
        destination_provider,
        source_provider_data,
        source_vms_namespace,
        source_provider_inventory,
        labeled_worker_node,
        target_vm_labels,
        vm_ssh_connections,
    ):
        """Validate migrated VMs with comprehensive feature configuration."""
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

> **Note:** For basic migration tests (cold, warm, copy-offload), only the required parameters are passed. Optional parameters like `labeled_worker_node` and `target_vm_labels` are used in comprehensive test classes.

---

## SSH Connectivity Checks

### check_ssh_connectivity()

Tests SSH access to a migrated VM using provider-configured credentials with automatic retry.

```python
def check_ssh_connectivity(
    vm_name: str,
    vm_ssh_connections: SSHConnectionManager,
    source_provider_data: dict[str, Any],
    source_vm_info: dict[str, Any],
    timeout: int = 300,
    retry_delay: int = 15,
) -> None:
```

**Flow:**

1. Resolves SSH credentials using `get_ssh_credentials_from_provider_config()` (Linux or Windows based on VM OS type).
2. Creates an SSH connection via `SSHConnectionManager.create()`.
3. Uses `TimeoutSampler` to retry connectivity with a 300-second default timeout and 15-second intervals.
4. Each attempt opens a context-managed SSH session and calls `is_connective()`.
5. Raises `TimeoutExpiredError` if connectivity is not established within the timeout.

**Activation condition** — `check_vms()` only calls this when:
- `vm_ssh_connections` is provided (not `None`)
- The destination VM power state is `"on"`

```python
# Inside check_vms():
if vm_ssh_connections and destination_vm.get("power_state") == "on":
    check_ssh_connectivity(
        vm_name=vm_name,
        vm_ssh_connections=vm_ssh_connections,
        source_provider_data=source_provider_data,
        source_vm_info=source_vm,
    )
```

> **Warning:** SSH connectivity checks require the `virtctl` binary to be installed and available in `PATH`. The underlying `VMSSHConnection` uses `virtctl port-forward` to tunnel SSH traffic to VMs running on OpenShift/KubeVirt.

### get_ssh_credentials_from_provider_config()

Resolves SSH username and password from the provider configuration based on the VM's OS type.

```python
def get_ssh_credentials_from_provider_config(
    source_provider_data: dict[str, Any],
    source_vm_info: dict[str, Any],
) -> tuple[str, str]:
```

**Credential lookup:**

| VM OS | Required Provider Config Keys |
|-------|-------------------------------|
| Windows (`win_os: True`) | `guest_vm_win_user`, `guest_vm_win_password` |
| Linux (default) | `guest_vm_linux_user`, `guest_vm_linux_password` |

Raises `ValueError` if the required credential keys are missing from the provider configuration.

### SSHConnectionManager

The `SSHConnectionManager` class (defined in `utilities/ssh_utils.py`) manages the lifecycle of SSH connections to migrated VMs. It is provided to tests via the `vm_ssh_connections` fixture.

```python
class SSHConnectionManager:
    def __init__(
        self,
        provider: BaseProvider,
        namespace: str,
        fixture_store: dict[str, Any],
        ocp_client: DynamicClient,
    ) -> None: ...

    def create(
        self,
        vm_name: str,
        username: str,
        password: str | None = None,
        private_key_path: str | None = None,
        **kwargs: Any,
    ) -> VMSSHConnection: ...

    def cleanup_all(self) -> None: ...
```

**Fixture definition:**

```python
@pytest.fixture(scope="class")
def vm_ssh_connections(fixture_store, destination_provider, prepared_plan, ocp_admin_client):
    """Manage SSH connections to migrated VMs using python-rrmngmnt."""
    vm_namespace = prepared_plan["_vm_target_namespace"]
    manager = SSHConnectionManager(
        provider=destination_provider,
        namespace=vm_namespace,
        fixture_store=fixture_store,
        ocp_client=ocp_admin_client,
    )
    yield manager
    manager.cleanup_all()
```

> **Tip:** `SSHConnectionManager` extracts the OCP token on-demand via a `@property` to minimize the exposure window of authentication tokens.

---

## Static IP Validation

### check_static_ip_preservation()

Verifies that static IP addresses are preserved on Windows VMs after migration from vSphere.

```python
def check_static_ip_preservation(
    vm_name: str,
    vm_ssh_connections: SSHConnectionManager,
    source_vm_data: dict[str, Any],
    source_provider_data: dict[str, Any],
    timeout: int = 600,
    retry_delay: int = 30,
) -> None:
```

**Validation flow:**

1. Extracts static IP interfaces from `source_vm_data` using `_extract_static_interfaces()`.
2. For each static interface, connects to the destination VM via SSH.
3. Runs `ipconfig /all` on the Windows VM.
4. Parses the output using the `jc` library (`_parse_windows_network_config()`).
5. Matches interfaces by name or MAC address.
6. Validates IP address, subnet mask, and gateway match the source.

**Activation conditions** — `check_vms()` only calls this when:
- `vm_ssh_connections` is provided
- Destination VM is powered on
- Source VM data has `win_os: True`
- Source provider is vSphere

```python
# Inside check_vms():
if (
    source_vm_data
    and source_vm_data.get("win_os")
    and source_provider.type == Provider.ProviderType.VSPHERE
):
    check_static_ip_preservation(
        vm_name=vm_name,
        vm_ssh_connections=vm_ssh_connections,
        source_vm_data=source_vm_data,
        source_provider_data=source_provider_data,
    )
```

> **Note:** Static IP verification is currently implemented only for Windows VMs. Calling it on non-Windows VMs raises `NotImplementedError`.

### Configuration

Enable static IP preservation in the test config:

```python
"test_cold_migration_comprehensive": {
    "virtual_machines": [
        {
            "name": "mtv-win2019-3disks",
            "source_vm_power": "off",
            "guest_agent": True,
        },
    ],
    "preserve_static_ips": True,
    # ...
}
```

When `preserve_static_ips` is `True` and the source provider is vSphere, `check_vms()` requires `source_vms_data` to be populated by the `prepared_plan` fixture. A `ValueError` is raised if this data is missing.

### Helper Functions

#### _extract_static_interfaces()

Extracts interfaces with static IP configuration from source VM data:

```python
def _extract_static_interfaces(source_vm_data: dict[str, Any]) -> list[dict[str, Any]]:
```

Returns a flat list of dictionaries, one per static IP, containing `name`, `macAddress`, `ip_address`, `subnet_mask`, `gateway`, and `is_static_ip`.

#### _parse_windows_network_config()

Parses Windows `ipconfig /all` output using the `jc` library:

```python
def _parse_windows_network_config(ipconfig_output: str) -> dict[str, dict[str, Any]]:
```

Returns a dictionary mapping interface names to their configuration including `ip_addresses`, `subnet_mask`, `macAddress`, and `gateway`.

#### _verify_subnet_mask()

Compares subnet masks using `ipaddress.IPv4Network` for proper normalization:

```python
def _verify_subnet_mask(
    interface_name: str,
    expected_subnet: str,
    matching_interface: dict[str, Any],
) -> None:
```

#### _verify_gateway()

Validates gateway configuration matches between source and destination:

```python
def _verify_gateway(
    interface_name: str,
    expected_gateway: str,
    matching_interface: dict[str, Any] | None,
) -> None:
```

---

## PVC Name Verification

### check_pvc_names()

Validates that PVC names on destination VMs match the expected `pvcNameTemplate` Go template pattern from the migration plan.

```python
def check_pvc_names(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    pvc_name_template: str | None,
    use_generate_name: bool = False,
    source_provider: BaseProvider | None = None,
    source_provider_inventory: ForkliftInventory | None = None,
) -> None:
```

### Template Variables

The function supports Go template syntax with Sprig functions, evaluated using the `py-go-template` library:

| Variable | Description | Provider Requirement |
|----------|-------------|---------------------|
| `{{.VmName}}` | Source VM name | Any |
| `{{.DiskIndex}}` | 0-based disk index (by controller position) | vSphere only |
| `{{.FileName}}` | VMDK filename without path/extension | vSphere only |

**Sprig functions** are supported for string manipulation:

```
{{ .FileName | trimSuffix ".vmdk" | replace "_" "-" }}
{{ .VmName | lower }}-{{ .DiskIndex }}
{{ mustRegexReplaceAll "[^a-z0-9]" .VmName "-" }}
```

### Name Length Handling

PVC names are subject to Kubernetes name length limits:

| Mode | Max Length | Constant |
|------|-----------|----------|
| Exact name (`use_generate_name: false`) | 63 characters | `KUBERNETES_MAX_NAME_LENGTH` |
| Prefix (`use_generate_name: true`) | 58 characters | `KUBERNETES_MAX_GENERATE_NAME_PREFIX_LENGTH` |

Names exceeding these limits are automatically truncated, and the verification adjusts accordingly.

### Matching Modes

- **Exact match** (`use_generate_name: false`): PVC name must exactly equal the rendered template result.
- **Prefix match** (`use_generate_name: true`): PVC name must start with the rendered template result. Kubernetes adds a random suffix.

### Disk Ordering

For VMware/vSphere providers, source disks are sorted by `(controller_key, unit_number)` to establish the correct disk index for template evaluation. The function also verifies that destination disk `unit_number` matches the source disk index to ensure disk order preservation.

### Configuration

```python
"test_cold_migration_comprehensive": {
    "pvc_name_template": "{{.VmName}}-disk-{{.DiskIndex}}",
    "pvc_name_template_use_generate_name": False,
    # ...
}
```

### Invocation in check_vms()

```python
# Inside check_vms():
if plan.get("pvc_name_template"):
    check_pvc_names(
        source_vm=plan.get("source_vms_data", {}).get(vm["name"], source_vm),
        destination_vm=destination_vm,
        pvc_name_template=plan["pvc_name_template"],
        use_generate_name=plan.get("pvc_name_template_use_generate_name", False),
        source_provider=source_provider,
        source_provider_inventory=source_provider_inventory,
    )
```

> **Note:** When `source_vms_data` is available in the plan (populated by the `prepared_plan` fixture), it is preferred over the live `source_vm` data for PVC name verification. This ensures disk ordering metadata captured before migration is used.

---

## Power State Validation

### check_vms_power_state()

Verifies the destination VM's power state matches expectations after migration.

```python
def check_vms_power_state(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    source_power_before_migration: str | None,
    target_power_state: str | None = None,
) -> None:
```

**Logic:**

1. If `target_power_state` is specified in the plan, the destination VM must match it exactly (e.g., force VMs to `"on"` even if they were `"off"` before migration).
2. If no `target_power_state`, the destination VM should preserve the source VM's power state from before migration.
3. Raises `ValueError` if `source_power_before_migration` is not `"on"` or `"off"`.

### Configuration

```python
# Force VMs on after migration (regardless of source state)
"test_cold_migration_comprehensive": {
    "virtual_machines": [{"name": "vm", "source_vm_power": "off"}],
    "target_power_state": "on",
}

# Preserve source power state (default behavior, no target_power_state key)
"test_sanity_cold_migration": {
    "virtual_machines": [{"name": "vm", "source_vm_power": "on"}],
}
```

---

## Resource Validation Checks

### check_cpu()

Compares CPU cores and sockets between source and destination VMs.

```python
def check_cpu(source_vm: dict[str, Any], destination_vm: dict[str, Any]) -> None:
```

Collects all mismatches (cores and sockets) and reports them in a single `pytest.fail()` call.

### check_memory()

Asserts memory size in MB matches between source and destination.

```python
def check_memory(source_vm: dict[str, Any], destination_vm: dict[str, Any]) -> None:
```

### check_network()

Validates network interface mappings. For each source NIC, resolves the expected destination network from the `NetworkMap` CR and verifies the destination NIC (matched by MAC address) uses the correct network.

```python
def check_network(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    network_migration_map: NetworkMap,
) -> None:
```

> **Note:** Network checks are skipped for OpenShift-to-OpenShift migrations (source provider type `OPENSHIFT`).

### check_storage()

Validates disk count matches and storage class is correctly applied. For `ocs-storagecluster-ceph-rbd` storage, also verifies access modes based on the StorageMap configuration.

```python
def check_storage(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    storage_map_resource: StorageMap,
) -> None:
```

### get_destination()

Helper function that resolves the destination entry from a `NetworkMap` or `StorageMap` for a given source NIC/disk.

```python
def get_destination(
    map_resource: NetworkMap | StorageMap,
    source_vm_nic: dict[str, Any],
) -> dict[str, Any]:
```

Matches by `source.type`, `source.name`, or `source.id`. Returns `{"name": "pod"}` for pod-network destinations.

---

## Provider-Specific Checks

### check_serial_preservation() (vSphere)

Verifies VMware UUID is preserved as the VM serial number after migration. The expected format depends on the OpenShift cluster version:

| OCP Version | Serial Format | Example |
|-------------|---------------|---------|
| 4.20+ | VMware BIOS serial | `VMware-12 34 56 78 12 34 12 34-12 34 12 34 56 78 90 12` |
| < 4.20 | Plain UUID | `12345678-1234-1234-1234-123456789012` |

```python
def check_serial_preservation(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    destination_provider: BaseProvider,
) -> None:
```

Uses `get_cluster_version()` to detect the OCP version and `_format_uuid_to_vmware_serial()` for BIOS serial formatting.

### check_snapshots() (vSphere)

Compares VM snapshots before and after migration, verifying `create_time`, `id`, `name`, and `state` are preserved.

```python
def check_snapshots(
    snapshots_before_migration: list[dict[str, Any]],
    snapshots_after_migration: list[dict[str, Any]],
) -> None:
```

Activated only when `snapshots_before_migration` is present in the VM config.

### check_false_vm_power_off() (RHV)

For RHV providers, verifies that the `USER_STOP_VM` event (event code 33) was **not** triggered, confirming the VM was not forcibly powered off.

```python
def check_false_vm_power_off(
    source_provider: OvirtProvider,
    source_vm: dict[str, Any],
) -> None:
```

### check_guest_agent()

Asserts the QEMU guest agent is running on the destination VM.

```python
def check_guest_agent(destination_vm: dict[str, Any]) -> None:
```

Only runs when `guest_agent: True` is set in the VM configuration.

---

## Placement and Scheduling Checks

### check_vm_node_placement()

Verifies the migrated VM is scheduled on the expected labeled worker node.

```python
def check_vm_node_placement(
    destination_vm: dict[str, Any],
    expected_node: str,
) -> None:
```

Activated when `target_node_selector` is present in the plan and `labeled_worker_node` is provided.

### check_vm_labels()

Validates that expected labels are set on the migrated VM's metadata. Reports missing and incorrect labels separately.

```python
def check_vm_labels(
    destination_vm: dict[str, Any],
    expected_labels: dict[str, str],
) -> None:
```

> **Note:** Kubernetes API returns `ResourceField` objects for labels. This function uses `.to_dict()` for recursive conversion before comparison, since the `in` operator does not work on `ResourceField`.

### check_vm_affinity()

Validates VM affinity configuration matches the expected spec via deep dictionary comparison.

```python
def check_vm_affinity(
    destination_vm: dict[str, Any],
    expected_affinity: dict[str, Any],
) -> None:
```

### Configuration Example

```python
"test_cold_migration_comprehensive": {
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
}
```

---

## SSL Configuration Check

### check_ssl_configuration()

Verifies the source provider's Kubernetes Secret has an `insecureSkipVerify` value matching the global `source_provider_insecure_skip_verify` config setting.

```python
def check_ssl_configuration(source_provider: BaseProvider) -> None:
```

**Flow:**

1. Reads `source_provider_insecure_skip_verify` from `py_config`.
2. Retrieves the provider's Secret reference from `spec.secret`.
3. Decodes the Base64-encoded `insecureSkipVerify` field.
4. Asserts the decoded value matches the expected setting.

Applies to VMware, RHV, and OpenStack providers. Runs automatically at the beginning of `check_vms()`.

---

## Error Handling Strategy

`check_vms()` uses a **collect-all-failures** pattern rather than fail-fast. Each individual check runs inside a `try/except Exception` block:

```python
res: dict[str, list[str]] = {}

for vm in plan["virtual_machines"]:
    vm_name = vm["name"]
    res[vm_name] = []

    try:
        check_vms_power_state(...)
    except Exception as exp:
        res[vm_name].append(f"check_vms_power_state - {str(exp)}")

    try:
        check_cpu(...)
    except Exception as exp:
        res[vm_name].append(f"check_cpu - {str(exp)}")

    # ... additional checks ...

# Report all failures at the end
for _vm_name, _errors in res.items():
    if _errors:
        should_fail = True
        LOGGER.error(f"VM {_vm_name} failed checks: {_errors}")

if should_fail:
    pytest.fail("Some of the VMs did not match")
```

This ensures all validations execute even when early checks fail, providing a complete picture of migration fidelity in a single test run.

> **Tip:** Check the test logs for per-VM error details. The `LOGGER.error()` calls emit the full list of failed checks for each VM before the final `pytest.fail()`.

---

## Check Execution Matrix

Which checks run depends on the provider type and plan configuration:

| Check | vSphere | RHV | OpenStack | OCP | Condition |
|-------|---------|-----|-----------|-----|-----------|
| `check_ssl_configuration` | Yes | Yes | Yes | No | Always for supported providers |
| `check_vms_power_state` | Yes | Yes | Yes | Yes | Always |
| `check_cpu` | Yes | Yes | Yes | Yes | Always |
| `check_memory` | Yes | Yes | Yes | Yes | Always |
| `check_network` | Yes | Yes | Yes | No | Skipped for OCP-to-OCP |
| `check_storage` | Yes | Yes | Yes | Yes | Always |
| `check_pvc_names` | Yes | Yes | Yes | Yes | `pvc_name_template` in plan |
| `check_snapshots` | Yes | No | No | No | `snapshots_before_migration` in VM config |
| `check_serial_preservation` | Yes | No | No | No | vSphere only |
| `check_guest_agent` | Yes | Yes | Yes | Yes | `guest_agent: True` in VM config |
| `check_ssh_connectivity` | Yes | Yes | Yes | Yes | `vm_ssh_connections` provided + VM powered on |
| `check_static_ip_preservation` | Yes | No | No | No | Windows VM + vSphere + `source_vms_data` present |
| `check_vm_node_placement` | Yes | Yes | Yes | Yes | `target_node_selector` in plan |
| `check_vm_labels` | Yes | Yes | Yes | Yes | `target_labels` in plan |
| `check_vm_affinity` | Yes | Yes | Yes | Yes | `target_affinity` in plan |
| `check_false_vm_power_off` | No | Yes | No | No | RHV provider only |
