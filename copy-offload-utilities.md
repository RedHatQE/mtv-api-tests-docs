# Copy-Offload Utilities

API reference for the copy-offload utility modules used by MTV API tests: credential management, plan secret synchronization, and supported storage vendor constants.

## Module Overview

The copy-offload utilities are split across two modules:

| Module | Purpose |
|--------|---------|
| `utilities/copyoffload_migration.py` | Runtime functions for credential resolution and plan secret polling |
| `utilities/copyoffload_constants.py` | Immutable constants for storage vendor validation |

These modules support the MTV copy-offload feature, which uses storage-array-level operations (such as XCOPY) to migrate VM disks directly from vSphere datastores to OpenShift PVCs, bypassing the traditional v2v data transfer path.

---

## `copyoffload_constants.py`

### `SUPPORTED_VENDORS`

```python
SUPPORTED_VENDORS = (
    "ontap",
    "vantara",
    "primera3par",
    "pureFlashArray",
    "powerflex",
    "powermax",
    "powerstore",
    "infinibox",
    "flashsystem",
)
```

An immutable tuple of all storage vendor product identifiers recognized by the copy-offload populator. Each value corresponds to a `storage_vendor_product` key in the provider's `copyoffload` configuration block.

| Constant Value | Storage Vendor | Vendor-Specific Fields Required |
|---------------|----------------|-------------------------------|
| `"ontap"` | NetApp ONTAP | `ontap_svm` |
| `"vantara"` | Hitachi Vantara | `vantara_storage_id`, `vantara_storage_port`, `vantara_hostgroup_id_list` |
| `"primera3par"` | HPE Primera/3PAR | None (base credentials only) |
| `"pureFlashArray"` | Pure Storage FlashArray | `pure_cluster_prefix` |
| `"powerflex"` | Dell PowerFlex | `powerflex_system_id` |
| `"powermax"` | Dell PowerMax | `powermax_symmetrix_id` |
| `"powerstore"` | Dell PowerStore | None (base credentials only) |
| `"infinibox"` | Infinidat InfiniBox | None (base credentials only) |
| `"flashsystem"` | IBM FlashSystem | None (base credentials only) |

**Usage in the codebase:**

`SUPPORTED_VENDORS` is imported in `conftest.py` to validate the `storage_vendor_product` value during the `copyoffload_storage_secret` fixture setup. The fixture also uses a runtime assertion to ensure its internal `vendor_specific_fields` mapping stays in sync with this constant:

```python
# From conftest.py — enforces that vendor_specific_fields keys match SUPPORTED_VENDORS
assert set(vendor_specific_fields.keys()) == set(SUPPORTED_VENDORS), (
    f"vendor_specific_fields keys must match SUPPORTED_VENDORS. "
    f"Missing in vendor_specific_fields: {set(SUPPORTED_VENDORS) - set(vendor_specific_fields.keys())}. "
    f"Extra in vendor_specific_fields: {set(vendor_specific_fields.keys()) - set(SUPPORTED_VENDORS)}"
)
```

> **Warning:** If you add a new storage vendor to `SUPPORTED_VENDORS`, you must also add a corresponding entry to the `vendor_specific_fields` dictionary in the `copyoffload_storage_secret` fixture in `conftest.py`. The assertion above will catch any drift between the two.

---

## `copyoffload_migration.py`

### `get_copyoffload_credential()`

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
```

Resolves a copy-offload credential value using a two-tier lookup strategy: environment variables take precedence over configuration file values.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `credential_name` | `str` | Name of the credential (e.g., `"storage_hostname"`, `"ontap_svm"`) |
| `copyoffload_config` | `dict[str, Any]` | The `copyoffload` section from the provider's `.providers.json` configuration |

**Returns:** `str | None` — The credential value from the environment variable or config dictionary, or `None` if not found in either location.

**Environment variable naming convention:**

The environment variable name is constructed by uppercasing the credential name and prefixing it with `COPYOFFLOAD_`:

```
COPYOFFLOAD_{CREDENTIAL_NAME_UPPER}
```

| `credential_name` | Environment Variable |
|--------------------|---------------------|
| `"storage_hostname"` | `COPYOFFLOAD_STORAGE_HOSTNAME` |
| `"storage_username"` | `COPYOFFLOAD_STORAGE_USERNAME` |
| `"storage_password"` | `COPYOFFLOAD_STORAGE_PASSWORD` |
| `"ontap_svm"` | `COPYOFFLOAD_ONTAP_SVM` |
| `"esxi_host"` | `COPYOFFLOAD_ESXI_HOST` |
| `"esxi_user"` | `COPYOFFLOAD_ESXI_USER` |
| `"esxi_password"` | `COPYOFFLOAD_ESXI_PASSWORD` |
| `"vantara_hostgroup_id_list"` | `COPYOFFLOAD_VANTARA_HOSTGROUP_ID_LIST` |

**Implementation:**

```python
env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

> **Tip:** Use environment variables to override credentials at runtime without modifying `.providers.json`. This is particularly useful in CI/CD pipelines and OpenShift Job-based test execution where secrets are mounted as environment variables.

**Usage examples from the codebase:**

The function is used extensively in the `copyoffload_storage_secret` and `setup_copyoffload_ssh` fixtures in `conftest.py`:

```python
# conftest.py — copyoffload_storage_secret fixture
copyoffload_cfg = source_provider_data["copyoffload"]

storage_hostname = get_copyoffload_credential("storage_hostname", copyoffload_cfg)
storage_username = get_copyoffload_credential("storage_username", copyoffload_cfg)
storage_password = get_copyoffload_credential("storage_password", copyoffload_cfg)
```

```python
# conftest.py — setup_copyoffload_ssh fixture (ESXi SSH credentials)
esxi_host = get_copyoffload_credential("esxi_host", copyoffload_cfg)
esxi_user = get_copyoffload_credential("esxi_user", copyoffload_cfg)
esxi_password = get_copyoffload_credential("esxi_password", copyoffload_cfg)
```

```python
# conftest.py — vendor-specific fields resolution
for config_key, secret_key, required in vendor_specific_fields[storage_vendor]:
    value = get_copyoffload_credential(config_key, copyoffload_cfg)
    if value:
        secret_data[secret_key] = value
    elif required:
        env_var_name = f"COPYOFFLOAD_{config_key.upper()}"
        raise ValueError(
            f"Required vendor-specific field '{config_key}' not found for vendor '{storage_vendor}'. "
            f"Add it to .providers.json copyoffload section or set environment variable: {env_var_name}"
        )
```

It is also used in the `copyoffload_config` validation fixture to check that required credentials are available before tests begin:

```python
# conftest.py — copyoffload_config fixture (early validation)
required_credentials = ["storage_hostname", "storage_username", "storage_password"]
missing_credentials = []

for cred in required_credentials:
    if not get_copyoffload_credential(cred, config):
        missing_credentials.append(cred)

if missing_credentials:
    pytest.fail(
        f"Required storage credentials not found: {missing_credentials}. "
        f"Add them to .providers.json copyoffload section or set environment variables: "
        f"{', '.join([f'COPYOFFLOAD_{c.upper()}' for c in missing_credentials])}"
    )
```

---

### `wait_for_plan_secret()`

```python
def wait_for_plan_secret(
    ocp_admin_client: DynamicClient,
    namespace: str,
    plan_name: str,
) -> None:
```

Polls for the existence of a Forklift-generated plan-specific secret used in copy-offload migrations. When a Plan CR is created with copy-offload configuration, the ForkliftController automatically creates a secret containing storage credentials. This function handles the race condition between Plan creation and secret availability.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ocp_admin_client` | `DynamicClient` | OpenShift admin dynamic client |
| `namespace` | `str` | Namespace where the plan and secret reside |
| `plan_name` | `str` | Name of the Plan CR (the secret will be named `{plan_name}-*`) |

**Returns:** `None`

**Behavior:**

- Polls every 2 seconds for up to 60 seconds
- Checks for any `Secret` resource in the given namespace whose name starts with `{plan_name}-`
- On timeout, logs a warning but does **not** raise an exception — execution continues
- If the secret is genuinely missing, the migration itself will fail with a clearer error

**Implementation:**

```python
LOGGER.info("Copy-offload: waiting for Forklift to create plan-specific secret...")
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
    LOGGER.warning(f"Timeout waiting for plan secret '{plan_name}-*' - continuing anyway")
```

> **Note:** The 60-second timeout with a non-fatal warning is intentional. In some environments, the secret may take longer to appear, but stopping the entire migration pipeline would be premature. The migration controller itself will surface a clear error if the secret is truly absent.

**Usage in the codebase:**

This function is called from `create_plan_resource()` in `utilities/mtv_migration.py`, but only when the `copyoffload` parameter is `True`:

```python
# From utilities/mtv_migration.py — create_plan_resource()
plan = create_and_store_resource(**plan_kwargs)

try:
    plan.wait_for_condition(
        condition=Plan.Condition.READY,
        status=Plan.Condition.Status.TRUE,
        timeout=360,
    )
except TimeoutExpiredError:
    LOGGER.error(f"Plan {plan.name} failed to reach status {Plan.Condition.Status.TRUE}")
    raise

# Wait for Forklift to create plan-specific secret for copy-offload (race condition)
if copyoffload:
    wait_for_plan_secret(ocp_admin_client, target_namespace, plan.name)
```

---

## Integration with Fixtures

The copy-offload utility functions are consumed by several session-scoped and class-scoped fixtures in `conftest.py`. The diagram below shows the dependency chain:

```
copyoffload_config (session)          ← validates provider type, config, credentials
    ├── copyoffload_storage_secret (session)  ← creates OCP Secret with storage creds
    │       uses: get_copyoffload_credential(), SUPPORTED_VENDORS
    └── setup_copyoffload_ssh (session)       ← installs SSH key on ESXi (if SSH method)
            uses: get_copyoffload_credential()
```

### `copyoffload_config` Fixture

Session-scoped fixture that validates the entire copy-offload configuration before any test runs. It checks:

1. The source provider is a vSphere provider
2. A `copyoffload` section exists in the provider data
3. Required storage credentials (`storage_hostname`, `storage_username`, `storage_password`) are available via `get_copyoffload_credential()`
4. Required parameters (`storage_vendor_product`, `datastore_id`) are present

### `copyoffload_storage_secret` Fixture

Session-scoped fixture that creates an OpenShift `Secret` containing storage credentials for the copy-offload populator. It:

1. Retrieves base credentials using `get_copyoffload_credential()`
2. Validates `storage_vendor_product` against `SUPPORTED_VENDORS`
3. Adds vendor-specific fields (also resolved via `get_copyoffload_credential()`)
4. Creates the secret using `create_and_store_resource()` for proper lifecycle tracking

### `setup_copyoffload_ssh` Fixture

Session-scoped fixture that installs an SSH key on the ESXi host when the SSH clone method is configured. It uses `get_copyoffload_credential()` to resolve `esxi_host`, `esxi_user`, and `esxi_password`.

---

## Integration with `create_plan_resource()`

The `create_plan_resource()` function in `utilities/mtv_migration.py` accepts a `copyoffload` boolean parameter. When set to `True`, two things happen:

1. **PVC naming template** — Sets `pvc_name_template` to `"pvc"` so the volume populator framework can generate consistent PVC names
2. **Plan secret wait** — Calls `wait_for_plan_secret()` after the Plan reaches `READY` status

Test classes pass this flag from the test configuration:

```python
# From tests/test_copyoffload_migration.py
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
    copyoffload=prepared_plan.get("copyoffload", False),
)
```

The `copyoffload: True` flag in test configurations (defined in `tests/tests_config/config.py`) flows through `class_plan_config` / `prepared_plan` fixtures and is passed to `create_plan_resource()`.

---

## Test Configuration

Copy-offload test parameters are defined in `tests/tests_config/config.py`. Each test configuration includes `"copyoffload": True` to enable copy-offload behavior:

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

All copy-offload tests use the `@pytest.mark.copyoffload` marker (registered in `pytest.ini`) and reference the following fixtures via `@pytest.mark.usefixtures`:

```python
@pytest.mark.copyoffload
@pytest.mark.incremental
@pytest.mark.usefixtures("multus_network_name", "copyoffload_config", "setup_copyoffload_ssh", "cleanup_migrated_vms")
class TestCopyoffloadThinMigration:
    ...
```

The available copy-offload test classes cover a range of scenarios:

| Test Configuration Key | Scenario |
|----------------------|----------|
| `test_copyoffload_thin_migration` | Thin provisioned disk |
| `test_copyoffload_thick_lazy_migration` | Thick lazy zeroed disk |
| `test_copyoffload_multi_disk_migration` | VM with multiple disks |
| `test_copyoffload_multi_disk_different_path_migration` | Multi-disk across different datastores |
| `test_copyoffload_rdm_virtual_disk_migration` | RDM virtual disk |
| `test_copyoffload_thin_snapshots_migration` | Disk with snapshots |
| `test_copyoffload_multi_datastore_migration` | Multi-datastore configuration |
| `test_copyoffload_independent_persistent_disk_migration` | Independent persistent disk |
| `test_copyoffload_independent_nonpersistent_disk_migration` | Independent non-persistent disk |
| `test_copyoffload_10_mixed_disks_migration` | 10 mixed-type disks |
| `test_copyoffload_large_vm_migration` | Large VM with big disks |
| `test_copyoffload_warm_migration` | Warm (live) migration with copy-offload |
| `test_copyoffload_2tb_vm_snapshots_migration` | 2TB VM with snapshots |
| `test_copyoffload_mixed_datastore_migration` | Mixed XCOPY/non-XCOPY datastores |
| `test_simultaneous_copyoffload_migrations` | Concurrent migration plans |
| `test_copyoffload_nonconforming_name_migration` | VM with non-Kubernetes-conforming name |
| `test_copyoffload_fallback_large_migration` | Fallback to standard transfer for large disks |
| `test_copyoffload_scale_migration` | Scale test with multiple VMs |
| `test_concurrent_xcopy_vddk_migration` | Concurrent XCOPY and VDDK migrations |

---

## StorageMap Configuration for Copy-Offload

Copy-offload tests create a `StorageMap` with an `offloadPluginConfig` section that references the storage secret:

```python
# From tests/test_copyoffload_migration.py — test_create_storagemap
copyoffload_config_data = source_provider_data["copyoffload"]
storage_vendor_product = copyoffload_config_data["storage_vendor_product"]

offload_plugin_config = {
    "vsphereXcopyConfig": {
        "secretRef": copyoffload_storage_secret.name,
        "storageVendorProduct": storage_vendor_product,
    }
}

self.__class__.storage_map = get_storage_migration_map(
    fixture_store=fixture_store,
    target_namespace=target_namespace,
    source_provider=source_provider,
    destination_provider=destination_provider,
    ocp_admin_client=ocp_admin_client,
    source_provider_inventory=source_provider_inventory,
    vms=vms_names,
    storage_class=storage_class,
    datastore_id=datastore_id,
    offload_plugin_config=offload_plugin_config,
    access_mode="ReadWriteOnce",
    volume_mode="Block",
)
```

> **Note:** Copy-offload requires `volume_mode="Block"` and `access_mode="ReadWriteOnce"`. NFS-backed storage is not supported for copy-offload migrations.

---

## VMware Provider Integration

The `VMWareProvider` class in `libs/providers/vmware.py` stores the copy-offload configuration passed during provider creation:

```python
class VMWareProvider(BaseProvider):
    def __init__(self, host, username, password, ocp_resource=None, **kwargs):
        self.copyoffload_config = kwargs.pop("copyoffload", {})
        super().__init__(...)
```

This configuration is used by the provider for:

- **ESXi clone method** — Patches the provider's `esxiCloneMethod` setting when SSH is configured
- **VM cloning** — Uses `copyoffload_config["datastore_id"]` and `copyoffload_config["esxi_host"]` as defaults for clone target location
- **RDM disk attachment** — Reads `copyoffload_config["rdm_lun_uuid"]` and `copyoffload_config["datastore_id"]` for RDM operations
- **Multi-datastore support** — Resolves `secondary_datastore_id` and `non_xcopy_datastore_id` from the copy-offload config for disk placement during cloning

The provider also gets an annotation when copy-offload is configured:

```python
# From conftest.py — provider creation
if vmware_provider(provider_data=source_provider_data_copy) and has_copyoffload:
    provider_annotations["forklift.konveyor.io/empty-vddk-init-image"] = "yes"
```

This annotation tells the ForkliftController that VDDK is not needed for copy-offload migrations.
