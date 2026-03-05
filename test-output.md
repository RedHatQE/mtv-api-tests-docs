# Test Output and Reporting

This page covers the test output mechanisms, diagnostic collection, resource tracking, and optional AI-powered failure analysis available in the MTV API Tests framework.

## JUnit XML Reports

### Default Configuration

JUnit XML reporting is enabled by default via `pytest.ini`:

```ini
[pytest]
addopts =
  --junit-xml=junit-report.xml
  ...

junit_logging = all
```

Every test run produces a `junit-report.xml` file in the working directory. The `junit_logging = all` setting captures all log output (DEBUG, INFO, WARNING, ERROR, CRITICAL) inside the XML report, providing full diagnostic context alongside test results.

### What the Report Contains

Each JUnit XML report includes:

- **Test results** — pass, fail, skip, and xfail outcomes for every collected test
- **Execution times** — per-test and aggregate timing
- **Error messages and stack traces** — full tracebacks for failed tests
- **All log output** — captured via `junit_logging = all`
- **Test naming context** — each test name is suffixed with the source provider and storage class (e.g., `test_create_plan-vsphere-8.0.1-ocs-storagecluster-ceph-rbd`)

Test name enrichment happens in the `pytest_collection_modifyitems` hook:

```python
def pytest_collection_modifyitems(session, config, items):
    for item in items:
        item.name = f"{item.name}-{py_config.get('source_provider')}-{py_config.get('storage_class')}"
```

### Containerized Execution

When running inside the container image, the Dockerfile defines an output directory:

```dockerfile
ENV JUNITFILE=${APP_DIR}/output/
RUN mkdir -p ${APP_DIR}/output
```

The JUnit XML file is generated at `/app/junit-report.xml` (from `pytest.ini` defaults). To retrieve it from a container or OpenShift pod:

```bash
# From a local container
podman cp <container_id>:/app/junit-report.xml ./junit-report.xml

# From an OpenShift pod
oc cp <pod>:/app/junit-report.xml ./junit-report.xml
```

### Overriding the Output Path

Override the default output path with the `--junit-xml` flag:

```bash
uv run pytest -m tier0 --junit-xml=/custom/path/results.xml
```

### Incremental Test Support

Tests marked with `@pytest.mark.incremental` track failures across a test class. When a test fails, subsequent tests in the same class are marked as `xfail` with a reference to the failed test. This is reflected in the JUnit XML output:

```python
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    setattr(item, "rep_" + rep.when, rep)

    if "incremental" in item.keywords and rep.when == "call" and rep.failed:
        item.parent._previousfailed = item
```

When a subsequent test is set up and detects a prior failure, it is automatically skipped:

```python
def pytest_runtest_setup(item):
    if "incremental" in item.keywords:
        previousfailed = getattr(item.parent, "_previousfailed", None)
        if previousfailed is not None:
            pytest.xfail(f"previous test failed ({previousfailed.name})")
```

### Console Status Reporting

Alongside the XML report, the framework provides colored console output with visual separators for each test phase (SETUP, CALL, TEARDOWN):

```python
def pytest_report_teststatus(report, config):
    test_name = report.head_line
    if report.passed:
        if when == "call":
            BASIC_LOGGER.info(f"\nTEST: {test_name} STATUS: \033[0;32mPASSED\033[0m")
    elif report.skipped:
        BASIC_LOGGER.info(f"\nTEST: {test_name} STATUS: \033[1;33mSKIPPED\033[0m")
    elif report.failed:
        if when != "call":
            BASIC_LOGGER.info(f"\nTEST: {test_name} [{when}] STATUS: \033[0;31mERROR\033[0m")
        else:
            BASIC_LOGGER.info(f"\nTEST: {test_name} STATUS: \033[0;31mFAILED\033[0m")
```

### Log File

All log output is also written to a rotating log file via a `QueueHandler`/`QueueListener` architecture:

- **Default path:** `pytest-tests.log`
- **Max file size:** 100 MB per file
- **Backup count:** 20 rotated files
- **Colored output:** Console handler includes color-coded log levels (cyan for DEBUG, green for INFO, yellow for WARNING, red for ERROR)

Override the log file path with `--log-file`:

```bash
uv run pytest -m tier0 --log-file=/var/log/mtv-tests.log
```

---

## Must-Gather Collection on Failures

The framework automatically collects MTV diagnostic data using `oc adm must-gather` when tests fail. This is implemented in `utilities/must_gather.py` and triggered by pytest hooks in `conftest.py`.

### When Must-Gather Runs

Must-gather is triggered in two scenarios:

1. **On test exception** — The `pytest_exception_interact` hook runs when any test raises an exception during setup, call, or teardown. It collects must-gather data scoped to the specific migration plan associated with the failing test.

2. **On session teardown failure** — If the session-level resource cleanup (`session_teardown`) fails, must-gather runs to capture the full cluster state.

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

### How the Must-Gather Image Is Resolved

Rather than hardcoding a must-gather image, the framework dynamically resolves it from the installed MTV operator:

1. Looks up the MTV `Subscription` resource to get the operator channel (e.g., `release-v2.11`)
2. Reads the installed `ClusterServiceVersion` (CSV) to extract the `MUST_GATHER_IMAGE` environment variable and its SHA digest
3. Derives the `ImageDigestMirrorSet` (IDMS) name from the channel (e.g., `release-v2.11` becomes `devel-testing-for-2-11`)
4. Queries the IDMS resource for the mirror URL (preferring `quay` mirrors)
5. Combines the mirror URL with the SHA: `{mirror_url}@{sha}`

```python
def _resolve_must_gather_image(
    ocp_admin_client: DynamicClient,
    mtv_subs: Subscription,
    mtv_csv: ClusterServiceVersion,
) -> str:
    csv_image = _get_csv_must_gather_image(mtv_csv=mtv_csv)
    channel = mtv_subs.instance.spec.channel
    idms_name = _get_idms_name(channel=channel)

    idms = ImageDigestMirrorSet(client=ocp_admin_client, name=idms_name, ensure_exists=True)
    must_gather_mirror_url = _get_must_gather_mirror_url(idms=idms)

    sha = csv_image.split("@")[1]
    resolved_image = f"{must_gather_mirror_url}@{sha}"
    return resolved_image
```

### Scoped vs. Full Collection

The must-gather command runs in two modes depending on whether a migration plan is associated with the failure:

**Plan-scoped (targeted)** — collects diagnostics for a specific migration plan:

```bash
oc adm must-gather --image=<resolved-image> --dest-dir=<path> \
  -- NS=<plan_namespace> PLAN=<plan_name> /usr/bin/targeted
```

**Namespace-scoped (full)** — collects all MTV namespace data when no specific plan is available:

```bash
oc adm must-gather --image=<resolved-image> --dest-dir=<path> \
  -- -- NS=<mtv_namespace>
```

> **Note:** Must-gather errors are logged but never fail the test run. If the image resolution or `oc adm must-gather` command fails, the exception is captured and the test session continues.

### Disabling Must-Gather

To skip all data collection including must-gather:

```bash
uv run pytest -m tier0 --skip-data-collector
```

---

## Data-Collector Resource Tracking

The data-collector system tracks every OpenShift resource created during a test session and writes a manifest to disk. This enables post-run diagnostics, manual cleanup, and automated resource recovery.

### Command-Line Options

| Option | Default | Description |
|---|---|---|
| `--skip-data-collector` | `False` | Disable all resource tracking and must-gather collection |
| `--data-collector-path` | `.data-collector` | Directory for collected data output |

### How Resources Are Tracked

Every OpenShift resource created through `create_and_store_resource()` is recorded in the session's `fixture_store["teardown"]` dictionary:

```python
def create_and_store_resource(
    client: "DynamicClient",
    fixture_store: dict[str, Any],
    resource: type[Resource],
    test_name: str | None = None,
    **kwargs: Any,
) -> Any:
    # ... resource creation logic ...

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

### Output Structure

At the end of a test session, the `pytest_sessionfinish` hook writes all tracked resources to `resources.json`:

```python
def collect_created_resources(session_store: dict[str, Any], data_collector_path: Path) -> None:
    resources = session_store["teardown"]
    if resources:
        with open(data_collector_path / "resources.json", "w") as fd:
            json.dump(session_store["teardown"], fd)
```

The resulting directory layout:

```
.data-collector/
├── resources.json                          # All tracked resources
├── test_create_plan-vsphere-8.0.1-ceph/    # Must-gather for failed test
│   └── ... (oc adm must-gather output)
├── test_migrate_vms-vsphere-8.0.1-ceph/    # Must-gather for another failure
│   └── ...
└── ...
```

### resources.json Format

The `resources.json` file maps resource kinds to lists of resource metadata:

```json
{
  "Namespace": [
    {"name": "auto-abc123-ns", "namespace": null, "module": "ocp_resources.namespace"}
  ],
  "Secret": [
    {"name": "auto-abc123-secret", "namespace": "openshift-mtv", "module": "ocp_resources.secret"}
  ],
  "Plan": [
    {"name": "auto-abc123-cold", "namespace": "openshift-mtv", "module": "ocp_resources.plan", "test_name": "test_create_plan"}
  ],
  "StorageMap": [
    {"name": "auto-abc123-map", "namespace": "openshift-mtv", "module": "ocp_resources.storage_map"}
  ]
}
```

Each entry includes the `module` field, which allows the cleanup tool to dynamically import the correct resource class.

### Manual Resource Cleanup

If a test run is interrupted or teardown is skipped (via `--skip-teardown`), leftover resources can be cleaned up using the `resources.json` manifest:

```bash
uv run tools/clean_cluster.py .data-collector/resources.json
```

The cleanup tool dynamically loads each resource class from its module and calls `clean_up()`:

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

> **Tip:** Use `--skip-teardown` during debugging to preserve resources for inspection, then run the cleanup tool when done:
> ```bash
> uv run pytest -m tier0 --skip-teardown
> # ... inspect cluster state ...
> uv run tools/clean_cluster.py .data-collector/resources.json
> ```

### Session Teardown

When `--skip-teardown` is **not** set, the framework performs automatic cleanup in `pytest_sessionfinish`. The teardown sequence is:

1. **Cancel all Migrations** — active migrations are cancelled
2. **Archive all Plans** — plans are archived before deletion
3. **Delete resources in order** — Migrations, Plans, Providers, Hosts, Secrets, NetworkAttachmentDefinitions, StorageMaps, NetworkMaps, VirtualMachines, Pods
4. **Clean up cloned VMs** — VMware, OpenStack, and RHV cloned VMs are deleted from source providers
5. **Namespace cleanup** — target namespaces are deleted last
6. **Leftover detection** — any resources that failed to delete are reported, and `SessionTeardownError` is raised

If teardown fails, must-gather is automatically triggered to capture diagnostics.

---

## AI-Powered Failure Analysis with Jenkins Job Insight

The framework integrates with [Jenkins Job Insight (JJI)](https://github.com/myk-org/jenkins-job-insight) to provide AI-powered analysis of test failures. When enabled, the JUnit XML report is enriched with failure classifications, root cause analysis, and suggested fixes.

### Prerequisites

- A running JJI server instance
- Network connectivity from the test runner to the JJI server
- The `JJI_SERVER_URL` environment variable set

### Enabling AI Analysis

Add the `--analyze-with-ai` flag to your pytest invocation:

```bash
uv run pytest -m tier0 --analyze-with-ai \
  --tc=source_provider:vsphere-8.0.1 \
  --tc=storage_class:ocs-storagecluster-ceph-rbd
```

### Configuration

Configure the AI analysis via environment variables or a `.env` file:

| Variable | Required | Default | Description |
|---|---|---|---|
| `JJI_SERVER_URL` | Yes | — | URL of the Jenkins Job Insight server |
| `JJI_AI_PROVIDER` | No | `claude` | AI provider to use for analysis |
| `JJI_AI_MODEL` | No | `claude-opus-4-6[1m]` | Specific AI model identifier |
| `JJI_TIMEOUT` | No | `600` | Request timeout in seconds |

Example `.env` file:

```env
JJI_SERVER_URL=http://jji-server.example.com
JJI_AI_PROVIDER=claude
JJI_AI_MODEL=claude-opus-4-6[1m]
JJI_TIMEOUT=600
```

> **Note:** Environment variables can be set directly or via a `.env` file in the project root. The framework uses `python-dotenv` to load `.env` automatically during setup.

### How It Works

The AI analysis pipeline has two phases:

#### 1. Setup Phase (`pytest_sessionstart`)

During session initialization, `setup_ai_analysis()` validates the configuration:

```python
def setup_ai_analysis(session: pytest.Session) -> None:
    if is_dry_run(session.config):
        session.config.option.analyze_with_ai = False
        return

    load_dotenv()

    if not os.environ.get("JJI_SERVER_URL"):
        LOGGER.warning("JJI_SERVER_URL is not set. Analyze with AI features will be disabled.")
        session.config.option.analyze_with_ai = False
    else:
        if not os.environ.get("JJI_AI_PROVIDER"):
            os.environ["JJI_AI_PROVIDER"] = "claude"
        if not os.environ.get("JJI_AI_MODEL"):
            os.environ["JJI_AI_MODEL"] = "claude-opus-4-6[1m]"
```

- Automatically disabled during dry runs (`--collectonly`, `--setupplan`)
- Disabled with a warning if `JJI_SERVER_URL` is not set
- Sets default AI provider and model if not explicitly configured

#### 2. Enrichment Phase (`pytest_sessionfinish`)

After all tests complete, if failures were detected (exit status != 0), the JUnit XML is sent to the JJI server:

```python
if session.config.getoption("analyze_with_ai"):
    if exitstatus == 0:
        LOGGER.info("No test failures (exit code %d), skipping AI analysis", exitstatus)
    else:
        try:
            enrich_junit_xml(session)
        except Exception:
            LOGGER.exception("Failed to enrich JUnit XML, original preserved")
```

The `enrich_junit_xml()` function reads the generated JUnit XML and POSTs it to the JJI server:

```python
def enrich_junit_xml(session: pytest.Session) -> None:
    xml_path = Path(getattr(session.config.option, "xmlpath", None))
    raw_xml = xml_path.read_text()

    timeout_value = int(os.environ.get("JJI_TIMEOUT", "600"))

    response = requests.post(
        f"{server_url.rstrip('/')}/analyze-failures",
        json={
            "raw_xml": raw_xml,
            "ai_provider": ai_provider,
            "ai_model": ai_model,
        },
        timeout=timeout_value,
    )
    response.raise_for_status()
    result = response.json()

    if enriched_xml := result.get("enriched_xml"):
        xml_path.write_text(enriched_xml)
```

The enriched XML replaces the original file in-place. If the enrichment request fails for any reason, the original JUnit XML is preserved.

### What the Enriched Report Contains

The JJI server analyzes each test failure and adds properties to the JUnit XML including:

- **Failure classification** — categorization of the failure type
- **Root cause analysis** — AI-generated explanation of what went wrong
- **Suggested code fixes** — recommended changes to resolve the failure
- **Bug report information** — structured data for filing issues
- **Human-readable summaries** — plain-language descriptions of failures

### Error Handling

The AI analysis feature is designed to be non-disruptive:

- If `JJI_SERVER_URL` is not set, the feature is silently disabled with a log warning
- If all tests pass (exit code 0), analysis is skipped
- If the JJI server is unreachable or returns an error, the original JUnit XML is preserved
- If the timeout is exceeded (default: 600 seconds), the request fails gracefully
- Any exception during enrichment is logged but does not affect the test exit code

> **Warning:** The AI analysis timeout defaults to 600 seconds (10 minutes). For large test suites with many failures, you may need to increase `JJI_TIMEOUT` to allow the server sufficient time to analyze all failures.

---

## Parallel Execution and Reporting

When tests run in parallel via `pytest-xdist` (`--dist=loadscope`), each worker maintains its own `fixture_store`. The `pytest-harvest` plugin handles merging results across workers using a pickle-based file exchange mechanism:

```
.xdist_results/
├── gw0.pkl    # Worker 0 fixture store and session items
├── gw1.pkl    # Worker 1 fixture store and session items
└── ...
```

Workers dump their results during teardown, and the controller process loads and merges them. The temporary pickle files are cleaned up after loading. The final JUnit XML report consolidates results from all workers into a single file.

> **Note:** Parallel safety is ensured by unique namespaces per session (via `session_uuid`), isolated `fixture_store` per worker, and unique resource names generated by `create_and_store_resource()`.

---

## Complete Data Flow

The following diagram shows how test output and reporting flows through the framework:

```
Session Start
│
├─ Validate config (storage_class, source_provider)
├─ Initialize fixture_store["teardown"] = {}
├─ Prepare .data-collector/ directory
├─ Setup logging (QueueHandler → console + rotating file)
└─ Setup AI analysis (if --analyze-with-ai)

Test Execution
│
├─ Each test phase logged with separators (SETUP → CALL → TEARDOWN)
├─ Test status reported with color coding (green/yellow/red)
├─ Resources tracked in fixture_store["teardown"]
│
└─ On test failure:
   └─ pytest_exception_interact triggers
      └─ run_must_gather() → .data-collector/<test_name>/

Session Finish
│
├─ Write resources.json to .data-collector/
├─ Run session_teardown() (delete all tracked resources)
│  └─ On teardown failure: run_must_gather() → .data-collector/
├─ Generate junit-report.xml (pytest built-in)
│
└─ If --analyze-with-ai AND failures exist:
   └─ enrich_junit_xml()
      ├─ POST raw XML → JJI /analyze-failures
      └─ Write enriched XML back to junit-report.xml
```

### Output Files Summary

| File | Description |
|---|---|
| `junit-report.xml` | JUnit XML test results (optionally enriched with AI analysis) |
| `pytest-tests.log` | Rotating log file with full test output (100 MB max, 20 backups) |
| `.data-collector/resources.json` | Manifest of all OpenShift resources created during the session |
| `.data-collector/<test_name>/` | Must-gather diagnostic output for each failed test |
| `.xdist_results/*.pkl` | Temporary worker result files (parallel execution only, auto-cleaned) |
