# Test Markers and Selection

This page covers the pytest markers available in mtv-api-tests, how they control test selection, and how to combine them using `-m` expressions for targeted test runs.

## Registered Markers

All markers are registered in `pytest.ini` with `--strict-markers` enabled, which means only declared markers can be used — any typo or unregistered marker causes an immediate error.

```ini
# pytest.ini
markers =
    tier0: Core functionality tests (smoke tests)
    remote: Remote cluster migration tests
    warm: Warm migration tests
    copyoffload: Copy-offload (XCOPY) tests
    incremental: marks tests as incremental (xfail on previous failure)
    min_mtv_version: mark test to require minimum MTV version (e.g., @pytest.mark.min_mtv_version("2.6.0"))
```

## Category Markers

Category markers classify tests by the migration feature they exercise. Use them with `pytest -m <marker>` to select which tests to run.

### `tier0` — Smoke Tests

Core functionality tests that validate critical migration paths. These are the fastest tests and should be the first tests run against a new environment.

**What it covers:** Basic cold migration, basic warm migration, comprehensive migration scenarios, and post-hook behavior.

```python
# tests/test_mtv_cold_migration.py
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
```

```bash
# Run all smoke tests
uv run pytest -m tier0 -v
```

### `warm` — Warm Migration Tests

Tests for warm (live) migration where VMs remain running during data transfer. Warm migration is only supported for certain source providers, so these test modules use `pytestmark` with `skipif` to skip automatically when the source provider doesn't support it.

```python
# tests/test_mtv_warm_migration.py
pytestmark = [
    pytest.mark.skipif(
        _SOURCE_PROVIDER_TYPE
        in (Provider.ProviderType.OPENSTACK, Provider.ProviderType.OPENSHIFT, Provider.ProviderType.OVA),
        reason=f"{_SOURCE_PROVIDER_TYPE} warm migration is not supported.",
    ),
]
```

Individual test classes within warm migration modules combine the `warm` marker with other markers:

```python
# tests/test_mtv_warm_migration.py — tier0 warm migration
@pytest.mark.tier0
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [
        pytest.param(
            py_config["tests_params"]["test_sanity_warm_mtv_migration"],
        )
    ],
    indirect=True,
    ids=["rhel8"],
)
@pytest.mark.usefixtures("precopy_interval_forkliftcontroller", "cleanup_migrated_vms")
class TestSanityWarmMtvMigration:
```

```bash
# Run all warm migration tests
uv run pytest -m warm -v
```

### `copyoffload` — Copy-Offload (XCOPY) Tests

Tests for copy-offload migration, which uses shared storage arrays (XCOPY) for faster data transfer. Some copy-offload tests are further restricted to specific source providers using `skipif`:

```python
# tests/test_copyoffload_migration.py
@pytest.mark.copyoffload
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_copyoffload_thin_migration"])],
    indirect=True,
    ids=["MTV-559:copyoffload-thin"],
)
@pytest.mark.usefixtures("multus_network_name", "copyoffload_config", "setup_copyoffload_ssh", "cleanup_migrated_vms")
class TestCopyoffloadThinMigration:
```

Some copy-offload tests also carry the `warm` marker when testing warm copy-offload migration:

```python
# tests/test_copyoffload_migration.py
@pytest.mark.copyoffload
@pytest.mark.warm
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_copyoffload_warm_migration"])],
    indirect=True,
    ids=["MTV-577:copyoffload-warm"],
)
@pytest.mark.usefixtures(
    "multus_network_name", "precopy_interval_forkliftcontroller", "copyoffload_config", "cleanup_migrated_vms"
)
class TestCopyoffloadWarmMigration:
```

```bash
# Run all copy-offload tests
uv run pytest -m copyoffload -v
```

### `remote` — Remote Cluster Tests

Tests that migrate VMs to a remote OpenShift cluster (as opposed to the local cluster). These tests use `skipif` with a configuration check to skip when no remote cluster is configured:

```python
# tests/test_mtv_cold_migration.py
@pytest.mark.remote
@pytest.mark.incremental
@pytest.mark.skipif(not get_value_from_py_config("remote_ocp_cluster"), reason="No remote OCP cluster provided")
@pytest.mark.parametrize(
    "class_plan_config",
    [
        pytest.param(
            py_config["tests_params"]["test_cold_remote_ocp"],
        )
    ],
    indirect=True,
    ids=["MTV-79"],
)
@pytest.mark.usefixtures("cleanup_migrated_vms")
class TestColdRemoteMigration:
```

```bash
# Run all remote cluster tests
uv run pytest -m remote -v
```

> **Note:** Remote tests require the `remote_ocp_cluster` configuration to be set. Without it, the tests are automatically skipped regardless of marker selection.

## Behavioral Markers

These markers control how tests execute rather than selecting test categories.

### `incremental` — Sequential Test Dependencies

The `incremental` marker enables sequential test execution within a class. When a test method fails, all subsequent test methods in the same class are marked as `xfail` (expected failure) instead of running and failing with cascading errors.

This marker is used on **every test class** in the project because all tests follow a 5-step sequential pattern (create storagemap → create networkmap → create plan → migrate → check VMs), where each step depends on the previous one succeeding.

**How it works:**

Two pytest hooks in `conftest.py` implement this behavior:

1. **`pytest_runtest_makereport`** — Tracks which test failed:

```python
# conftest.py, line 113
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    setattr(item, "rep_" + rep.when, rep)

    # Incremental test support - track failures for class-based tests
    if "incremental" in item.keywords and rep.when == "call" and rep.failed:
        item.parent._previousfailed = item
```

2. **`pytest_runtest_setup`** — Skips subsequent tests after a failure:

```python
# conftest.py, line 175
def pytest_runtest_setup(item):
    # Incremental test support - xfail if previous test in class failed
    if "incremental" in item.keywords:
        previousfailed = getattr(item.parent, "_previousfailed", None)
        if previousfailed is not None:
            pytest.xfail(f"previous test failed ({previousfailed.name})")
```

**Example output when a test fails in an incremental class:**

```
TestSanityColdMtvMigration::test_create_storagemap     PASSED
TestSanityColdMtvMigration::test_create_networkmap      PASSED
TestSanityColdMtvMigration::test_create_plan            FAILED
TestSanityColdMtvMigration::test_migrate_vms            XFAIL (previous test failed (test_create_plan))
TestSanityColdMtvMigration::test_check_vms              XFAIL (previous test failed (test_create_plan))
```

> **Warning:** The `incremental` marker must be applied at the **class level**, not on individual test methods. It relies on `item.parent` to track failures across sibling tests.

### `min_mtv_version` — Minimum MTV Version Requirement

The `min_mtv_version` marker gates test execution based on the MTV (Migration Toolkit for Virtualization) version installed on the target cluster. Tests marked with a minimum version are skipped when the cluster runs an older MTV version.

**Usage:**

```python
@pytest.mark.usefixtures("mtv_version_checker")
@pytest.mark.min_mtv_version("2.10.0")
@pytest.mark.incremental
class TestFeatureRequiringMtv210:
    """This test requires MTV 2.10.0 or later."""
    ...
```

> **Warning:** Both the `@pytest.mark.min_mtv_version("X.Y.Z")` marker **and** `@pytest.mark.usefixtures("mtv_version_checker")` are required. The marker stores the version string; the fixture reads it and performs the version check.

**Implementation:**

The `mtv_version_checker` fixture in `conftest.py` reads the marker and compares versions:

```python
# conftest.py, line 995
@pytest.fixture(scope="class")
def mtv_version_checker(request: pytest.FixtureRequest, ocp_admin_client: DynamicClient) -> None:
    """Check if test requires minimum MTV version and skip if not met."""
    marker = request.node.get_closest_marker("min_mtv_version")
    if marker:
        min_version = marker.args[0]
        from utilities.utils import has_mtv_minimum_version

        if not has_mtv_minimum_version(min_version, client=ocp_admin_client):
            pytest.skip(f"Test requires MTV {min_version}+")
```

The actual version comparison uses PEP 440 version parsing in `utilities/utils.py`:

```python
# utilities/utils.py, line 719
def has_mtv_minimum_version(min_version: str, client: DynamicClient) -> bool:
    """Check if MTV version meets minimum requirement."""
    try:
        current_version = get_mtv_version(client=client)
        return Version(current_version) >= Version(min_version)
    except (ValueError, InvalidVersion, AttributeError, KeyError, ConnectionError) as e:
        LOGGER.warning(f"Failed to check MTV version: {e}")
        return False
```

> **Note:** If the MTV version cannot be determined (connection error, missing resource, etc.), the function returns `False` and the test is skipped with a warning.

## Combining Markers with `-m` Expressions

Pytest's `-m` flag accepts boolean expressions to combine markers. This allows fine-grained test selection.

### Boolean Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `or` | Run tests matching either marker | `-m "tier0 or warm"` |
| `and` | Run tests matching both markers | `-m "copyoffload and warm"` |
| `not` | Exclude tests with a marker | `-m "not remote"` |
| Parentheses | Group expressions | `-m "tier0 and (not remote)"` |

### Common Selection Patterns

```bash
# Run smoke tests only (fastest feedback)
uv run pytest -m tier0 -v

# Run all warm migration tests
uv run pytest -m warm -v

# Run all copy-offload tests
uv run pytest -m copyoffload -v

# Run smoke OR warm tests
uv run pytest -m "tier0 or warm" -v

# Run only warm copy-offload tests (tests with both markers)
uv run pytest -m "copyoffload and warm" -v

# Run copy-offload tests that are NOT warm
uv run pytest -m "copyoffload and not warm" -v

# Run everything except remote cluster tests
uv run pytest -m "not remote" -v

# Run smoke tests, excluding remote scenarios
uv run pytest -m "tier0 and not remote" -v
```

### Combining `-m` with `-k` for Fine-Grained Selection

The `-k` flag filters by test name (substring or expression), and can be combined with `-m` for precise targeting:

```bash
# Run a specific copy-offload test by name
uv run pytest -m copyoffload -k test_copyoffload_thin_migration

# Run only storagemap creation tests within tier0
uv run pytest -m tier0 -k test_create_storagemap

# Run warm tests but only the sanity class
uv run pytest -m warm -k TestSanityWarmMtvMigration
```

> **Tip:** Use `-m` for category-level selection (what kind of tests) and `-k` for name-level filtering (which specific tests). They work together — both conditions must be satisfied.

## Module-Level Marker with `pytestmark`

Some test modules apply markers to all tests in the file using the `pytestmark` module variable. This is used primarily for `skipif` conditions that apply to an entire migration type:

```python
# tests/test_mtv_warm_migration.py
pytestmark = [
    pytest.mark.skipif(
        _SOURCE_PROVIDER_TYPE
        in (Provider.ProviderType.OPENSTACK, Provider.ProviderType.OPENSHIFT, Provider.ProviderType.OVA),
        reason=f"{_SOURCE_PROVIDER_TYPE} warm migration is not supported.",
    ),
]
```

This ensures all tests in the warm migration modules are skipped when the configured source provider does not support warm migration, regardless of which markers are used for selection.

## Conditional Skipping with `skipif`

While not a custom marker, `@pytest.mark.skipif` is used extensively alongside category markers to conditionally skip tests based on runtime configuration:

| Condition | Pattern | Used In |
|-----------|---------|---------|
| No remote cluster configured | `skipif(not get_value_from_py_config("remote_ocp_cluster"), ...)` | Remote migration tests |
| Provider doesn't support warm | `skipif(_SOURCE_PROVIDER_TYPE in (...), ...)` | Warm migration modules |
| Provider-specific feature | `skipif(_SOURCE_PROVIDER_TYPE != Provider.ProviderType.VSPHERE, ...)` | Copy-offload snapshot tests |

Example combining a category marker with `skipif`:

```python
# tests/test_copyoffload_migration.py
@pytest.mark.copyoffload
@pytest.mark.incremental
@pytest.mark.parametrize(...)
@pytest.mark.skipif(
    _SOURCE_PROVIDER_TYPE != Provider.ProviderType.VSPHERE,
    reason="Snapshots copy-offload test is only applicable to vSphere source providers",
)
@pytest.mark.usefixtures("multus_network_name", "copyoffload_config", "setup_copyoffload_ssh", "cleanup_migrated_vms")
class TestCopyoffloadThinSnapshotsMigration(CopyoffloadSnapshotBase):
```

> **Note:** `skipif` conditions are evaluated at collection time. A test selected by `-m copyoffload` will still be skipped if its `skipif` condition is met. The test appears in output as `SKIPPED` with the reason message.

## Parallel Execution and Markers

Tests run in parallel using `pytest-xdist` with `--dist=loadscope` (configured in `pytest.ini`). This distributes tests by **scope** — all tests within a class go to the same worker. This is essential because the `incremental` marker tracks failures within a class, and the 5-step test pattern requires sequential execution within each class.

```ini
# pytest.ini
addopts =
  --dist=loadscope
```

Markers do not affect parallelization strategy. When you select tests with `-m`, the selected test classes are still distributed across workers by scope.

## Marker Summary

| Marker | Type | Applied To | Purpose |
|--------|------|------------|---------|
| `tier0` | Category | Class | Smoke tests — critical path validation |
| `warm` | Category | Class | Warm (live) migration tests |
| `copyoffload` | Category | Class | Copy-offload (XCOPY) migration tests |
| `remote` | Category | Class | Remote cluster migration tests |
| `incremental` | Behavioral | Class | Xfail remaining tests after a failure |
| `min_mtv_version` | Behavioral | Class | Skip if MTV version is too old |

## Quick Reference

```bash
# Smoke tests (recommended for first validation)
uv run pytest -m tier0 -v

# Warm migration
uv run pytest -m warm -v

# Copy-offload
uv run pytest -m copyoffload -v

# Remote cluster
uv run pytest -m remote -v

# Multiple categories
uv run pytest -m "tier0 or warm" -v

# Exclude a category
uv run pytest -m "not remote" -v

# Dry run — collect without executing
uv run pytest -m tier0 --collect-only

# List all registered markers
uv run pytest --markers
```
