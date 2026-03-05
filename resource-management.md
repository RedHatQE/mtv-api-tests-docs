# Resource Management

This page covers the resource lifecycle in mtv-api-tests: how OpenShift/Kubernetes resources are created, named, tracked, and cleaned up. The central mechanism revolves around the `create_and_store_resource()` function and the `fixture_store` dictionary that tracks every resource for automatic teardown.

## Core Principle

Every OpenShift resource created during a test session must go through `create_and_store_resource()`. This ensures:

- Unique, deterministic naming with UUID suffixes
- Kubernetes-compliant name lengths (max 63 characters)
- Automatic registration in the `fixture_store` for teardown
- Graceful handling of naming conflicts

Direct resource instantiation and deployment bypasses tracking and will leave orphaned resources on the cluster.

```python
# Correct - tracked and cleaned up automatically
namespace = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Namespace,
    client=ocp_admin_client,
    name="my-namespace",
)

# Wrong - bypasses tracking, resource will not be cleaned up
namespace = Namespace(client=ocp_admin_client, name="my-namespace")
namespace.deploy()
```

## create_and_store_resource()

**Location:** `utilities/resources.py`

```python
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `client` | `DynamicClient` | OpenShift client for API interactions |
| `fixture_store` | `dict[str, Any]` | Session-scoped store for resource tracking |
| `resource` | `type[Resource]` | The `ocp_resources` class to instantiate (e.g., `Namespace`, `Plan`, `StorageMap`) |
| `test_name` | `str \| None` | Optional test name for associating resources with specific tests |
| `**kwargs` | `Any` | Passed through to the resource constructor (e.g., `name`, `namespace`, `mapping`, `kind_dict`) |

### Name Resolution

The function resolves the resource name through a priority chain:

```
1. Explicit `name` kwarg  →  use as-is
2. `yaml_file` kwarg      →  parse YAML, extract metadata.name
3. `kind_dict` kwarg       →  extract metadata.name from dict
4. None of the above       →  auto-generate with UUID
```

> **Warning:** You cannot specify both `yaml_file` and `kind_dict`. The function raises `ValueError` if both are provided.

#### Auto-Generated Names

When no name is provided, the function generates one using the `base_resource_name` stored in `fixture_store`:

```python
_resource_name = generate_name_with_uuid(name=fixture_store["base_resource_name"])
```

The `generate_name_with_uuid()` function (from `utilities/naming.py`) appends a 4-character random suffix:

```python
def generate_name_with_uuid(name: str) -> str:
    _name = f"{name}-{shortuuid.ShortUUID().random(length=4).lower()}"
    _name = _name.replace("_", "-").replace(".", "-").lower()
    return _name
```

For `Migration` and `Plan` resources, a migration type suffix is also appended:

```python
if resource.kind in (Migration.kind, Plan.kind):
    _resource_name = f"{_resource_name}-{'warm' if kwargs.get('warm_migration') else 'cold'}"
```

**Example name progression:**

```
session_uuid:       "auto-x7km"
base_resource_name: "auto-x7km-source-vsphere-8-0"
auto-generated:     "auto-x7km-source-vsphere-8-0-ab2f"
plan resource:      "auto-x7km-source-vsphere-8-0-ab2f-cold"
```

### 63-Character Truncation

Kubernetes DNS-1123 limits resource names to 63 characters. When an auto-generated name exceeds this limit, the function truncates from the left, keeping the last 63 characters:

```python
if len(_resource_name) > 63:
    LOGGER.warning(f"'{_resource_name=}' is too long ({len(_resource_name)} > 63). Truncating.")
    _resource_name = _resource_name[-63:]
```

> **Note:** Truncation preserves the right side of the name (UUID suffix and migration type), ensuring uniqueness is maintained even after truncation.

The target namespace fixture applies the same 63-character constraint directly:

```python
unique_namespace_name = f"{session_uuid}{_target_namespace}"[:63]
```

### Conflict Handling

If a resource with the same name already exists on the cluster, the function catches the `ConflictError` (HTTP 409) and reuses the existing resource instead of failing:

```python
try:
    _resource.deploy(wait=True)
except ConflictError:
    LOGGER.warning(f"{_resource.kind} {_resource_name} already exists, reusing it.")
    _resource.wait()
```

The resource is still registered in `fixture_store` for teardown, even when reused. This ensures cleanup happens regardless of whether the resource was freshly created or pre-existing.

### fixture_store Tracking

After deployment (or reuse), the resource metadata is appended to `fixture_store["teardown"]` under its resource kind:

```python
_resource_dict = {
    "name": _resource.name,
    "namespace": _resource.namespace,
    "module": _resource.__module__,
}

if test_name:
    _resource_dict["test_name"] = test_name

fixture_store["teardown"].setdefault(_resource.kind, []).append(_resource_dict)
```

The `test_name` field is used to associate Plan resources with specific test methods, enabling per-test data collection on failure.

## fixture_store Structure

The `fixture_store` is a session-scoped dictionary provided by the `pytest-harvest` plugin. It is initialized in the `pytest_sessionstart` hook:

```python
def pytest_sessionstart(session):
    _session_store = get_fixture_store(session)
    _session_store["teardown"] = {}
```

As fixtures and test setup run, the store accumulates metadata:

```python
{
    "session_uuid": "auto-x7km",
    "base_resource_name": "auto-x7km-source-vsphere-8-0",
    "target_namespace": "auto-x7km-mtv",
    "source_provider_data": { ... },
    "teardown": {
        "Namespace":  [{"name": "auto-x7km-mtv", "namespace": None, "module": "..."}],
        "Secret":     [{"name": "...", "namespace": "...", "module": "..."}],
        "Provider":   [{"name": "...", "namespace": "...", "module": "..."}],
        "StorageMap": [{"name": "...", "namespace": "...", "module": "..."}],
        "NetworkMap": [{"name": "...", "namespace": "...", "module": "..."}],
        "Plan":       [{"name": "...", "namespace": "...", "module": "...", "test_name": "..."}],
        "Migration":  [{"name": "...", "namespace": "...", "module": "..."}],
        "VirtualMachine": [{"name": "...", "namespace": "...", "module": "..."}],
        "Pod":        [{"name": "...", "namespace": "...", "module": "..."}],
        "vsphere":    [{"name": "cloned-vm-1"}],
        "openstack":  [{"name": "cloned-vm-2"}],
        "ovirt":      [{"name": "cloned-vm-3"}],
        "VolumeSnapshot": [{"id": "snap-id", "name": "snap-name"}],
    },
}
```

> **Note:** Provider-specific cloned VMs (keyed by `Provider.ProviderType.VSPHERE`, etc.) and `VolumeSnapshot` entries are added directly by provider implementations, not through `create_and_store_resource()`.

### session_uuid and base_resource_name

These two fixtures establish the naming foundation for the entire test session:

```python
@pytest.fixture(scope="session")
def session_uuid(fixture_store):
    _session_uuid = generate_name_with_uuid(name="auto")
    fixture_store["session_uuid"] = _session_uuid
    return _session_uuid


@pytest.fixture(scope="session")
def base_resource_name(fixture_store, session_uuid, source_provider_data):
    _name = f"{session_uuid}-source-{source_provider_data['type']}-{source_provider_data['version'].replace('.', '-')}"

    if "copyoffload" in source_provider_data:
        _name = f"{_name}-xcopy"

    fixture_store["base_resource_name"] = _name
```

The `session_uuid` uses a 4-character `ShortUUID` suffix (e.g., `auto-x7km`), giving each test session a unique identifier that propagates through all resource names.

## get_or_create_namespace()

**Location:** `utilities/resources.py`

A convenience function for namespace management that avoids creating duplicate namespaces:

```python
def get_or_create_namespace(
    fixture_store: dict[str, Any],
    ocp_admin_client: "DynamicClient",
    namespace_name: str,
) -> str:
```

### Behavior

1. **Check existence** — creates a `Namespace` object with the given name and checks `ns.exists`
2. **Reuse or create** — if the namespace exists, logs and skips creation. If not, calls `create_and_store_resource()` with standard security labels
3. **Wait for ACTIVE** — always waits for the namespace to reach `ACTIVE` status, whether new or existing
4. **Returns** — the namespace name as a string (not the Namespace object)

```python
ns = Namespace(name=namespace_name, client=ocp_admin_client)
if ns.exists:
    LOGGER.info(f"Namespace {namespace_name} already exists, using it")
else:
    LOGGER.info(f"Creating namespace {namespace_name}")
    ns = create_and_store_resource(
        fixture_store=fixture_store,
        resource=Namespace,
        client=ocp_admin_client,
        name=namespace_name,
        label={
            "pod-security.kubernetes.io/enforce": "restricted",
            "pod-security.kubernetes.io/enforce-version": "latest",
            "mutatevirtualmachines.kubemacpool.io": "ignore",
        },
    )
ns.wait_for_status(status=ns.Status.ACTIVE)
return namespace_name
```

> **Tip:** Only newly created namespaces are added to `fixture_store["teardown"]`. Pre-existing namespaces are reused without being registered for teardown, preventing accidental deletion of shared namespaces.

### Usage Examples

Custom Multus namespace for network attachment definitions:

```python
nad_namespace = get_or_create_namespace(
    fixture_store=fixture_store,
    ocp_admin_client=ocp_admin_client,
    namespace_name=multus_namespace,
)
```

Custom target namespace for VM migration:

```python
get_or_create_namespace(
    fixture_store=fixture_store,
    ocp_admin_client=ocp_admin_client,
    namespace_name=vm_target_namespace,
)
```

## Teardown Ordering

Resource teardown happens at two levels: class-level (per test class) and session-level (end of test run). The ordering is carefully designed to respect resource dependencies.

### Class-Level Teardown: cleanup_migrated_vms

The `cleanup_migrated_vms` fixture runs after each test class completes, cleaning up VMs that were migrated during the tests:

```python
@pytest.fixture(scope="class")
def cleanup_migrated_vms(request, ocp_admin_client, target_namespace, prepared_plan):
    yield

    if request.config.getoption("skip_teardown"):
        LOGGER.info("Skipping VM cleanup due to --skip-teardown flag")
        return

    vm_namespace = prepared_plan.get("_vm_target_namespace", target_namespace)

    for vm in prepared_plan["virtual_machines"]:
        vm_name = vm["name"]
        vm_obj = VirtualMachine(
            client=ocp_admin_client,
            name=vm_name,
            namespace=vm_namespace,
        )
        if vm_obj.exists:
            LOGGER.info(f"Cleaning up migrated VM: {vm_name} from namespace: {vm_namespace}")
            vm_obj.clean_up()
```

Applied via decorator at the test class level:

```python
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestVSphereMigration:
    ...
```

### Session-Level Teardown

The `pytest_sessionfinish` hook triggers the full session teardown, orchestrated by `session_teardown()` in `utilities/pytest_utils.py`. Resources are deleted in a specific dependency-aware order:

```
Phase 1: Cancel & Archive (graceful shutdown)
├── Cancel all active Migrations
└── Archive all Plans

Phase 2: Delete test-created resources
├── Migrations     (wait for deletion)
├── Plans          (wait for deletion)
├── Providers      (wait for deletion)
├── Hosts          (wait for deletion)
├── Secrets        (wait for deletion)
├── NADs           (wait for deletion)
├── StorageMaps    (wait for deletion)
└── NetworkMaps    (wait for deletion)

Phase 3: Delete migration-created resources
├── VirtualMachines  (check exists, then delete)
├── Pods             (check exists, then delete)
├── Bulk pod cleanup (parallel, by session_uuid)
└── DataVolumes / PVCs / PVs verification

Phase 4: Delete namespaces
└── Namespaces     (deleted last, after all contained resources)

Phase 5: Clean provider-side resources
├── VMware cloned VMs
├── OpenStack cloned VMs
├── RHV cloned VMs
└── OpenStack volume snapshots
```

> **Warning:** The deletion order matters. Migrations must be cancelled before Plans can be archived. All resources within a namespace must be deleted before the namespace itself. Provider-side cloned VMs are cleaned up by connecting back to the source providers.

### Leftover Detection

After teardown, any resources that failed to delete are collected into a `leftovers` dictionary. If leftovers exist, a `SessionTeardownError` is raised:

```python
leftovers = teardown_resources(
    session_store=session_store,
    ocp_client=ocp_client,
    target_namespace=session_store.get("target_namespace"),
)
if leftovers:
    raise SessionTeardownError(f"Failed to clean up the following resources: {leftovers}")
```

### Parallel Pod Cleanup

Session teardown uses a `ThreadPoolExecutor` to delete pods in parallel, improving cleanup speed when many migration pods exist:

```python
pods_to_wait = [
    _pod for _pod in Pod.get(client=ocp_client, namespace=target_namespace)
    if session_uuid in _pod.name
]

with ThreadPoolExecutor(max_workers=min(len(pods_to_wait), 10)) as executor:
    future_to_pod = {executor.submit(wait_for_pod_deletion, pod): pod for pod in pods_to_wait}
    for future in as_completed(future_to_pod):
        result = future.result()
        if not result["success"]:
            leftovers = append_leftovers(leftovers=leftovers, resource=result["pod"])
```

### Skip Teardown

The `--skip-teardown` CLI flag skips all resource cleanup, useful for debugging failed test runs:

```python
if session.config.getoption("skip_teardown"):
    LOGGER.warning("User requested to skip teardown of resources")
```

Both class-level and session-level teardown respect this flag.

## Parallel Execution Safety

Resource management is designed for parallel test execution with `pytest-xdist`:

- **Unique names per worker**: Each worker gets its own `session_uuid`, so resource names never collide across workers
- **Isolated fixture_store**: Each xdist worker maintains a separate `fixture_store` instance
- **Worker data persistence**: Fixture stores are serialized via `pickle` for cross-worker data collection

```python
def pytest_harvest_xdist_worker_dump(worker_id, session_items, fixture_store):
    with open(RESULTS_PATH / (f"{worker_id}.pkl"), "wb") as f:
        pickle.dump((session_items, fixture_store), f)
```

## Common Usage Patterns

### Creating a Provider with kind_dict

```python
kind_dict = {
    "apiVersion": "forklift.konveyor.io/v1beta1",
    "kind": "Provider",
    "metadata": {"name": f"{session_uuid}-local-ocp-provider", "namespace": target_namespace},
    "spec": {"secret": {}, "type": "openshift", "url": ""},
}

provider = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Provider,
    kind_dict=kind_dict,
    client=ocp_admin_client,
)
```

### Creating a StorageMap

```python
storage_map = create_and_store_resource(
    fixture_store=fixture_store,
    resource=StorageMap,
    client=ocp_admin_client,
    namespace=target_namespace,
    mapping=storage_map_list,
    source_provider_name=source_provider.ocp_resource.name,
    source_provider_namespace=source_provider.ocp_resource.namespace,
    destination_provider_name=destination_provider.ocp_resource.name,
    destination_provider_namespace=destination_provider.ocp_resource.namespace,
)
```

### Creating a Migration

```python
create_and_store_resource(
    client=ocp_admin_client,
    fixture_store=fixture_store,
    resource=Migration,
    namespace=target_namespace,
    plan_name=plan.name,
    plan_namespace=plan.namespace,
    cut_over=cut_over,
)
```

### Creating a Target Namespace

```python
namespace = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Namespace,
    client=ocp_admin_client,
    name=unique_namespace_name,
    label={
        "pod-security.kubernetes.io/enforce": "restricted",
        "pod-security.kubernetes.io/enforce-version": "latest",
        "mutatevirtualmachines.kubemacpool.io": "ignore",
    },
)
namespace.wait_for_status(status=namespace.Status.ACTIVE)
```

## Name Sanitization

For VM names imported from source providers, the `sanitize_kubernetes_name()` function (in `utilities/naming.py`) ensures Kubernetes DNS-1123 compliance:

```python
def sanitize_kubernetes_name(name: str, max_length: int = 63) -> str:
```

Rules applied:
- Converts to lowercase
- Replaces `_` and `.` with `-`
- Removes characters that are not alphanumeric or `-`
- Strips leading/trailing non-alphanumeric characters
- Truncates to `max_length` (default 63)
- Raises `InvalidVMNameError` if no valid characters remain
