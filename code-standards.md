# Code Standards

This page documents the coding conventions and standards enforced across the mtv-api-tests project. These rules ensure consistency, maintainability, and reliability of the test suite.

## Type Annotations

All new functions and any functions with signature changes **must** have complete type annotations. The project uses Python 3.12+ and enforces type checking via [mypy](https://mypy-lang.org/) in pre-commit hooks.

### Rules

- Use built-in Python types: `dict`, `list`, `tuple` instead of `typing.Dict`, `typing.List`, `typing.Tuple`
- Use `|` union syntax instead of `typing.Union` (e.g., `str | None` not `Optional[str]`)
- Import `TYPE_CHECKING` from `typing` for imports only needed at type-check time
- Use string annotations (`"DynamicClient"`) for forward references

### Real Example

From `utilities/resources.py`:

```python
from typing import TYPE_CHECKING, Any

from ocp_resources.resource import Resource

if TYPE_CHECKING:
    from kubernetes.dynamic import DynamicClient


def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
```

From `utilities/naming.py`:

```python
def sanitize_kubernetes_name(name: str, max_length: int = 63) -> str:
```

### mypy Configuration

The project's `pyproject.toml` enables strict mypy checks:

```toml
[tool.mypy]
disallow_any_generics = false
disallow_incomplete_defs = true
no_implicit_optional = true
show_error_codes = true
warn_unused_ignores = true
```

Type stubs are installed for third-party libraries (`types-pyvmomi`, `types-requests`, `types-PyYAML`, `types-paramiko`, etc.) via the pre-commit mypy hook.

---

## Docstring Format

All new functions must include docstrings with `Args`, `Returns`, and `Raises` sections. Each parameter should include its type in parentheses.

### Format

```python
def function_name(param: Type) -> ReturnType:
    """Brief description of what the function does.

    Args:
        param (Type): Description of the parameter

    Returns:
        ReturnType: Description of the return value

    Raises:
        ExceptionType: When this exception is raised
    """
```

### Real Example

From `utilities/mtv_migration.py`:

```python
def _find_migration_for_plan(plan: Plan) -> Migration:
    """Find Migration CR for Plan.

    Args:
        plan (Plan): The Plan resource

    Returns:
        Migration: The Migration CR owned by the Plan

    Raises:
        MigrationNotFoundError: If Migration CR cannot be found
    """
```

From `utilities/naming.py`:

```python
def sanitize_kubernetes_name(name: str, max_length: int = 63) -> str:
    """Sanitize a VM name to comply with Kubernetes DNS-1123 naming conventions.

    Rules:
    - lowercase alphanumeric characters and '-'
    - must start and end with an alphanumeric character
    - max `max_length` characters

    Args:
        name (str): The VM name to sanitize
        max_length (int): Maximum length for the sanitized name. Defaults to 63.

    Returns:
        str: The sanitized name compliant with Kubernetes DNS-1123 conventions

    Raises:
        InvalidVMNameError: If the name cannot be sanitized to a valid DNS-1123 name
            (i.e., contains no alphanumeric characters)
    """
```

---

## Fail-Fast Validation

Code must never allow `None` to propagate silently when `None` is not a valid value. Validate early and raise descriptive errors.

### Rules

- Use `if value is None:` for explicit None checks
- Use `if not value:` only when empty containers and `False` are also invalid
- Validate in fixtures where values originate (see [Validate at Source](#validate-at-source))

### Real Examples

From `utilities/mtv_migration.py` — validating provider state before plan creation:

```python
def create_plan_resource(
    ocp_admin_client: DynamicClient,
    source_provider: BaseProvider,
    destination_provider: OCPProvider,
    ...
) -> Plan:
    if not source_provider.ocp_resource:
        raise ValueError("source_provider.ocp_resource is not set")

    if not destination_provider.ocp_resource:
        raise ValueError("destination_provider.ocp_resource is not set")
```

From `utilities/naming.py` — validating sanitization produced a usable result:

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

From `conftest.py` — validating provider configuration exists:

```python
@pytest.fixture(scope="session")
def source_provider_data(source_providers: dict[str, dict[str, Any]], fixture_store: dict[str, Any]) -> dict[str, Any]:
    if not source_providers:
        raise MissingProvidersFileError()

    requested_provider = py_config["source_provider"]
    if requested_provider not in source_providers:
        raise ValueError(
            f"Source provider '{requested_provider}' not found in '.providers.json'. "
            f"Available providers: {sorted(source_providers.keys())}"
        )
```

---

## Pass Objects Over Values

Functions should receive objects and extract needed values internally. This reduces parameter count, simplifies call sites, and makes refactoring easier.

### Real Example

From `utilities/mtv_migration.py` — `create_plan_resource` receives provider and map objects, then extracts names and namespaces internally:

```python
def create_plan_resource(
    source_provider: BaseProvider,       # Object, not name + namespace
    destination_provider: OCPProvider,   # Object, not name + namespace
    storage_map: StorageMap,             # Object, not name + namespace
    network_map: NetworkMap,             # Object, not name + namespace
    ...
) -> Plan:
    # Values extracted inside the function
    plan_kwargs: dict[str, Any] = {
        "source_provider_name": source_provider.ocp_resource.name,
        "source_provider_namespace": source_provider.ocp_resource.namespace,
        "destination_provider_name": destination_provider.ocp_resource.name,
        "destination_provider_namespace": destination_provider.ocp_resource.namespace,
        "storage_map_name": storage_map.name,
        "storage_map_namespace": storage_map.namespace,
        "network_map_name": network_map.name,
        "network_map_namespace": network_map.namespace,
        ...
    }
```

Callers pass objects directly:

```python
self.__class__.plan_resource = create_plan_resource(
    ocp_admin_client=ocp_admin_client,
    fixture_store=fixture_store,
    source_provider=source_provider,
    destination_provider=destination_provider,
    storage_map=self.storage_map,
    network_map=self.network_map,
    virtual_machines_list=prepared_plan["virtual_machines"],
    target_namespace=target_namespace,
)
```

> **Note:** This rule applies when you control both the function signature and the caller. Existing external APIs that require extracted values may accept values directly.

---

## No Unnecessary Variables

Avoid intermediate variables that add no clarity. Return or yield values directly when the intent is obvious.

### Real Examples

From `conftest.py` — direct yield without storing in a temporary:

```python
@pytest.fixture(scope="session")
def target_namespace(ocp_admin_client, fixture_store, session_uuid):
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

From `utilities/resources.py` — direct return:

```python
def get_or_create_namespace(
    fixture_store: dict[str, Any],
    ocp_admin_client: "DynamicClient",
    namespace_name: str,
) -> str:
    ns = Namespace(name=namespace_name, client=ocp_admin_client)
    if ns.exists:
        LOGGER.info(f"Namespace {namespace_name} already exists, using it")
    else:
        ns = create_and_store_resource(...)
    ns.wait_for_status(status=ns.Status.ACTIVE)
    return namespace_name
```

> **Tip:** The litmus test: if removing a variable and inlining the value doesn't hurt readability, remove it.

---

## Variables Must Have Consistent Types

A variable must always hold the same type throughout its lifetime. Use empty container defaults to avoid `None` creeping in.

### Real Examples

From `libs/forklift_inventory.py` — using empty container defaults:

```python
attached_volumes = vm.get("attachedVolumes", [])
if not attached_volumes:
    return False
```

```python
vm_addresses = vm.get("addresses", {})
if not vm_addresses:
    LOGGER.debug(f"VM '{vm_name}' found but has no addresses yet.")
    return False
```

From `utilities/mtv_migration.py`:

```python
num_added_disks = len(vm_config.get("add_disks", []))
```

---

## Deterministic Tests

Tests must be deterministic and reproducible. No randomness is allowed.

### Rules

- Never use `random.choice()`, `random.sample()`, or any random selection
- Use index-based selection (`[0]`) when order does not matter
- Use deterministic naming with UUIDs generated from `shortuuid` for uniqueness without randomness in test logic

### Real Example

From `utilities/naming.py` — name generation uses ShortUUID for uniqueness but selection logic is always deterministic:

```python
def generate_name_with_uuid(name: str) -> str:
    _name = f"{name}-{shortuuid.ShortUUID().random(length=4).lower()}"
    _name = _name.replace("_", "-").replace(".", "-").lower()
    return _name
```

### No Defaults for Our Configuration

For configurations the project controls (`py_config`, `plan`, `tests_params`), never use fallback defaults. If a key is missing, it should fail immediately:

```python
# Wrong - hides missing config
storage_class = py_config.get("storage_class", "default-storage")

# Correct - fails fast if missing
storage_class = py_config["storage_class"]
```

For external or provider data, `.get()` with validation is acceptable:

```python
vm_id = provider_data.get("vm_id")
if not vm_id:
    raise ValueError(f"VM ID not found for VM '{vm_name}'")
```

Optional feature flags are the sole exception:

```python
if plan.get("warm_migration", False):  # OK - this is an optional feature flag
    setup_warm_migration()
```

---

## OpenShift Resource Interaction Rules

### Required: `openshift-python-wrapper`

All cluster interactions must use the [`openshift-python-wrapper`](https://github.com/RedHatQE/openshift-python-wrapper) library. Direct usage of the `kubernetes` Python package at runtime is forbidden.

```python
# Correct imports
from ocp_resources.namespace import Namespace
from ocp_resources.secret import Secret
from ocp_resources.virtual_machine import VirtualMachine
from ocp_resources.provider import Provider
from ocp_resources.resource import ResourceEditor

# Forbidden at runtime
from kubernetes import client              # Never
from kubernetes.dynamic import DynamicClient  # Never instantiate directly
```

### `DynamicClient` Usage Rules

| Usage | Allowed |
|-------|---------|
| Import inside `TYPE_CHECKING` block | Yes |
| String annotation `"DynamicClient"` | Yes |
| Instantiate `DynamicClient(...)` directly | **No** — use `get_client()` |
| Use in `isinstance()` or runtime checks | **No** |
| Import other `kubernetes.*` modules | **No** |

Real example from `conftest.py`:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from kubernetes.dynamic import DynamicClient

# In fixture:
@pytest.fixture(scope="session")
def ocp_admin_client():
    _client = get_cluster_client()  # Uses openshift-python-wrapper
    yield _client
```

### Resource Creation: `create_and_store_resource()`

Every OpenShift resource must be created through the centralized `create_and_store_resource()` function in `utilities/resources.py`. This ensures:

- Automatic unique name generation with session UUID
- Deployment with `wait=True`
- Registration in `fixture_store["teardown"]` for cleanup
- Conflict handling (reuses existing resources)
- Name truncation to 63 characters (Kubernetes DNS-1123 limit)

```python
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
```

Real usage from `conftest.py`:

```python
provider = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Provider,
    kind_dict=kind_dict,
    client=ocp_admin_client,
)
```

```python
namespace = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Namespace,
    client=ocp_admin_client,
    name=unique_namespace_name,
    label=label,
)
```

> **Warning:** Never create resources directly by calling `resource.deploy()`. This bypasses tracking and will leave orphaned resources on the cluster.

```python
# Wrong - bypasses tracking and cleanup
namespace = Namespace(client=ocp_admin_client, name="my-namespace")
namespace.deploy()

# Correct - tracked for automatic cleanup
namespace = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Namespace,
    client=ocp_admin_client,
    name="my-namespace",
)
```

---

## Exception Types

Use specific exception types instead of generic `RuntimeError`. Domain-specific custom exceptions are encouraged.

### Standard Exception Mapping

| Scenario | Exception Type |
|----------|---------------|
| Invalid input or configuration | `ValueError` |
| Missing resource | `ValueError` |
| Type issues | `TypeError` |
| Key not found | `KeyError` (let propagate) |
| Connection failures | `ConnectionError` |
| Domain-specific errors | Custom exception class (preferred) |

> **Note:** `RuntimeError` is allowed **only** in pytest hooks for infrastructure failures (e.g., cluster unreachable). Use `ValueError` for configuration errors.

### Custom Exceptions

All custom exceptions are defined in `exceptions/exceptions.py`. This centralizes exception definitions for discoverability.

Real examples from the codebase:

```python
class MigrationNotFoundError(Exception):
    """Raised when Migration CR cannot be found for a Plan."""

    def __init__(self, message: str) -> None:
        self.message = message
        super().__init__(message)


class MigrationStatusError(Exception):
    """Raised when Migration CR has no status or incomplete status."""

    def __init__(self, migration_name: str) -> None:
        self.migration_name = migration_name
        super().__init__(f"Migration CR '{migration_name}' has no status or incomplete status")


class VmMigrationStepMismatchError(Exception):
    """Raised when VMs in the same plan fail at different migration steps."""

    def __init__(self, plan_name: str, failed_steps: dict[str, str | None]) -> None:
        self.plan_name = plan_name
        self.failed_steps = failed_steps
        super().__init__(f"VMs in plan '{plan_name}' failed at different steps: {failed_steps}")


class MissingProvidersFileError(Exception):
    def __init__(self) -> None:
        super().__init__("'.providers.json' file is missing or empty")
```

Usage from `utilities/mtv_migration.py`:

```python
raise MigrationNotFoundError(
    f"Migration CR not found for Plan '{plan.name}' in namespace '{plan.namespace}'"
)
```

---

## Validate at Source

Validation (verifying required inputs and configuration values are present and valid) must happen in **fixtures** where values originate — not in utility functions or test methods.

### Rules

- Fixtures validate their own construction ("is the value I produce valid?")
- Utility functions may validate external/provider data that varies at runtime
- Test methods should **never** contain validation logic

### Real Example

From `conftest.py` — validation happens in the fixture:

```python
@pytest.fixture(scope="session")
def source_provider_data(source_providers: dict[str, dict[str, Any]], fixture_store: dict[str, Any]) -> dict[str, Any]:
    if not source_providers:
        raise MissingProvidersFileError()

    requested_provider = py_config["source_provider"]
    if requested_provider not in source_providers:
        raise ValueError(
            f"Source provider '{requested_provider}' not found in '.providers.json'. "
            f"Available providers: {sorted(source_providers.keys())}"
        )
    return source_providers[requested_provider]
```

From `conftest.py` — the multus fixture validates that networks exist:

```python
@pytest.fixture(scope="class")
def multus_network_name(...) -> dict[str, str]:
    networks = source_provider.get_vm_or_template_networks(names=vms, inventory=source_provider_inventory)

    if not networks:
        raise ValueError(f"No networks found for VMs {vms}. VMs must have at least one network interface.")
```

> **Warning:** Never use `pytest.skip()` or `pytest.fail()` for validation inside fixtures. Use `@pytest.mark.skipif` at the class or test level for conditional skipping.

---

## Context Managers for Cleanup

Use context managers to ensure proper resource cleanup, especially when modifying cluster state temporarily.

### Real Examples

From `conftest.py` — temporary StorageProfile modification using `ResourceEditor`:

```python
@pytest.fixture(scope="session")
def nfs_storage_profile(ocp_admin_client):
    nfs = StorageClass.Types.NFS
    if py_config["storage_class"] == nfs:
        storage_profile = StorageProfile(client=ocp_admin_client, name=nfs, ensure_exists=True)

        with ResourceEditor(
            patches={
                storage_profile: {
                    "spec": {
                        "claimPropertySets": [
                            {
                                "accessModes": ["ReadWriteOnce"],
                                "volumeMode": "Filesystem",
                            }
                        ]
                    }
                }
            }
        ):
            yield
    else:
        yield
```

From `conftest.py` — temporary ForkliftController modification:

```python
with ResourceEditor(
    patches={
        forklift_controller: {
            "spec": {
                "controller_precopy_interval": str(snapshots_interval),
            }
        }
    }
):
    forklift_controller.wait_for_condition(...)
    yield
```

From `utilities/utils.py` — provider connection lifecycle:

```python
@contextmanager
def create_source_provider(
    source_provider_data: dict[str, Any],
    ...
) -> Generator[BaseProvider, None, None]:
    ...
    with source_provider(ocp_resource=ocp_resource_provider, **provider_args) as _source_provider:
        if not _source_provider.test:
            pytest.fail(f"{source_provider.type} provider {provider_args['host']} is not available.")
        yield _source_provider
```

---

## Logging Format

Use f-strings for all logging by default. Use parameterized format (`%s`) **only** for expensive operations where lazy evaluation matters.

### Real Examples

```python
LOGGER.info(f"VM {vm_name} failed at step '{step_name}': {step_error}")
LOGGER.info(f"Plan '{plan.name}' migration status: '{sample}'")
LOGGER.warning(f"'{_resource_name=}' is too long ({len(_resource_name)} > 63). Truncating.")
LOGGER.info(f"Storing {_resource.kind} {_resource.name} in fixture store")
LOGGER.info(f"Using Windows credentials for VM: {username}")
```

---

## Function Size and Naming

### Single Responsibility

If you would write "and" in the docstring, split the function. Extract helpers with a `_` prefix for sub-tasks.

From `utilities/mtv_migration.py` — private helpers for sub-tasks:

```python
def _find_migration_for_plan(plan: Plan) -> Migration:
    """Find Migration CR for Plan."""
    ...

def _get_failed_migration_step(plan: Plan, vm_name: str) -> str:
    """Get step where VM migration failed."""
    ...

def _get_all_vms_failed_steps(plan_resource: Plan, vm_names: list[str]) -> dict[str, str | None]:
    """Map VM names to their failed step names."""
    ...
```

### Descriptive Names

Function names must clearly describe **what** they do:

```python
# Good
sanitize_kubernetes_name()
create_plan_resource()
get_storage_migration_map()
get_ssh_credentials_from_provider_config()

# Bad
process()
handle_data()
do_stuff()
```

### Size Guideline

Keep functions under 50 lines when possible. Longer functions need clear section comments.

---

## No Duplicate Code

Extract common logic into shared functions. Create a helper when identical code appears 2+ times.

### Real Example

The `create_and_store_resource()` function in `utilities/resources.py` centralizes all OpenShift resource creation logic — name generation, deployment, conflict handling, and teardown registration — avoiding duplication across dozens of call sites.

Similarly, `get_storage_migration_map()` and `get_network_migration_map()` in `utilities/mtv_migration.py` encapsulate map creation logic used by every test class.

> **Tip:** Don't create abstractions for single-use or merely "similar" code. The 2+ copy-paste threshold is the indicator.

---

## Trust Required Arguments

Don't check if required function arguments exist — Python's function signature guarantees they were passed.

```python
# Wrong - unnecessary check
def compare_labels(expected_labels: dict, actual_labels: dict) -> bool:
    if expected_labels and actual_labels:
        return expected_labels == actual_labels
    return False

# Correct - trust the signature
def compare_labels(expected_labels: dict, actual_labels: dict) -> bool:
    return expected_labels == actual_labels
```

> **Note:** "Trust" means don't check if required arguments were passed. "Validate at Source" means fixtures validate the *value* they produce is valid. These are complementary rules.

---

## Pre-commit and Linting

All code must pass pre-commit checks before committing. Never use `--no-verify`.

### Running Pre-commit

```bash
pre-commit run --all-files
```

### Configured Hooks

| Tool | Version | Purpose |
|------|---------|---------|
| pre-commit-hooks | v6.0.0 | AST validation, merge conflicts, debug statements, trailing whitespace |
| flake8 | 7.3.0 | Mutable default argument detection (`M511`) |
| detect-secrets | v1.5.0 | Secret detection in code |
| ruff | v0.15.4 | Linting (`PLC0415`) and formatting (line-length: 120) |
| gitleaks | v8.30.0 | Git secret scanning |
| mypy | v1.19.1 | Static type checking |
| markdownlint-cli2 | v0.21.0 | Markdown linting with auto-fix |

### Ruff Configuration

From `pyproject.toml`:

```toml
[tool.ruff]
preview = true
line-length = 120
fix = true
output-format = "grouped"

[tool.ruff.lint]
select = ["PLC0415"]
```

---

## Fixture Rules

### Naming

Fixtures use **noun names** — they represent resources, not actions:

```python
# Correct - nouns
source_provider
target_namespace
ocp_admin_client
multus_network_name

# Wrong - verbs
setup_provider
create_namespace
```

### Autouse

Only the `autouse_fixtures` fixture in the root `conftest.py` uses `autouse=True`. All other fixtures must be requested explicitly via parameters or `@pytest.mark.usefixtures()`.

### Request Patterns

| Pattern | When to Use |
|---------|-------------|
| Method parameter | When the test needs the fixture's return value |
| `@pytest.mark.usefixtures()` | For side-effect fixtures (setup/teardown) where the return value isn't needed |

Never request a fixture via both parameter **and** `usefixtures`.

### No Magic Skip

Use `@pytest.mark.skipif` at the test or class level for conditional skipping. Never use `pytest.skip()` inside fixtures.

```python
# Correct
@pytest.mark.skipif(
    not get_value_from_py_config("remote_ocp_cluster"),
    reason="No remote cluster configured"
)
class TestRemoteMigration:
    ...
```
