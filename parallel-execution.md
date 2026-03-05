# Parallel Execution

Running tests in parallel with pytest-xdist for faster feedback cycles. This page covers the distribution strategy, worker isolation mechanisms, unique naming, and coordinated teardown that make parallel execution safe and reliable.

## Overview

mtv-api-tests uses [pytest-xdist](https://pytest-xdist.readthedocs.io/) to run test classes in parallel across multiple workers. Each worker operates in complete isolation with its own unique identifiers, namespaces, and resource tracking -- ensuring zero cross-worker interference on a shared OpenShift cluster.

**Key components:**

| Component | Purpose |
|-----------|---------|
| `--dist=loadscope` | Distributes entire test classes (not individual methods) to workers |
| `session_uuid` | Per-worker unique identifier prefixed to all resource names |
| `target_namespace` | Unique Kubernetes namespace per worker |
| `fixture_store` | Per-worker dictionary tracking all created resources for teardown |
| `pytest-harvest` | Serializes worker state for aggregated teardown and reporting |

## Distribution Strategy: `--dist=loadscope`

The distribution mode is configured in `pytest.ini`:

```ini
[pytest]
addopts =
  -s
  -o log_cli=true
  -p no:logging
  --tc-file=tests/tests_config/config.py
  --tc-format=python
  --junit-xml=junit-report.xml
  --basetemp=/tmp/pytest
  --show-progress
  --strict-markers
  --jira
  --dist=loadscope
```

The `loadscope` strategy groups tests by their **scope** -- for class-based tests, all methods in a class are sent to the same worker. This is critical because mtv-api-tests uses the `@pytest.mark.incremental` marker and shared class-level state (`self.__class__.attribute`) to implement the 5-step migration test pattern:

```
test_create_storagemap → test_create_networkmap → test_create_plan → test_migrate_vms → test_check_vms
```

These methods must execute sequentially on the same worker because each step depends on resources created by previous steps.

> **Warning:** Do not change the distribution strategy to `loadfile` or `load`. The `loadscope` strategy is required to keep incremental test classes together on a single worker. Using other strategies will break class-level state sharing and cause test failures.

### How `loadscope` distributes work

With 4 workers and 4 test classes, each class runs on a separate worker:

```
Worker gw0: TestSanityColdMigration     (5 test methods, sequential)
Worker gw1: TestWarmMigration           (5 test methods, sequential)
Worker gw2: TestRemoteMigration         (5 test methods, sequential)
Worker gw3: TestCopyOffloadMigration    (5 test methods, sequential)
```

All 5 methods within each class execute sequentially on their assigned worker, while different classes run in parallel across workers.

## Worker Isolation via `session_uuid`

Each xdist worker gets its own pytest session, which generates a unique identifier using `shortuuid`:

```python
# conftest.py
@pytest.fixture(scope="session")
def session_uuid(fixture_store):
    _session_uuid = generate_name_with_uuid(name="auto")
    fixture_store["session_uuid"] = _session_uuid
    return _session_uuid
```

```python
# utilities/naming.py
def generate_name_with_uuid(name: str) -> str:
    _name = f"{name}-{shortuuid.ShortUUID().random(length=4).lower()}"
    _name = _name.replace("_", "-").replace(".", "-").lower()
    return _name
```

This produces identifiers like `auto-x7k2` or `auto-m9pf`. The 4-character random suffix from `shortuuid` provides sufficient uniqueness across concurrent workers.

The `session_uuid` flows through to the `base_resource_name` fixture, which incorporates the provider type and version:

```python
# conftest.py
@pytest.fixture(scope="session")
def base_resource_name(fixture_store, session_uuid, source_provider_data):
    _name = f"{session_uuid}-source-{source_provider_data['type']}-{source_provider_data['version'].replace('.', '-')}"

    # Add copyoffload indicator for Plan/StorageMap/NetworkMap names
    if "copyoffload" in source_provider_data:
        _name = f"{_name}-xcopy"

    fixture_store["base_resource_name"] = _name
```

**Example `base_resource_name` values:**

| Worker | base_resource_name |
|--------|--------------------|
| gw0 | `auto-x7k2-source-vsphere-8-0` |
| gw1 | `auto-m9pf-source-vsphere-8-0` |
| gw2 | `auto-r3jn-source-vsphere-8-0-xcopy` |

Every OpenShift resource created through `create_and_store_resource()` uses this base name, ensuring globally unique resource names across all workers.

## Unique Namespaces

Each worker creates its own Kubernetes namespace by combining the `session_uuid` with the configured namespace prefix:

```python
# conftest.py
@pytest.fixture(scope="session")
def target_namespace(fixture_store, session_uuid, ocp_admin_client):
    """create the target namespace for MTV migrations"""
    label: dict[str, str] = {
        "pod-security.kubernetes.io/enforce": "restricted",
        "pod-security.kubernetes.io/enforce-version": "latest",
        "mutatevirtualmachines.kubemacpool.io": "ignore",
    }

    _target_namespace: str = py_config["target_namespace_prefix"]

    # Remove prefix value to avoid duplication - session_uuid is already generated from this prefix
    _target_namespace = _target_namespace.replace("auto", "")

    # Generate a unique namespace name to avoid conflicts and support run multiple runs with the same provider configs
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

If `target_namespace_prefix` is configured as `"auto-mtv-tests"`, each worker produces a namespace like:

- Worker gw0: `auto-x7k2-mtv-tests`
- Worker gw1: `auto-m9pf-mtv-tests`

The name is truncated to 63 characters to comply with the Kubernetes DNS-1123 naming limit.

> **Note:** Never hardcode namespace names in tests. Always use the `target_namespace` fixture to ensure proper isolation in parallel runs.

## `fixture_store` Per Worker

The `fixture_store` is a per-session dictionary provided by `pytest-harvest` via `get_fixture_store(session)`. In an xdist run, each worker has its own independent session, so each worker gets its own `fixture_store`.

### Initialization

The store is initialized in the `pytest_sessionstart` hook:

```python
# conftest.py
def pytest_sessionstart(session):
    _session_store = get_fixture_store(session)
    _session_store["teardown"] = {}
```

### Structure

During test execution, the `fixture_store` accumulates this structure:

```python
{
    "session_uuid": "auto-x7k2",
    "base_resource_name": "auto-x7k2-source-vsphere-8-0",
    "target_namespace": "auto-x7k2-mtv-tests",
    "vms_for_current_session": {"rhel8-vm", "centos9-vm"},
    "teardown": {
        "Namespace": [
            {"name": "auto-x7k2-mtv-tests", "namespace": "", "module": "ocp_resources.namespace"}
        ],
        "StorageMap": [
            {"name": "auto-x7k2-source-vsphere-8-0-ab1c", "namespace": "auto-x7k2-mtv-tests", "module": "..."}
        ],
        "NetworkMap": [
            {"name": "auto-x7k2-source-vsphere-8-0-de2f", "namespace": "auto-x7k2-mtv-tests", "module": "..."}
        ],
        "Plan": [
            {"name": "auto-x7k2-source-vsphere-8-0-gh3i-cold", "namespace": "auto-x7k2-mtv-tests", "module": "...", "test_name": "test_create_plan"}
        ],
    }
}
```

### Resource tracking

Every resource created through `create_and_store_resource()` is automatically registered in `fixture_store["teardown"]`:

```python
# utilities/resources.py
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    # ... name generation and resource deployment ...

    fixture_store["teardown"].setdefault(_resource.kind, []).append(_resource_dict)

    return _resource
```

This ensures every resource created by any worker is tracked for cleanup, regardless of which worker created it.

## Unique Resource Name Generation

The `create_and_store_resource()` function generates unique names by combining the `base_resource_name` with an additional short UUID:

```python
# utilities/resources.py
if not _resource_name:
    _resource_name = generate_name_with_uuid(name=fixture_store["base_resource_name"])

    if resource.kind in (Migration.kind, Plan.kind):
        _resource_name = f"{_resource_name}-{'warm' if kwargs.get('warm_migration') else 'cold'}"

if len(_resource_name) > 63:
    LOGGER.warning(f"'{_resource_name=}' is too long ({len(_resource_name)} > 63). Truncating.")
    _resource_name = _resource_name[-63:]
```

**Naming example for a cold migration Plan on worker gw0:**

```
auto-x7k2-source-vsphere-8-0-ab1c-cold
│         │                   │    │
│         │                   │    └── migration type suffix
│         │                   └── additional shortuuid (4 chars)
│         └── provider type and version
└── session_uuid (unique per worker)
```

### Class-level resource naming

For resources scoped to a test class (such as NetworkAttachmentDefinitions), a SHA-256 hash of the pytest node ID provides deterministic, class-unique names:

```python
# utilities/utils.py
def generate_class_hash_prefix(nodeid: str, length: int = 6) -> str:
    """Generate a FIPS-compliant hash prefix for class-based resource naming."""
    return hashlib.sha256(nodeid.encode()).hexdigest()[:length]
```

```python
# conftest.py (multus_network_name fixture)
hash_prefix = generate_class_hash_prefix(request.node.nodeid)
base_name = f"cb-{hash_prefix}"  # e.g., "cb-a1b2c3"

# Creates NADs like: cb-a1b2c3-1, cb-a1b2c3-2
for i in range(1, multus_count + 1):
    nad_name = f"{base_name}-{i}"
    create_and_store_resource(
        fixture_store=fixture_store,
        resource=NetworkAttachmentDefinition,
        client=ocp_admin_client,
        namespace=nad_namespace,
        config=config,
        name=nad_name,
    )
```

> **Tip:** The `cb-` prefix and 6-character hash keep NAD names within the 15-character Linux bridge interface name limit while maintaining uniqueness across parallel test classes.

## Cross-Worker Coordination with pytest-harvest

When running with xdist, each worker's `fixture_store` must be collected by the main process for aggregated teardown and reporting. This is handled by `pytest-harvest`'s xdist hooks:

```python
# conftest.py
RESULTS_PATH = Path("./.xdist_results/")
RESULTS_PATH.mkdir(exist_ok=True)


def pytest_harvest_xdist_init():
    # reset the recipient folder
    if RESULTS_PATH.exists():
        rmtree(RESULTS_PATH)
    RESULTS_PATH.mkdir(exist_ok=False)
    return True


def pytest_harvest_xdist_worker_dump(worker_id, session_items, fixture_store):
    # persist session_items and fixture_store in the file system
    with open(RESULTS_PATH / (f"{worker_id}.pkl"), "wb") as f:
        try:
            pickle.dump((session_items, fixture_store), f)
        except Exception as exp:
            LOGGER.warning(
                f"Error while pickling worker {worker_id}'s harvested results: [{exp.__class__}] {exp}"
            )
    return True


def pytest_harvest_xdist_load():
    # restore the saved objects from file system
    workers_saved_material = {}
    for pkl_file in RESULTS_PATH.glob("*.pkl"):
        wid = pkl_file.stem
        with pkl_file.open("rb") as f:
            workers_saved_material[wid] = pickle.load(f)
    return workers_saved_material


def pytest_harvest_xdist_cleanup():
    # delete all temporary pickle files
    rmtree(RESULTS_PATH)
    return True
```

**Lifecycle:**

1. **`pytest_harvest_xdist_init`** -- Main process creates `.xdist_results/` before workers start
2. **Workers execute tests** -- Each worker maintains its own `fixture_store` in memory
3. **`pytest_harvest_xdist_worker_dump`** -- Each worker serializes its `fixture_store` to `.xdist_results/{worker_id}.pkl`
4. **`pytest_harvest_xdist_load`** -- Main process deserializes all worker results from pickle files
5. **`pytest_harvest_xdist_cleanup`** -- Temporary `.xdist_results/` directory is removed

## Shared Resource Coordination

Some resources must be shared across workers without duplication. The `virtctl_binary` fixture demonstrates the file-locking pattern for safe cross-worker coordination:

```python
# conftest.py
@pytest.fixture(scope="session")
def virtctl_binary(ocp_admin_client: "DynamicClient") -> Path:
    """Download and configure virtctl binary from the cluster.

    Uses file locking to handle pytest-xdist parallel execution safely.
    The binary is downloaded to a shared directory that all workers can access.
    """
    cluster_version_str = get_cluster_version_str(ocp_admin_client)

    shared_dir = Path(tempfile.gettempdir()) / "pytest-shared-virtctl" / cluster_version_str.replace(".", "-")
    shared_dir.mkdir(mode=0o700, parents=True, exist_ok=True)

    # Security: check for symlink hijack attack
    if shared_dir.is_symlink():
        raise PermissionError(
            f"Security error: shared directory {shared_dir} is a symlink. This may indicate a hijack attempt."
        )

    lock_file = shared_dir / "virtctl.lock"
    virtctl_path = shared_dir / "virtctl"

    try:
        # File lock ensures only one process downloads
        with filelock.FileLock(lock_file, timeout=600):
            if not virtctl_path.is_file() or not os.access(virtctl_path, os.X_OK):
                download_virtctl_from_cluster(client=ocp_admin_client, download_dir=shared_dir)
    except filelock.Timeout as err:
        raise TimeoutError(
            f"Timeout (600s) waiting for virtctl lock at {lock_file}. Another process may be stuck."
        ) from err

    add_to_path(str(shared_dir))
    return virtctl_path
```

**Key patterns:**
- **Persistent shared directory** in `/tmp/` (not pytest's `tmp_path`) so all workers can access it
- **`filelock.FileLock`** ensures only one worker downloads the binary; others wait
- **Version-keyed cache** (`/tmp/pytest-shared-virtctl/4-15-0/`) for automatic invalidation across cluster versions
- **Security checks** for symlink hijack and ownership validation

## Teardown in Parallel Context

Teardown happens at two levels:

### Class-level teardown

The `cleanup_migrated_vms` fixture runs after each test class completes on its worker:

```python
# conftest.py
@pytest.fixture(scope="class")
def cleanup_migrated_vms(
    request: pytest.FixtureRequest,
    ocp_admin_client: DynamicClient,
    target_namespace: str,
    prepared_plan: dict[str, Any],
) -> Generator[None, None, None]:
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

### Session-level teardown

After all workers finish, the main process runs `session_teardown()` which cleans up all tracked resources:

```python
# utilities/pytest_utils.py
def session_teardown(session_store: dict[str, Any]) -> None:
    LOGGER.info("Running teardown to delete all created resources")

    ocp_client = get_cluster_client()

    # When running in parallel (-n auto) `session_store` can be empty.
    if session_teardown_resources := session_store.get("teardown"):
        for migration_name in session_teardown_resources.get(Migration.kind, []):
            migration = Migration(name=migration_name["name"], namespace=migration_name["namespace"], client=ocp_client)
            cancel_migration(migration=migration)

        for plan_name in session_teardown_resources.get(Plan.kind, []):
            plan = Plan(name=plan_name["name"], namespace=plan_name["namespace"], client=ocp_client)
            archive_plan(plan=plan)

        leftovers = teardown_resources(
            session_store=session_store,
            ocp_client=ocp_client,
            target_namespace=session_store.get("target_namespace"),
        )
        if leftovers:
            raise SessionTeardownError(f"Failed to clean up the following resources: {leftovers}")
```

The teardown processes resources in dependency order: Migrations, Plans, Providers, Hosts, Secrets, NetworkMaps, StorageMaps, NetworkAttachmentDefinitions, VirtualMachines, Pods, and finally Namespaces.

> **Note:** The comment `"When running in parallel (-n auto) session_store can be empty"` reflects that individual workers may not have created resources if test collection was empty for that worker. The guard clause prevents errors in this case.

## Incremental Tests and Parallel Safety

The `@pytest.mark.incremental` marker ensures that if one test method fails, subsequent methods in the same class are marked as `xfail` rather than running against broken state:

```python
# conftest.py
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    setattr(item, "rep_" + rep.when, rep)

    # Incremental test support - track failures for class-based tests
    if "incremental" in item.keywords and rep.when == "call" and rep.failed:
        item.parent._previousfailed = item


def pytest_runtest_setup(item):
    # Incremental test support - xfail if previous test in class failed
    if "incremental" in item.keywords:
        previousfailed = getattr(item.parent, "_previousfailed", None)
        if previousfailed is not None:
            pytest.xfail(f"previous test failed ({previousfailed.name})")
```

This is parallel-safe because `_previousfailed` is stored on the test class's parent node (`item.parent`), which is local to each worker. Different workers running different test classes never share this state.

## Rules for Writing Parallel-Safe Tests

1. **Always use fixtures for namespaces** -- never hardcode namespace names
2. **Never share mutable state between test classes** -- each class should be self-contained
3. **Always use `create_and_store_resource()`** -- it generates unique names and tracks resources for teardown
4. **Use class-scoped fixtures for shared resources** -- `loadscope` guarantees class methods run on the same worker
5. **Use `@pytest.mark.usefixtures("cleanup_migrated_vms")`** -- ensures VM cleanup runs after each test class
6. **Use file locking for shared external resources** -- follow the `virtctl_binary` pattern with `filelock.FileLock`

## Quick Reference

### Running tests in parallel

```bash
# Auto-detect number of workers (one per CPU core)
pytest -n auto

# Specify exact number of workers
pytest -n 4

# Run a specific marker with parallel execution
pytest -n auto -m tier0

# Skip teardown for debugging (resources persist after tests)
pytest -n auto --skip-teardown
```

> **Tip:** The `--dist=loadscope` flag is already set in `pytest.ini`, so you only need to pass `-n <workers>` to enable parallel execution. Without `-n`, tests run sequentially on a single process.

### Isolation summary by scope

| Scope | Isolation Mechanism | Example |
|-------|---------------------|---------|
| Worker session | `session_uuid` via `shortuuid` | `auto-x7k2` |
| Namespace | `session_uuid` + config prefix | `auto-x7k2-mtv-tests` |
| Resource names | `base_resource_name` + `shortuuid` | `auto-x7k2-source-vsphere-8-0-ab1c-cold` |
| Class resources (NADs) | SHA-256 hash of pytest node ID | `cb-a1b2c3-1` |
| Fixture state | `pytest-harvest` per-worker `fixture_store` | Serialized to `.xdist_results/gw0.pkl` |
| Shared binaries | `filelock.FileLock` in `/tmp/` | `virtctl.lock` with 600s timeout |
