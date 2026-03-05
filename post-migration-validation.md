# Post-Migration Validation

The `check_vms` function in `utilities/post_migration.py` is the central orchestrator for post-migration validation. After a VM migration completes, it runs a comprehensive suite of checks against each migrated VM to verify that the migration preserved all critical properties — from basic resource allocation (CPU, memory) to advanced features like static IP preservation and custom PVC naming.

## Overview

`check_vms` iterates through every VM defined in the migration plan and runs up to 16 distinct validation checks. Errors are accumulated per VM rather than failing on the first issue, so a single test run surfaces all problems at once.

```python
from utilities.post_migration import check_vms

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

### Error Accumulation Pattern

Rather than halting on the first failure, `check_vms` collects all errors and reports them together at the end:

```python
res: dict[str, list[str]] = {}

for vm in plan["virtual_machines"]:
    vm_name = vm["name"]
    res[vm_name] = []

    try:
        check_vms_power_state(...)
    except Exception as exp:
        res[vm_name].append(f"check_vms_power_state - {str(exp)}")

    # ... more checks ...

should_fail = any(errors for errors in res.values())
if should_fail:
    pytest.fail("Some of the VMs did not match")
```

This ensures that all validation failures for all VMs are logged before the test fails, making debugging significantly easier.

## Validation Checks

The following table summarizes every check performed by `check_vms` and when each check is executed:

| Check | Condition | Provider |
|-------|-----------|----------|
| [SSL Configuration](#ssl-configuration) | VMware, RHV, or OpenStack provider | VMware, RHV, OpenStack |
| [Power State](#power-state-verification) | Always | All |
| [CPU](#cpu-verification) | Always | All |
| [Memory](#memory-verification) | Always | All |
| [Network Configuration](#network-configuration) | Not OCP-to-OCP migration | All except OpenShift |
| [Storage & Disk Count](#storage-and-disk-count-verification) | Always | All |
| [PVC Name Templates](#pvc-name-template-validation) | `pvc_name_template` set in plan | All (FileName/DiskIndex: VMware only) |
| [Snapshots](#snapshot-preservation) | VMware + snapshots recorded | VMware |
| [Serial Number](#serial-number-preservation) | VMware source | VMware |
| [Guest Agent](#guest-agent-verification) | `guest_agent: True` in VM config | All |
| [SSH Connectivity](#ssh-connectivity) | `vm_ssh_connections` provided + VM powered on | All |
| [Static IP Preservation](#static-ip-preservation) | Windows VM + VMware source + SSH available | VMware |
| [Node Placement](#node-placement) | `target_node_selector` + `labeled_worker_node` | All |
| [VM Labels](#vm-label-verification) | `target_labels` + `target_vm_labels` | All |
| [VM Affinity](#vm-affinity-verification) | `target_affinity` set in plan | All |
| [RHV Power Off Event](#rhv-power-off-event-check) | RHV source provider | RHV |

> **Note:** OVA provider VMs are skipped entirely — `check_vms` returns immediately when the source provider type is OVA, as OVA VMs do not expose runtime statistics.

---

## Power State Verification

**Function:** `check_vms_power_state` (`utilities/post_migration.py:799`)

Validates that the destination VM's power state matches expectations after migration. Supports two modes:

1. **Explicit target power state** — When `target_power_state` is set in the plan, the destination VM must match it exactly.
2. **Source-mirrored power state** — When only `source_vm_power` is configured, the destination VM should retain the same power state as the source had before migration.

```python
def check_vms_power_state(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    source_power_before_migration: str | None,
    target_power_state: str | None = None,
) -> None:
    if target_power_state:
        actual_power_state = destination_vm["power_state"]
        assert actual_power_state == target_power_state, (
            f"VM power state mismatch: expected {target_power_state}, got {actual_power_state}"
        )
    elif source_power_before_migration:
        if source_power_before_migration not in ("on", "off"):
            raise ValueError(f"Invalid source_vm_power '{source_power_before_migration}'. Must be 'on' or 'off'")
        assert destination_vm["power_state"] == source_power_before_migration
```

### Configuration

```python
# Mirror source power state (default behavior)
"virtual_machines": [{"name": "my-vm", "source_vm_power": "on"}]

# Override: force destination to a specific power state
"target_power_state": "on"
```

> **Tip:** Use `target_power_state` when migrating a powered-off VM but you need it running after migration (e.g., for SSH-based post-migration checks).

---

## CPU Verification

**Function:** `check_cpu` (`utilities/post_migration.py:481`)

Verifies that CPU core count and socket count are preserved between source and destination VMs.

```python
def check_cpu(source_vm: dict[str, Any], destination_vm: dict[str, Any]) -> None:
    failed_checks = {}

    src_vm_num_cores = source_vm["cpu"]["num_cores"]
    dst_vm_num_cores = destination_vm["cpu"]["num_cores"]

    src_vm_num_sockets = source_vm["cpu"]["num_sockets"]
    dst_vm_num_sockets = destination_vm["cpu"]["num_sockets"]

    if src_vm_num_cores and not src_vm_num_cores == dst_vm_num_cores:
        failed_checks["cpu number of cores"] = (
            f"source_vm cpu cores: {src_vm_num_cores} != destination_vm cpu cores: {dst_vm_num_cores}"
        )

    if src_vm_num_sockets and not src_vm_num_sockets == dst_vm_num_sockets:
        failed_checks["cpu number of sockets"] = (
            f"source_vm cpu sockets: {src_vm_num_sockets} != destination_vm cpu sockets: {dst_vm_num_sockets}"
        )

    if failed_checks:
        pytest.fail(f"CPU failed checks: {failed_checks}")
```

---

## Memory Verification

**Function:** `check_memory` (`utilities/post_migration.py:504`)

A straightforward assertion that memory (in MB) is identical between source and destination:

```python
def check_memory(source_vm: dict[str, Any], destination_vm: dict[str, Any]) -> None:
    assert source_vm["memory_in_mb"] == destination_vm["memory_in_mb"]
```

---

## Network Configuration

**Function:** `check_network` (`utilities/post_migration.py:512`)

Validates that each network interface on the source VM is correctly mapped to the expected destination network, as defined in the `NetworkMap` resource. Matching is performed by MAC address.

```python
def check_network(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    network_migration_map: NetworkMap,
) -> None:
    for source_vm_nic in source_vm["network_interfaces"]:
        expected_network = get_destination(network_migration_map, source_vm_nic)
        assert expected_network, "Network not found in migration map"

        expected_network_name = expected_network["name"]
        destination_vm_nic = get_nic_by_mac(
            nics=destination_vm["network_interfaces"],
            mac_address=source_vm_nic["macAddress"],
        )
        assert destination_vm_nic["network"] == expected_network_name
```

The helper `get_nic_by_mac` finds a NIC in the destination VM by matching its MAC address:

```python
def get_nic_by_mac(nics: list[dict[str, Any]], mac_address: str) -> dict[str, Any]:
    return [nic for nic in nics if nic["macAddress"] == mac_address][0]
```

> **Note:** Network validation is skipped for OCP-to-OCP migrations (`source_provider.type == Provider.ProviderType.OPENSHIFT`).

---

## Storage and Disk Count Verification

**Function:** `check_storage` (`utilities/post_migration.py:527`)

Performs three storage-related validations:

1. **Disk count** — The number of destination disks must equal the number of source disks.
2. **Storage class** — Each destination disk's storage class must match `py_config["storage_class"]`.
3. **Access mode (OCS Ceph RBD only)** — When using `ocs-storagecluster-ceph-rbd`, verifies the correct access mode (RWO vs RWX) based on the StorageMap mapping configuration.

```python
def check_storage(
    source_vm: dict[str, Any],
    destination_vm: dict[str, Any],
    storage_map_resource: StorageMap,
) -> None:
    destination_disks = destination_vm["disks"]
    source_vm_disks_storage = [disk["storage"]["name"] for disk in source_vm["disks"]]

    assert len(destination_disks) == len(source_vm["disks"]), "disks count"

    for destination_disk in destination_disks:
        assert destination_disk["storage"]["name"] == py_config["storage_class"], "storage class"
        if destination_disk["storage"]["name"] == "ocs-storagecluster-ceph-rbd":
            for mapping in storage_map_resource.instance.spec.map:
                if mapping.source.name in source_vm_disks_storage:
                    if mapping.destination.get("accessMode"):
                        assert destination_disk["storage"]["access_mode"][0] == DataVolume.AccessMode.RWO
                    else:
                        assert destination_disk["storage"]["access_mode"][0] == DataVolume.AccessMode.RWX
```

---

## PVC Name Template Validation

**Function:** `check_pvc_names` (`utilities/post_migration.py:545`)

The most complex validation function. It verifies that PVC names on the destination cluster follow the Go template pattern specified in the migration plan via `pvc_name_template`.

### Template Variables

The PVC name template uses Go template syntax with the following variables:

| Variable | Description | Provider Support |
|----------|-------------|-----------------|
| `{{.VmName}}` | Source VM name | All providers |
| `{{.DiskIndex}}` | 0-based disk index (sorted by controller position) | VMware only |
| `{{.FileName}}` | VMDK filename without path (e.g., `vm-name_1.vmdk`) | VMware only |

### Sprig Function Support

Templates support the full set of [Sprig template functions](http://masterminds.github.io/sprig/):

- `trimSuffix` — Remove a suffix from a string
- `replace` — String replacement
- `lower` / `upper` — Case conversion
- `mustRegexReplaceAll` — Regex-based replacement

### Template Rendering

Templates are rendered using the `go_template` Python library, which evaluates Go templates with Sprig function support:

```python
template_values = {
    "VmName": source_vm["name"],
    "DiskIndex": source_index,
    "FileName": filename,
}

with tempfile.NamedTemporaryFile(mode="w", suffix=".tmpl", delete=False) as tmp_file:
    tmp_file.write(pvc_name_template)
    tmp_file_path = tmp_file.name

result = go_template.render(Path(tmp_file_path), template_values)
expected_pvc_name = result.decode("utf-8") if isinstance(result, bytes) else result
```

### Kubernetes Name Length Limits

PVC names are subject to Kubernetes naming constraints. The validation automatically truncates rendered names:

```python
KUBERNETES_MAX_NAME_LENGTH: int = 63
KUBERNETES_MAX_GENERATE_NAME_PREFIX_LENGTH: int = 58
```

- **With `generateName`**: Truncated to **58 characters** (Kubernetes reserves 5 characters for the random suffix)
- **Without `generateName`**: Truncated to **63 characters** (standard Kubernetes name limit)

### Matching Modes

The `pvc_name_template_use_generate_name` flag controls how PVC names are matched:

- **`True` (generateName)**: Kubernetes appends a random suffix. Validation uses **prefix matching** — the PVC name must start with the rendered template.
- **`False` (exact)**: Validation uses **exact matching** — the PVC name must equal the rendered template exactly.

```python
if use_generate_name:
    # Prefix match: "my-vm-disk-0-xk4j2" starts with "my-vm-disk-0-"
    if dest_pvc_name.startswith(expected_pvc_name):
        matching_pvc = dest_pvc
else:
    # Exact match: "my-vm-disk-0" == "my-vm-disk-0"
    if dest_pvc_name == expected_pvc_name:
        matching_pvc = dest_pvc
```

### Disk Order Preservation

For VMware sources, disks are sorted by `(controller_key, unit_number)` to establish a deterministic order. The validation then confirms that each destination PVC's `unit_number` matches the expected source disk index:

```python
# Sort source disks by their position (controller_key, unit_number)
if source_provider and source_provider.type == Provider.ProviderType.VSPHERE:
    source_disks_ordered = sorted(
        source_disks,
        key=lambda d: (d["controller_key"], d["unit_number"]),
    )
```

### Inventory File Path Extraction

For the `{{.FileName}}` variable, disk filenames are extracted from the Forklift inventory API. The raw path format from Forklift is parsed to extract just the filename:

```python
# Format: "[datastore1] vm-name/vm-name_1.vmdk"
# Extracted: "vm-name_1.vmdk"
full_path = disk["file"]
if "]" in full_path:
    full_path = full_path.split("]", 1)[1].strip()
filename = full_path.split("/")[-1]
```

### Configuration Examples

```python
# Exact match with VM name and disk index
"pvc_name_template": "{{.VmName}}-disk-{{.DiskIndex}}",
"pvc_name_template_use_generate_name": False,
# Result: "my-vm-disk-0", "my-vm-disk-1", "my-vm-disk-2"

# generateName with Sprig functions
"pvc_name_template": '{{ .FileName | trimSuffix ".vmdk" | replace "_" "-" }}-{{.DiskIndex}}',
"pvc_name_template_use_generate_name": True,
# Result: "vm-name-1-0-xk4j2", "vm-name-2-1-ab3cd" (random suffixes appended)
```

> **Warning:** The `{{.FileName}}` and `{{.DiskIndex}}` variables are only supported for VMware (vSphere) providers. If used with other providers, PVC name validation is skipped with a warning.

---

## SSH Connectivity

**Function:** `check_ssh_connectivity` (`utilities/post_migration.py:81`)

Validates that SSH access to the migrated VM is functional. This check only runs when:
- An `SSHConnectionManager` is provided
- The destination VM is powered on

### Connection Flow

1. **Credential lookup** — Retrieves username/password from provider configuration based on the VM's OS type (Windows or Linux).
2. **Connection creation** — Creates an SSH connection via `virtctl port-forward`, which tunnels through the Kubernetes API server.
3. **Retry loop** — Uses `TimeoutSampler` with a 300-second timeout and 15-second retry interval to wait for connectivity.

```python
def check_ssh_connectivity(
    vm_name: str,
    vm_ssh_connections: SSHConnectionManager,
    source_provider_data: dict[str, Any],
    source_vm_info: dict[str, Any],
    timeout: int = 300,
    retry_delay: int = 15,
) -> None:
    ssh_username, ssh_password = get_ssh_credentials_from_provider_config(
        source_provider_data, source_vm_info
    )
    ssh_conn = vm_ssh_connections.create(
        vm_name=vm_name, username=ssh_username, password=ssh_password
    )

    def _test_connectivity() -> bool:
        try:
            with ssh_conn:
                if ssh_conn.is_connective(tcp_timeout=10):
                    LOGGER.info(f"SSH connectivity to VM {vm_name} verified successfully")
                    return True
                return False
        except (SSHException, AuthenticationException, NoValidConnectionsError, ChannelException):
            return False

    for sample in TimeoutSampler(wait_timeout=timeout, sleep=retry_delay, func=_test_connectivity):
        if sample:
            return
```

### SSH Credential Configuration

Credentials are sourced from the provider configuration file (`.providers.json`):

| VM OS Type | Username Key | Password Key |
|------------|-------------|--------------|
| Windows | `guest_vm_win_user` | `guest_vm_win_password` |
| Linux | `guest_vm_linux_user` | `guest_vm_linux_password` |

```python
def get_ssh_credentials_from_provider_config(
    source_provider_data: dict[str, Any], source_vm_info: dict[str, Any]
) -> tuple[str, str]:
    is_windows = source_vm_info.get("win_os", False)

    if is_windows:
        username = source_provider_data["guest_vm_win_user"]
        password = source_provider_data["guest_vm_win_password"]
    else:
        username = source_provider_data["guest_vm_linux_user"]
        password = source_provider_data["guest_vm_linux_password"]

    return username, password
```

> **Note:** SSH connectivity is accessed via `virtctl port-forward`, which creates a tunnel through the Kubernetes API server. No external IP or ingress is required on the migrated VM.

---

## Static IP Preservation

**Function:** `check_static_ip_preservation` (`utilities/post_migration.py:284`)

Verifies that static IP addresses configured on source VMs are preserved after migration. This is a critical validation for enterprise workloads where IP addresses are tied to DNS records, firewall rules, or application configurations.

### Applicability

This check runs only when **all** of the following conditions are met:
- SSH connections are available (`vm_ssh_connections` provided)
- The destination VM is powered on
- The source VM is a Windows VM (`source_vm_data["win_os"]` is `True`)
- The source provider is VMware vSphere

> **Note:** Static IP verification is currently implemented for Windows VMs only. Non-Windows VMs raise `NotImplementedError`.

### Verification Process

1. **Extract static interfaces** — Identifies all network interfaces with static IP configurations from the source VM data.

```python
def _extract_static_interfaces(source_vm_data: dict[str, Any]) -> list[dict[str, Any]]:
    static_interfaces = []
    for interface in source_vm_data.get("network_interfaces", []):
        for ip_config in interface.get("ip_addresses", []):
            if ip_config.get("is_static_ip") is True:
                static_interface = {
                    "name": interface["name"],
                    "macAddress": interface["macAddress"],
                    "ip_address": ip_config["ip_address"],
                    "subnet_mask": ip_config["subnet_mask"],
                    "gateway": ip_config.get("gateway", ""),
                }
                static_interfaces.append(static_interface)
    return static_interfaces
```

2. **Run `ipconfig /all` via SSH** — Executes the Windows command on the destination VM and parses the output using the `jc` library.

3. **Match interfaces** — Finds the corresponding destination interface by name or MAC address (normalized for format differences).

```python
source_mac = interface.get("macAddress", "").lower().replace("-", ":").replace(".", ":")
dest_mac = iface_config.get("macAddress", "").lower().replace("-", ":").replace(".", ":")
mac_match = source_mac == dest_mac and source_mac != ""
```

4. **Validate IP, subnet mask, and gateway**:
   - **IP address**: The expected static IP must be present in the destination interface's IP list
   - **Subnet mask**: Validated using `ipaddress.IPv4Network` for correct netmask comparison
   - **Gateway**: Compared directly if configured on the source

### Retry Behavior

Network configuration takes time to stabilize after migration, especially for secondary IPs. The check uses a `TimeoutSampler` with:
- **Timeout**: 600 seconds (10 minutes)
- **Retry interval**: 30 seconds

### Configuration

```python
"test_warm_migration_comprehensive": {
    "virtual_machines": [
        {"name": "mtv-win2022-ip-3disks", "source_vm_power": "on", "guest_agent": True},
    ],
    "preserve_static_ips": True,
    # ...
}
```

> **Warning:** When `preserve_static_ips` is enabled for a vSphere source, the `source_vms_data` dict must be populated by the `prepared_plan` fixture. If it is missing, `check_vms` raises a `ValueError` immediately rather than silently skipping verification.

---

## Serial Number Preservation

**Function:** `check_serial_preservation` (`utilities/post_migration.py:875`)

Validates that the VMware VM's UUID is correctly preserved as the BIOS serial number on the destination OpenShift VM. The format depends on the OpenShift cluster version.

### OCP Version-Dependent Behavior

| OCP Version | Serial Format | Example |
|-------------|--------------|---------|
| >= 4.20 | VMware BIOS format | `VMware-12 34 56 78 12 34 12 34-12 34 12 34 56 78 90 12` |
| < 4.20 | Plain UUID | `12345678-1234-1234-1234-123456789012` |

### UUID Formatting

The helper function `_format_uuid_to_vmware_serial` converts a standard UUID into the VMware BIOS serial format:

```python
def _format_uuid_to_vmware_serial(uuid: str) -> str:
    """
    Converts: "12345678-1234-1234-1234-123456789012"
    To: "VMware-12 34 56 78 12 34 12 34-12 34 12 34 56 78 90 12"
    """
    uuid_no_hyphens = uuid.replace("-", "").upper()
    return (
        f"VMware-{' '.join([uuid_no_hyphens[i : i + 2] for i in range(0, 16, 2)])}-"
        f"{' '.join([uuid_no_hyphens[i : i + 2] for i in range(16, 32, 2)])}"
    )
```

Comparison is case-insensitive:

```python
assert str(dest_serial).lower() == expected_serial_420.lower()
```

> **Note:** This check only applies to VMware (vSphere) source providers. The OCP cluster version is detected at runtime via `get_cluster_version()`.

---

## Guest Agent Verification

**Function:** `check_guest_agent` (`utilities/post_migration.py:820`)

When a VM is configured with `"guest_agent": True`, this check verifies that the QEMU guest agent is running on the destination VM:

```python
def check_guest_agent(destination_vm: dict[str, Any]) -> None:
    assert destination_vm.get("guest_agent_running"), "checking guest agent."
```

---

## Snapshot Preservation

**Function:** `check_snapshots` (`utilities/post_migration.py:831`)

For VMware VMs that had snapshots before migration, this check validates that snapshot metadata is preserved. Each snapshot is compared on four fields:

- `id` — Snapshot identifier
- `name` — Snapshot display name
- `state` — Snapshot state
- `create_time` — Creation timestamp (compared at `YYYY-MM-DD HH:MM` precision)

```python
def check_snapshots(
    snapshots_before_migration: list[dict[str, Any]],
    snapshots_after_migration: list[dict[str, Any]],
) -> None:
    snapshots_before_migration.sort(key=lambda x: x["id"])
    snapshots_after_migration.sort(key=lambda x: x["id"])

    time_format: str = "%Y-%m-%d %H:%M"

    for before_snapshot, after_snapshot in zip(snapshots_before_migration, snapshots_after_migration):
        if (
            before_snapshot["create_time"].strftime(time_format) != after_snapshot["create_time"].strftime(time_format)
            or before_snapshot["id"] != after_snapshot["id"]
            or before_snapshot["name"] != after_snapshot["name"]
            or before_snapshot["state"] != after_snapshot["state"]
        ):
            failed_snapshots.append(...)
```

---

## Node Placement

**Function:** `check_vm_node_placement` (`utilities/post_migration.py:947`)

When a `target_node_selector` is configured in the plan, this check verifies that the migrated VM is scheduled on the expected worker node:

```python
def check_vm_node_placement(
    destination_vm: dict[str, Any],
    expected_node: str,
) -> None:
    actual_node = destination_vm.get("node_name")
    if not actual_node:
        pytest.fail(f"VM {vm_name} has no node assignment")
    if actual_node != expected_node:
        pytest.fail(
            f"VM {vm_name} not scheduled on expected node. Expected: {expected_node}, Got: {actual_node}"
        )
```

---

## VM Label Verification

**Function:** `check_vm_labels` (`utilities/post_migration.py:975`)

Validates that all expected labels are present on the destination VM's metadata with correct values. The function handles Kubernetes `ResourceField` objects by converting them to plain dicts:

```python
actual_labels: dict[str, str] = actual_labels_raw.to_dict() if actual_labels_raw else {}

for label_key, expected_value in expected_labels.items():
    if label_key not in actual_labels:
        missing_labels.append(f"{label_key}=<missing>")
    elif actual_labels[label_key] != expected_value:
        incorrect_labels.append(
            f"{label_key}={actual_labels[label_key]} (expected: {expected_value})"
        )
```

### Configuration

Labels support auto-generated values (using `session_uuid`) and static values:

```python
"target_labels": {
    "mtv-comprehensive-test": None,     # None = auto-generate with session_uuid
    "static-label": "static-value",     # Static value
}
```

---

## VM Affinity Verification

**Function:** `check_vm_affinity` (`utilities/post_migration.py:1027`)

Performs a deep comparison between expected and actual affinity rules on the destination VM. Like label verification, it handles `ResourceField` conversion:

```python
actual_affinity: dict[str, Any] = actual_affinity_raw.to_dict() if actual_affinity_raw else {}

if actual_affinity != expected_affinity:
    pytest.fail(
        f"VM {vm_name} affinity verification failed:\n"
        f"  Expected affinity: {expected_affinity}\n"
        f"  Actual affinity: {actual_affinity}"
    )
```

> **Tip:** Use `.to_dict()` (not `dict()`) for `ResourceField` objects. The `dict()` built-in only converts the top level, leaving nested `ResourceField` objects that break comparison.

---

## RHV Power Off Event Check

**Function:** `check_false_vm_power_off` (`utilities/post_migration.py:824`)

Specific to RHV (oVirt) source providers. Validates that no `USER_STOP_VM` event (event code 33) was generated during the migration, which would indicate an unexpected VM shutdown:

```python
def check_false_vm_power_off(source_provider: OvirtProvider, source_vm: dict[str, Any]) -> None:
    """Checking that USER_STOP_VM (event.code=33) was not performed"""
    assert not source_provider.check_for_power_off_event(source_vm["provider_vm_api"])
```

---

## SSL Configuration Check

**Function:** `check_ssl_configuration` (`utilities/post_migration.py:1067`)

Verifies that the provider's `insecureSkipVerify` setting in its Kubernetes Secret matches the global test configuration (`source_provider_insecure_skip_verify`). This check applies to VMware, RHV, and OpenStack providers.

The actual secret value is base64-decoded and compared:

```python
actual_value = base64.b64decode(secret.instance.data["insecureSkipVerify"]).decode("utf-8")
assert actual_value == expected_value
```

> **Note:** SSL configuration check failures are logged but do not block the remaining per-VM validations. They are accumulated alongside per-VM errors under the `_provider` key.

---

## Complete Test Configuration Example

The comprehensive migration tests demonstrate the full range of `check_vms` validations:

```python
# From tests/tests_config/config.py
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
        "mtv-comprehensive-test": None,
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

This configuration triggers the following validations: power state, CPU, memory, network, storage/disk count, PVC name template (with generateName), snapshots, serial number, guest agent, SSH connectivity, static IP preservation, VM labels, and VM affinity.
