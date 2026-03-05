# Custom Exceptions

All custom exceptions for the MTV API Tests framework are centralized in `exceptions/exceptions.py`. This module provides domain-specific exception classes organized into four categories: migration errors, VM errors, provider errors, and infrastructure errors.

> **Note:** Per project convention, all custom exceptions **must** be defined in `exceptions/exceptions.py` and never scattered across other modules. This centralizes exception definitions for better discoverability.

## Overview

| Exception | Category | Description |
|-----------|----------|-------------|
| `MigrationPlanExecError` | Migration | Plan execution failed or timed out |
| `MigrationNotFoundError` | Migration | Migration CR not found for a Plan |
| `MigrationStatusError` | Migration | Migration CR has no status or incomplete status |
| `VmMigrationStepMismatchError` | Migration | VMs in the same plan failed at different steps |
| `VmCloneError` | VM | VM cloning operation failed |
| `VmNotFoundError` | VM | VM not found on source provider or in migration status |
| `VmMissingVmxError` | VM | VMware VM is missing its VMX file |
| `VmBadDatastoreError` | VM | VM has a bad datastore status |
| `VmPipelineError` | VM | VM pipeline is missing or has no failed step |
| `InvalidVMNameError` | VM | VM name cannot be sanitized to valid DNS-1123 |
| `OvirtMTVDatacenterNotFoundError` | Provider | MTV-CNV datacenter not found in RHV |
| `OvirtMTVDatacenterStatusError` | Provider | MTV-CNV datacenter status is not "up" |
| `MissingProvidersFileError` | Provider | `.providers.json` file is missing or empty |
| `ForkliftPodsNotRunningError` | Infrastructure | Forklift controller pod not found or pods not running |
| `RemoteClusterAndLocalCluterNamesError` | Infrastructure | Remote cluster name doesn't match local cluster |
| `SessionTeardownError` | Infrastructure | Session teardown failed to clean up resources |
| `ResourceNameNotStartedWithSessionUUIDError` | Infrastructure | Resource name doesn't start with session UUID |

---

## Migration Errors

Exceptions related to MTV migration plan execution and status tracking.

### `MigrationPlanExecError`

Raised when a migration plan fails to execute successfully or times out waiting for completion.

```python
class MigrationPlanExecError(Exception):
    pass
```

This is the primary exception surfaced during `execute_migration()`. It is raised in two scenarios: when the plan status reaches `FAILED`, or when the plan times out before reaching a terminal state.

**Where it's raised** (`utilities/mtv_migration.py`):

```python
def wait_for_migration_complate(plan: Plan) -> None:
    try:
        last_status: str = ""

        for sample in TimeoutSampler(
            func=get_plan_migration_status,
            sleep=1,
            wait_timeout=py_config.get("plan_wait_timeout", 600),
            plan=plan,
        ):
            if sample != last_status:
                LOGGER.info(f"Plan '{plan.name}' migration status: '{sample}'")
                last_status = sample

            if sample == Plan.Status.SUCCEEDED:
                return

            elif sample == Plan.Status.FAILED:
                raise MigrationPlanExecError()

    except (TimeoutExpiredError, MigrationPlanExecError):
        raise MigrationPlanExecError(
            f"Plan {plan.name} failed to reach the expected condition. \nstatus:\n\t{plan.instance}"
        )
```

**Test usage** — tests can catch this exception to handle expected migration failures (`tests/test_post_hook_retain_failed_vm.py`):

```python
from exceptions.exceptions import MigrationPlanExecError

class TestPostHookRetainFailedVm:
    """Test PostHook with VM retention - migration fails but VMs should be retained."""
    # MigrationPlanExecError is imported for expected-failure test scenarios
```

---

### `MigrationNotFoundError`

Raised when a Migration Custom Resource cannot be found for a given Plan. Includes a descriptive message with the plan name and namespace.

```python
class MigrationNotFoundError(Exception):
    """Raised when Migration CR cannot be found for a Plan."""

    def __init__(self, message: str) -> None:
        self.message = message
        super().__init__(message)
```

**Where it's raised** (`utilities/mtv_migration.py`):

```python
def _find_migration_for_plan(plan: Plan) -> Migration:
    for migration in Migration.get(client=plan.client, namespace=plan.namespace):
        if migration.instance.metadata.ownerReferences:
            for owner_ref in migration.instance.metadata.ownerReferences:
                if owner_ref.get("kind") == "Plan" and owner_ref.get("name") == plan.name:
                    return migration

    raise MigrationNotFoundError(
        f"Migration CR not found for Plan '{plan.name}' in namespace '{plan.namespace}'"
    )
```

**Where it's caught** — in `get_plan_migration_status()`, this exception is caught to indicate that migration hasn't started yet:

```python
def get_plan_migration_status(plan: Plan) -> str:
    # ...
    try:
        _find_migration_for_plan(plan)
        return Plan.Status.EXECUTING
    except MigrationNotFoundError:
        return ""
```

---

### `MigrationStatusError`

Raised when a Migration CR has no status or an incomplete status (e.g., missing the `vms` field). Stores the migration name for diagnostic purposes.

```python
class MigrationStatusError(Exception):
    """Raised when Migration CR has no status or incomplete status."""

    def __init__(self, migration_name: str) -> None:
        self.migration_name = migration_name
        super().__init__(f"Migration CR '{migration_name}' has no status or incomplete status")
```

**Where it's raised** (`utilities/mtv_migration.py`):

```python
def _get_failed_migration_step(plan: Plan, vm_name: str) -> str:
    migration = _find_migration_for_plan(plan)

    if not hasattr(migration.instance, "status") or not migration.instance.status:
        raise MigrationStatusError(migration_name=migration.name)

    vms_status = getattr(migration.instance.status, "vms", None)
    if not vms_status:
        raise MigrationStatusError(migration_name=migration.name)
    # ...
```

---

### `VmMigrationStepMismatchError`

Raised when VMs within the same migration plan fail at different pipeline steps, or when no failed step can be identified. This is used during hook validation to ensure all VMs exhibit consistent failure behavior.

```python
class VmMigrationStepMismatchError(Exception):
    """Raised when VMs in the same plan fail at different migration steps."""

    def __init__(self, plan_name: str, failed_steps: dict[str, str | None]) -> None:
        self.plan_name = plan_name
        self.failed_steps = failed_steps
        super().__init__(f"VMs in plan '{plan_name}' failed at different steps: {failed_steps}")
```

**Where it's raised** (`utilities/hooks.py`):

```python
def validate_all_vms_same_step(plan_name: str, failed_steps: dict[str, str | None]) -> str:
    unique_steps = {step for step in failed_steps.values() if step is not None}

    if not unique_steps:
        raise VmMigrationStepMismatchError(plan_name, failed_steps)

    if len(unique_steps) > 1:
        raise VmMigrationStepMismatchError(plan_name, failed_steps)

    common_step = unique_steps.pop()
    LOGGER.info("All VMs failed at step '%s'", common_step)
    return common_step
```

---

## VM Errors

Exceptions related to virtual machine operations: lookup, cloning, health checks, and pipeline status.

### `VmNotFoundError`

A general-purpose exception raised when a VM cannot be found. Used across all provider implementations (VMware, RHV, OpenStack) and in migration status checking.

```python
class VmNotFoundError(Exception):
    pass
```

**Provider-specific usage examples:**

VMware (`libs/providers/vmware.py`):
```python
if not target_vm:
    raise VmNotFoundError(f"VM {target_vm_name} not found on host [{self.host}]")
```

RHV (`libs/providers/rhv.py`) — caught gracefully during VM deletion:
```python
except VmNotFoundError:
    LOGGER.warning(f"VM '{vm_name}' not found. Nothing to delete.")
    return
```

OpenStack (`libs/providers/openstack.py`):
```python
source_vm = self.get_instance_obj(name_filter=source_vm_name)
if not source_vm:
    raise VmNotFoundError(f"Source VM '{source_vm_name}' not found.")
```

Migration status (`utilities/mtv_migration.py`):
```python
raise VmNotFoundError(f"VM {vm_name} not found in Migration {migration.name} status")
```

---

### `VmCloneError`

Raised when a VM cloning operation fails. This is the most widely used exception in the codebase, covering scenarios from missing secrets and insufficient datastore capacity to vSphere task failures.

```python
class VmCloneError(Exception):
    pass
```

**Common scenarios** (`libs/providers/vmware.py`):

Secret not found during clone:
```python
except TimeoutExpiredError as exc:
    LOGGER.exception("Timed out waiting for secret '%s' to be created.", secret_name)
    raise VmCloneError(f"SSH public key secret '{secret_name}' not found.") from exc
```

vSphere task failure:
```python
raise VmCloneError(f"vSphere task failed: {error_msg}")
```

Missing SCSI controller:
```python
raise VmCloneError(
    f"Could not find a SCSI controller on VM '{source_vm.name}' to add new disks."
)
```

Insufficient datastore capacity:
```python
raise VmCloneError(
    format_insufficient_capacity_message(datastore.name, required_gb, available_space_gb),
)
```

**Retry handling** — `VmCloneError` is caught during clone operations to retry without MAC regeneration when a resource conflict occurs:

```python
try:
    res = self.wait_task(
        task=task,
        action_name=f"Cloning VM {clone_vm_name} from {source_vm_name}",
        wait_timeout=60 * 20,
        sleep=5,
    )
except VmCloneError as e:
    if regenerate_mac and "in use" in str(e).lower():
        LOGGER.warning("Clone failed with resource conflict, retrying without MAC regeneration")
        # ... retry logic
```

---

### `VmMissingVmxError`

Raised when a VMware VM is missing its VMX configuration file. Includes a custom `__str__` method for clear error messaging.

```python
class VmMissingVmxError(Exception):
    def __init__(self, vm: str) -> None:
        self.vm = vm

    def __str__(self) -> str:
        return f"VM is missing VMX file: {self.vm}"
```

**Where it's raised** (`libs/providers/vmware.py`):

```python
if self.is_vm_missing_vmx_file(vm=target_vm):
    raise VmMissingVmxError(vm=target_vm.name)
```

---

### `VmBadDatastoreError`

Raised when a VM has a bad datastore status or when a datastore ID cannot be resolved. Includes a custom `__str__` method.

```python
class VmBadDatastoreError(Exception):
    def __init__(self, vm: str) -> None:
        self.vm = vm

    def __str__(self) -> str:
        return f"VM have bad datastore status: {self.vm}"
```

**Where it's raised** (`libs/providers/vmware.py`):

```python
# During datastore lookup
datastore = self.get_obj([vim.Datastore], datastore_id)
if not datastore:
    raise VmBadDatastoreError(f"Datastore with ID '{datastore_id}' not found.")

# During VM health checks
if self.is_vm_with_bad_datastore(vm=target_vm):
    raise VmBadDatastoreError(vm=target_vm.name)
```

> **Note:** `VmBadDatastoreError` accepts a `vm` parameter in its `__init__`, but is also raised with a plain string message (bypassing the custom `__str__`). Both patterns are valid since the base `Exception` class handles string arguments.

---

### `VmPipelineError`

Raised when a VM's migration pipeline is missing or contains no failed step. Used during post-failure analysis.

```python
class VmPipelineError(Exception):
    """Raised when VM pipeline is missing or has no failed step."""

    def __init__(self, vm_name: str) -> None:
        self.vm_name = vm_name
        super().__init__(f"VM '{vm_name}' pipeline is missing or has no failed step")
```

**Where it's raised** (`utilities/mtv_migration.py`):

```python
pipeline = getattr(vm_status, "pipeline", None)
if not pipeline:
    raise VmPipelineError(vm_name=vm_name)

for step in pipeline:
    step_error = getattr(step, "error", None)
    if step_error:
        step_name = step.name
        LOGGER.info(f"VM {vm_name} failed at step '{step_name}': {step_error}")
        return step_name

raise VmPipelineError(vm_name=vm_name)
```

---

### `InvalidVMNameError`

Raised when a VM name cannot be sanitized to a valid Kubernetes DNS-1123 name (lowercase alphanumeric and hyphens, starting and ending with alphanumeric).

```python
class InvalidVMNameError(Exception):
    pass
```

**Where it's raised** (`utilities/naming.py`):

```python
def sanitize_kubernetes_name(name: str, max_length: int = 63) -> str:
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

**Where it's caught** (`libs/providers/openshift.py`) — caught and re-raised with additional logging context:

```python
try:
    cnv_vm_name = sanitize_kubernetes_name(cnv_vm_name)
except InvalidVMNameError:
    LOGGER.error(
        f"Failed to sanitize VM name for Kubernetes lookup. "
        f"original_name='{original_name}', namespace='{kwargs.get('namespace')}'"
    )
    raise
```

---

## Provider Errors

Exceptions specific to source provider connectivity and configuration.

### `OvirtMTVDatacenterNotFoundError`

Raised when the `MTV-CNV` datacenter cannot be found in the RHV (oVirt) environment.

```python
class OvirtMTVDatacenterNotFoundError(Exception):
    pass
```

**Where it's raised** (`libs/providers/rhv.py`):

```python
@property
def mtv_datacenter(self) -> ovirtsdk4.types.DataCenter:
    for dc in self.data_center_service.list():
        if dc.name == "MTV-CNV":
            return dc

    raise OvirtMTVDatacenterNotFoundError()
```

---

### `OvirtMTVDatacenterStatusError`

Raised when the MTV-CNV datacenter exists but its status is not `"up"`. This is checked during provider connection setup.

```python
class OvirtMTVDatacenterStatusError(Exception):
    pass
```

**Where it's raised** (`libs/providers/rhv.py`):

```python
def connect(self) -> Self:
    self.api = ovirtsdk4.Connection(
        url=self.host,
        username=self.username,
        password=self.password,
        ca_file=self.ca_file if not self.insecure else None,
        debug=self.debug,
        log=self.log,
        insecure=self.insecure,
    )
    if not self.is_mtv_datacenter_ok:
        raise OvirtMTVDatacenterStatusError()
    return self
```

> **Tip:** `OvirtMTVDatacenterNotFoundError` and `OvirtMTVDatacenterStatusError` form a validation chain: the `mtv_datacenter` property raises `NotFound` if the datacenter doesn't exist, while `connect()` raises `StatusError` if it exists but is unhealthy. The `is_mtv_datacenter_ok` property accesses `mtv_datacenter` internally, so a missing datacenter is caught first.

---

### `MissingProvidersFileError`

Raised when the `.providers.json` configuration file is missing or empty. Has a hardcoded message via `__init__`.

```python
class MissingProvidersFileError(Exception):
    def __init__(self) -> None:
        super().__init__("'.providers.json' file is missing or empty")
```

**Where it's raised** (`conftest.py`):

```python
@pytest.fixture(scope="session")
def source_provider_data(source_providers, fixture_store):
    if not source_providers:
        raise MissingProvidersFileError()
    # ...
```

---

## Infrastructure Errors

Exceptions related to test infrastructure: cluster state, pod health, session management, and resource naming.

### `ForkliftPodsNotRunningError`

Raised when forklift controller pods are not found or are not in a `RUNNING`/`SUCCEEDED` state. Used with `TimeoutSampler` to allow pods time to reach a ready state before failing.

```python
class ForkliftPodsNotRunningError(Exception):
    pass
```

**Where it's raised** (`conftest.py`):

```python
@pytest.fixture(scope="session")
def forklift_pods_state(ocp_admin_client: DynamicClient) -> None:
    def _get_not_running_pods(_admin_client: DynamicClient) -> bool:
        controller_pod: str | None = None
        not_running_pods: list[str] = []

        for pod in Pod.get(client=_admin_client, namespace=py_config["mtv_namespace"]):
            if pod.name.startswith("forklift-"):
                if pod.name.startswith("forklift-controller"):
                    controller_pod = pod

                if pod.status not in (pod.Status.RUNNING, pod.Status.SUCCEEDED):
                    not_running_pods.append(pod.name)

        if not controller_pod:
            raise ForkliftPodsNotRunningError("Forklift controller pod not found")

        if not_running_pods:
            raise ForkliftPodsNotRunningError(
                f"Some of the forklift pods are not running: {not_running_pods}"
            )

        return True

    for sample in TimeoutSampler(
        func=_get_not_running_pods,
        _admin_client=ocp_admin_client,
        sleep=1,
        wait_timeout=60 * 5,
        exceptions_dict={ForkliftPodsNotRunningError: [], NotFoundError: []},
    ):
        if sample:
            break
```

> **Note:** `ForkliftPodsNotRunningError` is listed in the `TimeoutSampler`'s `exceptions_dict`, which means the sampler will retry on this exception rather than immediately propagating it. The exception only surfaces if pods fail to become ready within the 5-minute timeout.

---

### `RemoteClusterAndLocalCluterNamesError`

Raised when the remote OCP cluster name configured in `py_config` does not match the local cluster that the client is connected to.

```python
class RemoteClusterAndLocalCluterNamesError(Exception):
    pass
```

**Where it's raised** (`conftest.py`):

```python
@pytest.fixture(scope="session")
def ocp_admin_client():
    _client = get_cluster_client()

    if remote_cluster_name := get_value_from_py_config("remote_ocp_cluster"):
        if remote_cluster_name not in _client.configuration.host:
            raise RemoteClusterAndLocalCluterNamesError(
                "Remote cluster must be the same as local cluster."
            )

    yield _client
```

---

### `SessionTeardownError`

Raised when session teardown fails to clean up all created resources, indicating leftover resources remain in the cluster.

```python
class SessionTeardownError(Exception):
    pass
```

**Where it's raised** (`utilities/pytest_utils.py`):

```python
leftovers = teardown_resources(
    session_store=session_store,
    ocp_client=ocp_client,
    target_namespace=session_store.get("target_namespace"),
)
if leftovers:
    raise SessionTeardownError(
        f"Failed to clean up the following resources: {leftovers}"
    )
```

> **Warning:** A `SessionTeardownError` indicates resources were leaked in the cluster. Investigate the leftover resources listed in the error message and clean them up manually if necessary.

---

### `ResourceNameNotStartedWithSessionUUIDError`

Reserved exception for enforcing that resource names start with the session UUID. Currently defined but not used in the codebase.

```python
class ResourceNameNotStartedWithSessionUUIDError(Exception):
    pass
```

> **Note:** This exception is defined but not currently raised anywhere. It exists as a safeguard for future enforcement of the naming convention where all created resources must be prefixed with the session UUID (e.g., `auto-abc123-`).

---

## Exception Hierarchy

All custom exceptions inherit directly from Python's built-in `Exception` class. There is no shared base exception for the MTV framework.

```
Exception
├── MigrationPlanExecError
├── MigrationNotFoundError
├── MigrationStatusError
├── VmMigrationStepMismatchError
├── VmCloneError
├── VmNotFoundError
├── VmMissingVmxError
├── VmBadDatastoreError
├── VmPipelineError
├── InvalidVMNameError
├── OvirtMTVDatacenterNotFoundError
├── OvirtMTVDatacenterStatusError
├── MissingProvidersFileError
├── ForkliftPodsNotRunningError
├── RemoteClusterAndLocalCluterNamesError
├── SessionTeardownError
└── ResourceNameNotStartedWithSessionUUIDError
```

---

## Usage Guidelines

### Importing Exceptions

Always import from the centralized module:

```python
from exceptions.exceptions import MigrationPlanExecError, VmNotFoundError
```

### When to Use Custom vs Built-in Exceptions

Per project conventions in `CLAUDE.md`:

| Scenario | Exception to Use |
|----------|-----------------|
| Migration plan failed | `MigrationPlanExecError` |
| VM not found on provider | `VmNotFoundError` |
| VM clone operation failed | `VmCloneError` |
| Invalid input or config | `ValueError` |
| Type mismatch | `TypeError` |
| Key not found | `KeyError` (let propagate) |
| Connection failure | `ConnectionError` |
| Any other domain-specific error | Create a new custom exception |

> **Warning:** `RuntimeError` should **not** be used for domain-specific errors. It is allowed **only** in pytest hooks for infrastructure failures (e.g., cluster unreachable, API timeout). Use `ValueError` for configuration errors.

### Adding New Exceptions

1. Define the exception class in `exceptions/exceptions.py`
2. Include a docstring explaining when it should be raised
3. Use typed constructor parameters for structured error data (see `VmMigrationStepMismatchError` for a good example)
4. Import and use from the centralized module

```python
# In exceptions/exceptions.py
class ProviderConnectionError(Exception):
    """Raised when a connection to a source provider fails."""

    def __init__(self, provider_name: str, reason: str) -> None:
        self.provider_name = provider_name
        self.reason = reason
        super().__init__(f"Failed to connect to provider '{provider_name}': {reason}")
```
