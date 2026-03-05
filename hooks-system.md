# Pre/Post Migration Hooks

Migration hooks allow you to execute custom Ansible playbooks at specific points during the VM migration lifecycle. The MTV (Migration Toolkit for Virtualization) framework supports two hook types:

- **PreHook** — Runs before VM migration begins (e.g., shutting down services, taking snapshots)
- **PostHook** — Runs after VM migration completes (e.g., reconfiguring networks, validating services)

Hooks are implemented as Forklift `Hook` Custom Resources (CRs) that reference base64-encoded Ansible playbooks. The test framework provides utilities for creating hooks, validating hook failures, and determining whether migrated VMs should be retained after a hook failure.

## Hook CR Creation

Hook CRs are created through the `utilities/hooks.py` module using the `create_and_store_resource()` function, which ensures proper resource tracking and teardown.

### The `create_hook_for_plan` Function

This is the core function for creating a Hook CR:

```python
from ocp_resources.hook import Hook
from utilities.resources import create_and_store_resource

def create_hook_for_plan(
    hook_config: dict[str, Any],
    hook_type: str,
    fixture_store: dict[str, Any],
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
) -> tuple[str, str]:
    """Create a Hook CR based on plan configuration.

    Returns:
        tuple[str, str]: Tuple of (hook_name, hook_namespace)
    """
    validate_hook_config(hook_config, hook_type)

    expected_result = hook_config.get("expected_result")
    custom_playbook = hook_config.get("playbook_base64")

    if custom_playbook:
        validate_custom_playbook(custom_playbook, hook_type)
        playbook = custom_playbook
    else:
        playbook = HOOK_PLAYBOOK_FAIL if expected_result == "fail" else HOOK_PLAYBOOK_SUCCESS

    hook = create_and_store_resource(
        client=ocp_admin_client,
        fixture_store=fixture_store,
        resource=Hook,
        namespace=target_namespace,
        playbook=playbook,
    )

    return hook.name, hook.namespace
```

### Automatic Hook Creation in Fixtures

Hooks are created automatically during the `prepared_plan` fixture in `conftest.py`. If the test configuration includes `pre_hook` or `post_hook` keys, the corresponding Hook CRs are created and their references stored in the plan dict:

```python
# conftest.py — inside the prepared_plan fixture
create_hook_if_configured(plan, "pre_hook", "pre", fixture_store, ocp_admin_client, target_namespace)
create_hook_if_configured(plan, "post_hook", "post", fixture_store, ocp_admin_client, target_namespace)
```

The `create_hook_if_configured` function checks whether a hook key exists in the plan config and, if so, creates the Hook CR and stores the name/namespace references for later use:

```python
def create_hook_if_configured(
    plan: dict[str, Any],
    hook_key: str,        # "pre_hook" or "post_hook"
    hook_type: str,       # "pre" or "post"
    fixture_store: dict[str, Any],
    ocp_admin_client: "DynamicClient",
    target_namespace: str,
) -> None:
    hook_config = plan.get(hook_key)
    if hook_config:
        hook_name, hook_namespace = create_hook_for_plan(
            hook_config=hook_config,
            hook_type=hook_type,
            fixture_store=fixture_store,
            ocp_admin_client=ocp_admin_client,
            target_namespace=target_namespace,
        )
        plan[f"_{hook_type}_hook_name"] = hook_name
        plan[f"_{hook_type}_hook_namespace"] = hook_namespace
```

After creation, the hook references are stored as internal keys (`_pre_hook_name`, `_pre_hook_namespace`, `_post_hook_name`, `_post_hook_namespace`) and later passed to `create_plan_resource()`:

```python
self.__class__.plan_resource = create_plan_resource(
    # ...other args...
    pre_hook_name=prepared_plan["_pre_hook_name"],
    pre_hook_namespace=prepared_plan["_pre_hook_namespace"],
    after_hook_name=prepared_plan["_post_hook_name"],
    after_hook_namespace=prepared_plan["_post_hook_namespace"],
)
```

> **Note:** The Plan CR uses `after_hook_name` / `after_hook_namespace` for the post-migration hook, while the test configuration uses `post_hook`. The `prepared_plan` fixture bridges this naming difference.

## Base64-Encoded Ansible Playbooks

Hook CRs require Ansible playbooks encoded in base64. The framework provides two predefined playbooks and supports custom playbooks.

### Predefined Playbooks

Two built-in playbooks are defined in `utilities/hooks.py`:

**Success playbook** (`HOOK_PLAYBOOK_SUCCESS`) — A debug task that logs a success message:

```yaml
- name: Successful-hook
  hosts: localhost
  tasks:
    - name: Success task
      debug:
        msg: "Hook executed successfully"
```

**Failure playbook** (`HOOK_PLAYBOOK_FAIL`) — A task that intentionally fails:

```yaml
- name: Failing-post-migration
  hosts: localhost
  tasks:
    - name: Task that will fail
      fail:
        msg: "This hook is designed to fail for testing purposes"
```

These are stored as base64-encoded constants:

```python
HOOK_PLAYBOOK_SUCCESS: str = (
    "LSBuYW1lOiBTdWNjZXNzZnVsLWhvb2sKICBob3N0czogbG9jYWxob3N0CiAgdGFza3M6CiAgICAt"
    "IG5hbWU6IFN1Y2Nlc3MgdGFzawogICAgICBkZWJ1ZzoKICAgICAgICBtc2c6ICJIb29rIGV4ZWN1"
    "dGVkIHN1Y2Nlc3NmdWxseSI="
)

HOOK_PLAYBOOK_FAIL: str = (
    "LSBuYW1lOiBGYWlsaW5nLXBvc3QtbWlncmF0aW9uCiAgaG9zdHM6IGxvY2FsaG9zdAogIHRhc2tz"
    "OgogICAgLSBuYW1lOiBUYXNrIHRoYXQgd2lsbCBmYWlsCiAgICAgIGZhaWw6CiAgICAgICAgbXNn"
    "OiAiVGhpcyBob29rIGlzIGRlc2lnbmVkIHRvIGZhaWwgZm9yIHRlc3RpbmcgcHVycG9zZXMi"
)
```

### Custom Playbooks

Custom Ansible playbooks can be provided via the `playbook_base64` configuration key. Before use, custom playbooks undergo multi-layer validation through `validate_custom_playbook()`:

1. **Base64 decoding** — Validates the string is proper base64 encoding
2. **UTF-8 decoding** — Ensures decoded bytes are valid UTF-8 text
3. **YAML syntax** — Parses the text as valid YAML
4. **Ansible structure** — Confirms the result is a non-empty list (Ansible playbook format)

```python
def validate_custom_playbook(playbook_base64: str, hook_type: str) -> None:
    # Step 1: Validate base64
    decoded = base64.b64decode(playbook_base64, validate=True)

    # Step 2: Validate UTF-8
    playbook_text = decoded.decode("utf-8")

    # Step 3: Validate YAML
    playbook_data = yaml.safe_load(playbook_text)

    # Step 4: Validate Ansible playbook structure
    if not isinstance(playbook_data, list) or not playbook_data:
        raise ValueError(
            f"Invalid {hook_type} hook playbook_base64: "
            f"Ansible playbook must be a non-empty list of plays"
        )
```

> **Warning:** Custom playbooks and predefined playbooks are mutually exclusive. Specifying both `expected_result` and `playbook_base64` in the same hook configuration raises a `ValueError`.

## Hook Configuration

Hooks are configured in the test parameters within `tests/tests_config/config.py`. Each hook supports two mutually exclusive modes:

### Predefined Mode

Use `expected_result` with either `"succeed"` or `"fail"` to select a built-in playbook:

```python
"test_post_hook_retain_failed_vm": {
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8",
            "source_vm_power": "on",
            "guest_agent": True,
        },
    ],
    "warm_migration": False,
    "target_power_state": "off",
    "pre_hook": {"expected_result": "succeed"},
    "post_hook": {"expected_result": "fail"},
    "expected_migration_result": "fail",
}
```

### Custom Playbook Mode

Use `playbook_base64` to supply a custom Ansible playbook:

```python
"pre_hook": {
    "playbook_base64": "<base64-encoded Ansible playbook>"
}
```

### Configuration Validation

The `validate_hook_config()` function enforces these rules:

| Rule | Description |
|------|-------------|
| Type check | `hook_config` must be a `dict` |
| Mutual exclusivity | Cannot specify both `expected_result` and `playbook_base64` |
| Required field | Must specify at least one of `expected_result` or `playbook_base64` |
| No empty strings | Neither field can be empty or whitespace-only |

## Hook Failure Validation

When a migration is expected to fail due to a hook, the framework provides a pipeline for validating exactly where and why the failure occurred.

### Migration Pipeline Steps

The MTV migration pipeline executes steps in order. Each VM's pipeline status is tracked in the Migration CR:

```
PreHook → DiskTransfer → ... → PostHook
```

When a hook fails, the framework inspects the Migration CR status to identify which pipeline step has an error:

```python
def _get_failed_migration_step(plan: Plan, vm_name: str) -> str:
    """Get step where VM migration failed.

    Examines the Migration status (not Plan) to find which pipeline step failed.
    """
    migration = _find_migration_for_plan(plan)
    # ...
    for step in pipeline:
        step_error = getattr(step, "error", None)
        if step_error:
            step_name = step.name
            LOGGER.info(f"VM {vm_name} failed at step '{step_name}': {step_error}")
            return step_name
```

### Multi-VM Consistency Validation

When a plan migrates multiple VMs, `validate_all_vms_same_step()` ensures all VMs failed at the same pipeline step. If VMs fail at different steps, a `VmMigrationStepMismatchError` is raised:

```python
class VmMigrationStepMismatchError(Exception):
    """Raised when VMs in the same plan fail at different migration steps."""

    def __init__(self, plan_name: str, failed_steps: dict[str, str | None]) -> None:
        self.plan_name = plan_name
        self.failed_steps = failed_steps
        super().__init__(f"VMs in plan '{plan_name}' failed at different steps: {failed_steps}")
```

### Expected vs Actual Failure Validation

For predefined mode hooks, `validate_expected_hook_failure()` verifies the actual failure step matches expectations. PreHook is checked first since it runs before PostHook in the pipeline:

```python
def validate_expected_hook_failure(
    actual_failed_step: str,
    plan_config: dict[str, Any],
) -> None:
    pre_hook_expected = pre_hook_config.get("expected_result") if pre_hook_config else None
    post_hook_expected = post_hook_config.get("expected_result") if post_hook_config else None

    # PreHook runs before PostHook, so check PreHook first
    if pre_hook_expected == "fail":
        expected_step = "PreHook"
    elif post_hook_expected == "fail":
        expected_step = "PostHook"
    else:
        return  # Custom mode - skip validation

    if actual_failed_step != expected_step:
        raise AssertionError(
            f"Migration failed at step '{actual_failed_step}' "
            f"but expected to fail at '{expected_step}'"
        )
```

> **Tip:** For custom playbook mode (no `expected_result` set), step validation is a no-op. Only the multi-VM consistency check is performed.

### The Orchestrator: `validate_hook_failure_and_check_vms`

This function ties together the full validation pipeline and returns a boolean indicating whether VMs should be validated post-migration:

```python
def validate_hook_failure_and_check_vms(
    plan_resource: "Plan",
    prepared_plan: dict[str, Any],
) -> bool:
    vm_names = [vm["name"] for vm in prepared_plan["virtual_machines"]]
    failed_steps = _get_all_vms_failed_steps(plan_resource, vm_names)
    actual_failed_step = validate_all_vms_same_step(plan_resource.name, failed_steps)
    validate_expected_hook_failure(actual_failed_step, prepared_plan)

    if actual_failed_step == "PostHook":
        return True   # VMs migrated — validate them
    elif actual_failed_step == "PreHook":
        return False  # VMs never migrated — skip checks
    else:
        raise ValueError(f"Unexpected failure step: {actual_failed_step}")
```

The return value semantics are critical:

| Failed Step | VMs Migrated? | Return Value | VM Checks |
|-------------|---------------|--------------|-----------|
| `PreHook`   | No            | `False`      | Skipped   |
| `PostHook`  | Yes           | `True`       | Performed |

## The `TestPostHookRetainFailedVm` Test Pattern

This test validates a specific scenario: the PreHook succeeds, migration completes, but the PostHook fails. Even though the overall migration is marked as failed, the VMs should be retained on the target cluster because they were already migrated before the PostHook ran.

### Test Configuration

```python
"test_post_hook_retain_failed_vm": {
    "virtual_machines": [
        {
            "name": "mtv-tests-rhel8",
            "source_vm_power": "on",
            "guest_agent": True,
        },
    ],
    "warm_migration": False,
    "target_power_state": "off",
    "pre_hook": {"expected_result": "succeed"},
    "post_hook": {"expected_result": "fail"},
    "expected_migration_result": "fail",
}
```

Key configuration points:
- `pre_hook` with `"succeed"` — Uses the success playbook (migration proceeds)
- `post_hook` with `"fail"` — Uses the failure playbook (migration marked as failed after VMs migrate)
- `expected_migration_result: "fail"` — The test expects `MigrationPlanExecError` to be raised

### Test Class Structure

The test follows the standard 5-step incremental pattern with hook-specific logic:

```python
@pytest.mark.tier0
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_post_hook_retain_failed_vm"])],
    indirect=True,
    ids=["post-hook-retain-failed-vm"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestPostHookRetainFailedVm:
    """Test PostHook with VM retention - migration fails but VMs should be retained."""

    storage_map: StorageMap | None = None
    network_map: NetworkMap | None = None
    plan_resource: Plan | None = None
    should_check_vms: bool = False
```

> **Note:** The `should_check_vms` class attribute defaults to `False`. It is set during `test_migrate_vms` based on where the hook failure occurred.

### Step 1–3: Resource Creation

The first three test methods (`test_create_storagemap`, `test_create_networkmap`, `test_create_plan`) follow the standard pattern. The key difference is in `test_create_plan`, which passes the hook references to the Plan CR:

```python
def test_create_plan(self, prepared_plan, fixture_store, ocp_admin_client,
                     source_provider, destination_provider, target_namespace,
                     source_provider_inventory):
    populate_vm_ids(prepared_plan, source_provider_inventory)

    self.__class__.plan_resource = create_plan_resource(
        # ...standard args...
        pre_hook_name=prepared_plan["_pre_hook_name"],
        pre_hook_namespace=prepared_plan["_pre_hook_namespace"],
        after_hook_name=prepared_plan["_post_hook_name"],
        after_hook_namespace=prepared_plan["_post_hook_namespace"],
    )
    assert self.plan_resource, "Plan creation failed"
```

### Step 4: Migration Execution with Expected Failure

The `test_migrate_vms` method handles the expected failure and determines whether VMs should be validated:

```python
def test_migrate_vms(self, prepared_plan, fixture_store, ocp_admin_client,
                     target_namespace):
    expected_result = prepared_plan["expected_migration_result"]

    if expected_result == "fail":
        with pytest.raises(MigrationPlanExecError):
            execute_migration(
                ocp_admin_client=ocp_admin_client,
                fixture_store=fixture_store,
                plan=self.plan_resource,
                target_namespace=target_namespace,
            )
        self.__class__.should_check_vms = validate_hook_failure_and_check_vms(
            self.plan_resource, prepared_plan
        )
    else:
        execute_migration(
            ocp_admin_client=ocp_admin_client,
            fixture_store=fixture_store,
            plan=self.plan_resource,
            target_namespace=target_namespace,
        )
        self.__class__.should_check_vms = True
```

The flow for this test:
1. `execute_migration()` raises `MigrationPlanExecError` (PostHook failure)
2. `pytest.raises` catches the expected exception
3. `validate_hook_failure_and_check_vms()` inspects the Migration CR, confirms all VMs failed at `PostHook`, and returns `True`
4. `should_check_vms` is set to `True` — VMs were migrated and should be validated

### Step 5: Conditional VM Validation

The final step checks VMs only if the hook failed after migration (PostHook), not before (PreHook):

```python
def test_check_vms(self, prepared_plan, source_provider, destination_provider,
                   source_provider_data, target_namespace, source_vms_namespace,
                   source_provider_inventory, vm_ssh_connections):
    # Runtime skip — decision based on previous test's migration result
    if not self.__class__.should_check_vms:
        pytest.skip("Skipping VM checks - hook failed before VM migration")

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

> **Note:** The `pytest.skip()` in `test_check_vms` is a runtime skip based on the outcome of a previous incremental test step, not a validation skip. This is an intentional exception to the "No Auto-Skip" rule — the decision of whether to check VMs can only be made after observing the migration result.

## Adding a New Hook Test

To add a new test with hooks:

1. **Add test configuration** in `tests/tests_config/config.py`:

```python
"test_pre_hook_failure": {
    "virtual_machines": [
        {"name": "my-vm", "source_vm_power": "on", "guest_agent": True},
    ],
    "warm_migration": False,
    "pre_hook": {"expected_result": "fail"},
    "expected_migration_result": "fail",
}
```

2. **Create a test file** following the `TestPostHookRetainFailedVm` pattern. The `prepared_plan` fixture will automatically create Hook CRs based on the `pre_hook`/`post_hook` configuration keys.

3. **Use the hook reference keys** when creating the Plan:
   - `prepared_plan["_pre_hook_name"]` / `prepared_plan["_pre_hook_namespace"]`
   - `prepared_plan["_post_hook_name"]` / `prepared_plan["_post_hook_namespace"]`

4. **Handle expected failures** in `test_migrate_vms` using `pytest.raises(MigrationPlanExecError)` and `validate_hook_failure_and_check_vms()`.

5. **Conditionally validate VMs** in `test_check_vms` based on `should_check_vms`.

## Architecture Summary

```
Test Config (config.py)
  └── pre_hook / post_hook keys
        │
        ▼
prepared_plan fixture (conftest.py)
  └── create_hook_if_configured()
        │
        ▼
utilities/hooks.py
  ├── validate_hook_config()        — mutual exclusivity check
  ├── validate_custom_playbook()    — base64/UTF-8/YAML/structure validation
  └── create_hook_for_plan()        — creates Hook CR via create_and_store_resource()
        │
        ▼
Test Class (test_*.py)
  ├── test_create_plan()            — passes hook name/namespace to Plan CR
  ├── test_migrate_vms()            — catches expected failure, calls validation
  │     └── validate_hook_failure_and_check_vms()
  │           ├── _get_all_vms_failed_steps()     — inspects Migration CR pipeline
  │           ├── validate_all_vms_same_step()    — ensures consistency
  │           └── validate_expected_hook_failure() — matches expected vs actual step
  └── test_check_vms()              — conditional VM validation based on failure step
```
