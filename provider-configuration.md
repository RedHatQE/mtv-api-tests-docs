# Provider Configuration

The MTV API Tests framework requires a `.providers.json` file in the project root to define source provider credentials and connection details. This file tells the test suite how to connect to each source virtualization platform for migration testing.

## File Location and Security

The configuration file must be placed at the project root as `.providers.json`. This file is already listed in `.gitignore` and must never be committed to version control.

```
mtv-api-tests/
├── .providers.json          # Your provider config (gitignored)
├── .providers.json.example  # Reference template
├── conftest.py
└── ...
```

> **Warning:** The `.providers.json` file contains sensitive credentials (passwords, API keys). Always set restrictive permissions: `chmod 600 .providers.json`. Delete the file from local systems when no longer needed.

## File Structure

The file is a JSON object where each key is a provider identifier and each value is a provider configuration object. The key name is how you reference the provider when running tests:

```bash
pytest --tc=source_provider:vsphere-8.0.1 ...
```

The key must exactly match a top-level key in your `.providers.json`:

```json
{
  "vsphere-8.0.1": { ... },
  "ovirt-4.4.9": { ... },
  "openstack-queens": { ... },
  "ova-1.0": { ... },
  "openshift-4.14": { ... }
}
```

> **Note:** You only need to configure the providers you intend to test against. A file with a single provider entry is perfectly valid.

## How Providers Are Loaded

The framework loads providers at session startup via `load_source_providers()` in `utilities/utils.py`:

```python
def load_source_providers() -> dict[str, dict[str, Any]]:
    providers_file = Path(".providers.json")
    if not providers_file.exists():
        return {}

    with open(providers_file) as fd:
        content = fd.read()
        if not content.strip():
            return {}
        return json.loads(content)
```

The `source_provider_data` fixture in `conftest.py` resolves the requested provider from the loaded configurations. If the file is missing or the requested provider key is not found, the test session fails immediately with a clear error:

```python
if not source_providers:
    raise MissingProvidersFileError()  # "'.providers.json' file is missing or empty"

requested_provider = py_config["source_provider"]
if requested_provider not in source_providers:
    raise ValueError(
        f"Source provider '{requested_provider}' not found in '.providers.json'. "
        f"Available providers: {sorted(source_providers.keys())}"
    )
```

## Common Fields

All provider types share these base fields:

| Field      | Type   | Required | Description                              |
|------------|--------|----------|------------------------------------------|
| `type`     | string | Yes      | Provider type identifier (see below)     |
| `version`  | string | Yes      | Provider/platform version                |
| `fqdn`     | string | Varies   | Provider FQDN or IP address              |
| `api_url`  | string | Varies   | API endpoint URL                         |
| `username` | string | Varies   | Authentication username                  |
| `password` | string | Varies   | Authentication password                  |

Valid `type` values correspond to the `Provider.ProviderType` constants from `ocp_resources`:

| Type         | Constant                        |
|--------------|---------------------------------|
| `"vsphere"`  | `Provider.ProviderType.VSPHERE` |
| `"ovirt"`    | `Provider.ProviderType.RHV`     |
| `"openstack"`| `Provider.ProviderType.OPENSTACK`|
| `"ova"`      | `Provider.ProviderType.OVA`     |
| `"openshift"`| `Provider.ProviderType.OPENSHIFT`|

## SSL/TLS Certificate Handling

Provider connections support both secure and insecure (skip verification) modes, controlled by the `source_provider_insecure_skip_verify` test configuration parameter in `tests/tests_config/config.py`:

```python
source_provider_insecure_skip_verify: str = "false"  # SSL verification for source provider
```

Override at runtime:

```bash
pytest --tc=source_provider_insecure_skip_verify:true ...
```

When secure mode is enabled (the default), the framework automatically fetches CA certificates from the provider's FQDN using `openssl s_client` and stores them in the MTV Secret for the provider connection.

> **Note:** For RHV/oVirt, the CA certificate is always fetched regardless of the insecure setting because the `imageio` connection requires it.

---

## VMware vSphere

**Type:** `"vsphere"` | **Implementation:** `libs/providers/vmware.py` (`VMWareProvider`)

VMware vSphere providers connect to a vCenter Server using the vSphere SDK (pyVmomi).

### Configuration

```json
{
  "vsphere-8.0.1": {
    "type": "vsphere",
    "version": "8.0.1",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "your-password",
    "guest_vm_linux_user": "root",
    "guest_vm_linux_password": "vm-linux-password",
    "guest_vm_win_user": "Administrator",
    "guest_vm_win_password": "vm-windows-password",
    "vddk_init_image": "quay.io/kubev2v/vddk-init:latest"
  }
}
```

### Field Reference

| Field                    | Required | Description                                          |
|--------------------------|----------|------------------------------------------------------|
| `type`                   | Yes      | Must be `"vsphere"`                                  |
| `version`                | Yes      | vCenter Server version (e.g., `"8.0.1"`, `"7.0.3"`) |
| `fqdn`                  | Yes      | vCenter FQDN or IP (used for SSL certificate fetch)  |
| `api_url`                | Yes      | vSphere SDK endpoint (typically `https://<fqdn>/sdk`)|
| `username`               | Yes      | vCenter administrator account                        |
| `password`               | Yes      | vCenter administrator password                       |
| `guest_vm_linux_user`    | No       | SSH username for Linux guest VMs                     |
| `guest_vm_linux_password`| No       | SSH password for Linux guest VMs                     |
| `guest_vm_win_user`      | No       | Username for Windows guest VMs                       |
| `guest_vm_win_password`  | No       | Password for Windows guest VMs                       |
| `vddk_init_image`        | No       | VDDK init container image path for the Provider CR   |

### How Credentials Are Used

The framework creates a Kubernetes Secret with the vSphere credentials:

```python
# From utilities/utils.py - create_source_provider()
secret_string_data = {
    "url": source_provider_data["api_url"],
    "user": source_provider_data["username"],
    "password": source_provider_data["password"],
    "insecureSkipVerify": "true" if insecure else "false",
}
```

The `VMWareProvider` connects using `pyVim.connect.SmartConnect`:

```python
# From libs/providers/vmware.py
self.api = SmartConnect(
    host=self.host,        # from "fqdn"
    user=self.username,    # from "username"
    pwd=self.password,     # from "password"
    disableSslCertVerification=True,
)
```

> **Tip:** The `api_url` field must end with `/sdk`. The `fqdn` field is used separately for the pyVmomi connection and for SSL certificate fetching.

---

## RHV/oVirt

**Type:** `"ovirt"` | **Implementation:** `libs/providers/rhv.py` (`OvirtProvider`)

RHV/oVirt providers connect to the oVirt Engine REST API using the `ovirtsdk4` library.

### Configuration

```json
{
  "ovirt-4.4.9": {
    "type": "ovirt",
    "version": "4.4.9",
    "fqdn": "ovirt-engine.example.com",
    "api_url": "https://ovirt-engine.example.com/ovirt-engine/api",
    "username": "admin@internal",
    "password": "your-password"
  }
}
```

### Field Reference

| Field      | Required | Description                                                          |
|------------|----------|----------------------------------------------------------------------|
| `type`     | Yes      | Must be `"ovirt"`                                                    |
| `version`  | Yes      | RHV/oVirt Engine version (e.g., `"4.4.9"`)                          |
| `fqdn`     | Yes      | oVirt Engine FQDN or IP (used for SSL certificate fetch)             |
| `api_url`  | Yes      | oVirt REST API endpoint (typically `https://<fqdn>/ovirt-engine/api`)|
| `username` | Yes      | oVirt administrator account (e.g., `"admin@internal"`)               |
| `password` | Yes      | oVirt administrator password                                         |

### How Credentials Are Used

The `OvirtProvider` connects using the `ovirtsdk4.Connection`:

```python
# From libs/providers/rhv.py
self.api = ovirtsdk4.Connection(
    url=self.host,         # from "api_url"
    username=self.username,
    password=self.password,
    ca_file=self.ca_file if not self.insecure else None,
    insecure=self.insecure,
)
```

### Requirements

- An **MTV-CNV datacenter** must exist and be in `"up"` status on the oVirt Engine. The provider validates this on connection:

```python
# From libs/providers/rhv.py
@property
def is_mtv_datacenter_ok(self) -> bool:
    return self.mtv_datacenter.status.value == "up"
```

If the datacenter is not found or not in the `"up"` state, the provider raises `OvirtMTVDatacenterNotFoundError` or `OvirtMTVDatacenterStatusError`.

- RHV uses **templates** (not VMs) as source machines for cloning. When tests specify `"clone": true`, the framework clones from a template, not a running VM.

> **Note:** The CA certificate is always fetched from the oVirt Engine, even when `insecure=true`. This is because the `imageio` connection requires a valid certificate. The `insecureSkipVerify` flag controls only the SDK connection validation.

---

## OpenStack

**Type:** `"openstack"` | **Implementation:** `libs/providers/openstack.py` (`OpenStackProvider`)

OpenStack providers connect to Keystone and the Nova/Cinder/Neutron APIs using the `openstacksdk` library.

### Configuration

```json
{
  "openstack-queens": {
    "type": "openstack",
    "version": "queens",
    "fqdn": "keystone.example.com",
    "api_url": "https://keystone.example.com:5000/v3",
    "username": "admin",
    "password": "your-password",
    "user_domain_name": "Default",
    "region_name": "RegionOne",
    "project_name": "admin",
    "user_domain_id": "default",
    "project_domain_id": "default",
    "guest_vm_linux_user": "cloud-user",
    "guest_vm_linux_password": "vm-password"
  }
}
```

### Field Reference

| Field                    | Required | Description                                              |
|--------------------------|----------|----------------------------------------------------------|
| `type`                   | Yes      | Must be `"openstack"`                                    |
| `version`                | Yes      | OpenStack release name (e.g., `"queens"`, `"rocky"`)     |
| `fqdn`                  | Yes      | Keystone FQDN or IP (used for SSL certificate fetch)     |
| `api_url`                | Yes      | Keystone identity endpoint (e.g., `https://<fqdn>:5000/v3`)|
| `username`               | Yes      | OpenStack user account                                   |
| `password`               | Yes      | OpenStack user password                                  |
| `user_domain_name`       | Yes      | Keystone user domain name                                |
| `region_name`            | Yes      | OpenStack region name                                    |
| `project_name`           | Yes      | OpenStack project/tenant name                            |
| `user_domain_id`         | Yes      | Keystone user domain ID                                  |
| `project_domain_id`      | Yes      | Keystone project domain ID                               |
| `guest_vm_linux_user`    | No       | SSH username for Linux guest VMs                         |
| `guest_vm_linux_password`| No       | SSH password for Linux guest VMs                         |

### How Credentials Are Used

The framework creates a Kubernetes Secret with OpenStack-specific fields:

```python
# From utilities/utils.py - create_source_provider()
secret_string_data["username"] = source_provider_data["username"]
secret_string_data["password"] = source_provider_data["password"]
secret_string_data["regionName"] = source_provider_data["region_name"]
secret_string_data["projectName"] = source_provider_data["project_name"]
secret_string_data["domainName"] = source_provider_data["user_domain_name"]
```

The `OpenStackProvider` connects using `openstack.connection.Connection`:

```python
# From libs/providers/openstack.py
self.api = Connection(
    auth_url=self.auth_url,
    project_name=self.project_name,
    username=self.username,
    password=self.password,
    user_domain_name=self.user_domain_name,
    region_name=self.region_name,
    user_domain_id=self.user_domain_id,
    project_domain_id=self.project_domain_id,
)
```

> **Tip:** The `api_url` field serves double duty: it populates both the MTV Secret's `url` field and the OpenStack SDK's `auth_url` parameter. Make sure it points to the Keystone v3 identity endpoint.

---

## OVA

**Type:** `"ova"` | **Implementation:** `libs/providers/ova.py` (`OVAProvider`)

OVA providers reference an NFS share containing exported OVA files. This is the simplest provider type -- it does not connect to an external hypervisor API.

### Configuration

```json
{
  "ova-1.0": {
    "type": "ova",
    "version": "1.0",
    "fqdn": "",
    "api_url": "nfs://nfs-server.example.com/exports/ova-share",
    "username": "",
    "password": ""
  }
}
```

### Field Reference

| Field      | Required | Description                                     |
|------------|----------|-------------------------------------------------|
| `type`     | Yes      | Must be `"ova"`                                 |
| `version`  | Yes      | Arbitrary version placeholder (e.g., `"1.0"`)   |
| `fqdn`     | No       | Not used -- leave empty                          |
| `api_url`  | Yes      | NFS share URL containing OVA files               |
| `username` | No       | NFS authentication username (if required)        |
| `password` | No       | NFS authentication password (if required)        |

### Behavior

- The `OVAProvider` does not establish external connections. Its `connect()` method is a no-op.
- All VMs from OVA sources are reported with `power_state: "off"`.
- Network information comes from the Forklift inventory, not from the provider directly.
- VM cloning and deletion are not supported (they are no-ops).

---

## OpenShift

**Type:** `"openshift"` | **Implementation:** `libs/providers/openshift.py` (`OCPProvider`)

OpenShift source providers enable on-cluster VM migrations (CNV-to-CNV). Unlike other providers, OpenShift uses the local cluster credentials -- no external connection details are needed.

### Configuration

```json
{
  "openshift-4.14": {
    "type": "openshift",
    "version": "4.14",
    "fqdn": "",
    "api_url": "",
    "username": "",
    "password": ""
  }
}
```

### Field Reference

| Field      | Required | Description                                     |
|------------|----------|-------------------------------------------------|
| `type`     | Yes      | Must be `"openshift"`                           |
| `version`  | Yes      | OpenShift version (informational, not validated)  |
| `fqdn`     | No       | Not used -- leave empty                          |
| `api_url`  | No       | Auto-populated from cluster -- leave empty       |
| `username` | No       | Not used -- leave empty                          |
| `password` | No       | Not used -- leave empty                          |

### Behavior

The framework auto-populates the API URL from the current cluster connection and reuses the destination OCP secret:

```python
# From utilities/utils.py - create_source_provider()
if ocp_provider(provider_data=source_provider_data_copy):
    source_provider = OCPProvider
    source_provider_data_copy["api_url"] = ocp_admin_client.configuration.host
    source_provider_data_copy["type"] = Provider.ProviderType.OPENSHIFT
    source_provider_secret = destination_ocp_secret
```

> **Note:** The OpenShift provider uses the cluster client established via the `ocp_admin_client` fixture. Cluster credentials are configured separately through `--tc=cluster_host:...`, `--tc=cluster_username:...`, and `--tc=cluster_password:...`.

---

## VMware Copy-Offload (XCOPY) Configuration

Copy-offload enables direct storage array disk transfers (XCOPY/VAAI) for VMware migrations, bypassing the network data path. This is configured by adding a `copyoffload` object to a VMware provider entry.

> **Tip:** Create a separate provider entry (e.g., `"vsphere-copy-offload"`) rather than adding copy-offload config to your standard vSphere entry. This lets you run standard and copy-offload tests independently.

### Configuration

```json
{
  "vsphere-copy-offload": {
    "type": "vsphere",
    "version": "8.0.1",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "your-password",
    "guest_vm_linux_user": "root",
    "guest_vm_linux_password": "vm-password",
    "guest_vm_win_user": "Administrator",
    "guest_vm_win_password": "win-password",
    "copyoffload": {
      "storage_vendor_product": "ontap",
      "datastore_id": "datastore-12345",
      "default_vm_name": "xcopy-template-test",
      "storage_hostname": "storage.example.com",
      "storage_username": "admin",
      "storage_password": "storage-password",
      "ontap_svm": "vserver-name"
    }
  }
}
```

> **Note:** When `copyoffload` is present, the framework automatically adds the `forklift.konveyor.io/empty-vddk-init-image: "yes"` annotation to the Provider CR and omits the `vddk_init_image` field.

### Common Fields

| Field                    | Required | Description                                           |
|--------------------------|----------|-------------------------------------------------------|
| `storage_vendor_product` | Yes      | Storage array vendor (see supported list below)       |
| `datastore_id`           | Yes      | Primary vSphere datastore MoRef ID (e.g., `"datastore-12345"`) |
| `storage_hostname`       | Yes      | Storage array management IP or FQDN                   |
| `storage_username`       | Yes      | Storage array admin username                           |
| `storage_password`       | Yes      | Storage array admin password                           |
| `default_vm_name`        | No       | Default template VM name for cloning tests             |
| `secondary_datastore_id` | No       | Secondary datastore for multi-datastore tests          |
| `non_xcopy_datastore_id` | No       | Non-XCOPY datastore for mixed datastore tests          |

### Supported Storage Vendors

Defined in `utilities/copyoffload_constants.py`:

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

### Vendor-Specific Fields

Each vendor may require additional configuration fields. Only configure the fields for your selected `storage_vendor_product`:

| Vendor            | Field                          | Required | Description                               |
|-------------------|--------------------------------|----------|-------------------------------------------|
| `ontap`           | `ontap_svm`                    | Yes      | NetApp ONTAP SVM (vserver) name            |
| `vantara`         | `vantara_storage_id`           | Yes      | Storage array serial number                |
| `vantara`         | `vantara_storage_port`         | Yes      | Storage API port (typically `"443"`)       |
| `vantara`         | `vantara_hostgroup_id_list`    | Yes      | IO ports and host group IDs (format: `"CL1-A,1:CL2-B,2"`) |
| `pureFlashArray`  | `pure_cluster_prefix`          | Yes      | Cluster prefix (e.g., `"px_a1b2c3d4"`)    |
| `powerflex`       | `powerflex_system_id`          | Yes      | PowerFlex system ID                        |
| `powermax`        | `powermax_symmetrix_id`        | Yes      | PowerMax Symmetrix ID                      |
| `primera3par`     | --                             | --       | No additional fields required              |
| `powerstore`      | --                             | --       | No additional fields required              |
| `infinibox`       | --                             | --       | No additional fields required              |
| `flashsystem`     | --                             | --       | No additional fields required              |

### ESXi SSH Cloning (Optional)

For SSH-based VM cloning (instead of the default VIB method):

| Field               | Required       | Description                                    |
|---------------------|----------------|------------------------------------------------|
| `esxi_clone_method` | No             | `"vib"` (default) or `"ssh"`                   |
| `esxi_host`         | If SSH method  | ESXi host FQDN or IP                           |
| `esxi_user`         | If SSH method  | ESXi SSH username (typically `"root"`)          |
| `esxi_password`     | If SSH method  | ESXi SSH password                               |

### RDM Disk Testing (Optional)

| Field           | Required | Description                                          |
|-----------------|----------|------------------------------------------------------|
| `rdm_lun_uuid`  | No       | RDM disk LUN UUID (format: `"naa.xxx..."`)           |

> **Note:** The `datastore_id` must point to a VMFS datastore for RDM disk support.

### Environment Variable Overrides

All copy-offload credentials can be overridden via environment variables. Environment variables take precedence over `.providers.json` values. The naming convention is `COPYOFFLOAD_{FIELD_NAME_UPPER}`:

```bash
# Storage credentials
export COPYOFFLOAD_STORAGE_HOSTNAME="storage.example.com"
export COPYOFFLOAD_STORAGE_USERNAME="admin"
export COPYOFFLOAD_STORAGE_PASSWORD="secret"

# Vendor-specific
export COPYOFFLOAD_ONTAP_SVM="vserver-name"
export COPYOFFLOAD_VANTARA_STORAGE_ID="123456789"

# ESXi SSH
export COPYOFFLOAD_ESXI_HOST="esxi-host.example.com"
export COPYOFFLOAD_ESXI_USER="root"
export COPYOFFLOAD_ESXI_PASSWORD="esxi-secret"
```

This is implemented in `utilities/copyoffload_migration.py`:

```python
def get_copyoffload_credential(
    credential_name: str,
    copyoffload_config: dict[str, Any],
) -> str | None:
    env_var_name = f"COPYOFFLOAD_{credential_name.upper()}"
    return os.getenv(env_var_name) or copyoffload_config.get(credential_name)
```

> **Tip:** Environment variable overrides are useful in CI/CD pipelines where secrets should not be stored in files.

---

## Complete Example

A `.providers.json` file with all provider types configured:

```json
{
  "vsphere-8.0.1": {
    "type": "vsphere",
    "version": "8.0.1",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "secure-password",
    "guest_vm_linux_user": "root",
    "guest_vm_linux_password": "linux-vm-pass",
    "guest_vm_win_user": "Administrator",
    "guest_vm_win_password": "windows-vm-pass",
    "vddk_init_image": "quay.io/kubev2v/vddk-init:latest"
  },
  "vsphere-copy-offload": {
    "type": "vsphere",
    "version": "8.0.1",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "administrator@vsphere.local",
    "password": "secure-password",
    "guest_vm_linux_user": "root",
    "guest_vm_linux_password": "linux-vm-pass",
    "guest_vm_win_user": "Administrator",
    "guest_vm_win_password": "windows-vm-pass",
    "copyoffload": {
      "storage_vendor_product": "ontap",
      "datastore_id": "datastore-12345",
      "storage_hostname": "netapp.example.com",
      "storage_username": "admin",
      "storage_password": "storage-pass",
      "ontap_svm": "svm-prod"
    }
  },
  "ovirt-4.4.9": {
    "type": "ovirt",
    "version": "4.4.9",
    "fqdn": "ovirt-engine.example.com",
    "api_url": "https://ovirt-engine.example.com/ovirt-engine/api",
    "username": "admin@internal",
    "password": "ovirt-pass"
  },
  "openstack-queens": {
    "type": "openstack",
    "version": "queens",
    "fqdn": "keystone.example.com",
    "api_url": "https://keystone.example.com:5000/v3",
    "username": "admin",
    "password": "openstack-pass",
    "user_domain_name": "Default",
    "region_name": "RegionOne",
    "project_name": "admin",
    "user_domain_id": "default",
    "project_domain_id": "default",
    "guest_vm_linux_user": "cloud-user",
    "guest_vm_linux_password": "cloud-pass"
  },
  "ova-1.0": {
    "type": "ova",
    "version": "1.0",
    "fqdn": "",
    "api_url": "nfs://nfs-server.example.com/exports/ova-share",
    "username": "",
    "password": ""
  },
  "openshift-4.14": {
    "type": "openshift",
    "version": "4.14",
    "fqdn": "",
    "api_url": "",
    "username": "",
    "password": ""
  }
}
```

## Selecting a Provider at Runtime

Pass the provider key to the test session using `--tc=source_provider:<key>`:

```bash
# Run tests against VMware vSphere
pytest --tc-file=tests/tests_config/config.py \
       --tc=source_provider:vsphere-8.0.1 \
       tests/

# Run copy-offload tests
pytest --tc-file=tests/tests_config/config.py \
       --tc=source_provider:vsphere-copy-offload \
       -m copyoffload \
       tests/

# Run against RHV with insecure SSL
pytest --tc-file=tests/tests_config/config.py \
       --tc=source_provider:ovirt-4.4.9 \
       --tc=source_provider_insecure_skip_verify:true \
       tests/
```

## Troubleshooting

### `MissingProvidersFileError`

```
'.providers.json' file is missing or empty
```

The `.providers.json` file does not exist in the project root or is empty. Create it from the example template:

```bash
cp .providers.json.example .providers.json
chmod 600 .providers.json
# Edit with your actual credentials
```

### Provider Not Found

```
Source provider 'my-provider' not found in '.providers.json'.
Available providers: ['ovirt-4.4.9', 'vsphere-8.0.1']
```

The `--tc=source_provider` value does not match any key in `.providers.json`. Check spelling and case sensitivity.

### `OvirtMTVDatacenterNotFoundError` / `OvirtMTVDatacenterStatusError`

The RHV provider requires a datacenter named `MTV-CNV` in `"up"` status on the oVirt Engine. Verify this exists before running RHV tests.

### SSL Certificate Failures

If certificate fetching fails (e.g., `Failed to download valid certificate from <fqdn>`), verify that:
1. The `fqdn` field is reachable on port 443
2. The server presents a valid certificate chain
3. For testing, use `--tc=source_provider_insecure_skip_verify:true` to skip validation
