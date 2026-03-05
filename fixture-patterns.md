# Fixture Patterns

This page documents the pytest fixture architecture used in the MTV API Tests. All fixtures are centralized in a single `conftest.py` at the project root. Understanding fixture scopes, the `fixture_store` resource tracker, and the `class_plan_config`/`prepared_plan` relationship is essential for writing and maintaining tests.

## Fixture Scopes Overview

The project uses three fixture scopes, each serving a distinct purpose:

| Scope | Lifetime | Created | Shared Across | Use Case |
|-------|----------|---------|---------------|----------|
| **session** | Entire test run | Once per pytest session | All test classes and methods | Cluster clients, provider connections, namespaces, configuration |
| **class** | One test class | Once per `@pytest.mark.parametrize` class | All test methods in that class | VM preparation, network setup, cleanup |
| **function** | One test method | Once per test method | Single test method only | Not used in this codebase |

> **Note:** Function-scoped fixtures are intentionally avoided. All fixtures are either session or class-scoped to maximize resource reuse and minimize test execution time.

## Session-Scoped Fixtures

Session-scoped fixtures are created once when the test session starts and are shared across all tests. They represent long-lived resources like cluster connections and provider configurations.

### Core Infrastructure Fixtures

#### `ocp_admin_client`

Creates the OpenShift cluster client used for all cluster interactions:

```python
@pytest.fixture(scope="session")
def ocp_admin_client():
    LOGGER.info(msg="Creating OCP admin Client")
    _client = get_cluster_client()

    if remote_cluster_name := get_value_from_py_config("remote_ocp_cluster"):
        if remote_cluster_name not in _client.configuration.host:
            raise RemoteClusterAndLocalCluterNamesError("Remote cluster must be the same as local cluster.")

    yield _client
```

#### `session_uuid`

Generates a unique identifier for the test session. This UUID is used to create unique resource names and prevent conflicts during parallel execution:

```python
@pytest.fixture(scope="session")
def session_uuid(fixture_store):
    _session_uuid = generate_name_with_uuid(name="auto")
    fixture_store["session_uuid"] = _session_uuid
    return _session_uuid
```

The generated UUID follows the pattern `auto-<short-uuid>` (e.g., `auto-a1b2c3`).

#### `target_namespace`

Creates a unique namespace for migration targets, with labels required for pod security and kubemacpool:

```python
@pytest.fixture(scope="session")
def target_namespace(fixture_store, session_uuid, ocp_admin_client):
    label: dict[str, str] = {
        "pod-security.kubernetes.io/enforce": "restricted",
        "pod-security.kubernetes.io/enforce-version": "latest",
        "mutatevirtualmachines.kubemacpool.io": "ignore",
    }

    _target_namespace: str = py_config["target_namespace_prefix"]
    _target_namespace = _target_namespace.replace("auto", "")

    unique_namespace_name = f"{session_uuid}{_target_namespace}"[:63]
    fixture_store["target_namespace"] = unique_namespace_name

    namespace = create_and_store_resource(
        fixture_store=fixture_store,
        resource=Namespace,
        client=ocp_admin_client,
        name=unique_namespace_name,
        label=label,
    )
    namespace.wait_for_status(status=namespace.Status.ACTIVE)
    yield namespace.name
```

> **Tip:** The namespace name is truncated to 63 characters to comply with the Kubernetes name length limit.

### Provider Fixtures

#### `source_provider_data`

Resolves provider configuration from `.providers.json`. This fixture validates the provider exists and stores it in `fixture_store` for session teardown:

```python
@pytest.fixture(scope="session")
def source_provider_data(source_providers, fixture_store):
    requested_provider = py_config["source_provider"]
    if requested_provider not in source_providers:
        raise ValueError(
            f"Source provider '{requested_provider}' not found in '.providers.json'. "
            f"Available providers: {sorted(source_providers.keys())}"
        )

    _source_provider = source_providers[requested_provider]
    fixture_store["source_provider_data"] = _source_provider
    return _source_provider
```

#### `source_provider`

Creates a provider connection using a context manager. The provider is automatically disconnected during teardown:

```python
@pytest.fixture(scope="session")
def source_provider(
    fixture_store, session_uuid, source_provider_data, target_namespace,
    ocp_admin_client, tmp_path_factory, destination_ocp_secret,
):
    with create_source_provider(
        fixture_store=fixture_store,
        session_uuid=session_uuid,
        source_provider_data=source_provider_data,
        namespace=target_namespace,
        admin_client=ocp_admin_client,
        tmp_dir=tmp_path_factory,
        ocp_admin_client=ocp_admin_client,
        destination_ocp_secret=destination_ocp_secret,
        insecure=get_value_from_py_config(value="source_provider_insecure_skip_verify"),
    ) as _source_provider:
        yield _source_provider

    _source_provider.disconnect()
```

#### `destination_provider`

Creates the destination OpenShift Provider resource as a Forklift custom resource:

```python
@pytest.fixture(scope="session")
def destination_provider(session_uuid, ocp_admin_client, target_namespace, fixture_store):
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

    return OCPProvider(ocp_resource=provider, fixture_store=fixture_store)
```

### Other Session Fixtures

| Fixture | Purpose |
|---------|---------|
| `base_resource_name` | Generates base name for all resources (e.g., `auto-a1b2c3-source-vsphere-8-0`) |
| `mtv_namespace` | Returns the MTV operator namespace from config |
| `source_providers` | Loads all provider configs from `.providers.json` |
| `source_provider_inventory` | Creates ForkliftInventory instance matching the provider type |
| `source_vms_namespace` | Creates namespace for source CNV VMs (OpenShift source only) |
| `multus_cni_config` | Returns Multus CNI bridge configuration JSON |
| `virtctl_binary` | Downloads and caches `virtctl` binary with file-lock safety for parallel workers |
| `forklift_pods_state` | Validates all Forklift pods are running before tests start |
| `nfs_storage_profile` | Configures NFS StorageProfile if NFS storage class is used |
| `destination_ocp_secret` | Creates Secret with OCP API token for destination cluster auth |
| `destination_ocp_provider` | Creates destination OCP Provider (alternative to `destination_provider` for remote clusters) |
| `precopy_interval_forkliftcontroller` | Sets snapshot interval in ForkliftController CR (warm migration only) |

## The Autouse Fixture

The project enforces a strict rule: **only one autouse fixture exists**, named `autouse_fixtures`. This is the sole session-scoped autouse fixture and acts as a gatekeeper that validates all prerequisites before any tests run:

```python
@pytest.fixture(scope="session", autouse=True)
def autouse_fixtures(
    source_provider_data, nfs_storage_profile, base_resource_name,
    forklift_pods_state, virtctl_binary
):
    # source_provider_data called here to fail fast in provider not found
    yield
```

By listing `source_provider_data` as its first dependency, the fixture ensures a fast failure if the requested provider is missing from `.providers.json`. The remaining dependencies trigger NFS configuration, resource name generation, Forklift pod validation, and virtctl binary download - all before any test method executes.

> **Warning:** Never create additional autouse fixtures. All prerequisite validation must be added as a dependency of `autouse_fixtures` rather than as a new autouse fixture.

## Class-Scoped Fixtures

Class-scoped fixtures are created once per test class and shared across all test methods in that class. They handle VM-specific setup and teardown.

### `class_plan_config` and `prepared_plan` Relationship

These two fixtures work together through pytest's indirect parametrization to drive the entire test configuration pipeline.

#### `class_plan_config`

A thin wrapper that extracts test configuration from `@pytest.mark.parametrize`:

```python
@pytest.fixture(scope="class")
def class_plan_config(request: pytest.FixtureRequest) -> dict[str, Any]:
    return request.param
```

This fixture receives its value through indirect parametrization at the class level:

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

The `indirect=True` flag tells pytest to pass the parametrized value as `request.param` to the `class_plan_config` fixture rather than directly to the test method.

#### `prepared_plan`

The core class-scoped fixture that transforms raw configuration into a fully prepared migration plan. It deep-copies the `class_plan_config`, clones VMs, updates names with unique suffixes, and populates source VM data:

```python
@pytest.fixture(scope="class")
def prepared_plan(
    request, class_plan_config, fixture_store, source_provider,
    source_vms_namespace, ocp_admin_client, multus_cni_config,
    source_provider_inventory, target_namespace,
) -> Generator[dict[str, Any], None, None]:
    plan: dict[str, Any] = deepcopy(class_plan_config)
    virtual_machines: list[dict[str, Any]] = plan["virtual_machines"]
    warm_migration = plan.get("warm_migration", False)

    # Initialize separate storage for source VM data
    plan["source_vms_data"] = {}

    # Handle custom VM target namespace
    vm_target_namespace = plan.get("vm_target_namespace")
    if vm_target_namespace:
        get_or_create_namespace(
            fixture_store=fixture_store,
            ocp_admin_client=ocp_admin_client,
            namespace_name=vm_target_namespace,
        )
        plan["_vm_target_namespace"] = vm_target_namespace
    else:
        plan["_vm_target_namespace"] = target_namespace

    # ... VM cloning, power state control, name updates ...

    for vm in virtual_machines:
        clone_options = {**vm, "enable_ctk": warm_migration}
        provider_vm_api = source_provider.get_vm_by_name(
            query=vm["name"],
            vm_name_suffix=vm_name_suffix,
            clone_vm=True,
            session_uuid=fixture_store["session_uuid"],
            clone_options=clone_options,
        )

        # Update VM name with cloned name
        source_vm_details = source_provider.vm_dict(...)
        vm["name"] = source_vm_details["name"]

        # Store complete source VM data separately
        plan["source_vms_data"][vm["name"]] = source_vm_details

    yield plan
```

#### Data flow diagram

```
tests_params (config.py)
    |
    v
@pytest.mark.parametrize("class_plan_config", [...], indirect=True)
    |
    v
class_plan_config fixture  -->  returns request.param (raw config dict)
    |
    v
prepared_plan fixture  -->  deep-copies config, clones VMs, updates names
    |
    v
Test methods receive prepared_plan with:
  - virtual_machines: list with updated (cloned) VM names
  - source_vms_data: dict with complete VM metadata
  - _vm_target_namespace: resolved target namespace
  - warm_migration, target_power_state, etc.
```

#### `prepared_plan` output structure

```python
{
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8-cold-a1b2c3",  # Updated with clone suffix
            "source_vm_power": "on",
            "guest_agent": True,
            "snapshots_before_migration": [...],
        },
    ],
    "source_vms_data": {
        "mtv-tests-rhel8-cold-a1b2c3": {
            "name": "...",
            "provider_vm_api": ...,
            "snapshots_data": [...],
        },
    },
    "_vm_target_namespace": "auto-a1b2c3-target",
    "warm_migration": False,
    # ... other keys from test config ...
}
```

> **Note:** `virtual_machines` is kept clean for Plan CR serialization. All provider-specific VM details go into `source_vms_data` to maintain a clean separation.

### Simultaneous Migration Fixtures

For tests that run two migrations concurrently, the `prepared_plan_1` and `prepared_plan_2` fixtures extract individual VMs from a multi-VM plan:

```python
@pytest.fixture(scope="class")
def prepared_plan_1(prepared_plan: dict[str, Any]) -> dict[str, Any]:
    return extract_vm_from_plan(prepared_plan, vm_index=0, fixture_name="prepared_plan_1")

@pytest.fixture(scope="class")
def prepared_plan_2(prepared_plan: dict[str, Any]) -> dict[str, Any]:
    return extract_vm_from_plan(prepared_plan, vm_index=1, fixture_name="prepared_plan_2")
```

### Other Class-Scoped Fixtures

| Fixture | Purpose |
|---------|---------|
| `multus_network_name` | Creates NetworkAttachmentDefinitions (NADs) with class-unique names (`cb-{hash}-1`, etc.) |
| `vm_ssh_connections` | Manages SSH connections to migrated VMs via `SSHConnectionManager` |
| `labeled_worker_node` | Labels a worker node for target scheduling tests; auto-cleaned via `ResourceEditor` |
| `target_vm_labels` | Generates VM labels from config; supports auto-generation with `session_uuid` |
| `mtv_version_checker` | Skips tests if MTV version is below the minimum specified by `@pytest.mark.min_mtv_version` |
| `mixed_datastore_config` | Validates mixed datastore configuration for copy-offload tests |

## fixture_store for Resource Tracking

The `fixture_store` is provided by the `pytest-harvest` plugin and serves as the central registry for all resources created during a test session. It enables deterministic cleanup at the end of the session.

### Initialization

The store is initialized during `pytest_sessionstart`:

```python
def pytest_sessionstart(session):
    _session_store = get_fixture_store(session)
    _session_store["teardown"] = {}
```

### Structure

Throughout the test session, `fixture_store` accumulates the following data:

```python
fixture_store = {
    # Identity and naming
    "session_uuid": "auto-a1b2c3",
    "base_resource_name": "auto-a1b2c3-source-vsphere-8-0",
    "target_namespace": "auto-a1b2c3-target",

    # Provider configuration (stored for session teardown)
    "source_provider_data": {
        "type": "vsphere",
        "version": "8.0",
        "fqdn": "vcenter.example.com",
        ...
    },

    # VM tracking for the current session
    "vms_for_current_session": {"mtv-tests-rhel8", "mtv-win2019-79"},

    # Resource cleanup registry
    "teardown": {
        "Namespace": [{"name": "auto-a1b2c3-target", "namespace": None, "module": "..."}],
        "Provider": [{"name": "auto-a1b2c3-local-ocp-provider", "namespace": "auto-a1b2c3-target", "module": "..."}],
        "Secret": [...],
        "StorageMap": [...],
        "NetworkMap": [...],
        "Plan": [{"name": "...", "namespace": "...", "module": "...", "test_name": "test_migrate_vms"}],
        "Migration": [...],
        "NetworkAttachmentDefinition": [...],
        "VirtualMachine": [...],
        # Provider-specific cloned VMs
        "vsphere": [{"name": "mtv-tests-rhel8-cold-a1b2c3"}],
        "openstack": [...],
        "ovirt": [...],
        "VolumeSnapshot": [...],
    },
}
```

### How `create_and_store_resource()` Populates It

Every OpenShift resource must be created through `create_and_store_resource()`, which automatically registers the resource for cleanup:

```python
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    # Auto-generate name if not provided
    if not _resource_name:
        _resource_name = generate_name_with_uuid(name=fixture_store["base_resource_name"])

        if resource.kind in (Migration.kind, Plan.kind):
            _resource_name = f"{_resource_name}-{'warm' if kwargs.get('warm_migration') else 'cold'}"

    # Truncate to Kubernetes 63-char limit
    if len(_resource_name) > 63:
        _resource_name = _resource_name[-63:]

    _resource = resource(**kwargs)

    try:
        _resource.deploy(wait=True)
    except ConflictError:
        LOGGER.warning(f"{_resource.kind} {_resource_name} already exists, reusing it.")
        _resource.wait()

    # Register for teardown
    _resource_dict = {
        "name": _resource.name,
        "namespace": _resource.namespace,
        "module": _resource.__module__,
    }
    if test_name:
        _resource_dict["test_name"] = test_name

    fixture_store["teardown"].setdefault(_resource.kind, []).append(_resource_dict)

    return _resource
```

> **Warning:** Never create OpenShift resources directly (e.g., `Namespace(...).deploy()`). Always use `create_and_store_resource()` to ensure proper cleanup tracking.

### Usage Example

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
```

## Cleanup and Teardown

Resource cleanup happens at two levels: class-level for migrated VMs and session-level for all tracked resources.

### `cleanup_migrated_vms` (Class-Level)

A teardown-only fixture that deletes VMs migrated during a test class. It runs after all test methods in the class complete:

```python
@pytest.fixture(scope="class")
def cleanup_migrated_vms(
    request: pytest.FixtureRequest,
    ocp_admin_client: DynamicClient,
    target_namespace: str,
    prepared_plan: dict[str, Any],
) -> Generator[None, None, None]:
    yield  # No setup - teardown only

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

This fixture is applied using `@pytest.mark.usefixtures()` at the class level because test methods do not need its return value:

```python
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestSanityColdMtvMigration:
    ...
```

> **Note:** The `--skip-teardown` CLI flag skips both class-level and session-level cleanup. This is useful for debugging failed migrations.

### Session-Level Teardown

At session end, `pytest_sessionfinish` invokes `session_teardown()` to clean up all resources tracked in `fixture_store["teardown"]`:

```python
def pytest_sessionfinish(session, exitstatus):
    _session_store = get_fixture_store(session)

    if session.config.getoption("skip_teardown"):
        LOGGER.warning("User requested to skip teardown of resources")
    else:
        session_teardown(session_store=_session_store)
```

The `session_teardown()` function in `utilities/pytest_utils.py` processes resources in a specific order:

1. **Cancel active Migrations** - Stop any in-progress migrations
2. **Archive Plans** - Mark Plans as archived before deletion
3. **Delete tracked resources** - Migrations, Plans, Providers, Secrets, NetworkMaps, StorageMaps, NADs, VMs, Pods
4. **Delete VMs in target namespace** - Catch any untracked VMs
5. **Wait for pod deletion** - Parallel wait using `ThreadPoolExecutor`
6. **Verify DV/PVC/PV cleanup** - Check that DataVolumes, PVCs, and PVs are gone
7. **Delete namespaces** - Remove test namespaces last
8. **Delete cloned VMs on source providers** - Connect back to VMware/OpenStack/RHV to delete clones

If any resources fail to clean up, a `SessionTeardownError` is raised with the list of leftovers.

### Cleanup Layers Summary

```
Test Class Ends
    |
    v
cleanup_migrated_vms  -->  Deletes migrated VMs from target namespace
    |
    v
Session Ends
    |
    v
session_teardown()  -->  Processes fixture_store["teardown"]
    |-- Cancel Migrations
    |-- Archive & delete Plans
    |-- Delete Providers, Secrets, Maps, NADs
    |-- Delete remaining VMs & Pods
    |-- Delete Namespaces
    |-- Delete cloned VMs on source providers (VMware/OpenStack/RHV)
```

## Fixture Request Patterns

There are two ways to request a fixture, and the choice depends on whether you need the fixture's return value.

### Method Parameters (when you need the value)

```python
def test_create_storagemap(
    self,
    prepared_plan,      # Need plan data for VM names
    fixture_store,      # Need store for resource tracking
    ocp_admin_client,   # Need client for API calls
    source_provider,    # Need provider for storage info
    ...
):
    vms = [vm["name"] for vm in prepared_plan["virtual_machines"]]
    self.__class__.storage_map = get_storage_migration_map(
        fixture_store=fixture_store,
        source_provider=source_provider,
        ...
    )
```

### `@pytest.mark.usefixtures()` (for side-effect fixtures)

Use this for fixtures that perform setup/teardown but whose return value is not needed:

```python
@pytest.mark.usefixtures("cleanup_migrated_vms")
@pytest.mark.usefixtures("precopy_interval_forkliftcontroller")
class TestSanityWarmMtvMigration:
    # cleanup_migrated_vms: runs VM cleanup on teardown
    # precopy_interval_forkliftcontroller: configures snapshot intervals
    ...
```

> **Warning:** Never request a fixture via both a method parameter AND `@pytest.mark.usefixtures()`. Choose one approach per fixture.

## Sharing State Between Test Methods

Test classes use `self.__class__` to share resources created in one test method with subsequent methods:

```python
@pytest.mark.incremental
class TestSanityColdMtvMigration:
    storage_map: StorageMap     # Type annotations for class attributes
    network_map: NetworkMap
    plan_resource: Plan

    def test_create_storagemap(self, ...):
        self.__class__.storage_map = get_storage_migration_map(...)
        assert self.storage_map

    def test_create_plan(self, ...):
        # Access storage_map created in test_create_storagemap
        self.__class__.plan_resource = create_plan_resource(
            storage_map=self.storage_map,
            ...
        )
```

The `@pytest.mark.incremental` marker ensures that if a test method fails, all subsequent methods in the class are marked as `xfail` rather than running and producing misleading errors:

```python
def pytest_runtest_setup(item):
    if "incremental" in item.keywords:
        previousfailed = getattr(item.parent, "_previousfailed", None)
        if previousfailed is not None:
            pytest.xfail(f"previous test failed ({previousfailed.name})")
```

## Parallel Execution Support (pytest-xdist)

Fixtures are designed for safe parallel execution with `pytest-xdist`:

- **Unique namespaces**: Each session gets a unique namespace via `session_uuid`, preventing resource conflicts between workers
- **Isolated fixture stores**: Each xdist worker gets its own `fixture_store` instance via `pytest-harvest`
- **File locking**: Shared resources like the `virtctl` binary use `filelock` to prevent race conditions:

```python
lock_file = shared_dir / "virtctl.lock"
with filelock.FileLock(lock_file, timeout=600):
    if not virtctl_path.is_file() or not os.access(virtctl_path, os.X_OK):
        download_virtctl_from_cluster(client=ocp_admin_client, download_dir=shared_dir)
```

- **Pickle-based serialization**: Worker results are shared via pickle files for cross-worker result aggregation:

```python
def pytest_harvest_xdist_worker_dump(worker_id, session_items, fixture_store):
    with open(RESULTS_PATH / (f"{worker_id}.pkl"), "wb") as f:
        pickle.dump((session_items, fixture_store), f)
    return True
```

## Complete Test Class Example

This example from `test_mtv_cold_migration.py` shows all fixture patterns working together:

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

    def test_create_storagemap(
        self, prepared_plan, fixture_store, ocp_admin_client,
        source_provider, destination_provider,
        source_provider_inventory, target_namespace,
    ):
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

    def test_create_networkmap(self, prepared_plan, fixture_store, ...):
        ...

    def test_create_plan(self, prepared_plan, fixture_store, ...):
        populate_vm_ids(prepared_plan, source_provider_inventory)
        self.__class__.plan_resource = create_plan_resource(
            storage_map=self.storage_map,
            network_map=self.network_map,
            ...
        )
        assert self.plan_resource, "Plan creation failed"

    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        execute_migration(
            plan=self.plan_resource,
            fixture_store=fixture_store,
            ocp_admin_client=ocp_admin_client,
            target_namespace=target_namespace,
        )

    def test_check_vms(self, prepared_plan, source_provider, ...):
        check_vms(
            plan=prepared_plan,
            network_map_resource=self.network_map,
            storage_map_resource=self.storage_map,
            ...
        )
```

With this corresponding test configuration in `tests/tests_config/config.py`:

```python
tests_params: dict = {
    "test_sanity_cold_mtv_migration": {
        "virtual_machines": [
            {"name": "mtv-tests-rhel8", "guest_agent": True},
        ],
        "warm_migration": False,
    },
}
```

## Fixture Dependency Graph

The following shows the key fixture dependency chains:

```
autouse_fixtures (session, autouse=True)
├── source_provider_data ← source_providers
├── base_resource_name ← session_uuid, fixture_store
├── nfs_storage_profile ← ocp_admin_client
├── forklift_pods_state ← ocp_admin_client
└── virtctl_binary ← ocp_admin_client

class_plan_config (class) ← @pytest.mark.parametrize(..., indirect=True)
    └── prepared_plan (class)
        ├── source_provider (session) ← source_provider_data, ocp_admin_client, ...
        ├── source_provider_inventory (session) ← source_provider, ocp_admin_client
        ├── target_namespace (session) ← session_uuid, ocp_admin_client
        └── fixture_store (session, from pytest-harvest)

cleanup_migrated_vms (class) ← prepared_plan, ocp_admin_client, target_namespace
```

## Rules Summary

1. **One autouse fixture only** - `autouse_fixtures` at session scope; add new prerequisites as dependencies
2. **Noun names** - Fixtures represent resources (`source_provider`, not `setup_provider`)
3. **No magic skips** - Use `@pytest.mark.skipif` at test/class level, not `pytest.skip()` in fixtures
4. **Validate at source** - Fixtures validate the values they produce; test methods should not contain validation logic
5. **Always use `create_and_store_resource()`** - Never bypass the resource tracking system
6. **Choose one request method** - Use parameter for value access, `usefixtures` for side-effects; never both
7. **No function-scoped fixtures** - Use class or session scope for everything
