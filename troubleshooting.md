# Troubleshooting

This page covers common issues encountered when running or developing mtv-api-tests, along with their causes and solutions.

---

## Provider Connection Failures

### Missing `.providers.json` File

**Symptom:** Test session exits immediately with `MissingProvidersFileError`.

```
MissingProvidersFileError: '.providers.json' file is missing or empty
```

**Cause:** The `.providers.json` file is either missing from the project root or contains no data. The file is loaded by `utilities/utils.py:load_source_providers()` and validated by the `source_provider_data` fixture.

**Solution:** Create `.providers.json` in the project root using the example file as a template:

```bash
cp .providers.json.example .providers.json
# Edit with your provider credentials
```

Each provider entry requires at minimum:

```json
{
  "vsphere": {
    "type": "vsphere",
    "version": "8.0",
    "fqdn": "vcenter.example.com",
    "api_url": "vcenter.example.com/sdk",
    "username": "admin@vsphere.local",
    "password": "your-password"
  }
}
```

> **Warning:** Never commit `.providers.json` to version control. It contains sensitive credentials.

### Provider Not Found in Configuration

**Symptom:** `ValueError` during fixture setup referencing the `source_provider` config key.

```
ValueError: Source provider 'my-vsphere' not found in '.providers.json'.
Available providers: ['ovirt', 'vsphere']
```

**Cause:** The `source_provider` value in `tests/tests_config/config.py` does not match any key in `.providers.json`. The `source_provider_data` fixture in `conftest.py` performs this validation:

```python
requested_provider = py_config["source_provider"]
if requested_provider not in source_providers:
    raise ValueError(
        f"Source provider '{requested_provider}' not found in '.providers.json'. "
        f"Available providers: {sorted(source_providers.keys())}"
    )
```

**Solution:** Ensure the `source_provider` value in `tests/tests_config/config.py` exactly matches a key in your `.providers.json` file.

### Remote Cluster Name Mismatch

**Symptom:** `RemoteClusterAndLocalCluterNamesError` during `ocp_admin_client` fixture setup.

```
RemoteClusterAndLocalCluterNamesError: Remote cluster must be the same as local cluster.
```

**Cause:** The `remote_ocp_cluster` config value does not match the cluster your kubeconfig points to. The `ocp_admin_client` fixture validates this:

```python
if remote_cluster_name := get_value_from_py_config("remote_ocp_cluster"):
    if remote_cluster_name not in _client.configuration.host:
        raise RemoteClusterAndLocalCluterNamesError(
            "Remote cluster must be the same as local cluster."
        )
```

**Solution:** Verify your kubeconfig context matches the configured `remote_ocp_cluster`:

```bash
oc whoami --show-server
```

### Certificate Download Failures

**Symptom:** `ValueError` when the framework cannot retrieve a valid TLS certificate from the provider.

```
ValueError: Failed to download valid certificate from vcenter.example.com
```

**Cause:** The `generate_ca_cert_file()` function in `utilities/utils.py` uses `openssl s_client` to download the provider's certificate. If the provider is unreachable or returns an invalid certificate, this validation fails:

```python
cert = check_output([
    "/bin/sh", "-c",
    f"openssl s_client -connect {provider_fqdn}:443 -showcerts < /dev/null",
], stderr=STDOUT)

if b"BEGIN CERTIFICATE" not in cert:
    raise ValueError(f"Failed to download valid certificate from {provider_fqdn}")
```

**Solution:**

1. Verify the provider FQDN is reachable: `openssl s_client -connect <fqdn>:443`
2. Check DNS resolution and network connectivity
3. If using insecure connections, set `source_provider_insecure_skip_verify` to `"true"` in your test config

---

## Forklift Pod State Errors

### Forklift Controller Pod Not Found

**Symptom:** `ForkliftPodsNotRunningError` during session startup.

```
ForkliftPodsNotRunningError: Forklift controller pod not found
```

**Cause:** The `forklift_pods_state` fixture checks that all Forklift pods are healthy in the MTV namespace before tests run. This is an autouse session fixture, so it runs before any test. The check polls for 5 minutes with 1-second intervals:

```python
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
```

**Solution:**

1. Verify the MTV operator is installed:
   ```bash
   oc get pods -n openshift-mtv | grep forklift
   ```
2. Check that `mtv_namespace` in `tests/tests_config/config.py` matches your installation (default: `openshift-mtv`)
3. Wait for pods to become ready or investigate pod events:
   ```bash
   oc describe pod -n openshift-mtv -l app=forklift-controller
   ```

### Forklift Pods Not Running

**Symptom:** `ForkliftPodsNotRunningError` listing specific pods.

```
ForkliftPodsNotRunningError: Some of the forklift pods are not running:
['forklift-validation-xyz-abc', 'forklift-api-xyz-def']
```

**Cause:** One or more `forklift-*` pods in the MTV namespace are not in `RUNNING` or `SUCCEEDED` status. The fixture retries for 5 minutes, tolerating both `ForkliftPodsNotRunningError` and `NotFoundError` during polling:

```python
for sample in TimeoutSampler(
    func=_get_not_running_pods,
    _admin_client=ocp_admin_client,
    sleep=1,
    wait_timeout=60 * 5,
    exceptions_dict={ForkliftPodsNotRunningError: [], NotFoundError: []},
):
    if sample:
        return
```

**Solution:**

1. Check pod logs for errors:
   ```bash
   oc logs -n openshift-mtv <pod-name>
   ```
2. Look for resource constraints (OOM, CPU limits):
   ```bash
   oc describe pod -n openshift-mtv <pod-name> | grep -A5 "State:"
   ```
3. If pods are in `CrashLoopBackOff`, check the ForkliftController CR status:
   ```bash
   oc get forkliftcontroller -n openshift-mtv -o yaml
   ```

### ForkliftController Not Ready

**Symptom:** `TimeoutExpiredError` when the `precopy_interval_forkliftcontroller` fixture waits for the ForkliftController condition.

**Cause:** The fixture patches the ForkliftController with a custom `controller_precopy_interval` and waits for it to reach `SUCCESSFUL` status:

```python
forklift_controller.wait_for_condition(
    status=forklift_controller.Condition.Status.TRUE,
    condition=forklift_controller.Condition.Type.SUCCESSFUL,
    timeout=300,
)
```

**Solution:**

1. Check the ForkliftController status:
   ```bash
   oc get forkliftcontroller forklift-controller -n openshift-mtv -o yaml
   ```
2. Verify the controller pod is restarting after the configuration change

---

## VM Clone Failures

### Generic Clone Error

**Symptom:** `VmCloneError` during the `prepared_plan` fixture setup.

**Cause:** VM cloning occurs during test preparation when `"clone": True` is set in the VM config. Multiple conditions can trigger clone failures depending on the provider.

### VMware-Specific Clone Failures

**SCSI controller not available:**
```
VmCloneError: No SCSI controller found on VM 'vm-name'
```

**No available unit on controller:**
```
VmCloneError: No available unit number on SCSI controller for VM 'vm-name'
```

**Datastore not found or misconfigured:**
```
VmCloneError: Datastore 'datastore-12345' not found in vSphere inventory
```

**SSH key secret missing (copy-offload):**
```
VmCloneError: SSH public key secret not found for ESXi cloning
```

**Solution:**
1. Verify the source VM exists and is accessible in vSphere
2. Check that the target datastore has sufficient space
3. Confirm vSphere credentials have clone permissions
4. For copy-offload cloning, verify ESXi SSH configuration in `.providers.json`:
   ```json
   "copyoffload": {
     "esxi_clone_method": "ssh",
     "esxi_host": "your-esxi-host.example.com",
     "esxi_user": "root",
     "esxi_password": "your-password"
   }
   ```

### RHV/oVirt Clone Failures

**Datacenter not found:**
```
OvirtMTVDatacenterNotFoundError: Datacenter not found for RHV provider
```

**Datacenter status error:**
```
OvirtMTVDatacenterStatusError: Datacenter status is not UP
```

**Solution:** Verify the RHV datacenter is accessible and in `UP` status via the oVirt admin portal.

### Invalid VM Name After Clone

**Symptom:** `InvalidVMNameError` during VM name sanitization.

```
InvalidVMNameError: VM name '!!!invalid!!!' cannot be sanitized to a valid DNS-1123 name.
The name must contain at least one alphanumeric character.
```

**Cause:** The `sanitize_kubernetes_name()` function in `utilities/naming.py` converts VM names to DNS-1123 compliant names. If the source VM name contains no alphanumeric characters, sanitization fails:

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

**Solution:** Ensure source VM names contain at least one alphanumeric character. Valid examples: `rhel9-template`, `win2022-server`. Invalid examples: `---`, `___`.

### VM Not Found in Source Provider

**Symptom:** `VmNotFoundError` when the provider cannot locate the VM.

**Cause:** The VM specified in `tests/tests_config/config.py` does not exist in the source provider.

**Solution:** Verify the VM name matches exactly in the source provider (vSphere, RHV, or OpenStack admin console).

---

## Inventory Sync Timeouts

### Provider Not Appearing in Inventory

**Symptom:** `TimeoutExpiredError` when the Forklift inventory cannot find the provider.

```
TimeoutExpiredError: Timed out waiting for provider <name> to appear in inventory.
```

**Cause:** After a Provider CR is created, the Forklift inventory service needs to sync. The `_provider_id` property polls the inventory API for 180 seconds:

```python
@property
def _provider_id(self) -> str:
    try:
        for sample in TimeoutSampler(
            wait_timeout=180,
            sleep=5,
            func=lambda: [
                _provider["id"]
                for _provider in self._request(url_path=self.provider_type)
                if _provider["name"] == self.provider_name
            ],
        ):
            if sample:
                return sample[0]
    except TimeoutExpiredError:
        LOGGER.error(
            f"Timed out waiting for provider {self.provider_name} to appear in inventory."
        )
        raise
```

**Solution:**

1. Check that the Provider CR is in `Ready` status:
   ```bash
   oc get providers -n <target-namespace> -o wide
   ```
2. Check the Forklift inventory route is accessible:
   ```bash
   oc get route forklift-inventory -n openshift-mtv
   ```
3. Verify provider credentials and connectivity from the cluster

### VM Not Appearing in Inventory After Clone

**Symptom:** `TimeoutExpiredError` after VM is cloned but doesn't appear in inventory.

```
TimeoutExpiredError: VM 'auto-abcd-rhel9-template-efgh' did not appear in
Forklift inventory after 300s. Available VMs: ['vm1', 'vm2', ...]
```

**Cause:** After cloning, the `prepared_plan` fixture calls `wait_for_vm()` which polls for 300 seconds. The Forklift inventory may not have synced the newly cloned VM:

```python
# From prepared_plan fixture
if source_provider.type != Provider.ProviderType.OVA:
    source_provider_inventory.wait_for_vm(name=vm["name"], timeout=300)
```

**Solution:**

1. Verify the VM was actually cloned in the source provider
2. Check the Forklift inventory sync frequency — the controller may need time to detect new VMs
3. Verify the provider connection is healthy:
   ```bash
   oc get providers -n <namespace> -o jsonpath='{.items[*].status.conditions}'
   ```

### OpenStack Volume/Network Sync Timeout

**Symptom:** `TimeoutExpiredError` indicating volumes or networks are not synced.

```
TimeoutExpiredError: VM 'my-vm' found in Forklift inventory but attached volumes
or networks did not sync after 300s. Attached volumes: [...], VM addresses: {...}
```

**Cause:** OpenStack VMs have additional sync requirements beyond basic VM presence. Volumes and networks are synced separately. The `wait_for_vm()` method performs extra checks for OpenStack providers:

```python
# For OpenStack, verify volumes and networks are synced
if self.provider_type == Provider.ProviderType.OPENSTACK:
    if not (
        self._check_openstack_volumes_synced(vm, name)
        and self._check_openstack_networks_synced(vm, name)
    ):
        return None
```

**Solution:**

1. Verify volumes are properly attached to the VM in OpenStack
2. Check that networks exist and are active in the OpenStack project
3. Look for sync-related warnings in logs:
   ```
   VM 'my-vm' found but volume '<id>' not yet queryable in inventory
   VM 'my-vm' found but networks {'net1'} not yet in inventory
   ```
4. Increase the timeout if working with large environments

### Storage/Network Mappings Not Found

**Symptom:** `ValueError` when the inventory cannot resolve storage or network mappings.

```
ValueError: Storages not found for VMs ['vm1'] on provider vsphere
ValueError: Networks not found for vms ['vm1'] on provider ovirt
```

**Cause:** The `vms_storages_mappings()` or `vms_networks_mappings()` methods on provider-specific inventory classes could not find the corresponding resources.

**Solution:**

1. Verify the VM has disks/network interfaces configured in the source provider
2. Check that the source provider datastores/networks are visible in the Forklift inventory
3. For RHV: verify NIC profiles and storage domains exist
4. For VMware: verify datastores are mounted and accessible
5. For OpenStack: verify volume types and networks exist in the project

---

## Resource Name Conflicts

### Kubernetes Name Length Limit

**Symptom:** Warning in logs about name truncation.

```
WARNING: '_resource_name=auto-abcd-source-vsphere-8-0-efgh-cold' is too long
(67 > 63). Truncating.
```

**Cause:** Kubernetes limits resource names to 63 characters (DNS-1123 standard). The `create_and_store_resource()` function in `utilities/resources.py` auto-truncates names, keeping the last 63 characters to preserve the UUID suffix:

```python
if len(_resource_name) > 63:
    LOGGER.warning(
        f"'{_resource_name=}' is too long ({len(_resource_name)} > 63). Truncating."
    )
    _resource_name = _resource_name[-63:]
```

> **Note:** This is a warning, not an error. The truncation is handled automatically. However, truncated names may be less readable in debugging output.

### Resource Already Exists (ConflictError)

**Symptom:** Warning in logs about a resource being reused.

```
WARNING: Plan auto-abcd-plan-cold already exists, reusing it.
```

**Cause:** The `create_and_store_resource()` function handles `ConflictError` gracefully by reusing existing resources:

```python
try:
    _resource.deploy(wait=True)
except ConflictError:
    LOGGER.warning(
        f"{_resource.kind} {_resource_name} already exists, reusing it."
    )
    _resource.wait()
```

This can happen when:
- A previous test run did not clean up properly (use `--skip-teardown` carefully)
- Parallel test workers (`pytest-xdist`) generate overlapping names
- A test is re-run without proper teardown

**Solution:**

1. Ensure previous test resources are cleaned up:
   ```bash
   oc get all -n <target-namespace> | grep auto-
   ```
2. Manually delete leftover resources if needed:
   ```bash
   oc delete plan,storagemap,networkmap,migration -n <namespace> --all
   ```
3. If intentional, the reuse behavior is safe — the framework will wait for the existing resource to be ready

### Session Teardown Leftovers

**Symptom:** `SessionTeardownError` at the end of a test session.

```
SessionTeardownError: Failed to clean up the following resources:
{'Plan': [{'name': 'auto-abcd-plan', 'namespace': 'auto-abcd-ns'}]}
```

**Cause:** The session teardown in `utilities/pytest_utils.py` follows a strict cleanup order (Migrations → Plans → all other resources → Namespaces) and raises `SessionTeardownError` if any resources remain:

```python
def session_teardown(session_store: dict[str, Any]) -> None:
    # ...
    leftovers = teardown_resources(...)
    if leftovers:
        raise SessionTeardownError(
            f"Failed to clean up the following resources: {leftovers}"
        )
```

**Solution:**

1. Check if resources are stuck in a finalizer loop:
   ```bash
   oc get plan <name> -n <namespace> -o jsonpath='{.metadata.finalizers}'
   ```
2. For stuck migrations, try manual cancellation:
   ```bash
   oc patch migration <name> -n <namespace> --type merge -p '{"spec":{"cancel":[]}}'
   ```
3. Use `--skip-teardown` to preserve resources for debugging, then clean up manually

### ResourceNameNotStartedWithSessionUUIDError

**Symptom:** Error indicating a resource name does not follow the session UUID naming convention.

**Cause:** All resources created by the test framework should include the session UUID prefix (e.g., `auto-abcd-`). This prevents name collisions between parallel test runs.

**Solution:** Always use `create_and_store_resource()` for resource creation, never create resources directly:

```python
# Correct - uses create_and_store_resource
namespace = create_and_store_resource(
    fixture_store=fixture_store,
    resource=Namespace,
    client=ocp_admin_client,
    name="my-namespace",
)

# Wrong - bypasses naming and tracking
namespace = Namespace(client=ocp_admin_client, name="my-namespace")
namespace.deploy()
```

---

## Migration Execution Failures

### Plan Not Reaching Ready Status

**Symptom:** `TimeoutExpiredError` when the Plan CR does not become ready.

```
Plan auto-abcd-plan-cold failed to reach status True
```

**Cause:** After creating a Plan CR, the framework waits 360 seconds for it to reach the `Ready` condition. The error output includes the full Plan and Provider instances for debugging:

```python
try:
    plan.wait_for_condition(
        condition=Plan.Condition.READY,
        status=Plan.Condition.Status.TRUE,
        timeout=360,
    )
except TimeoutExpiredError:
    LOGGER.error(f"Plan {plan.name} failed to reach status {Plan.Condition.Status.TRUE}\n\t{plan.instance}")
    LOGGER.error(f"Source provider: {source_provider.ocp_resource.instance}")
    LOGGER.error(f"Destination provider: {destination_provider.ocp_resource.instance}")
    raise
```

**Solution:**

1. Check the Plan status conditions:
   ```bash
   oc get plan <name> -n <namespace> -o yaml
   ```
2. Verify StorageMap and NetworkMap are valid
3. Verify both source and destination providers are in `Ready` status
4. Check for VM validation errors in the Plan status

### Migration Failed or Timed Out

**Symptom:** `MigrationPlanExecError` with the Plan status dump.

```
MigrationPlanExecError: Plan auto-abcd-plan-cold failed to reach the expected
condition. status: ...
```

**Cause:** The migration either explicitly failed or exceeded the configured timeout (default 600 seconds, configurable via `plan_wait_timeout`):

```python
for sample in TimeoutSampler(
    func=get_plan_migration_status,
    sleep=1,
    wait_timeout=py_config.get("plan_wait_timeout", 600),
    plan=plan,
):
    if sample == Plan.Status.SUCCEEDED:
        return
    elif sample == Plan.Status.FAILED:
        raise MigrationPlanExecError()
```

**Solution:**

1. Check the Migration CR status for specific VM pipeline failures:
   ```bash
   oc get migration -n <namespace> -o yaml
   ```
2. Increase `plan_wait_timeout` in `tests/tests_config/config.py` for large VMs
3. Examine the Forklift controller logs:
   ```bash
   oc logs -n openshift-mtv deployment/forklift-controller
   ```

### Migration CR Not Found

**Symptom:** `MigrationNotFoundError` when the framework cannot find the Migration CR for a Plan.

```
MigrationNotFoundError: Migration CR not found for Plan 'auto-abcd-plan-cold'
in namespace 'auto-abcd-ns'
```

**Cause:** The `_find_migration_for_plan()` function in `utilities/mtv_migration.py` searches for a Migration CR with an ownerReference pointing to the Plan. If no such Migration exists, this error is raised.

**Solution:**

1. Verify the Migration CR was created:
   ```bash
   oc get migration -n <namespace>
   ```
2. Check if the Plan's ownerReferences are set correctly

### VM Pipeline Failure

**Symptom:** `VmPipelineError` when a specific VM's migration pipeline is missing or has no identifiable failed step.

```
VmPipelineError: VM 'my-vm' pipeline is missing or has no failed step
```

**Cause:** The `_get_failed_migration_step()` function examines the Migration status to identify which pipeline step failed (e.g., `PreHook`, `DiskTransfer`, `PostHook`). This error occurs when the pipeline data is incomplete.

**Solution:**

1. Inspect the Migration CR's `.status.vms` section for the affected VM
2. Check the virt-v2v pod logs for disk transfer issues:
   ```bash
   oc logs -n <namespace> <virt-v2v-pod>
   ```

### VMs Failing at Different Steps

**Symptom:** `VmMigrationStepMismatchError` when multiple VMs in the same plan fail at different pipeline steps.

```
VmMigrationStepMismatchError: VMs in plan 'auto-abcd-plan-cold' failed at
different steps: {'vm1': 'PreHook', 'vm2': 'DiskTransfer'}
```

**Cause:** In hook testing scenarios, all VMs in a plan are expected to fail at the same step. The `validate_all_vms_same_step()` function in `utilities/hooks.py` enforces this consistency.

**Solution:** This usually indicates a hook configuration issue. Review the hook playbook and ensure it applies consistently to all VMs.

---

## Copy-Offload Configuration Issues

### Missing Copy-Offload Configuration

**Symptom:** Test failure with a message about missing copy-offload parameters.

```
Copy-offload configuration not found in source provider data.
Add 'copyoffload' section to your provider in .providers.json
```

**Solution:** Add the `copyoffload` section to your vSphere provider in `.providers.json`:

```json
{
  "vsphere-copy-offload": {
    "type": "vsphere",
    "version": "8.0",
    "fqdn": "vcenter.example.com",
    "api_url": "vcenter.example.com/sdk",
    "username": "admin@vsphere.local",
    "password": "password",
    "copyoffload": {
      "storage_vendor_product": "ontap",
      "datastore_id": "datastore-12345",
      "storage_hostname": "storage.example.com",
      "storage_username": "admin",
      "storage_password": "password"
    }
  }
}
```

### Missing Storage Credentials

**Symptom:** Test failure listing missing credentials.

```
Required storage credentials not found: ['storage_hostname', 'storage_password'].
Add them to .providers.json copyoffload section or set environment variables:
COPYOFFLOAD_STORAGE_HOSTNAME, COPYOFFLOAD_STORAGE_PASSWORD
```

**Cause:** Copy-offload credentials can be provided via environment variables or `.providers.json`. Environment variables take precedence. The `get_copyoffload_credential()` function in `utilities/copyoffload_migration.py` checks both:

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
    env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
    return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

**Solution:** Provide credentials either way:

```bash
# Via environment variables
export COPYOFFLOAD_STORAGE_HOSTNAME=storage.example.com
export COPYOFFLOAD_STORAGE_USERNAME=admin
export COPYOFFLOAD_STORAGE_PASSWORD=secret
```

Or in `.providers.json` under the `copyoffload` section.

### Plan-Specific Secret Timeout

**Symptom:** Warning about plan secret not being created.

```
WARNING: Timeout waiting for plan secret 'auto-abcd-plan-cold-*' - continuing anyway
```

**Cause:** After a copy-offload Plan is created, ForkliftController creates a plan-specific secret containing storage credentials. The `wait_for_plan_secret()` function polls for 60 seconds but continues regardless:

```python
try:
    for _ in TimeoutSampler(
        wait_timeout=60,
        sleep=2,
        func=lambda: any(
            s.name.startswith(f"{plan_name}-")
            for s in Secret.get(client=ocp_admin_client, namespace=namespace)
        ),
    ):
        break
except TimeoutExpiredError:
    LOGGER.warning(
        f"Timeout waiting for plan secret '{plan_name}-*' - continuing anyway"
    )
```

> **Note:** This warning does not immediately fail the test. If the secret is truly missing, the migration itself will fail with a clearer error.

**Solution:**

1. Check that ForkliftController is running and responding
2. Verify the Plan has copy-offload configured correctly
3. Check for errors in the ForkliftController logs

---

## Must-Gather Collection

### Automatic Must-Gather Triggers

Must-gather collection runs automatically in two scenarios:

1. **On test failure** — via the `pytest_exception_interact` hook, scoped to the failing plan:
   ```python
   def pytest_exception_interact(node, call, report):
       # ...
       run_must_gather(data_collector_path=_data_collector_path, plan=plan)
   ```

2. **On session teardown failure** — when resource cleanup fails and `SessionTeardownError` is raised:
   ```python
   try:
       session_teardown(session_store=_session_store)
   except Exception as exp:
       LOGGER.error(f"the following resources was left after tests are finished: {exp}")
       if not session.config.getoption("skip_data_collector"):
           run_must_gather(data_collector_path=_data_collector_path)
   ```

### Must-Gather Image Resolution Failures

**Symptom:** Exception logged but test run continues.

```
Failed to run must-gather. ValueError: No MUST_GATHER_IMAGE found in MTV
ClusterServiceVersion 'mtv-operator.v2.11.0'
```

**Cause:** The must-gather image is resolved through a multi-step process in `utilities/must_gather.py`:

1. Find the MTV Subscription → extract `installedCSV`
2. Find the CSV → extract `MUST_GATHER_IMAGE` from container env
3. Extract Subscription channel → derive IDMS name
4. Look up IDMS → get mirror URL
5. Combine mirror URL with image SHA

Failure at any step logs the error but does not fail the test run:

```python
try:
    # ... resolution and execution logic
except Exception as ex:
    LOGGER.exception(f"Failed to run must-gather. {ex}")
```

**Common causes and solutions:**

| Error | Cause | Solution |
|-------|-------|----------|
| `No MUST_GATHER_IMAGE found` | CSV has no `MUST_GATHER_IMAGE` env var | Verify MTV operator version |
| `Subscription channel is empty` | MTV Subscription has no channel set | Check Subscription CR |
| `No must-gather entry found in IDMS` | IDMS missing must-gather mirror | Verify ImageDigestMirrorSet exists |
| `CSV image does not contain a digest separator '@'` | Image reference uses tag instead of SHA | Expected in some development builds |

### Skipping Data Collection

Use `--skip-data-collector` to disable must-gather collection entirely:

```bash
uv run pytest tests/ --skip-data-collector
```

### Manual Must-Gather

If automatic collection fails, run must-gather manually:

```bash
oc adm must-gather \
  --image=<must-gather-image> \
  --dest-dir=./must-gather-output \
  -- NS=<namespace> PLAN=<plan-name> /usr/bin/targeted
```

> **Tip:** For a full namespace capture without plan scoping, omit the `PLAN` parameter:
> ```bash
> oc adm must-gather \
>   --image=<must-gather-image> \
>   --dest-dir=./must-gather-output \
>   -- -- NS=openshift-mtv
> ```

---

## Missing Required Configuration

### Required Config Keys

The session startup validates that critical configuration is present:

```python
def pytest_sessionstart(session):
    required_config = ("storage_class", "source_provider")
    # ...
    if missing_configs:
        pytest.exit(
            reason=f"Some required config is missing {required_config=} - {missing_configs=}",
            returncode=1,
        )
```

**Solution:** Ensure `storage_class` and `source_provider` are set in `tests/tests_config/config.py`.

### API Key Not Found

**Symptom:** `ValueError` from the `destination_ocp_secret` fixture.

```
ValueError: API key not found in configuration
```

**Cause:** The OCP client does not have a valid API key (Bearer token). The fixture extracts it from:

```python
api_key: str = ocp_admin_client.configuration.api_key.get("authorization")
if not api_key:
    raise ValueError("API key not found in configuration")
```

**Solution:** Ensure you are logged in to the cluster:

```bash
oc login <cluster-url> -u <username> -p <password>
# or
export KUBECONFIG=/path/to/kubeconfig
```

---

## Parallel Execution Issues (pytest-xdist)

### Virtctl Lock Timeout

**Symptom:** `TimeoutError` waiting for the virtctl file lock.

```
TimeoutError: Timeout (600s) waiting for virtctl lock at
/tmp/pytest-shared-virtctl/4-17-0/virtctl.lock. Another process may be stuck.
```

**Cause:** When running with `pytest-xdist` (`-n auto`), multiple workers share a single virtctl binary. File locking prevents concurrent downloads:

```python
try:
    with filelock.FileLock(lock_file, timeout=600):
        if not virtctl_path.is_file() or not os.access(virtctl_path, os.X_OK):
            download_virtctl_from_cluster(client=ocp_admin_client, download_dir=shared_dir)
except filelock.Timeout as err:
    raise TimeoutError(
        f"Timeout (600s) waiting for virtctl lock at {lock_file}. "
        "Another process may be stuck."
    ) from err
```

**Solution:**

1. Check for stale lock files and remove them:
   ```bash
   rm -f /tmp/pytest-shared-virtctl/*/virtctl.lock
   ```
2. Check if another test session is still running
3. Verify network connectivity for virtctl download

### Empty Session Store in Parallel Mode

**Symptom:** Teardown issues when running with `-n auto`.

**Cause:** When running in parallel, each xdist worker gets its own `fixture_store`. The `session_teardown` function handles this:

```python
# When running in parallel (-n auto) `session_store` can be empty.
if session_teardown_resources := session_store.get("teardown"):
    # ... cleanup logic
```

> **Tip:** Tests are parallel-safe by design because unique namespaces (via `session_uuid`) and unique resource names prevent collisions between workers.

---

## Debugging Tips

### Enable Debug Logging

Use the `--openshift-python-wrapper-log-debug` flag to enable verbose API logging:

```bash
uv run pytest tests/ --openshift-python-wrapper-log-debug
```

### Preserve Resources for Investigation

Use `--skip-teardown` to keep test resources after completion:

```bash
uv run pytest tests/ --skip-teardown
```

> **Warning:** Always clean up manually after using `--skip-teardown` to avoid resource leaks on the cluster.

### Check Data Collector Output

Failed test data is stored in the data collector path (default: `.data-collector/`):

```bash
ls .data-collector/<test-name>/
```

This includes must-gather output and resource dumps. Configure a custom path with:

```bash
uv run pytest tests/ --data-collector-path=/path/to/output
```

### AI-Powered Failure Analysis

Enable AI analysis of test failures by passing `--analyze-with-ai`:

```bash
export JJI_SERVER_URL=<your-jji-server>
uv run pytest tests/ --analyze-with-ai --junitxml=results.xml
```

This enriches the JUnit XML with AI-generated failure analysis. Requires `JJI_SERVER_URL` to be set.
