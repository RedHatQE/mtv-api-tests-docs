# Naming Conventions

This page documents how mtv-api-tests generates, sanitizes, and manages names for Kubernetes resources, VMs, namespaces, and network attachments. A layered naming strategy ensures uniqueness across parallel test sessions, DNS-1123 compliance, and traceable resource ownership.

## Overview

Every resource created during a test session carries a name derived from a **session UUID** — a short, random identifier generated once per session. This UUID propagates through a naming hierarchy:

```
session_uuid (auto-a1b2)
  └── base_resource_name (auto-a1b2-source-vsphere-8-0)
        ├── Plan       (auto-a1b2-source-vsphere-8-0-k7p5-cold)
        ├── StorageMap  (auto-a1b2-source-vsphere-8-0-t2k1)
        ├── NetworkMap  (auto-a1b2-source-vsphere-8-0-m3p1)
        └── Namespace   (auto-a1b2-mtv-test)
```

## Session UUID

The `session_uuid` is the root of all resource naming. It is generated once per pytest session and stored in the `fixture_store` for use by all downstream fixtures.

**Source:** `conftest.py`

```python
@pytest.fixture(scope="session")
def session_uuid(fixture_store):
    _session_uuid = generate_name_with_uuid(name="auto")
    fixture_store["session_uuid"] = _session_uuid
    return _session_uuid
```

This produces values like `auto-a1b2`, `auto-x9k2`, or `auto-m3p1` — always starting with `auto-` followed by 4 random characters.

### Why session UUID matters

- **Parallel safety:** When `pytest-xdist` runs multiple workers, each worker gets its own session with a unique UUID. Resources from different workers never collide.
- **Traceability:** Every resource on the cluster can be traced back to the session that created it.
- **Cleanup:** The `fixture_store["teardown"]` dictionary tracks all created resources by kind, enabling reliable session teardown.

## `generate_name_with_uuid()`

This is the core naming function used across the codebase. It appends a 4-character random suffix to any base name and normalizes the result for Kubernetes compatibility.

**Source:** `utilities/naming.py`

```python
def generate_name_with_uuid(name: str) -> str:
    _name = f"{name}-{shortuuid.ShortUUID().random(length=4).lower()}"
    _name = _name.replace("_", "-").replace(".", "-").lower()
    return _name
```

### Behavior

| Input | Output (example) |
|---|---|
| `"auto"` | `auto-a1b2` |
| `"auto-a1b2-source-vsphere-8-0"` | `auto-a1b2-source-vsphere-8-0-x9y2` |
| `"my_vm.name"` | `my-vm-name-k3m1` |

The function:
1. Appends `-{4 random lowercase chars}` using `shortuuid`
2. Replaces underscores (`_`) and periods (`.`) with hyphens (`-`)
3. Lowercases the entire result

### Where it is called

| Caller | Purpose |
|---|---|
| `conftest.py` — `session_uuid` fixture | Generate the session-wide UUID (`auto-xxxx`) |
| `utilities/resources.py` — `create_and_store_resource()` | Auto-generate names for Plans, StorageMaps, NetworkMaps, Secrets, etc. |
| `libs/base_provider.py` — `_generate_clone_vm_name()` | Generate unique names for cloned VMs |

## Base Resource Name

The `base_resource_name` is a session-scoped composite name that encodes the session UUID, provider type, and provider version. It serves as the prefix for all auto-generated resource names.

**Source:** `conftest.py`

```python
@pytest.fixture(scope="session")
def base_resource_name(fixture_store, session_uuid, source_provider_data):
    _name = f"{session_uuid}-source-{source_provider_data['type']}-{source_provider_data['version'].replace('.', '-')}"

    # Add copyoffload indicator for Plan/StorageMap/NetworkMap names
    if "copyoffload" in source_provider_data:
        _name = f"{_name}-xcopy"

    fixture_store["base_resource_name"] = _name
```

### Format

```
{session_uuid}-source-{provider_type}-{version_with_dashes}[-xcopy]
```

### Examples

| Provider | Version | Copy-offload | Result |
|---|---|---|---|
| vSphere | 8.0 | No | `auto-a1b2-source-vsphere-8-0` |
| RHV | 4.4 | No | `auto-x9k2-source-rhv-4-4` |
| OpenStack | zed | Yes | `auto-m3p1-source-openstack-zed-xcopy` |
| OpenShift | 4.12 | No | `auto-q5r7-source-openshift-4-12` |

## Resource Name Generation in `create_and_store_resource()`

The central resource creation function in `utilities/resources.py` implements a name resolution hierarchy with automatic fallbacks.

**Source:** `utilities/resources.py`

```python
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    # ...
    _resource_name = kwargs.get("name")
    _resource_dict = kwargs.get("kind_dict", {})
    _resource_yaml = kwargs.get("yaml_file")

    if not _resource_name:
        if _resource_yaml:
            with open(_resource_yaml) as fd:
                _resource_dict = yaml.safe_load(fd)
        _resource_name = _resource_dict.get("metadata", {}).get("name")

    if not _resource_name:
        _resource_name = generate_name_with_uuid(name=fixture_store["base_resource_name"])

        if resource.kind in (Migration.kind, Plan.kind):
            _resource_name = f"{_resource_name}-{'warm' if kwargs.get('warm_migration') else 'cold'}"

    if len(_resource_name) > 63:
        LOGGER.warning(f"'{_resource_name=}' is too long ({len(_resource_name)} > 63). Truncating.")
        _resource_name = _resource_name[-63:]
    # ...
```

### Name resolution order

1. **Explicit `name` kwarg** — used as-is if provided
2. **YAML or kind_dict metadata** — extracted from `metadata.name` if present
3. **Auto-generated** — `generate_name_with_uuid(base_resource_name)`, with `-warm` or `-cold` appended for Plan and Migration resources

### 63-character truncation

Kubernetes enforces a 63-character limit on resource names (DNS-1123 label). When an auto-generated name exceeds this limit, the function truncates from the **left**, keeping the last 63 characters:

```python
_resource_name = _resource_name[-63:]
```

This preserves the UUID suffix, which is the most important part for uniqueness.

> **Note:** The `ResourceNameNotStartedWithSessionUUIDError` exception class exists in `exceptions/exceptions.py` as a safeguard — resource names that lose their session UUID prefix due to truncation can be flagged during validation.

### Example resource names

| Resource Kind | Auto-generated Name |
|---|---|
| StorageMap | `auto-a1b2-source-vsphere-8-0-t2k1` |
| NetworkMap | `auto-a1b2-source-vsphere-8-0-m3p1` |
| Plan | `auto-a1b2-source-vsphere-8-0-k7p5-cold` |
| Migration | `auto-a1b2-source-vsphere-8-0-q5l2-warm` |
| Secret | `auto-a1b2-source-vsphere-8-0-x5l2` |

## `sanitize_kubernetes_name()` — DNS-1123 Compliance

Source VM names from external providers (VMware, RHV, OpenStack) frequently contain characters that are illegal in Kubernetes resource names. The `sanitize_kubernetes_name()` function transforms arbitrary strings into valid DNS-1123 labels.

**Source:** `utilities/naming.py`

```python
# Compiled regex patterns for performance (reused in sanitize_kubernetes_name)
_INVALID_CHARS_PATTERN = re.compile(r"[^a-z0-9-]+")
_LEADING_TRAILING_PATTERN = re.compile(r"^[^a-z0-9]+|[^a-z0-9]+$")


def sanitize_kubernetes_name(name: str, max_length: int = 63) -> str:
    """Sanitize a VM name to comply with Kubernetes DNS-1123 naming conventions.

    Rules:
    - lowercase alphanumeric characters and '-'
    - must start and end with an alphanumeric character
    - max `max_length` characters
    """
    sanitized = name.replace("_", "-").replace(".", "-").lower()
    sanitized = _INVALID_CHARS_PATTERN.sub("-", sanitized)
    sanitized = _LEADING_TRAILING_PATTERN.sub("", sanitized)
    sanitized = sanitized[:max_length].rstrip("-")
    if not sanitized:
        raise InvalidVMNameError(
            f"VM name '{name}' cannot be sanitized to a valid DNS-1123 name. "
            "The name must contain at least one alphanumeric character."
        )
    return sanitized
```

### DNS-1123 label rules

A valid DNS-1123 label must:
- Contain only **lowercase alphanumeric characters** and **hyphens** (`-`)
- **Start** with an alphanumeric character
- **End** with an alphanumeric character
- Be at most **63 characters** long

### Transformation pipeline

| Step | Operation | Example |
|---|---|---|
| 1 | Replace `_` and `.` with `-` | `My_VM.Name` → `My-VM-Name` |
| 2 | Lowercase | `My-VM-Name` → `my-vm-name` |
| 3 | Replace invalid chars with `-` | `vm@#$name` → `vm---name` |
| 4 | Strip leading/trailing non-alphanumeric | `---my-vm---` → `my-vm` |
| 5 | Truncate to max_length | (63 chars by default) |
| 6 | Remove trailing hyphens after truncation | `my-long-name-` → `my-long-name` |
| 7 | Raise error if empty | `@#$%` → `InvalidVMNameError` |

### Where it is used

The primary use case is in the OpenShift provider (`libs/providers/openshift.py`), where destination VM names are sanitized to match how the MTV operator converts source VM names:

```python
# The MTV operator converts source VM names (which may have capitals and underscores)
# to valid Kubernetes resource names (lowercase, hyphens instead of underscores)
original_name = cnv_vm_name
if not _source:
    try:
        cnv_vm_name = sanitize_kubernetes_name(cnv_vm_name)
    except InvalidVMNameError:
        LOGGER.error(
            f"Failed to sanitize VM name for Kubernetes lookup. original_name='{original_name}', namespace='{kwargs.get('namespace')}'"
        )
        raise
```

> **Warning:** The `InvalidVMNameError` exception is raised when a VM name contains no alphanumeric characters at all (e.g., `"@#$%"`). This is a fatal error — the VM cannot be represented as a Kubernetes resource.

## Class-Unique NAD Naming with SHA-256 Hashing

Network Attachment Definitions (NADs) require special naming considerations because:
1. NAD names become **Linux bridge interface names**, which are limited to **15 characters**
2. Tests run in parallel, so NAD names must be unique per test class
3. Names must be deterministic (same test class always gets the same name)

### `generate_class_hash_prefix()`

**Source:** `utilities/utils.py`

```python
def generate_class_hash_prefix(nodeid: str, length: int = 6) -> str:
    """Generate a FIPS-compliant hash prefix for class-based resource naming.

    Args:
        nodeid (str): The pytest node ID (e.g., request.node.nodeid).
        length (int): Length of the hash prefix (default: 6).

    Returns:
        str: A hex prefix of the specified length (e.g., "a1b2c3").
    """
    return hashlib.sha256(nodeid.encode()).hexdigest()[:length]
```

This function takes the full pytest node ID (e.g., `tests/test_cold_migration.py::TestColdMigration`) and produces a 6-character hex string from its SHA-256 hash.

> **Note:** SHA-256 is used instead of MD5 for FIPS 140-2 compliance. This matters in regulated environments where FIPS-approved algorithms are required.

### NAD naming pattern

```
cb-{6-char-sha256-hash}-{sequence-number}
```

Where:
- `cb` = "CNV Bridge" (fixed prefix)
- `{6-char-sha256-hash}` = first 6 hex characters of `SHA-256(pytest_node_id)`
- `{sequence-number}` = sequential index (1, 2, 3...) for multiple networks

### Example names

| Test Class | Node ID Hash | NADs Created |
|---|---|---|
| `TestColdMigration` | `a1b2c3` | `cb-a1b2c3-1`, `cb-a1b2c3-2` |
| `TestWarmMigration` | `d4e5f6` | `cb-d4e5f6-1` |
| `TestMultiNicMigration` | `7f8e9d` | `cb-7f8e9d-1`, `cb-7f8e9d-2`, `cb-7f8e9d-3` |

The longest possible name is `cb-ffffff-99` (12 characters), well under the 15-character Linux bridge limit.

### NAD creation flow

**Source:** `conftest.py` — `multus_network_name` fixture

```python
@pytest.fixture(scope="class")
def multus_network_name(
    fixture_store, target_namespace, ocp_admin_client, multus_cni_config,
    source_provider, source_provider_inventory, class_plan_config, request,
) -> dict[str, str]:
    hash_prefix = generate_class_hash_prefix(request.node.nodeid)
    base_name = f"cb-{hash_prefix}"

    # ... determine number of networks from source VMs ...

    for i in range(1, multus_count + 1):
        nad_name = f"{base_name}-{i}"

        if i == 1:
            config = multus_cni_config
        else:
            cni_config = {"cniVersion": "0.3.1", "type": "bridge", "bridge": nad_name}
            config = json.dumps(cni_config)

        create_and_store_resource(
            fixture_store=fixture_store,
            resource=NetworkAttachmentDefinition,
            client=ocp_admin_client,
            namespace=nad_namespace,
            config=config,
            name=nad_name,
        )

    return {"name": base_name, "namespace": nad_namespace}
```

Key behaviors:
- The **first NAD** uses the user-provided `multus_cni_config`
- **Additional NADs** auto-generate a bridge CNI config using the NAD name as the bridge identifier
- NADs pass an **explicit `name`** to `create_and_store_resource()`, bypassing auto-generation
- The fixture is **class-scoped**, so each test class gets its own set of NADs

### OpenShift source provider NADs

When the source provider is OpenShift, the `prepared_plan` fixture also creates NADs for source VMs using the same hashing strategy:

```python
# conftest.py — prepared_plan fixture
if openshift_source_provider:
    hash_prefix = generate_class_hash_prefix(request.node.nodeid)
    multus_network_name = f"cb-{hash_prefix}"

    create_and_store_resource(
        fixture_store=fixture_store,
        resource=NetworkAttachmentDefinition,
        client=ocp_admin_client,
        namespace=source_vms_namespace,
        config=multus_cni_config,
        name=multus_network_name,
    )
```

> **Tip:** Because the hash is derived from the pytest node ID, the same test class always generates the same NAD names. This makes debugging easier — you can predict which NADs belong to which test class.

## VM Clone Naming

When VMs are cloned from source providers for test isolation, the clone name combines the session UUID with the original VM name.

**Source:** `libs/base_provider.py`

```python
def _generate_clone_vm_name(self, session_uuid: str, base_name: str) -> str:
    """Generate a unique clone VM name with UUID and truncate if needed."""
    clone_vm_name = generate_name_with_uuid(f"{session_uuid}-{base_name}")
    if len(clone_vm_name) > 63:
        self.log.warning(f"VM name '{clone_vm_name}' is too long ({len(clone_vm_name)} > 63). Truncating.")
        clone_vm_name = clone_vm_name[-63:]
    return clone_vm_name
```

### VM suffix

Cloned VMs also receive a suffix encoding the storage class, OCP version, and migration type:

**Source:** `utilities/mtv_migration.py`

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

### Example VM name chain

| Stage | Name |
|---|---|
| Source VM | `centos-7-test-vm` |
| With suffix | `centos-7-test-vm-nfs-4-14-cold` |
| After clone | `auto-a1b2-centos-7-test-vm-nfs-4-14-cold-x9k2` |
| After sanitization (destination) | `auto-a1b2-centos-7-test-vm-nfs-4-14-cold-x9k2` |

## Target Namespace Naming

Each test session creates a unique target namespace combining the session UUID with a configurable prefix.

**Source:** `conftest.py`

```python
@pytest.fixture(scope="session")
def target_namespace(fixture_store, session_uuid, ocp_admin_client):
    _target_namespace: str = py_config["target_namespace_prefix"]

    # Remove prefix value to avoid duplication - session_uuid is already generated from this prefix
    _target_namespace = _target_namespace.replace("auto", "")

    # Generate a unique namespace name to avoid conflicts
    unique_namespace_name = f"{session_uuid}{_target_namespace}"[:63]
    fixture_store["target_namespace"] = unique_namespace_name

    namespace = create_and_store_resource(
        fixture_store=fixture_store,
        resource=Namespace,
        client=ocp_admin_client,
        name=unique_namespace_name,
        label=label,
    )
```

The `"auto"` prefix is stripped from `target_namespace_prefix` to avoid duplication, since `session_uuid` already starts with `auto-`.

| Config value | Session UUID | Final namespace |
|---|---|---|
| `auto-mtv-test` | `auto-a1b2` | `auto-a1b2-mtv-test` |
| `auto` | `auto-x9k2` | `auto-x9k2` |

## Summary: Naming by Resource Type

| Resource | Pattern | Max Length | Generator |
|---|---|---|---|
| Session UUID | `auto-{4-char}` | — | `generate_name_with_uuid("auto")` |
| Base resource name | `{uuid}-source-{type}-{ver}[-xcopy]` | — | `base_resource_name` fixture |
| Plan | `{base}-{4-char}-cold\|warm` | 63 | `create_and_store_resource()` |
| Migration | `{base}-{4-char}-cold\|warm` | 63 | `create_and_store_resource()` |
| StorageMap / NetworkMap | `{base}-{4-char}` | 63 | `create_and_store_resource()` |
| Secret / Provider | `{base}-{4-char}` | 63 | `create_and_store_resource()` |
| NAD | `cb-{6-char-hash}-{n}` | 15 | `multus_network_name` fixture |
| Target namespace | `{uuid}{prefix}` | 63 | `target_namespace` fixture |
| Cloned VM | `{uuid}-{vm}-{suffix}-{4-char}` | 63 | `_generate_clone_vm_name()` |
| Destination VM | DNS-1123 sanitized source name | 63 | `sanitize_kubernetes_name()` |
