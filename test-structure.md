# Test Class Structure

Every migration test in mtv-api-tests follows a standardized 5-step class-based pattern. Each test class creates the OpenShift resources needed for a migration, executes it, and validates the results. This page explains the structure in detail.

## Overview

A migration test class orchestrates five sequential steps:

| Step | Method | Purpose |
|------|--------|---------|
| 1 | `test_create_storagemap` | Map source storage to destination storage classes |
| 2 | `test_create_networkmap` | Map source networks to destination networks |
| 3 | `test_create_plan` | Create the MTV Plan Custom Resource |
| 4 | `test_migrate_vms` | Execute the migration and wait for completion |
| 5 | `test_check_vms` | Validate migrated VMs match source configuration |

These steps must run in order because each depends on resources created by prior steps. The `@pytest.mark.incremental` marker enforces this — if any step fails, subsequent steps are automatically marked as `xfail`.

## Anatomy of a Test Class

Here is the simplest complete example, from `tests/test_mtv_cold_migration.py`:

```python
import pytest
from ocp_resources.network_map import NetworkMap
from ocp_resources.plan import Plan
from ocp_resources.storage_map import StorageMap
from pytest_testconfig import config as py_config

from utilities.mtv_migration import (
    create_plan_resource,
    execute_migration,
    get_network_migration_map,
    get_storage_migration_map,
)
from utilities.post_migration import check_vms
from utilities.utils import populate_vm_ids


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

    def test_create_storagemap(self, prepared_plan, fixture_store, ocp_admin_client,
                               source_provider, destination_provider,
                               source_provider_inventory, target_namespace):
        """Create StorageMap resource for migration."""
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

    def test_create_networkmap(self, prepared_plan, fixture_store, ocp_admin_client,
                               source_provider, destination_provider,
                               source_provider_inventory, target_namespace,
                               multus_network_name):
        """Create NetworkMap resource for migration."""
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

    def test_create_plan(self, prepared_plan, fixture_store, ocp_admin_client,
                         source_provider, destination_provider, target_namespace,
                         source_provider_inventory):
        """Create MTV Plan CR resource."""
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

    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        """Execute migration."""
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
        )

    def test_check_vms(self, prepared_plan, source_provider, destination_provider,
                       source_provider_data, target_namespace, source_vms_namespace,
                       source_provider_inventory, vm_ssh_connections):
        """Validate migrated VMs."""
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

## Class Declaration

Every test class requires four decorators and three class-level type annotations:

```python
@pytest.mark.tier0                          # Test tier marker
@pytest.mark.incremental                    # Sequential dependency enforcement
@pytest.mark.parametrize(                   # Configuration injection
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_name_here"])],
    indirect=True,
    ids=["descriptive-test-id"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")  # Automatic VM cleanup
class TestNameHere:
    """Test description."""

    storage_map: StorageMap      # Created in step 1
    network_map: NetworkMap      # Created in step 2
    plan_resource: Plan          # Created in step 3
```

### Required Decorators

| Decorator | Purpose |
|-----------|---------|
| `@pytest.mark.incremental` | Marks subsequent tests as `xfail` if a prior test fails |
| `@pytest.mark.parametrize("class_plan_config", ..., indirect=True)` | Injects test configuration into the `class_plan_config` fixture |
| `@pytest.mark.usefixtures("cleanup_migrated_vms")` | Registers teardown-only fixture for VM cleanup |
| `@pytest.mark.tier0` / `warm` / `remote` / `copyoffload` | Categorization marker (at least one required) |

### Optional Decorators

```python
# Conditional skip based on configuration
@pytest.mark.skipif(
    not get_value_from_py_config("remote_ocp_cluster"),
    reason="No remote OCP cluster provided"
)

# Minimum MTV version requirement
@pytest.mark.min_mtv_version("2.10.0")

# Additional setup fixtures (warm migration requires precopy interval)
@pytest.mark.usefixtures("precopy_interval_forkliftcontroller", "cleanup_migrated_vms")
```

## Shared State via `self.__class__`

Resources created in earlier test methods must be accessible to later ones. Since pytest creates a new `self` instance for each test method, direct instance attributes (`self.storage_map = ...`) would not persist across methods.

The solution is to store values on the **class itself** using `self.__class__`:

```python
# Step 1: Store on the class
self.__class__.storage_map = get_storage_migration_map(...)

# Step 3: Read from the class (via normal attribute lookup)
self.__class__.plan_resource = create_plan_resource(
    storage_map=self.storage_map,    # Works because Python checks class attrs
    network_map=self.network_map,    # after checking instance attrs
    ...
)
```

> **Note:** Reading uses `self.storage_map` (Python's attribute resolution checks the class after the instance), but writing must use `self.__class__.storage_map` to store on the class rather than the ephemeral instance.

The class-level type annotations (`storage_map: StorageMap`, `network_map: NetworkMap`, `plan_resource: Plan`) serve as documentation and enable IDE type checking. They do not create default values.

### Additional Shared State

Some test classes share more than the three standard resources. For example, `TestPostHookRetainFailedVm` uses a boolean flag to control whether VM validation runs:

```python
class TestPostHookRetainFailedVm:
    """Test PostHook with VM retention - migration fails but VMs should be retained."""

    storage_map: StorageMap | None = None
    network_map: NetworkMap | None = None
    plan_resource: Plan | None = None
    should_check_vms: bool = False   # Additional shared state

    def test_migrate_vms(self, ...):
        if expected_result == "fail":
            with pytest.raises(MigrationPlanExecError):
                execute_migration(...)
            self.__class__.should_check_vms = validate_hook_failure_and_check_vms(...)
        else:
            execute_migration(...)
            self.__class__.should_check_vms = True

    def test_check_vms(self, ...):
        if not self.__class__.should_check_vms:
            pytest.skip("Skipping VM checks - hook failed before VM migration")
        check_vms(...)
```

## The 5-Step Pattern in Detail

### Step 1: Create StorageMap

Maps source provider storage (datastores, storage domains) to destination OpenShift storage classes.

```python
def test_create_storagemap(
    self,
    prepared_plan,
    fixture_store,
    ocp_admin_client,
    source_provider,
    destination_provider,
    source_provider_inventory,
    target_namespace,
):
    """Create StorageMap resource for migration."""
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

The VM name list is extracted from `prepared_plan["virtual_machines"]`, which contains the cloned VM names (not the original template names). The `get_storage_migration_map()` utility queries the source provider inventory to discover which storage each VM uses and creates the appropriate `StorageMap` Custom Resource.

For copy-offload tests, additional parameters are passed:

```python
# From tests/test_copyoffload_migration.py
offload_plugin_config = {
    "vsphereXcopyConfig": {
        "secretRef": copyoffload_storage_secret.name,
        "storageVendorProduct": storage_vendor_product,
    }
}

self.__class__.storage_map = get_storage_migration_map(
    fixture_store=fixture_store,
    source_provider=source_provider,
    destination_provider=destination_provider,
    source_provider_inventory=source_provider_inventory,
    ocp_admin_client=ocp_admin_client,
    target_namespace=target_namespace,
    vms=vms_names,
    datastore_id=datastore_id,
    storage_class=storage_class,
    offload_plugin_config=offload_plugin_config,
)
```

### Step 2: Create NetworkMap

Maps source provider networks to destination OpenShift networks (pod network or Multus NADs).

```python
def test_create_networkmap(
    self,
    prepared_plan,
    fixture_store,
    ocp_admin_client,
    source_provider,
    destination_provider,
    source_provider_inventory,
    target_namespace,
    multus_network_name,
):
    """Create NetworkMap resource for migration."""
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

The `multus_network_name` fixture is class-scoped and creates Network Attachment Definitions (NADs) with unique names per test class to prevent conflicts during parallel execution.

### Step 3: Create Plan

Creates the MTV `Plan` Custom Resource that ties together the source provider, destination provider, storage map, network map, and VM list.

```python
def test_create_plan(
    self,
    prepared_plan,
    fixture_store,
    ocp_admin_client,
    source_provider,
    destination_provider,
    target_namespace,
    source_provider_inventory,
):
    """Create MTV Plan CR resource."""
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

`populate_vm_ids()` enriches each VM entry with its provider-specific ID (looked up from the Forklift inventory). This call must happen before `create_plan_resource()`.

For comprehensive tests, the plan creation accepts many more parameters:

```python
# From tests/test_cold_migration_comprehensive.py
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

### Step 4: Migrate VMs

Executes the migration by creating a `Migration` Custom Resource and waiting for completion.

```python
def test_migrate_vms(
    self,
    fixture_store,
    ocp_admin_client,
    target_namespace,
):
    """Execute migration."""
    execute_migration(
        ocp_admin_client=ocp_admin_client,
        fixture_store=fixture_store,
        plan=self.plan_resource,
        target_namespace=target_namespace,
    )
```

This is the simplest step — it has no assertion because `execute_migration()` raises an exception if the migration fails or times out.

For warm migrations, a `cut_over` parameter schedules the cutover time:

```python
# From tests/test_mtv_warm_migration.py
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

### Step 5: Validate VMs

Validates that the migrated VMs match their source configuration — disks, networks, firmware, guest agent, and more.

```python
def test_check_vms(
    self,
    prepared_plan,
    source_provider,
    destination_provider,
    source_provider_data,
    target_namespace,
    source_vms_namespace,
    source_provider_inventory,
    vm_ssh_connections,
):
    """Validate migrated VMs."""
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

The `check_vms()` function performs all assertions internally. It compares the migrated VM on OpenShift against the source VM data stored in `prepared_plan["source_vms_data"]`.

For comprehensive tests, additional validation fixtures are passed:

```python
# From tests/test_cold_migration_comprehensive.py
check_vms(
    plan=prepared_plan,
    source_provider=source_provider,
    destination_provider=destination_provider,
    source_provider_data=source_provider_data,
    network_map_resource=self.network_map,
    storage_map_resource=self.storage_map,
    source_vms_namespace=source_vms_namespace,
    source_provider_inventory=source_provider_inventory,
    labeled_worker_node=labeled_worker_node,       # Node selector validation
    target_vm_labels=target_vm_labels,              # Label validation
    vm_ssh_connections=vm_ssh_connections,
)
```

## Class-Level Parametrization with `indirect=True`

The `indirect=True` mechanism is central to how test configuration reaches the test class. Here is the flow:

```
┌──────────────────────────────────┐
│  tests/tests_config/config.py    │
│                                  │
│  tests_params = {                │
│    "test_sanity_cold_...": {     │
│      "virtual_machines": [...],  │
│      "warm_migration": False,    │
│    },                            │
│  }                               │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  @pytest.mark.parametrize(       │
│    "class_plan_config",          │
│    [pytest.param(                │
│      py_config["tests_params"]   │
│        ["test_sanity_cold_..."], │
│    )],                           │
│    indirect=True,                │
│    ids=["rhel8"],                │
│  )                               │
└──────────┬───────────────────────┘
           │ indirect=True routes the value
           │ to the fixture, not the test
           ▼
┌──────────────────────────────────┐
│  @pytest.fixture(scope="class")  │
│  def class_plan_config(request): │
│      return request.param        │
│                                  │
│  # Returns the raw config dict   │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  @pytest.fixture(scope="class")  │
│  def prepared_plan(              │
│      class_plan_config, ...):    │
│                                  │
│  # Deep copies config            │
│  # Clones VMs on source provider │
│  # Updates VM names              │
│  # Stores source VM details      │
│  # Yields enriched plan dict     │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  Test methods receive            │
│  prepared_plan as a parameter    │
└──────────────────────────────────┘
```

### `class_plan_config` Fixture

The simplest fixture in the chain — it receives the parametrized value via `request.param` and returns it unchanged:

```python
@pytest.fixture(scope="class")
def class_plan_config(request: pytest.FixtureRequest) -> dict[str, Any]:
    """Get plan configuration for class-based tests."""
    return request.param
```

### `prepared_plan` Fixture

The workhorse fixture that transforms raw config into a ready-to-use plan:

```python
@pytest.fixture(scope="class")
def prepared_plan(
    request: pytest.FixtureRequest,
    class_plan_config: dict[str, Any],
    fixture_store: dict[str, Any],
    source_provider: Any,
    source_vms_namespace: str,
    ocp_admin_client: DynamicClient,
    multus_cni_config: str,
    source_provider_inventory: ForkliftInventory,
    target_namespace: str,
) -> Generator[dict[str, Any], None, None]:
    """Prepare plan with cloned VMs for class-based tests."""

    # Deep copy the plan config to avoid mutation
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

    # For each VM: clone, set power state, collect source details
    for vm in virtual_machines:
        provider_vm_api = source_provider.get_vm_by_name(
            query=vm["name"],
            clone_vm=True,
            session_uuid=fixture_store["session_uuid"],
            ...
        )

        # Power state control
        source_vm_power = vm.get("source_vm_power")
        if source_vm_power == "on":
            source_provider.start_vm(provider_vm_api)
        elif source_vm_power == "off":
            source_provider.stop_vm(provider_vm_api)

        # Collect source VM details and update name to cloned name
        source_vm_details = source_provider.vm_dict(...)
        vm["name"] = source_vm_details["name"]

        # Store complete source VM data for post-migration validation
        plan["source_vms_data"][vm["name"]] = source_vm_details

    yield plan
```

**Key transformations `prepared_plan` performs:**

1. **Deep copies** `class_plan_config` to prevent mutation across test methods
2. **Initializes** `plan["source_vms_data"]` for storing source VM details separately from the VM list
3. **Creates custom namespace** if `vm_target_namespace` is specified in config
4. **Clones VMs** on the source provider with session-unique names
5. **Controls power state** based on `source_vm_power` config ("on", "off", or unchanged)
6. **Collects source VM details** (disks, networks, firmware) for post-migration comparison
7. **Updates VM names** from template names to actual cloned names
8. **Waits for inventory sync** so Forklift can discover the cloned VMs

### Difference Between `class_plan_config` and `prepared_plan`

| Aspect | `class_plan_config` | `prepared_plan` |
|--------|-------------------|----------------|
| Scope | Raw config from `config.py` | Enriched, ready-to-use plan |
| VM names | Template names (`"mtv-tests-rhel8"`) | Cloned names (`"auto-abc123-mtv-tests-rhel8-cold"`) |
| `source_vms_data` | Not present | Dict of detailed source VM data |
| `_vm_target_namespace` | Not present | Resolved target namespace |
| VMs on provider | Not yet cloned | Cloned and in correct power state |

> **Warning:** Test methods should use `prepared_plan`, not `class_plan_config`. The only fixture that reads `class_plan_config` directly is `prepared_plan`.

## Incremental Test Execution

The `@pytest.mark.incremental` marker is implemented via two pytest hooks in `conftest.py`:

```python
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    setattr(item, "rep_" + rep.when, rep)

    # Track failures for class-based incremental tests
    if "incremental" in item.keywords and rep.when == "call" and rep.failed:
        item.parent._previousfailed = item


def pytest_runtest_setup(item):
    # Xfail if previous test in class failed
    if "incremental" in item.keywords:
        previousfailed = getattr(item.parent, "_previousfailed", None)
        if previousfailed is not None:
            pytest.xfail(f"previous test failed ({previousfailed.name})")
```

**Behavior:**

- When a test method fails during the `"call"` phase, the test item is stored as `item.parent._previousfailed`
- Before each subsequent test method runs, `pytest_runtest_setup` checks for `_previousfailed`
- If a previous failure exists, the current test is marked `xfail` instead of running

This prevents cascading failures. For example, if `test_create_storagemap` fails, `test_create_networkmap` through `test_check_vms` are all reported as `xfail` rather than failing with `AttributeError` when trying to access `self.storage_map`.

```
test_create_storagemap    FAILED     ← Actual failure
test_create_networkmap    XFAIL      ← Skipped: previous test failed (test_create_storagemap)
test_create_plan          XFAIL      ← Skipped: previous test failed (test_create_storagemap)
test_migrate_vms          XFAIL      ← Skipped: previous test failed (test_create_storagemap)
test_check_vms            XFAIL      ← Skipped: previous test failed (test_create_storagemap)
```

## Cleanup: `cleanup_migrated_vms`

This class-scoped, teardown-only fixture deletes migrated VMs after all test methods in the class complete:

```python
@pytest.fixture(scope="class")
def cleanup_migrated_vms(
    request: pytest.FixtureRequest,
    ocp_admin_client: DynamicClient,
    target_namespace: str,
    prepared_plan: dict[str, Any],
) -> Generator[None, None, None]:
    """Cleanup migrated VMs after test class completes."""
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
        else:
            LOGGER.info(f"VM {vm_name} already deleted, skipping cleanup")
```

**Key points:**

- Requested via `@pytest.mark.usefixtures("cleanup_migrated_vms")`, not as a parameter (its return value is `None`)
- The `yield` with no value means it is purely a teardown fixture
- Honors `--skip-teardown` for debugging failed migrations
- Uses `prepared_plan["_vm_target_namespace"]` to find VMs in custom namespaces

> **Tip:** Use `--skip-teardown` when debugging migration failures to keep the migrated VMs alive for manual inspection.

## Test Configuration

Test parameters live in `tests/tests_config/config.py` and are loaded via `pytest-testconfig`:

```python
# Global configuration
insecure_verify_skip: str = "true"
target_namespace_prefix: str = "auto"
mtv_namespace: str = "openshift-mtv"
plan_wait_timeout: int = 3600

# Per-test configuration
tests_params: dict = {
    "test_sanity_cold_mtv_migration": {
        "virtual_machines": [
            {"name": "mtv-tests-rhel8", "guest_agent": True},
        ],
        "warm_migration": False,
    },
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
}
```

### VM Configuration Options

| Option | Required | Type | Description |
|--------|----------|------|-------------|
| `name` | Yes | `str` | VM name (or template name) on the source provider |
| `source_vm_power` | No | `"on"` or `"off"` | Desired power state before migration. If omitted, power state is unchanged |
| `guest_agent` | No | `bool` | Whether the VM has a guest agent installed |
| `clone` | No | `bool` | Whether to clone the VM before migration |
| `disk_type` | No | `str` | Disk provisioning type: `"thin"`, `"thick-lazy"`, `"thick-eager"` |
| `add_disks` | No | `list[dict]` | Additional disks to add before migration |
| `snapshots` | No | `int` | Number of snapshots to create before migration |

### Plan-Level Options

| Option | Required | Type | Description |
|--------|----------|------|-------------|
| `warm_migration` | Yes | `bool` | Whether to use warm (incremental) migration |
| `copyoffload` | No | `bool` | Enable vSphere XCOPY copy-offload |
| `preserve_static_ips` | No | `bool` | Preserve static IP addresses during migration |
| `pvc_name_template` | No | `str` | Template for PVC naming |
| `target_labels` | No | `dict` | Labels to apply to migrated VMs |
| `target_affinity` | No | `dict` | Affinity rules for migrated VMs |
| `vm_target_namespace` | No | `str` | Custom namespace for migrated VMs |
| `target_power_state` | No | `str` | Desired power state after migration |

## Warm Migration Variant

Warm migration tests follow the same 5-step pattern with two differences:

1. **Additional fixture:** `precopy_interval_forkliftcontroller` configures the Forklift controller for incremental copies
2. **Cutover parameter:** `test_migrate_vms` passes a `cut_over` time to `execute_migration()`

```python
@pytest.mark.tier0
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_sanity_warm_mtv_migration"])],
    indirect=True,
    ids=["rhel8"],
)
@pytest.mark.usefixtures("precopy_interval_forkliftcontroller", "cleanup_migrated_vms")
class TestSanityWarmMtvMigration:
    """Warm migration sanity test."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    # Steps 1-3 are identical to cold migration...

    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        """Execute warm migration with cutover."""
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
            cut_over=get_cutover_value(),  # Schedules cutover
        )

    # Step 5 is identical to cold migration
```

> **Note:** Warm migration is not supported for all source providers. The `test_mtv_warm_migration.py` file uses a module-level `pytestmark` to skip for unsupported providers (OpenStack, OpenShift, OVA).

## Remote Cluster Variant

Remote OCP tests migrate VMs to a separate OpenShift cluster. The only difference is using `destination_ocp_provider` instead of `destination_provider`:

```python
@pytest.mark.remote
@pytest.mark.incremental
@pytest.mark.skipif(
    not get_value_from_py_config("remote_ocp_cluster"),
    reason="No remote OCP cluster provided"
)
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_cold_remote_ocp"])],
    indirect=True,
    ids=["MTV-79"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestColdRemoteOcp:
    """Cold migration test to remote OCP cluster."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    def test_create_storagemap(self, ..., destination_ocp_provider, ...):
        self.__class__.storage_map = get_storage_migration_map(
            destination_provider=destination_ocp_provider,  # Remote cluster
            ...
        )
```

## Fixture Dependencies per Step

Each test method requests only the fixtures it needs:

| Step | Key Fixtures |
|------|-------------|
| `test_create_storagemap` | `prepared_plan`, `fixture_store`, `ocp_admin_client`, `source_provider`, `destination_provider`, `source_provider_inventory`, `target_namespace` |
| `test_create_networkmap` | Same as storagemap + `multus_network_name` |
| `test_create_plan` | Same as storagemap + `source_provider_inventory` (uses `self.storage_map`, `self.network_map` from class) |
| `test_migrate_vms` | `fixture_store`, `ocp_admin_client`, `target_namespace` (uses `self.plan_resource` from class) |
| `test_check_vms` | `prepared_plan`, `source_provider`, `destination_provider`, `source_provider_data`, `source_vms_namespace`, `source_provider_inventory`, `vm_ssh_connections` (uses `self.network_map`, `self.storage_map` from class) |

### Fixture Scopes

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `ocp_admin_client` | Session | OpenShift cluster connection |
| `fixture_store` | Session | Resource tracking for teardown |
| `target_namespace` | Session | Migration target namespace |
| `source_provider` | Session | Source infrastructure provider |
| `destination_provider` | Session | Destination OpenShift provider |
| `source_provider_inventory` | Session | Forklift inventory for source |
| `class_plan_config` | Class | Raw parametrized test config |
| `prepared_plan` | Class | Enriched plan with cloned VMs |
| `multus_network_name` | Class | Per-class unique NAD names |
| `cleanup_migrated_vms` | Class | VM teardown after class completes |

## Resource Tracking with `create_and_store_resource()`

All OpenShift resources created during tests are tracked via `create_and_store_resource()` in `utilities/resources.py`:

```python
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    kwargs["client"] = client

    _resource_name = kwargs.get("name")
    # ... name resolution from yaml_file or kind_dict ...

    if not _resource_name:
        _resource_name = generate_name_with_uuid(name=fixture_store["base_resource_name"])

        if resource.kind in (Migration.kind, Plan.kind):
            _resource_name = f"{_resource_name}-{'warm' if kwargs.get('warm_migration') else 'cold'}"

    if len(_resource_name) > 63:
        _resource_name = _resource_name[-63:]

    kwargs["name"] = _resource_name
    _resource = resource(**kwargs)

    try:
        _resource.deploy(wait=True)
    except ConflictError:
        _resource.wait()

    fixture_store["teardown"].setdefault(_resource.kind, []).append({
        "name": _resource.name,
        "namespace": _resource.namespace,
        "module": _resource.__module__,
    })

    return _resource
```

**Features:**

- **Auto-naming:** Generates unique names using session UUID if no name is provided
- **Type suffixing:** Appends `-warm` or `-cold` to Plan and Migration resources
- **Kubernetes compliance:** Truncates names to 63 characters
- **Conflict handling:** Reuses existing resources on `ConflictError`
- **Teardown tracking:** Registers every resource in `fixture_store["teardown"]` for automatic cleanup

> **Warning:** Never create OpenShift resources directly with `resource.deploy()`. Always use `create_and_store_resource()` to ensure proper cleanup.

## Parallel Execution

Tests use `pytest-xdist` with `--dist=loadscope` (configured in `pytest.ini`), which groups tests by module or class and distributes groups across workers. Each worker gets its own session, so:

- **Session UUID** is unique per worker, preventing resource name collisions
- **`fixture_store`** is isolated per worker
- **Multus NAD names** use a hash of the test class node ID for uniqueness

```ini
# pytest.ini
[pytest]
addopts =
  --dist=loadscope
```

This means all 5 methods of a single test class always run on the same worker (required by the incremental pattern and shared class state), but different test classes can run on different workers in parallel.

## Adding a New Test Class

1. **Add configuration** in `tests/tests_config/config.py`:

```python
tests_params: dict = {
    # ...existing tests...
    "test_my_new_feature": {
        "virtual_machines": [
            {
                "name": "mtv-tests-rhel8",
                "source_vm_power": "on",
                "guest_agent": True,
            },
        ],
        "warm_migration": False,
    },
}
```

2. **Create test file** `tests/test_my_new_feature.py`:

```python
import pytest
from ocp_resources.network_map import NetworkMap
from ocp_resources.plan import Plan
from ocp_resources.storage_map import StorageMap
from pytest_testconfig import config as py_config

from utilities.mtv_migration import (
    create_plan_resource,
    execute_migration,
    get_network_migration_map,
    get_storage_migration_map,
)
from utilities.post_migration import check_vms
from utilities.utils import populate_vm_ids


@pytest.mark.tier0
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_my_new_feature"])],
    indirect=True,
    ids=["my-new-feature"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestMyNewFeature:
    """Description of what this test validates."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    def test_create_storagemap(self, prepared_plan, fixture_store, ocp_admin_client,
                               source_provider, destination_provider,
                               source_provider_inventory, target_namespace):
        """Create StorageMap resource for migration."""
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

    def test_create_networkmap(self, prepared_plan, fixture_store, ocp_admin_client,
                               source_provider, destination_provider,
                               source_provider_inventory, target_namespace,
                               multus_network_name):
        """Create NetworkMap resource for migration."""
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

    def test_create_plan(self, prepared_plan, fixture_store, ocp_admin_client,
                         source_provider, destination_provider, target_namespace,
                         source_provider_inventory):
        """Create MTV Plan CR resource."""
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

    def test_migrate_vms(self, fixture_store, ocp_admin_client, target_namespace):
        """Execute migration."""
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
        )

    def test_check_vms(self, prepared_plan, source_provider, destination_provider,
                       source_provider_data, target_namespace, source_vms_namespace,
                       source_provider_inventory, vm_ssh_connections):
        """Validate migrated VMs."""
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

> **Tip:** Copy an existing test class (e.g., `TestSanityColdMtvMigration`) as a starting point rather than writing from scratch. The 5-step structure is intentionally consistent across all test files.
