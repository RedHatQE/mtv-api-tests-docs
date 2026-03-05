# Cluster Configuration

This page covers how to configure the target OpenShift cluster for MTV API tests, including cluster connectivity, admin credentials, storage classes, and namespace settings.

## Configuration Mechanism

MTV API tests uses [pytest-testconfig](https://pypi.org/project/pytest-testconfig/) for runtime configuration. Default values are defined in `tests/tests_config/config.py` and can be overridden at invocation time using `--tc` CLI flags.

The `pytest.ini` file registers the config file and format automatically:

```ini
addopts =
  --tc-file=tests/tests_config/config.py
  --tc-format=python
```

All configuration values are accessed at runtime through `py_config` (a dictionary provided by pytest-testconfig) and the helper function `get_value_from_py_config()` in `utilities/utils.py`:

```python
def get_value_from_py_config(value: str) -> Any:
    config_value = py_config.get(value)

    if not config_value:
        return config_value

    if isinstance(config_value, str):
        if config_value.lower() == "true":
            return True

        if config_value.lower() == "false":
            return False

        return config_value

    return config_value
```

This helper automatically converts string values `"true"` and `"false"` to Python booleans, which is important since `--tc` flags always pass values as strings.

## Required Configuration

The test framework validates required configuration at session startup and exits immediately if any are missing:

```python
def pytest_sessionstart(session):
    required_config = ("storage_class", "source_provider")

    missing_configs: list[str] = []
    for _req in required_config:
        if not py_config.get(_req):
            missing_configs.append(_req)

    if missing_configs:
        pytest.exit(
            reason=f"Some required config is missing {required_config=} - {missing_configs=}",
            returncode=1,
        )
```

> **Warning:** Tests will not collect or run if `storage_class` or `source_provider` are not provided. These have no defaults and must be explicitly set via `--tc` flags.

## Cluster Connection

### Authentication Parameters

The target OpenShift cluster connection requires three parameters, all passed via `--tc` flags:

| Parameter | Flag | Description |
|-----------|------|-------------|
| `cluster_host` | `--tc=cluster_host:<url>` | OpenShift API server URL (e.g., `https://api.cluster.example.com:6443`) |
| `cluster_username` | `--tc=cluster_username:<user>` | Cluster admin username (typically `kubeadmin`) |
| `cluster_password` | `--tc=cluster_password:<password>` | Cluster admin password |

### How the Client is Created

The `get_cluster_client()` function in `utilities/utils.py` builds a Kubernetes `DynamicClient` from the configuration:

```python
def get_cluster_client() -> DynamicClient:
    host = get_value_from_py_config("cluster_host")
    username = get_value_from_py_config("cluster_username")
    password = get_value_from_py_config("cluster_password")
    insecure_verify_skip = get_value_from_py_config("insecure_verify_skip")
    _client = get_client(
        host=host, username=username, password=password,
        verify_ssl=not insecure_verify_skip,
    )

    if isinstance(_client, DynamicClient):
        return _client
    raise ValueError("Failed to get client for cluster")
```

The `get_client()` call comes from the `ocp_utilities` library (`openshift-python-utilities`), which handles the OAuth token exchange internally. No kubeconfig file is needed.

### SSL Verification

SSL verification for the cluster API connection is controlled by the `insecure_verify_skip` setting:

```python
insecure_verify_skip: str = "true"  # default in config.py
```

Override at runtime:

```
--tc=insecure_verify_skip:false
```

When set to `"true"` (the default), SSL certificate verification is skipped. Set to `"false"` for production clusters with valid certificates.

### The `ocp_admin_client` Fixture

The cluster client is exposed to all tests as a session-scoped fixture:

```python
@pytest.fixture(scope="session")
def ocp_admin_client():
    LOGGER.info(msg="Creating OCP admin Client")
    _client = get_cluster_client()

    if remote_cluster_name := get_value_from_py_config("remote_ocp_cluster"):
        if remote_cluster_name not in _client.configuration.host:
            raise RemoteClusterAndLocalCluterNamesError(
                "Remote cluster must be the same as local cluster."
            )

    yield _client
```

This fixture is created once per session and shared across all tests and workers. When `remote_ocp_cluster` is configured, it additionally validates that the client is connected to the expected cluster.

> **Tip:** When running tests for a remote cluster migration scenario, set `--tc=remote_ocp_cluster:<cluster-name>` to enable the cluster name validation check.

## Storage Class

### Setting the Storage Class

The `storage_class` parameter specifies which OpenShift StorageClass to use for migration target volumes:

```
--tc=storage_class:ocs-storagecluster-ceph-rbd
```

This is a **required** parameter with no default value. Common choices include:

- `ocs-storagecluster-ceph-rbd` - ODF (OpenShift Data Foundation) block storage
- `ocs-storagecluster-ceph-rbd-virtualization` - ODF block storage optimized for VMs
- `nfs` - NFS-based storage

### NFS Storage Profile Handling

When using NFS storage, the framework automatically patches the `StorageProfile` custom resource with appropriate defaults. This is handled by the `nfs_storage_profile` fixture:

```python
@pytest.fixture(scope="session")
def nfs_storage_profile(ocp_admin_client):
    nfs = StorageClass.Types.NFS
    if py_config["storage_class"] == nfs:
        storage_profile = StorageProfile(
            client=ocp_admin_client, name=nfs, ensure_exists=True,
        )

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

> **Note:** The NFS storage profile patch is applied automatically as part of the `autouse_fixtures` fixture and is reverted when the session ends.

### Storage Class in Test Names

The storage class value is appended to each test name for traceability in reports:

```python
def pytest_collection_modifyitems(session, config, items):
    for item in items:
        item.name = f"{item.name}-{py_config.get('source_provider')}-{py_config.get('storage_class')}"
```

This ensures test results clearly indicate which storage class was used, which is especially helpful when comparing runs across different storage backends.

## Namespace Configuration

### Target Namespace

The migration target namespace is auto-generated per session to support parallel execution and prevent conflicts between test runs.

#### Namespace Prefix

The prefix for namespace names is controlled by:

```python
target_namespace_prefix: str = "auto"  # default in config.py
```

Override at runtime:

```
--tc=target_namespace_prefix:my-prefix
```

#### How Namespace Names are Generated

The `target_namespace` fixture combines a UUID-based session identifier with the prefix:

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

The resulting namespace name follows the pattern `auto-<uuid><suffix>`, truncated to 63 characters (the Kubernetes maximum). For example: `auto-a1b2c3d4`.

#### Security Labels

Every target namespace is created with three labels:

| Label | Value | Purpose |
|-------|-------|---------|
| `pod-security.kubernetes.io/enforce` | `restricted` | Enforces the restricted Pod Security Standard |
| `pod-security.kubernetes.io/enforce-version` | `latest` | Uses the latest enforcement version |
| `mutatevirtualmachines.kubemacpool.io` | `ignore` | Disables KubeMacPool MAC address mutation for VMs |

### MTV Operator Namespace

The namespace where the MTV (Migration Toolkit for Virtualization) operator is installed:

```python
mtv_namespace: str = "openshift-mtv"  # default in config.py
```

Override if the operator was installed in a custom namespace:

```
--tc=mtv_namespace:custom-mtv-namespace
```

The `forklift_pods_state` fixture validates at session startup that the forklift controller pod is running in this namespace:

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
```

> **Warning:** If the forklift controller pod is not found or any forklift pods are not running, the session will fail before any tests execute.

## Full Configuration Reference

### All Cluster-Related `--tc` Flags

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--tc=cluster_host:<url>` | Yes | *(none)* | OpenShift API server URL |
| `--tc=cluster_username:<user>` | Yes | *(none)* | Cluster admin username |
| `--tc=cluster_password:<pass>` | Yes | *(none)* | Cluster admin password |
| `--tc=insecure_verify_skip:<bool>` | No | `"true"` | Skip SSL certificate verification |
| `--tc=storage_class:<name>` | Yes | *(none)* | Target StorageClass for migration volumes |
| `--tc=source_provider:<name>` | Yes | *(none)* | Source provider key from `.providers.json` |
| `--tc=target_namespace_prefix:<prefix>` | No | `"auto"` | Prefix for the auto-generated target namespace |
| `--tc=mtv_namespace:<namespace>` | No | `"openshift-mtv"` | Namespace where MTV operator is installed |
| `--tc=remote_ocp_cluster:<name>` | No | `""` | Remote cluster name for cross-cluster validation |
| `--tc=plan_wait_timeout:<seconds>` | No | `3600` | Timeout in seconds for migration plan completion |

### Minimal Example

```bash
uv run pytest -m tier0 \
  --tc=cluster_host:https://api.my-cluster.example.com:6443 \
  --tc=cluster_username:kubeadmin \
  --tc=cluster_password:my-password \
  --tc=source_provider:vsphere-8.0.1 \
  --tc=storage_class:ocs-storagecluster-ceph-rbd
```

### Container Execution Example

When running from the container image, pass credentials via environment variables to avoid shell history exposure:

```bash
podman run --rm \
  -v $(pwd)/.providers.json:/app/.providers.json:ro \
  -e CLUSTER_PASSWORD \
  ghcr.io/redhatqe/mtv-api-tests:latest \
  uv run pytest -m tier0 -v \
    --tc=cluster_host:https://api.my-cluster.example.com:6443 \
    --tc=cluster_username:kubeadmin \
    --tc=cluster_password:${CLUSTER_PASSWORD} \
    --tc=source_provider:vsphere-8.0.1 \
    --tc=storage_class:ocs-storagecluster-ceph-rbd
```

> **Tip:** Set `CLUSTER_PASSWORD` as an environment variable before running the container (`export CLUSTER_PASSWORD=...`) to prevent the password from appearing in your shell history.

## Resource Teardown

By default, all resources created during the session (namespaces, VMs, plans, maps) are cleaned up when the session ends. To preserve resources for debugging:

```
--skip-teardown
```

This flag prevents the framework from deleting any created resources, including the target namespace and migrated VMs. This is useful for post-test inspection but requires manual cleanup afterward.

> **Warning:** Using `--skip-teardown` will leave resources on the cluster. Clean them up manually before running tests again to avoid conflicts.
