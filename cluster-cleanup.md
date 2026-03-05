# Cluster Cleanup

This page covers the mechanisms for cleaning up OpenShift cluster resources after test execution, including automated session teardown, manual cleanup via `tools/clean_cluster.py`, parallel pod and storage resource deletion, and diagnostic data collection.

## Overview

The mtv-api-tests framework provides a multi-layered cleanup system that ensures all resources created during test runs are properly removed from the cluster. Cleanup operates at three levels:

1. **Class-level** — the `cleanup_migrated_vms` fixture removes migrated VMs after each test class
2. **Session-level** — the `session_teardown()` function orchestrates full resource deletion when the pytest session ends
3. **Manual** — the `tools/clean_cluster.py` script deletes resources from a previously saved `resources.json` file

Every resource created through `create_and_store_resource()` is automatically registered in the `fixture_store["teardown"]` dictionary, which serves as the source of truth for what needs to be cleaned up.

## Automated Teardown via `session_teardown()`

### How It Triggers

When the pytest session finishes, the `pytest_sessionfinish` hook in `conftest.py` orchestrates cleanup:

```python
def pytest_sessionfinish(session, exitstatus):
    _session_store = get_fixture_store(session)
    _data_collector_path = Path(session.config.getoption("data_collector_path"))

    # 1. Export created resources to resources.json
    if not session.config.getoption("skip_data_collector"):
        collect_created_resources(session_store=_session_store, data_collector_path=_data_collector_path)

    # 2. Run session teardown (unless --skip-teardown)
    if session.config.getoption("skip_teardown"):
        LOGGER.warning("User requested to skip teardown of resources")
    else:
        try:
            session_teardown(session_store=_session_store)
        except Exception as exp:
            LOGGER.error(f"the following resources was left after tests are finished: {exp}")
            if not session.config.getoption("skip_data_collector"):
                run_must_gather(data_collector_path=_data_collector_path)

    # 3. Cleanup temp directory
    shutil.rmtree(path=session.config.option.basetemp, ignore_errors=True)
```

The data collector always runs first (unless `--skip-data-collector` is set), so `resources.json` is available for manual cleanup even if automated teardown fails.

### The `session_teardown()` Function

Defined in `utilities/pytest_utils.py`, this function is the main entry point for session-level cleanup:

```python
def session_teardown(session_store: dict[str, Any]) -> None:
    LOGGER.info("Running teardown to delete all created resources")
    ocp_client = get_cluster_client()

    # When running in parallel (-n auto) `session_store` can be empty.
    if session_teardown_resources := session_store.get("teardown"):
        # Phase 1: Cancel active migrations
        for migration_name in session_teardown_resources.get(Migration.kind, []):
            migration = Migration(name=migration_name["name"], namespace=migration_name["namespace"], client=ocp_client)
            cancel_migration(migration=migration)

        # Phase 2: Archive plans
        for plan_name in session_teardown_resources.get(Plan.kind, []):
            plan = Plan(name=plan_name["name"], namespace=plan_name["namespace"], client=ocp_client)
            archive_plan(plan=plan)

        # Phase 3: Delete all tracked resources
        leftovers = teardown_resources(
            session_store=session_store,
            ocp_client=ocp_client,
            target_namespace=session_store.get("target_namespace"),
        )
        if leftovers:
            raise SessionTeardownError(f"Failed to clean up the following resources: {leftovers}")
```

> **Note:** When running with `pytest-xdist` (`-n auto`), each worker has its own isolated `fixture_store`. The walrus operator (`if ... :=`) safely handles the case where the store is empty.

### Resource Deletion Order

The `teardown_resources()` function deletes resources in a specific order to respect Kubernetes dependency chains:

| Phase | Resources | Method |
|-------|-----------|--------|
| 1 | Migrations | Sequential — `clean_up(wait=True)` |
| 2 | Plans | Sequential — `clean_up(wait=True)` |
| 3 | Providers | Sequential — `clean_up(wait=True)` |
| 4 | Hosts | Sequential — `clean_up(wait=True)` |
| 5 | Secrets | Sequential — `clean_up(wait=True)` |
| 6 | NetworkAttachmentDefinitions | Sequential — `clean_up(wait=True)` |
| 7 | StorageMaps | Sequential — `clean_up(wait=True)` |
| 8 | NetworkMaps | Sequential — `clean_up(wait=True)` |
| 9 | VirtualMachines | Sequential — existence check + `clean_up(wait=True)` |
| 10 | Pods (tracked) | Sequential — existence check + `clean_up(wait=True)` |
| 11 | Target namespace VMs | `delete_all_vms()` — sweep of all VMs in namespace |
| 12 | Target namespace Pods | **Parallel** — `ThreadPoolExecutor` (max 10 workers) |
| 13 | DataVolumes, PVCs, PVs | **Parallel** — `check_dv_pvc_pv_deleted()` |
| 14 | Namespaces | Sequential — `clean_up(wait=True)` |
| 15 | Source provider cloned VMs | Sequential per provider (VMware, OpenStack, RHV) |
| 16 | OpenStack volume snapshots | Sequential via OpenStack API |

> **Warning:** Namespaces are deleted last intentionally. Deleting a namespace before its contained resources can lead to finalizer deadlocks. The framework first removes all namespace-scoped resources, then deletes the namespace itself.

### Migration Cancellation

Before deleting resources, active migrations must be cancelled. The `cancel_migration()` function in `utilities/migration_utils.py` patches the Migration CR:

```python
def cancel_migration(migration: Migration) -> None:
    for condition in migration.instance.status.conditions:
        if condition.type == migration.Condition.Type.RUNNING and condition.status == migration.Condition.Status.TRUE:
            LOGGER.info(f"Canceling migration {migration.name}")

            migration_spec = migration.instance.spec
            plan = Plan(client=migration.client, name=migration_spec.plan.name, namespace=migration_spec.plan.namespace)

            ResourceEditor(
                patches={
                    migration: {
                        "spec": {
                            "cancel": plan.instance.spec.vms,
                        }
                    }
                }
            ).update()

            try:
                migration.wait_for_condition(
                    condition=migration.Condition.CANCELED, status=migration.Condition.Status.TRUE
                )
                check_dv_pvc_pv_deleted(
                    ocp_client=migration.client,
                    target_namespace=plan.instance.spec.targetNamespace,
                    partial_name=migration.name,
                )
            except TimeoutExpiredError:
                LOGGER.error(f"Failed to cancel migration {migration.name}")
```

After cancellation, `check_dv_pvc_pv_deleted()` runs immediately to verify that the associated storage resources are cleaned up.

### Plan Archiving

Plans must be archived before deletion. The `archive_plan()` function patches the Plan CR and waits for associated pods to terminate:

```python
def archive_plan(plan: Plan) -> None:
    LOGGER.info(f"Archiving plan {plan.name}")
    ResourceEditor(
        patches={
            plan: {
                "spec": {
                    "archived": True,
                }
            }
        }
    ).update()

    try:
        plan.wait_for_condition(condition=plan.Condition.ARCHIVED, status=plan.Condition.Status.TRUE)
        for _pod in Pod.get(client=plan.client, namespace=plan.instance.spec.targetNamespace):
            if plan.name in _pod.name:
                if not _pod.wait_deleted():
                    LOGGER.error(f"Pod {_pod.name} was not deleted after plan {plan.name} was archived")
    except TimeoutExpiredError:
        LOGGER.error(f"Failed to archive plan {plan.name}")
```

## Parallel Pod Cleanup

After deleting individually tracked resources, the framework performs a sweep of the target namespace to catch any remaining pods created by migrations. This uses `ThreadPoolExecutor` with up to 10 concurrent workers:

```python
if target_namespace:
    # Make sure all pods related to the test session are deleted (in parallel)
    pods_to_wait = [
        _pod for _pod in Pod.get(client=ocp_client, namespace=target_namespace)
        if session_uuid in _pod.name
    ]

    if pods_to_wait:
        LOGGER.info(f"Waiting for {len(pods_to_wait)} pods to be deleted in parallel...")

        def wait_for_pod_deletion(pod):
            """Helper function to wait for a single pod deletion."""
            try:
                if not pod.wait_deleted():
                    return {"success": False, "pod": pod, "error": None}
                return {"success": True, "pod": pod, "error": None}
            except Exception as exc:
                LOGGER.error(f"Failed to wait for pod {pod.name} deletion: {exc}")
                return {"success": False, "pod": pod, "error": exc}

        with ThreadPoolExecutor(max_workers=min(len(pods_to_wait), 10)) as executor:
            future_to_pod = {executor.submit(wait_for_pod_deletion, pod): pod for pod in pods_to_wait}
            for future in as_completed(future_to_pod):
                result = future.result()
                if not result["success"]:
                    leftovers = append_leftovers(leftovers=leftovers, resource=result["pod"])
```

Pods are filtered by `session_uuid` to ensure only pods from the current test session are targeted, making this safe for parallel test execution with `pytest-xdist`.

## DV/PVC/PV Verification

The `check_dv_pvc_pv_deleted()` function in `utilities/migration_utils.py` verifies that all storage resources associated with a migration are properly cleaned up. It processes three resource types in strict sequential order (DVs, then PVCs, then PVs), but uses parallel deletion within each group.

```python
def check_dv_pvc_pv_deleted(
    ocp_client: DynamicClient,
    target_namespace: str,
    partial_name: str,
    leftovers: dict[str, list[dict[str, str]]] | None = None,
) -> dict[str, list[dict[str, str]]]:
    """
    Check and wait for DataVolumes, PVCs, and PVs to be deleted in parallel.
    Order is maintained: DVs → PVCs → PVs, but within each group resources are checked in parallel.
    """
    if leftovers is None:
        leftovers = {}

    def wait_for_resource_deletion(resource, resource_type):
        """Helper function to wait for a single resource deletion."""
        try:
            with contextlib.suppress(NotFoundError):
                if not resource.wait_deleted(timeout=60 * 2):  # 2-minute timeout
                    return {"success": False, "resource": resource, "type": resource_type}
            return {"success": True, "resource": resource, "type": resource_type}
        except Exception as exc:
            LOGGER.error(f"Failed to wait for {resource_type} {resource.name} deletion: {exc}")
            return {"success": False, "resource": resource, "type": resource_type}
```

### DataVolume Cleanup

```python
dvs_to_wait = [
    _dv for _dv in DataVolume.get(client=ocp_client, namespace=target_namespace)
    if partial_name in _dv.name
]
if dvs_to_wait:
    LOGGER.info(f"Waiting for {len(dvs_to_wait)} DataVolumes to be deleted in parallel...")
    with ThreadPoolExecutor(max_workers=min(len(dvs_to_wait), 10)) as executor:
        future_to_dv = {executor.submit(wait_for_resource_deletion, dv, "DataVolume"): dv for dv in dvs_to_wait}
        for future in as_completed(future_to_dv):
            result = future.result()
            if not result["success"]:
                leftovers = append_leftovers(leftovers=leftovers, resource=result["resource"])
```

### PVC Cleanup

PVCs are filtered identically — by `partial_name` within the `target_namespace` — and waited on in parallel with the same thread pool pattern (max 10 workers, 2-minute timeout per resource).

### PV Cleanup

PVs are cluster-scoped, so they require special filtering. The function checks each PV's `claimRef` to find those associated with the test session, and excludes PVs that are already in `RELEASED` status:

```python
pvs_to_wait = []
for _pv in PersistentVolume.get(client=ocp_client):
    with contextlib.suppress(NotFoundError):
        _pv_spec = _pv.instance.spec.to_dict()
        if partial_name in _pv_spec.get("claimRef", {}).get("name", ""):
            if _pv.instance.status.phase != _pv.Status.RELEASED:
                pvs_to_wait.append(_pv)
```

> **Tip:** The `partial_name` parameter is typically the `session_uuid`, which is embedded in every resource name created by `create_and_store_resource()`. This makes it a reliable identifier for filtering resources belonging to a specific test session.

### When DV/PVC/PV Verification Runs

This function is called in two contexts:

1. **After migration cancellation** — `cancel_migration()` calls it immediately to verify the cancelled migration's storage resources are cleaned up
2. **During session teardown** — `teardown_resources()` calls it as part of the namespace cleanup phase to catch any remaining storage resources

## Class-Level VM Cleanup

The `cleanup_migrated_vms` fixture provides early cleanup of migrated VMs at the class scope, before the session-level teardown runs:

```python
@pytest.fixture(scope="class")
def cleanup_migrated_vms(
    request: pytest.FixtureRequest,
    ocp_admin_client: DynamicClient,
    target_namespace: str,
    prepared_plan: dict[str, Any],
) -> Generator[None, None, None]:
    yield  # Tests run here

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
            LOGGER.info(f"VM {vm_name} already deleted from namespace: {vm_namespace}, skipping cleanup")
```

This fixture is applied to test classes via `@pytest.mark.usefixtures("cleanup_migrated_vms")` and runs as a teardown-only fixture (it yields immediately, then performs cleanup after all tests in the class complete). Session-level teardown acts as a safety net for any VMs this fixture misses.

## Data Collector and `resources.json`

### Resource Collection

Before teardown, the framework serializes all tracked resources to a JSON file for later use:

```python
def collect_created_resources(session_store: dict[str, Any], data_collector_path: Path) -> None:
    resources = session_store["teardown"]
    if resources:
        try:
            LOGGER.info(f"Write created resources data to {data_collector_path}/resources.json")
            with open(data_collector_path / "resources.json", "w") as fd:
                json.dump(session_store["teardown"], fd)
        except Exception as ex:
            LOGGER.error(f"Failed to store resources.json due to: {ex}")
```

The output file at `.data-collector/resources.json` (default path) has this structure:

```json
{
  "Namespace": [
    {"name": "auto-abc123-test", "namespace": null, "module": "ocp_resources.namespace"}
  ],
  "Plan": [
    {"name": "auto-abc123-vsphere-cold", "namespace": "auto-abc123-test", "module": "ocp_resources.plan"}
  ],
  "Migration": [
    {"name": "auto-abc123-vsphere-cold", "namespace": "auto-abc123-test", "module": "ocp_resources.migration"}
  ],
  "Provider": [
    {"name": "auto-abc123-vsphere", "namespace": "auto-abc123-test", "module": "ocp_resources.provider"}
  ]
}
```

Each resource entry includes the `module` path, which allows `clean_cluster.py` to dynamically import the correct resource class.

### Must-Gather Integration

Diagnostic data is collected automatically via `oc adm must-gather` in two scenarios:

1. **On test failure** — the `pytest_exception_interact` hook triggers a targeted must-gather scoped to the failing test's plan:

```python
def pytest_exception_interact(node, call, report):
    if not node.session.config.getoption("skip_data_collector"):
        _session_store = get_fixture_store(node.session)
        _data_collector_path = Path(f"{node.session.config.getoption('data_collector_path')}/{node.name}")
        test_name = node._pyfuncitem.name if hasattr(node, "_pyfuncitem") else node.name
        plans = _session_store["teardown"].get("Plan", [])
        plan = [plan for plan in plans if plan.get("test_name", "") == test_name]
        plan = plan[0] if plan else None
        run_must_gather(data_collector_path=_data_collector_path, plan=plan)
```

2. **On session teardown failure** — when `session_teardown()` raises `SessionTeardownError`, a full must-gather runs against the MTV namespace.

## Using `tools/clean_cluster.py`

The `clean_cluster.py` script provides manual cleanup using the `resources.json` file produced by the data collector. This is useful when:

- Automated teardown was skipped (`--skip-teardown`)
- Session teardown failed and left resources behind
- A test run was interrupted (e.g., killed, lost SSH connection)

### Usage

```bash
python tools/clean_cluster.py .data-collector/resources.json
```

### How It Works

The script reads the JSON file, dynamically imports each resource class using `importlib`, instantiates the resource object, and calls `.clean_up()`:

```python
def clean_cluster_by_resources_file(resources_file: str) -> None:
    with open(resources_file, "r") as fd:
        data: dict[str, list[dict[str, str]]] = json.load(fd)

    for _resource_kind, _resources_list in data.items():
        for _resource in _resources_list:
            _resource_module = importlib.import_module(_resource["module"])
            _resource_class = getattr(_resource_module, _resource_kind)
            _kwargs = {"name": _resource["name"]}
            if _resource.get("namespace"):
                _kwargs["namespace"] = _resource["namespace"]

            _resource_class(**_kwargs).clean_up()
```

> **Warning:** The script does not follow the dependency-aware deletion order that `session_teardown()` uses. It processes resources in the order they appear in the JSON file. If cleanup fails due to dependency issues (e.g., trying to delete a namespace with existing resources), you may need to run the script multiple times or manually delete blocking resources first.

> **Note:** The script requires a valid `KUBECONFIG` environment variable or `~/.kube/config` file to authenticate with the cluster, since it uses `openshift-python-wrapper` under the hood.

## Source Provider Cleanup

The teardown process also cleans up cloned VMs on source providers. These are VMs that were cloned in the source environment (VMware, OpenStack, or RHV) before migration, and must be deleted from the source provider after tests complete.

Source provider cloned VMs are tracked in `fixture_store["teardown"]` under provider-type keys (`vsphere`, `openstack`, `rhv`). The framework connects to each provider using stored credentials and deletes the VMs via provider-specific APIs:

```python
# VMware cleanup
if vmware_cloned_vms:
    with VMWareProvider(
        host=source_provider_data["fqdn"],
        username=source_provider_data["username"],
        password=source_provider_data["password"],
    ) as vmware_provider:
        for _vm in vmware_cloned_vms:
            vmware_provider.delete_vm(vm_name=_vm["name"])

# OpenStack volume snapshots
if openstack_volume_snapshots:
    with OpenStackProvider(...) as openstack_provider:
        for snapshot in openstack_volume_snapshots:
            openstack_provider.api.block_storage.delete_snapshot(snapshot["id"], ignore_missing=True)
```

## Leftovers Tracking

Throughout the teardown process, resources that fail to delete are tracked in a `leftovers` dictionary using the `append_leftovers()` helper:

```python
def append_leftovers(
    leftovers: dict[str, list[dict[str, str]]], resource: Resource | NamespacedResource
) -> dict[str, list[dict[str, str]]]:
    leftovers.setdefault(resource.kind, []).append({
        "name": resource.name,
        "namespace": resource.namespace,
    })
    return leftovers
```

If any leftovers remain after all cleanup phases complete, `session_teardown()` raises a `SessionTeardownError`. This error is caught by `pytest_sessionfinish`, which triggers a must-gather to collect diagnostic data about the stuck resources.

## Command-Line Options

| Option | Default | Description |
|--------|---------|-------------|
| `--skip-teardown` | `False` | Skip all resource cleanup after tests complete. Resources remain on the cluster for inspection. |
| `--skip-data-collector` | `False` | Skip writing `resources.json` and collecting must-gather data on failures. |
| `--data-collector-path` | `.data-collector` | Directory path for `resources.json` output and must-gather artifacts. |

### Examples

```bash
# Normal run — full automated cleanup
uv run pytest tests/

# Keep resources for debugging — skip teardown but save resources.json
uv run pytest tests/ --skip-teardown

# Later, clean up manually using the saved resources.json
python tools/clean_cluster.py .data-collector/resources.json

# Custom data collector output path
uv run pytest tests/ --data-collector-path /tmp/test-artifacts

# Maximum speed — skip data collection entirely
uv run pytest tests/ --skip-data-collector
```

## Parallel Execution Safety

When running tests with `pytest-xdist` (`-n auto`), cleanup is inherently safe because:

- Each worker gets its own isolated `fixture_store` with a unique `session_uuid`
- All resource names embed the `session_uuid`, ensuring no cross-worker conflicts
- Resource filtering during cleanup uses the `session_uuid` as a `partial_name` match
- Worker results are serialized via pickle files and merged at session end

## Complete Cleanup Sequence

```
pytest_sessionfinish()
├── collect_created_resources() → .data-collector/resources.json
└── session_teardown()
    ├── cancel_migration() for each active Migration
    │   └── check_dv_pvc_pv_deleted() (parallel)
    ├── archive_plan() for each Plan
    │   └── wait for plan-related pods to terminate
    └── teardown_resources()
        ├── Delete Migrations          (sequential)
        ├── Delete Plans               (sequential)
        ├── Delete Providers           (sequential)
        ├── Delete Hosts               (sequential)
        ├── Delete Secrets             (sequential)
        ├── Delete NADs                (sequential)
        ├── Delete StorageMaps         (sequential)
        ├── Delete NetworkMaps         (sequential)
        ├── Delete VirtualMachines     (sequential)
        ├── Delete Pods (tracked)      (sequential)
        ├── delete_all_vms()           (sweep target namespace)
        ├── Pod wait                   (parallel, max 10 workers)
        ├── check_dv_pvc_pv_deleted()
        │   ├── DataVolumes            (parallel, max 10 workers, 2-min timeout)
        │   ├── PVCs                   (parallel, max 10 workers, 2-min timeout)
        │   └── PVs                    (parallel, max 10 workers, 2-min timeout)
        ├── Delete Namespaces          (sequential)
        ├── Delete VMware cloned VMs   (via provider API)
        ├── Delete OpenStack cloned VMs (via provider API)
        ├── Delete RHV cloned VMs      (via provider API)
        └── Delete OpenStack snapshots (via provider API)

On failure → SessionTeardownError → run_must_gather()
```
