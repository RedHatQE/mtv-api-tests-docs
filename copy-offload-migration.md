# Copy-Offload Migration

Copy-offload is an MTV (Migration Toolkit for Virtualization) feature that uses the storage array to directly copy VM disks from vSphere datastores to OpenShift PVCs using offload operations such as XCOPY, volume CoW, or host-based copy. This bypasses the traditional VDDK-based transfer path, resulting in significantly faster migration times for large-scale VM migrations.

Copy-offload requires shared SAN/Block storage infrastructure between vSphere and OpenShift, VAAI (vSphere APIs for Array Integration) enabled on ESXi hosts, and a StorageMap configured with offload plugin settings. The feature is implemented by the [vsphere-xcopy-volume-populator](https://github.com/kubev2v/forklift/tree/main/cmd/vsphere-xcopy-volume-populator), a component of the Forklift project.

> **Note:** NFS storage is **not supported** for copy-offload. Only SAN/Block storage (iSCSI or Fibre Channel) is supported.

---

## Supported Storage Vendors

The test framework validates against nine storage vendor products, defined in `utilities/copyoffload_constants.py`:

```python
SUPPORTED_VENDORS = (
    "ontap",           # NetApp ONTAP
    "vantara",         # Hitachi Vantara
    "primera3par",     # HPE Primera/3PAR
    "pureFlashArray",  # Pure Storage FlashArray
    "powerflex",       # Dell PowerFlex
    "powermax",        # Dell PowerMax
    "powerstore",      # Dell PowerStore
    "infinibox",       # Infinidat InfiniBox
    "flashsystem",     # IBM FlashSystem
)
```

Each vendor requires a base set of storage credentials (hostname, username, password) and may require additional vendor-specific fields. The following table summarizes the requirements:

| Vendor | `storage_vendor_product` | Additional Required Fields |
|--------|--------------------------|---------------------------|
| NetApp ONTAP | `ontap` | `ontap_svm` (SVM/vServer name) |
| Hitachi Vantara | `vantara` | `vantara_storage_id`, `vantara_storage_port`, `vantara_hostgroup_id_list` |
| Pure Storage | `pureFlashArray` | `pure_cluster_prefix` |
| Dell PowerFlex | `powerflex` | `powerflex_system_id` |
| Dell PowerMax | `powermax` | `powermax_symmetrix_id` |
| Dell PowerStore | `powerstore` | None (base credentials only) |
| HPE Primera/3PAR | `primera3par` | None (base credentials only) |
| Infinidat InfiniBox | `infinibox` | None (base credentials only) |
| IBM FlashSystem | `flashsystem` | None (base credentials only) |

### Vendor-Specific Field Details

**NetApp ONTAP** -- Requires the SVM (Storage Virtual Machine) name that serves the datastores:
```json
"ontap_svm": "vserver-name"
```

**Hitachi Vantara** -- Requires storage array identification and host group mapping. The `vantara_hostgroup_id_list` uses the format `port,hostgroup_id` separated by colons:
```json
"vantara_storage_id": "123456",
"vantara_storage_port": "443",
"vantara_hostgroup_id_list": "CL1-A,1:CL2-B,2:CL4-A,1:CL6-A,1"
```

**Pure Storage FlashArray** -- Requires the cluster prefix, which can be retrieved from the OpenShift cluster:
```bash
printf "px_%.8s" $(oc get storagecluster -A \
  -o=jsonpath='{.items[?(@.spec.cloudStorage.provider=="pure")].status.clusterUid}')
```

**Dell PowerFlex** -- Requires the system ID, available from the `vxflexos-config` ConfigMap:
```json
"powerflex_system_id": "abc123def456"
```

**Dell PowerMax** -- Requires the Symmetrix ID (12-digit format):
```json
"powermax_symmetrix_id": "000123456789"
```

---

## Datastore Configuration

Copy-offload tests support up to three datastore types to cover various migration scenarios:

| Datastore Type | Config Key | Purpose |
|---------------|-----------|---------|
| **Primary** | `datastore_id` | XCOPY-capable datastore where test VMs are cloned and stored. Required for all copy-offload tests. |
| **Secondary** | `secondary_datastore_id` | Second XCOPY-capable datastore on the **same storage array**. Used for multi-datastore tests. |
| **Non-XCOPY** | `non_xcopy_datastore_id` | A datastore that does **not** support XCOPY. Used for mixed-datastore and fallback tests. |

To find a datastore ID in vSphere: **Datacenter > Storage > Datastore > Summary > More Objects ID** (format: `datastore-123`).

> **Warning:** If `secondary_datastore_id` is configured, both the primary and secondary datastores must be on the **same physical storage array** and support VAAI XCOPY primitives. If this requirement is not met, copy-offload will fall back to an alternative transfer method.

### Multi-Datastore Scenarios

The test suite validates several multi-datastore configurations:

- **Multi-datastore migration** (`TestCopyoffloadMultiDatastoreMigration`) -- VM with disks distributed across primary and secondary XCOPY-capable datastores
- **Mixed datastore migration** (`TestCopyoffloadMixedDatastoreMigration`) -- VM with disks on both XCOPY-capable and non-XCOPY datastores; validates that XCOPY-capable disks use acceleration while others fall back gracefully
- **Fallback migration** (`TestCopyoffloadFallbackLargeMigration`) -- VM with all disks on a non-XCOPY datastore, verifying fallback behavior

The `mixed_datastore_config` fixture in `conftest.py` validates that `non_xcopy_datastore_id` is configured before mixed-datastore tests run:

```python
@pytest.fixture(scope="class")
def mixed_datastore_config(source_provider_data: dict[str, Any]) -> None:
    copyoffload_config_data: dict[str, Any] = source_provider_data.get("copyoffload", {})
    non_xcopy_datastore_id: str | None = copyoffload_config_data.get("non_xcopy_datastore_id")

    if not non_xcopy_datastore_id:
        raise ValueError(
            "Mixed datastore test requires 'non_xcopy_datastore_id' to be configured in copyoffload section. "
            "This should be a datastore that does NOT support XCOPY."
        )
```

### RDM (Raw Device Mapping) Support

Copy-offload also supports RDM virtual disk migrations. Configure `rdm_lun_uuid` in the copyoffload section:

```json
"rdm_lun_uuid": "naa.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

> **Warning:** RDM copy-offload is **currently supported only for Pure Storage**. The `datastore_id` must be a **VMFS datastore** -- RDM disks are not supported on vSAN or NFS datastores.

---

## Offload Plugin Setup

Copy-offload migrations require three components configured on the OpenShift cluster: a storage secret, the offload plugin configuration within the StorageMap, and the provider annotation.

### Storage Secret

The `copyoffload_storage_secret` fixture (session-scoped) creates a Kubernetes Secret containing storage array credentials. Every vendor requires the base credentials, plus vendor-specific fields as applicable:

```python
# Base secret data (required for all vendors)
secret_data = {
    "STORAGE_HOSTNAME": storage_hostname,
    "STORAGE_USERNAME": storage_username,
    "STORAGE_PASSWORD": storage_password,
}
```

Vendor-specific fields are added based on `storage_vendor_product`. The fixture maps config keys to secret keys as follows (from `conftest.py`):

```python
vendor_specific_fields = {
    "ontap": [("ontap_svm", "ONTAP_SVM", True)],
    "vantara": [
        ("vantara_storage_id", "STORAGE_ID", True),
        ("vantara_storage_port", "STORAGE_PORT", True),
        ("vantara_hostgroup_id_list", "HOSTGROUP_ID_LIST", True),
    ],
    "primera3par": [],  # Only basic credentials required
    "pureFlashArray": [("pure_cluster_prefix", "PURE_CLUSTER_PREFIX", True)],
    "powerflex": [("powerflex_system_id", "POWERFLEX_SYSTEM_ID", True)],
    "powermax": [("powermax_symmetrix_id", "POWERMAX_SYMMETRIX_ID", True)],
    "powerstore": [],  # Only basic credentials required
    "infinibox": [],  # Only basic credentials required
    "flashsystem": [],  # Only basic credentials required
}
```

The secret is created via `create_and_store_resource()` in the target namespace, ensuring proper lifecycle tracking and teardown.

### Credential Resolution

All credentials support a dual-source resolution pattern: environment variables take precedence over `.providers.json` config values. This is implemented in `utilities/copyoffload_migration.py`:

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
    env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
    return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

Available environment variable overrides:

| Environment Variable | Config Key |
|---------------------|-----------|
| `COPYOFFLOAD_STORAGE_HOSTNAME` | `storage_hostname` |
| `COPYOFFLOAD_STORAGE_USERNAME` | `storage_username` |
| `COPYOFFLOAD_STORAGE_PASSWORD` | `storage_password` |
| `COPYOFFLOAD_ONTAP_SVM` | `ontap_svm` |
| `COPYOFFLOAD_VANTARA_STORAGE_ID` | `vantara_storage_id` |
| `COPYOFFLOAD_VANTARA_STORAGE_PORT` | `vantara_storage_port` |
| `COPYOFFLOAD_VANTARA_HOSTGROUP_ID_LIST` | `vantara_hostgroup_id_list` |
| `COPYOFFLOAD_PURE_CLUSTER_PREFIX` | `pure_cluster_prefix` |
| `COPYOFFLOAD_POWERFLEX_SYSTEM_ID` | `powerflex_system_id` |
| `COPYOFFLOAD_POWERMAX_SYMMETRIX_ID` | `powermax_symmetrix_id` |
| `COPYOFFLOAD_ESXI_HOST` | `esxi_host` |
| `COPYOFFLOAD_ESXI_USER` | `esxi_user` |
| `COPYOFFLOAD_ESXI_PASSWORD` | `esxi_password` |

### StorageMap Offload Plugin Configuration

Each copy-offload StorageMap entry includes an `offloadPlugin` block that references the storage secret and specifies the vendor. In the tests, this is constructed as:

```python
offload_plugin_config = {
    "vsphereXcopyConfig": {
        "secretRef": copyoffload_storage_secret.name,
        "storageVendorProduct": storage_vendor_product,
    }
}
```

The `get_storage_migration_map()` function in `utilities/mtv_migration.py` attaches this configuration to each datastore mapping entry:

```python
storage_map_list.append({
    "destination": destination_config,
    "source": {"id": ds_id},
    "offloadPlugin": offload_plugin_config,
})
```

For copy-offload StorageMaps, the destination is configured with block-mode access:

- `accessMode`: `ReadWriteOnce`
- `volumeMode`: `Block`

### Provider Annotation

When copy-offload is configured, the vSphere Provider resource receives a special annotation that tells Forklift to skip VDDK image validation:

```python
provider_annotations["forklift.konveyor.io/empty-vddk-init-image"] = "yes"
```

This annotation is automatically set during provider creation in `utilities/utils.py` when a `copyoffload` section is present in the provider configuration.

---

## The vsphere-xcopy-volume-populator

The [vsphere-xcopy-volume-populator](https://github.com/kubev2v/forklift/tree/main/cmd/vsphere-xcopy-volume-populator) is the Forklift component that performs the actual storage-level disk copy operations. It runs as a pod in the `openshift-mtv` namespace and coordinates with the storage array to execute XCOPY (or equivalent) operations.

### How It Integrates with Tests

When a Plan is created with `copyoffload=True`, two Plan-level behaviors are activated in `utilities/mtv_migration.py`:

1. **PVC naming template** -- The Plan spec is configured with `pvc_name_template: "pvc"` to generate consistent PVC names that the volume populator framework expects:

```python
if copyoffload:
    plan_kwargs["pvc_name_template"] = "pvc"
```

2. **Plan secret polling** -- After Plan creation, the framework waits for Forklift to auto-create a plan-specific secret (up to 60 seconds) containing the storage credentials needed by the volume populator:

```python
def wait_for_plan_secret(ocp_admin_client: DynamicClient, namespace: str, plan_name: str) -> None:
    LOGGER.info("Copy-offload: waiting for Forklift to create plan-specific secret...")
    try:
        for _ in TimeoutSampler(
            wait_timeout=60,
            sleep=2,
            func=lambda: any(
                s.name.startswith(f"{plan_name}-") for s in Secret.get(client=ocp_admin_client, namespace=namespace)
            ),
        ):
            break
    except TimeoutExpiredError:
        LOGGER.warning(f"Timeout waiting for plan secret '{plan_name}-*' - continuing anyway")
```

### Clone Methods

The volume populator supports two methods for cloning VMDK files on ESXi hosts:

**SSH method** (`esxi_clone_method: "ssh"`) -- Recommended. Requires SSH access to ESXi hosts. The `setup_copyoffload_ssh` fixture automates SSH key deployment:

1. Retrieves the public key from the provider-created secret (`offload-ssh-keys-{provider_name}-public`)
2. Installs the key on the ESXi host via SSH using `install_ssh_key_on_esxi()`
3. Removes the key during teardown via `remove_ssh_key_from_esxi()`

**VIB method** (`esxi_clone_method: "vib"` or omitted) -- Requires ESXi host permissions to allow community-level VIB installation. The VIB is installed automatically by the volume populator.

> **Tip:** The SSH method is recommended as it does not require VIB installation permissions on ESXi hosts.

### Debugging the Volume Populator

To inspect volume populator logs during or after a migration:

```bash
# Check volume populator pod logs
oc logs -n openshift-mtv -l app=vsphere-xcopy-volume-populator --tail=100

# Check MTV Forklift controller logs
oc logs -n openshift-mtv deployment/forklift-controller --tail=100

# Inspect the migration plan status
oc get plan -n openshift-mtv <plan-name> -o yaml
```

---

## Provider Configuration

Add the `copyoffload` section to your provider definition in `.providers.json`:

```json
{
  "vsphere-8.0.3.00400": {
    "type": "vsphere",
    "version": "8.0.3.00400",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "your-vcenter-password",
    "guest_vm_linux_user": "root",
    "guest_vm_linux_password": "your-vm-password",
    "copyoffload": {
      "storage_vendor_product": "ontap",
      "datastore_id": "datastore-123",
      "secondary_datastore_id": "datastore-456",
      "default_vm_name": "rhel9-cloud-init-template",
      "storage_hostname": "storage.example.com",
      "storage_username": "admin",
      "storage_password": "your-storage-password",
      "ontap_svm": "vserver-name",
      "esxi_clone_method": "ssh",
      "esxi_host": "esxi01.example.com",
      "esxi_user": "root",
      "esxi_password": "your-esxi-password",
      "rdm_lun_uuid": "naa.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  }
}
```

> **Note:** The provider key (`vsphere-8.0.3.00400`) is passed directly via `--tc=source_provider:vsphere-8.0.3.00400`. The value must match exactly what is defined in your `.providers.json` file.

### Configuration Validation

The `copyoffload_config` fixture (session-scoped) validates the entire configuration before any copy-offload test runs:

- Verifies the source provider is vSphere
- Checks the `copyoffload` section exists in provider data
- Validates that required storage credentials are available (from environment variables or config)
- Validates `storage_vendor_product` and `datastore_id` are present

```python
@pytest.fixture(scope="session")
def copyoffload_config(source_provider, source_provider_data):
    if source_provider.type != Provider.ProviderType.VSPHERE:
        pytest.fail(
            f"Copy-offload tests require vSphere provider, but got '{source_provider.type}'. "
            "Check your provider configuration in .providers.json"
        )

    if "copyoffload" not in source_provider_data:
        pytest.fail(
            "Copy-offload configuration not found in source provider data. "
            "Add 'copyoffload' section to your provider in .providers.json"
        )
    # ... credential and parameter validation
```

---

## Test Configuration

Copy-offload test configurations are defined in `tests/tests_config/config.py`. Each test entry includes `"copyoffload": True` to enable the copy-offload migration path.

### Basic Test Configuration

```python
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
```

### Multi-Disk Configuration

Additional disks can be added with various provisioning types and disk modes:

```python
"test_copyoffload_multi_disk_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "add_disks": [
                {"size_gb": 30, "disk_mode": "persistent", "provision_type": "thick-lazy"},
            ],
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

### Snapshot Configuration

Snapshot tests specify the number of snapshots to create before migration:

```python
"test_copyoffload_thin_snapshots_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thin",
            "snapshots": 2,
        },
    ],
    "warm_migration": False,
    "copyoffload": True,
},
```

### Scale Configuration

Scale tests configure multiple VMs for concurrent migration:

```python
"test_copyoffload_scale_migration": {
    "virtual_machines": [
        {
            "name": "xcopy-template-test",
            "source_vm_power": "off",
            "guest_agent": True,
            "clone": True,
            "disk_type": "thick-lazy",
            "add_disks": [{"size_gb": 30, "provision_type": "thick-lazy", "disk_mode": "persistent"}],
        },
        # ... repeated for 5 VMs total
    ],
    "warm_migration": False,
    "copyoffload": True,
    "guest_agent_timeout": 600,
},
```

---

## Test Suite

All copy-offload tests are marked with `@pytest.mark.copyoffload` and follow the standard 5-step migration test pattern: StorageMap > NetworkMap > Plan > Migrate > Check VMs.

### Test Classes

| Test Class | Description | Special Requirements |
|-----------|------------|---------------------|
| `TestCopyoffloadThinMigration` | Thin-provisioned disk migration | -- |
| `TestCopyoffloadThickLazyMigration` | Thick lazy-zeroed disk migration | -- |
| `TestCopyoffloadThinSnapshotsMigration` | Thin disk with 2 snapshots | vSphere only |
| `TestCopyoffload2TbVmSnapshotsMigration` | 2TB VM with snapshots | vSphere only |
| `TestCopyoffloadMultiDiskMigration` | Multiple disks on same datastore | -- |
| `TestCopyoffloadMultiDiskDifferentPathMigration` | Multiple disks on different datastores | `secondary_datastore_id` |
| `TestCopyoffloadRdmVirtualDiskMigration` | RDM virtual disk | `rdm_lun_uuid`, Pure Storage only |
| `TestCopyoffloadMultiDatastoreMigration` | Disks across multiple XCOPY datastores | `secondary_datastore_id` |
| `TestCopyoffloadMixedDatastoreMigration` | Mix of XCOPY and non-XCOPY datastores | `non_xcopy_datastore_id` |
| `TestCopyoffloadFallbackLargeMigration` | Large VM on non-XCOPY datastore (fallback) | `non_xcopy_datastore_id` |
| `TestCopyoffloadIndependentPersistentDiskMigration` | Independent persistent disk mode | -- |
| `TestCopyoffloadIndependentNonpersistentDiskMigration` | Independent non-persistent disk mode | -- |
| `TestCopyoffload10MixedDisksMigration` | 10 mixed thin/thick disks | -- |
| `TestCopyoffloadLargeVmMigration` | Large VM (1TB disk) | -- |
| `TestCopyoffloadNonconformingNameMigration` | VM name with capitals/underscores | -- |
| `TestCopyoffloadWarmMigration` | Warm migration with copy-offload | VM powered on |
| `TestCopyoffloadScaleMigration` | 5 concurrent VMs | Extended `guest_agent_timeout` |
| `TestSimultaneousCopyoffloadMigrations` | Two simultaneous migration plans | -- |
| `TestConcurrentXcopyVddkMigration` | Concurrent XCOPY + VDDK migrations | -- |

### Test Structure Example

Every copy-offload test class follows this structure (from `tests/test_copyoffload_migration.py`):

```python
@pytest.mark.copyoffload
@pytest.mark.incremental
@pytest.mark.parametrize(
    "class_plan_config",
    [pytest.param(py_config["tests_params"]["test_copyoffload_thin_migration"])],
    indirect=True,
    ids=["MTV-559:copyoffload-thin"],
)
@pytest.mark.usefixtures(
    "multus_network_name",
    "copyoffload_config",
    "setup_copyoffload_ssh",
    "cleanup_migrated_vms",
)
class TestCopyoffloadThinMigration:
    """Copy-offload migration test - thin disk."""

    storage_map: StorageMap
    network_map: NetworkMap
    plan_resource: Plan

    def test_create_storagemap(self, ...):
        """Create StorageMap with copy-offload configuration."""
        ...

    def test_create_networkmap(self, ...):
        """Create NetworkMap resource."""
        ...

    def test_create_plan(self, ...):
        """Create MTV Plan CR resource."""
        ...

    def test_migrate_vms(self, ...):
        """Execute migration."""
        ...

    def test_check_vms(self, ...):
        """Validate migrated VMs."""
        ...
```

### Snapshot Tests

The `CopyoffloadSnapshotBase` base class provides shared logic for snapshot-related tests. Before creating the StorageMap, it:

1. Powers on the VM
2. Creates the specified number of snapshots
3. Records snapshot data for post-migration verification
4. Powers off the VM for cold migration

```python
class CopyoffloadSnapshotBase:
    def test_create_storagemap(self, ...):
        """Create StorageMap with copy-offload configuration after creating snapshots."""
        vm_cfg = prepared_plan["virtual_machines"][0]
        provider_vm_api = prepared_plan["source_vms_data"][vm_cfg["name"]]["provider_vm_api"]

        source_provider.start_vm(provider_vm_api)
        source_provider.wait_for_vmware_guest_info(provider_vm_api, timeout=60)

        snapshots_to_create = int(vm_cfg["snapshots"])
        snapshot_prefix = f"{vm_cfg['name']}-{fixture_store['session_uuid']}-snapshot"

        for idx in range(1, snapshots_to_create + 1):
            source_provider.create_snapshot(
                vm=provider_vm_api,
                name=f"{snapshot_prefix}-{idx}",
                description="mtv-api-tests copy-offload snapshots migration test",
                memory=False,
                quiesce=False,
                wait_timeout=60 * 10,
            )

        vm_cfg["snapshots_before_migration"] = source_provider.vm_dict(
            provider_vm_api=provider_vm_api
        )["snapshots_data"]

        source_provider.stop_vm(provider_vm_api)
        # ... continues with standard StorageMap creation
```

---

## Running Copy-Offload Tests

### Using pytest

```bash
# Run all copy-offload tests
uv run pytest -m copyoffload -v \
  --tc=source_provider:vsphere-8.0.3.00400 \
  --tc=storage_class:my-block-storageclass

# Run a specific test
uv run pytest -m copyoffload -k test_copyoffload_thin_migration -v \
  --tc=source_provider:vsphere-8.0.3.00400 \
  --tc=storage_class:my-block-storageclass

# List available copy-offload tests
uv run pytest --collect-only -m copyoffload
```

### Using OpenShift Jobs

For consistent execution, deploy tests as an OpenShift Job:

```bash
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: mtv-copyoffload-tests
  namespace: mtv-tests
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: tests
        image: ghcr.io/redhatqe/mtv-api-tests:latest
        env:
        - name: CLUSTER_HOST
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_host
              optional: true
        - name: CLUSTER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mtv-test-config
              key: cluster_password
              optional: true
        command:
          - /bin/bash
          - -c
          - |
            uv run pytest -m copyoffload \
              -v \
              \${CLUSTER_HOST:+--tc=cluster_host:\${CLUSTER_HOST}} \
              \${CLUSTER_PASSWORD:+--tc=cluster_password:\${CLUSTER_PASSWORD}} \
              --tc=source_provider:vsphere-8.0.3.00400 \
              --tc=storage_class:my-block-storageclass
        volumeMounts:
        - name: config
          mountPath: /app/.providers.json
          subPath: providers.json
      volumes:
      - name: config
        secret:
          secretName: mtv-test-config
EOF
```

### Monitoring and Cleanup

```bash
# Follow test logs
oc logs -n mtv-tests job/mtv-copyoffload-tests -f

# Retrieve JUnit report
POD_NAME=$(oc get pods -n mtv-tests -l job-name=mtv-copyoffload-tests \
  -o jsonpath='{.items[0].metadata.name}')
oc cp mtv-tests/$POD_NAME:/app/junit-report.xml ./junit-report.xml

# Skip automatic cleanup for debugging
# Add --skip-teardown to the pytest command in the Job definition
```

> **Tip:** The `cleanup_migrated_vms` fixture automatically cleans up migrated VMs after each test class completes. Use `--skip-teardown` to preserve resources for debugging.

---

## Fixture Dependency Chain

The copy-offload fixtures form a validation chain that ensures all prerequisites are met before tests execute:

```
copyoffload_config (session)
├── Validates vSphere provider type
├── Validates copyoffload section exists
├── Validates storage credentials
└── Validates storage_vendor_product and datastore_id

copyoffload_storage_secret (session, depends on copyoffload_config)
├── Resolves credentials from env vars or config
├── Adds vendor-specific secret fields
└── Creates Kubernetes Secret via create_and_store_resource()

setup_copyoffload_ssh (session, depends on copyoffload_config)
├── Skips if esxi_clone_method != "ssh"
├── Retrieves SSH public key from provider
├── Installs key on ESXi host
└── Removes key during teardown

mixed_datastore_config (class)
└── Validates non_xcopy_datastore_id exists
```

---

## MTV Version Considerations

For MTV versions before 2.11, copy-offload must be explicitly enabled by adding the feature flag to the ForkliftController spec:

```yaml
spec:
  feature_copy_offload: 'true'
```

In MTV 2.11 and later, copy-offload is available without explicit feature flag configuration.
