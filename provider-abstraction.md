# Provider Abstraction Layer

The Provider Abstraction Layer is the foundational architecture that enables mtv-api-tests to interact with multiple virtualization platforms through a single, unified interface. It abstracts the differences between VMware vSphere, Red Hat Virtualization (RHV/oVirt), OpenStack, OpenShift Virtualization (CNV), and OVA sources behind a common `BaseProvider` contract.

## Architecture Overview

```
                        ┌──────────────────┐
                        │   BaseProvider    │  (Abstract Base Class)
                        │                  │
                        │  connect()       │
                        │  disconnect()    │
                        │  vm_dict()       │
                        │  clone_vm()      │
                        │  delete_vm()     │
                        │  get_vm_or_      │
                        │  template_       │
                        │  networks()      │
                        └────────┬─────────┘
                                 │
          ┌──────────┬───────────┼───────────┬──────────┐
          │          │           │           │          │
    ┌─────┴────┐ ┌───┴───┐ ┌────┴─────┐ ┌───┴───┐ ┌───┴───┐
    │ VMWare   │ │ Ovirt │ │OpenStack │ │  OCP  │ │  OVA  │
    │ Provider │ │Provider│ │ Provider │ │Provider│ │Provider│
    │          │ │       │ │          │ │       │ │       │
    │ pyVmomi  │ │oVirt  │ │OpenStack │ │ocp-   │ │(static│
    │          │ │SDK    │ │SDK       │ │resources│ │files) │
    └──────────┘ └───────┘ └──────────┘ └───────┘ └───────┘
```

Each provider communicates with its source platform through a native SDK, but exposes a uniform API to the test framework. Tests interact only with the `BaseProvider` interface and never reference platform-specific types directly.

---

## BaseProvider Abstract Class

**Location:** `libs/base_provider.py`

The `BaseProvider` class defines the contract that all provider implementations must follow. It is an abstract base class using Python's `abc.ABC`.

### Class Attributes

```python
class BaseProvider(abc.ABC):
    # Unified Representation of a VM of All Provider Types
    VIRTUAL_MACHINE_TEMPLATE: dict[str, Any] = {
        "id": "",
        "name": "",
        "provider_type": "",  # "ovirt" / "vsphere" / "openstack"
        "provider_vm_api": None,
        "network_interfaces": [],
        "disks": [],
        "cpu": {},
        "memory_in_mb": 0,
        "snapshots_data": [],
        "power_state": "",
    }
```

### Constructor

The constructor accepts common parameters shared across all providers:

```python
def __init__(
    self,
    fixture_store: dict[str, Any] | None = None,
    ocp_resource: Provider | None = None,
    username: str | None = None,
    password: str | None = None,
    host: str | None = None,
    debug: bool = False,
    log: Logger | None = None,
) -> None:
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `fixture_store` | `dict \| None` | Shared store for tracking created resources and teardown |
| `ocp_resource` | `Provider \| None` | The OCP Provider custom resource (Forklift CR) |
| `username` | `str \| None` | Credentials for the source platform |
| `password` | `str \| None` | Credentials for the source platform |
| `host` | `str \| None` | Source platform endpoint |
| `debug` | `bool` | Enable debug logging for the SDK connection |
| `log` | `Logger \| None` | Custom logger instance |

### Abstract Methods

Every provider implementation must implement these methods:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `connect()` | `-> Any` | Establish connection to the source platform |
| `disconnect()` | `-> Any` | Close the platform connection |
| `test` | `@property -> bool` | Verify connectivity to the source platform |
| `vm_dict()` | `(**kwargs) -> dict[str, Any]` | Build a unified VM dictionary from platform-specific data |
| `clone_vm()` | `(source_vm_name, clone_vm_name, session_uuid, **kwargs) -> Any` | Clone a VM on the source platform |
| `delete_vm()` | `(vm_name) -> Any` | Delete a VM from the source platform |
| `get_vm_or_template_networks()` | `(names, inventory) -> list[dict[str, str]]` | Get network mappings for VMs or templates |

### Helper Methods

`_generate_clone_vm_name` produces unique, Kubernetes-safe clone names:

```python
def _generate_clone_vm_name(self, session_uuid: str, base_name: str) -> str:
    clone_vm_name = generate_name_with_uuid(f"{session_uuid}-{base_name}")
    if len(clone_vm_name) > 63:
        self.log.warning(f"VM name '{clone_vm_name}' is too long ({len(clone_vm_name)} > 63). Truncating.")
        clone_vm_name = clone_vm_name[-63:]
    return clone_vm_name
```

The `generate_name_with_uuid` function (from `utilities/naming.py`) appends a random 4-character suffix and normalizes the name:

```python
def generate_name_with_uuid(name: str) -> str:
    _name = f"{name}-{shortuuid.ShortUUID().random(length=4).lower()}"
    _name = _name.replace("_", "-").replace(".", "-").lower()
    return _name
```

> **Note:** Clone names are truncated to 63 characters (keeping the last 63) to comply with the Kubernetes DNS-1123 naming limit. The UUID suffix is preserved because it appears at the end.

---

## Context Manager Lifecycle

All providers implement Python's context manager protocol, enabling clean connection management:

```python
# BaseProvider implements __enter__ and __exit__
def __enter__(self):
    self.connect()
    return self

def __exit__(self, exc_type, exc_val, exc_tb):
    return
```

The `create_source_provider` factory function (in `utilities/utils.py`) uses this protocol:

```python
with source_provider(ocp_resource=ocp_resource_provider, **provider_args) as _source_provider:
    if not _source_provider.test:
        pytest.fail(f"{source_provider.type} provider is not available.")
    yield _source_provider
```

And the `source_provider` fixture calls `disconnect()` explicitly after the context manager exits:

```python
@pytest.fixture(scope="session")
def source_provider(fixture_store, session_uuid, source_provider_data, ...):
    with create_source_provider(
        fixture_store=fixture_store,
        session_uuid=session_uuid,
        source_provider_data=source_provider_data,
        ...
    ) as _source_provider:
        yield _source_provider

    _source_provider.disconnect()
```

The lifecycle flows as:

1. **`__enter__`** -- calls `connect()`, establishes SDK connection
2. **`test`** -- verifies the connection is functional
3. **`yield`** -- provider is used throughout the test session
4. **`disconnect()`** -- tears down the SDK connection

> **Warning:** The `__exit__` method in `BaseProvider` intentionally does nothing. Disconnection is handled explicitly by the fixture teardown because `disconnect()` must run after all tests complete, not when the `with` block exits.

---

## Provider Implementations

### VMWareProvider

**Location:** `libs/providers/vmware.py`
**SDK:** pyVmomi (`pyVim`, `pyVmomi.vim`)
**Provider Type:** `Provider.ProviderType.VSPHERE`

The VMware provider is the most feature-rich implementation, supporting disk provisioning types, copy-offload (XCOPY), RDM disks, Change Block Tracking for warm migrations, and VMware Tools guest info extraction.

#### Connection

```python
def connect(self) -> Self:
    self.api = SmartConnect(
        host=self.host,
        user=self.username,
        pwd=self.password,
        port=443,
        disableSslCertValidation=True,
    )
    return self

def disconnect(self) -> None:
    Disconnect(si=self.api)
```

The `test` property verifies connectivity by querying the authorization manager:

```python
@property
def test(self) -> bool:
    try:
        self.api.RetrieveContent().authorizationManager.description
        return True
    except Exception:
        return False
```

#### Copy-Offload Configuration

VMware supports XCOPY copy-offload for faster disk cloning. Configuration is passed via the `copyoffload` key in provider data:

```python
self.copyoffload_config = {
    "datastore_id": "datastore-123",        # Primary XCOPY-capable datastore
    "esxi_host": "esxi-01.example.com",     # Target ESXi host
    "secondary_datastore_id": "datastore-456",  # Secondary XCOPY datastore
    "non_xcopy_datastore_id": "datastore-789",  # Non-XCOPY datastore for comparison
    "rdm_lun_uuid": "...",                  # LUN UUID for RDM disks
    "esxi_clone_method": "ssh",             # Clone method: "ssh" or "vib" (default)
}
```

When copy-offload is configured, the provider patches the OCP Provider CR with the `esxiCloneMethod` setting and annotates it with `forklift.konveyor.io/empty-vddk-init-image`.

#### Disk Provisioning Types

```python
DISK_TYPE_MAP = {
    "thin": ("sparse", "Setting disk provisioning to 'thin' (sparse)."),
    "thick-lazy": ("flat", "Setting disk provisioning to 'thick-lazy' (flat)."),
    "thick-eager": ("eagerZeroedThick", "Setting disk provisioning to 'thick-eager' (eagerZeroedThick)."),
}
```

#### VM Dictionary Extensions

Beyond the base template fields, VMware adds:

- `uuid` -- VMware VM UUID
- `guest_agent_running` -- VMware Tools status (`toolsOk`)
- `win_os` -- Windows detection from `guestId`
- `ip_addresses` -- Full IP configuration per NIC (address, subnet, gateway, DNS, static/DHCP origin)
- `controller_key` and `unit_number` on disk entries

#### RDM Disk Support

The `add_rdm_disk_to_vm` method adds Raw Device Mapping disks post-clone:

```python
def add_rdm_disk_to_vm(self, vm: vim.VirtualMachine, rdm_type: Literal["virtual", "physical"]) -> None:
```

> **Note:** RDM disks must be added after cloning because they require a VMFS datastore. The method supports both `virtual` and `physical` compatibility modes.

---

### OvirtProvider

**Location:** `libs/providers/rhv.py`
**SDK:** ovirtsdk4
**Provider Type:** `Provider.ProviderType.RHV`

The oVirt provider has a key architectural difference from other providers: in RHV, VMs are cloned from **templates**, not from other VMs.

#### Connection

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

> **Warning:** The oVirt provider validates that the `MTV-CNV` datacenter exists and is in `up` status during connection. If this datacenter is unavailable, the connection raises `OvirtMTVDatacenterStatusError`.

#### CA Certificate Handling

The RHV provider always fetches the CA certificate, even when `insecure=True`. This is because the certificate is required for the imageio connection used during data transfer, regardless of the `insecureSkipVerify` setting on the Provider CR.

```python
# From create_source_provider in utilities/utils.py
# Always fetch CA certificate for RHV provider, even when insecure=True
cert_file = _fetch_and_store_cacert(source_provider_data_copy, secret_string_data, tmp_dir, session_uuid)

# Set ca_file in provider_args only when secure mode (for SDK connection)
if not insecure:
    provider_args["ca_file"] = str(cert_file)
else:
    provider_args["insecure"] = insecure
```

#### Template-Based Cloning

In RHV, `clone_vm` receives a template name (not a VM name) as `source_vm_name`:

```python
def clone_vm(self, source_vm_name: str, clone_vm_name: str, session_uuid: str, ...) -> types.Vm:
    clone_vm_name = self._generate_clone_vm_name(session_uuid=session_uuid, base_name=clone_vm_name)

    template = self.get_template_by_name(name=source_vm_name)

    new_vm = self.vms_services.add(
        vm=types.Vm(
            name=clone_vm_name,
            cluster=types.Cluster(id=template.cluster.id),
            template=types.Template(id=template.id),
            memory_policy=types.MemoryPolicy(guaranteed=0),
        ),
        clone=True,
    )
```

> **Note:** The `memory_policy` is set with `guaranteed=0` to prevent memory overcommit issues during testing.

#### Network Resolution

Because RHV templates are not present in the Forklift inventory, `get_vm_or_template_networks` queries the oVirt API directly instead of delegating to the inventory:

```python
def get_vm_or_template_networks(self, names, inventory):
    # Inventory is ignored for RHV - we use direct template API
    return self.get_template_networks(template_names=names)
```

The `get_template_networks` method resolves vNIC profiles to their associated networks:

```python
for nic in template_service.nics_service().list():
    nic_detail = self.api.follow_link(nic)
    network = self.api.follow_link(self.api.follow_link(nic_detail.vnic_profile).network)
```

---

### OpenStackProvider

**Location:** `libs/providers/openstack.py`
**SDK:** openstack (OpenStack SDK)
**Provider Type:** `Provider.ProviderType.OPENSTACK`

The OpenStack provider works with volume-backed instances, requiring a multi-step snapshot-based cloning process.

#### Connection

```python
def connect(self) -> Self:
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
    return self
```

The constructor requires several OpenStack-specific parameters beyond the base class:

| Parameter | Description |
|-----------|-------------|
| `auth_url` | Keystone authentication URL |
| `project_name` | OpenStack project/tenant name |
| `user_domain_name` | User domain name |
| `region_name` | OpenStack region |
| `user_domain_id` | User domain ID |
| `project_domain_id` | Project domain ID |

#### Clone Process

OpenStack cloning is the most complex operation, involving snapshots and volume recreation:

```
Source VM ──► Glance Snapshot ──► Volume Snapshots ──► New Volumes ──► New Server
                                                                         │
                                        Glance Snapshot Cleanup ◄────────┘
```

The key steps in `clone_vm`:

1. **Create server image snapshot** (Glance)
2. **Find volume snapshots** created by OpenStack from the image snapshot
3. **Sort volumes by device path** (`/dev/vda`, `/dev/vdb`, ...) to preserve attachment order
4. **Create new volumes** from snapshots with correct boot indices
5. **Build block device mapping** (BDM) ensuring exactly one boot volume at index 0
6. **Create new server** with the BDM
7. **Cleanup** Glance image snapshot; track volume snapshots for session cleanup

```python
# Validate exactly one boot volume exists
if sum(1 for item in bdm if item["boot_index"] == 0) != 1:
    raise ValueError(f"Expected exactly 1 boot volume for '{clone_vm_name}'")

# Ensure boot volume is first in BDM
bdm.sort(key=lambda x: x["boot_index"])
```

#### VM Dictionary Extensions

OpenStack adds:

- `win_os` -- Windows detection from volume image metadata (`os_type`) or Glance image metadata
- CPU topology is extracted from boot volume metadata (`hw_cpu_cores`, `hw_cpu_threads`, `hw_cpu_sockets`)
- Memory comes from the flavor object

---

### OCPProvider

**Location:** `libs/providers/openshift.py`
**SDK:** ocp_resources (openshift-python-wrapper)
**Provider Type:** `Provider.ProviderType.OPENSHIFT`

The OpenShift provider is used both as a **destination provider** (always) and optionally as a **source provider** for CNV-to-CNV migrations.

#### Connection

The OCP provider has no external connection -- it uses the existing OpenShift client:

```python
def connect(self) -> Self:
    return self

def disconnect(self) -> None:
    pass
```

#### No Cloning

OpenShift VMs are not cloned. The `clone_vm` and `delete_vm` methods are no-ops:

```python
def clone_vm(self, source_vm_name, clone_vm_name, session_uuid, **kwargs):
    return

def delete_vm(self, vm_name):
    return
```

Instead, source CNV VMs are created by `create_source_cnv_vms` in `utilities/utils.py` using `VirtualMachineFromInstanceType`.

#### VM Name Sanitization

When used as a destination provider, VM names are sanitized to comply with Kubernetes DNS-1123:

```python
if not _source:
    try:
        cnv_vm_name = sanitize_kubernetes_name(cnv_vm_name)
    except InvalidVMNameError:
        raise
```

The `sanitize_kubernetes_name` function converts uppercase, underscores, and dots to lowercase hyphens.

#### VM Dictionary Extensions

OpenShift adds several fields unique to CNV VMs:

- `serial` -- Firmware serial number (for serial preservation verification)
- `labels` -- Template labels from VMI metadata
- `affinity` -- VM affinity rules from template spec
- `node_name` -- The node where the VM is scheduled
- `guest_agent_running` -- QEMU guest agent status (via `AgentConnected` condition)

> **Note:** Cloud-init volumes (`cloudinitdisk`, `cloudInitNoCloud`, `cloudinit`) are excluded from the disk list.

#### Guest Agent Waiting

```python
def wait_for_cnv_vm_guest_agent(self, vm_dict: dict[str, Any], timeout: int = 301) -> bool:
    # Waits for the AgentConnected condition to become True on the VMI
    for sample in sampler:
        conditions = sample.get("status", {}).get("conditions", {})
        agent_status = [
            condition
            for condition in conditions
            if condition.get("type") == "AgentConnected" and condition.get("status") == "True"
        ]
        if agent_status:
            return True
```

---

### OVAProvider

**Location:** `libs/providers/ova.py`
**SDK:** None (static file-based provider)
**Provider Type:** `Provider.ProviderType.OVA`

The OVA provider is the simplest implementation. OVA files are static exports -- there are no live VMs to connect to, clone, or manage.

```python
class OVAProvider(BaseProvider):
    def connect(self) -> Self:
        return self

    def disconnect(self) -> None:
        return

    @property
    def test(self) -> bool:
        return True

    def vm_dict(self, **kwargs: Any) -> dict[str, Any]:
        result_vm_info = copy.deepcopy(self.VIRTUAL_MACHINE_TEMPLATE)
        result_vm_info["provider_type"] = self.type
        result_vm_info["power_state"] = "off"
        return result_vm_info

    def clone_vm(self, source_vm_name, clone_vm_name, session_uuid, **kwargs):
        return

    def delete_vm(self, vm_name):
        return
```

The `prepared_plan` fixture overrides the VM list for OVA providers:

```python
if source_provider.type == Provider.ProviderType.OVA:
    plan["virtual_machines"] = [{"name": "1nisim-rhel9-efi"}]
```

> **Tip:** OVA tests verify that Forklift can import and migrate VMs from OVA/OVF export files without needing a live source environment.

---

## Provider Factory

**Location:** `utilities/utils.py` -- `create_source_provider()`

The factory function instantiates the correct provider class based on the provider type in the configuration data. It also creates the OpenShift Provider CR and its associated Secret.

### Type Detection

Provider type is determined by helper functions:

```python
def vmware_provider(provider_data: dict[str, Any]) -> bool:
    return provider_data["type"] == Provider.ProviderType.VSPHERE

def rhv_provider(provider_data: dict[str, Any]) -> bool:
    return provider_data["type"] == Provider.ProviderType.RHV

def openstack_provider(provider_data: dict[str, Any]) -> bool:
    return provider_data["type"] == Provider.ProviderType.OPENSTACK
```

### Factory Flow

```python
def create_source_provider(source_provider_data, namespace, admin_client, ...) -> Generator[BaseProvider]:
    # 1. Determine provider class and build provider-specific args
    if ocp_provider(provider_data=source_provider_data_copy):
        source_provider = OCPProvider
    elif vmware_provider(provider_data=source_provider_data_copy):
        source_provider = VMWareProvider
        provider_args["host"] = source_provider_data_copy["fqdn"]
    elif rhv_provider(provider_data=source_provider_data_copy):
        source_provider = OvirtProvider
    elif openstack_provider(provider_data=source_provider_data_copy):
        source_provider = OpenStackProvider
    elif ova_provider(provider_data=source_provider_data_copy):
        source_provider = OVAProvider

    # 2. Create Secret CR with provider credentials
    source_provider_secret = create_and_store_resource(
        resource=Secret, client=admin_client, string_data=secret_string_data, ...
    )

    # 3. Create Provider CR and wait for Ready
    ocp_resource_provider = create_and_store_resource(
        resource=Provider, client=admin_client, provider_type=source_provider_data_copy["type"], ...
    )
    ocp_resource_provider.wait_for_status(Provider.Status.READY, timeout=600)

    # 4. Instantiate provider with context manager
    with source_provider(ocp_resource=ocp_resource_provider, **provider_args) as _source_provider:
        yield _source_provider
```

### Destination Provider

The destination provider is always an `OCPProvider` representing the local OpenShift cluster:

```python
@pytest.fixture(scope="session")
def destination_provider(session_uuid, ocp_admin_client, target_namespace, fixture_store):
    kind_dict = {
        "apiVersion": "forklift.konveyor.io/v1beta1",
        "kind": "Provider",
        "metadata": {"name": f"{session_uuid}-local-ocp-provider", "namespace": target_namespace},
        "spec": {"secret": {}, "type": "openshift", "url": ""},
    }

    provider = create_and_store_resource(
        fixture_store=fixture_store, resource=Provider, kind_dict=kind_dict, client=ocp_admin_client,
    )

    return OCPProvider(ocp_resource=provider, fixture_store=fixture_store)
```

---

## Unified VM Dictionary Template

The `VIRTUAL_MACHINE_TEMPLATE` defines a standard structure that all providers populate. This enables provider-agnostic post-migration validation -- the same `check_vms` function works regardless of source platform.

### Base Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Platform-specific VM identifier |
| `name` | `str` | VM name |
| `provider_type` | `str` | Provider type (`vsphere`, `ovirt`, `openstack`, `openshift`, `ova`) |
| `provider_vm_api` | `Any` | Native VM object (`vim.VirtualMachine`, `types.Vm`, `OSP_Server`, `VirtualMachine`) |
| `network_interfaces` | `list[dict]` | Network adapter details |
| `disks` | `list[dict]` | Disk/storage details |
| `cpu` | `dict` | CPU topology |
| `memory_in_mb` | `int` | Memory in megabytes |
| `snapshots_data` | `list[dict]` | Snapshot information |
| `power_state` | `str` | `"on"`, `"off"`, or `"other"` |

### Network Interface Structure

```python
{
    "name": "Network adapter 1",       # Device label
    "macAddress": "00:50:56:xx:xx:xx", # MAC address
    "network": {
        "name": "VM Network",          # Network name
        "id": "network-123",           # Network ID (provider-specific)
    },
    # VMware only:
    "ip_addresses": [
        {
            "ip_address": "192.168.1.10",
            "subnet_mask": 24,
            "gateway": "192.168.1.1",
            "dns_servers": ["8.8.8.8"],
            "ip_origin": "manual",
            "is_static_ip": True,
        }
    ],
}
```

### Disk Structure

```python
{
    "name": "Hard disk 1",
    "size_in_kb": 10485760,
    "storage": {
        "name": "datastore1",           # Datastore/storage domain/AZ/storage class
        "id": "datastore-123",          # Storage identifier (not all providers)
        "access_mode": ["ReadWriteOnce"],  # OpenShift only
    },
    "device_key": 2000,                # Provider-specific identifier
    "unit_number": 0,                  # VMware, OpenShift
    "controller_key": 1000,            # VMware only
}
```

### Provider-Specific Extensions

| Field | Providers | Description |
|-------|-----------|-------------|
| `uuid` | VMware | vSphere VM UUID |
| `win_os` | VMware, RHV, OpenStack | `True` if Windows guest OS detected |
| `guest_agent_running` | VMware, OpenShift | Guest agent/tools status |
| `serial` | OpenShift | Firmware serial number |
| `labels` | OpenShift | VMI template labels |
| `affinity` | OpenShift | VM affinity rules |
| `node_name` | OpenShift | Scheduled Kubernetes node |

---

## Clone Operations

Cloning is the process of creating a test-specific copy of a source VM (or template) before migration. Each provider implements cloning differently based on platform capabilities.

### Clone Workflow in Tests

The `prepared_plan` fixture orchestrates the clone process:

```python
for vm in virtual_machines:
    # 1. Clone or locate VM
    clone_options = {**vm, "enable_ctk": warm_migration}
    provider_vm_api = source_provider.get_vm_by_name(
        query=vm["name"],
        vm_name_suffix=vm_name_suffix,
        clone_vm=True,
        session_uuid=fixture_store["session_uuid"],
        clone_options=clone_options,
    )

    # 2. Set desired power state
    if source_vm_power == "on":
        source_provider.start_vm(provider_vm_api)
    elif source_vm_power == "off":
        source_provider.stop_vm(provider_vm_api)

    # 3. Build unified VM dict with VM in correct state
    source_vm_details = source_provider.vm_dict(
        provider_vm_api=provider_vm_api, ...
    )

    # 4. Wait for Forklift inventory sync
    source_provider_inventory.wait_for_vm(name=vm["name"], timeout=300)

    # 5. Store VM data in plan
    plan["source_vms_data"][vm["name"]] = source_vm_details
```

### Clone Comparison by Provider

| Aspect | VMware | RHV | OpenStack | OCP | OVA |
|--------|--------|-----|-----------|-----|-----|
| Source | VM | Template | Server | N/A | N/A |
| Mechanism | `CloneVM_Task` | `vms_service.add(clone=True)` | Snapshot → Volumes → Server | No cloning | No cloning |
| MAC Regeneration | Configurable | N/A | N/A | N/A | N/A |
| Disk Options | Thin/Thick-Lazy/Thick-Eager | Inherits template | Inherits volumes | N/A | N/A |
| Multi-disk | Yes (add_disks) | Inherits template | Preserves boot order | N/A | N/A |
| Post-clone | VMX health check, RDM | Memory policy reset | Glance cleanup | N/A | N/A |

### Cleanup Tracking

All cloned VMs are registered in `fixture_store["teardown"]` for automatic cleanup:

```python
# From OvirtProvider.clone_vm
if self.fixture_store:
    self.fixture_store["teardown"].setdefault(self.type, []).append({
        "name": clone_vm_name,
    })
```

This ensures cloned VMs are deleted when the test session ends, even if tests fail.

---

## Forklift Inventory Integration

**Location:** `libs/forklift_inventory.py`

The `ForkliftInventory` abstract class provides a separate abstraction for querying Forklift's inventory API. While `BaseProvider` interacts with source platforms directly, `ForkliftInventory` queries the data that Forklift has synced from those platforms.

### Inventory Classes

| Class | Provider Type | Storage Query | Network Query |
|-------|--------------|---------------|---------------|
| `VsphereForkliftInventory` | `VSPHERE` | Datastores | VM network references |
| `OvirtForkliftInventory` | `RHV` | Storage domains | vNIC profiles → networks |
| `OpenstackForliftinventory` | `OPENSTACK` | Volumes by type | VM addresses → networks |
| `OvaForkliftInventory` | `OVA` | Storage paths | VM network references |
| `OpenshiftForkliftInventory` | `OPENSHIFT` | PVC storage classes | Multus/pod networks |

### Key Methods

- `wait_for_vm(name)` -- Waits for a cloned VM to sync to inventory (critical for OpenStack which also waits for volume and network sync)
- `vms_storages_mappings(vms)` -- Returns storage mappings needed for StorageMap CR creation
- `vms_networks_mappings(vms)` -- Returns network mappings needed for NetworkMap CR creation

### Provider-Inventory Relationship

The `source_provider_inventory` fixture maps each provider type to its inventory class:

```python
@pytest.fixture(scope="session")
def source_provider_inventory(ocp_admin_client, mtv_namespace, source_provider):
    providers = {
        Provider.ProviderType.OVA: OvaForkliftInventory,
        Provider.ProviderType.RHV: OvirtForkliftInventory,
        Provider.ProviderType.VSPHERE: VsphereForkliftInventory,
        Provider.ProviderType.OPENSHIFT: OpenshiftForkliftInventory,
        Provider.ProviderType.OPENSTACK: OpenstackForliftinventory,
    }
    provider_instance = providers.get(source_provider.type)
    return provider_instance(
        client=ocp_admin_client,
        namespace=mtv_namespace,
        provider_name=source_provider.ocp_resource.name,
    )
```

---

## Provider Configuration

Source provider configuration is loaded from a `.providers.json` file at runtime. The `source_provider` key in `py_config` selects which provider to use.

### VMware Configuration Example

```json
{
  "vsphere-8": {
    "type": "vsphere",
    "fqdn": "vcenter.example.com",
    "api_url": "https://vcenter.example.com/sdk",
    "username": "admin@vsphere.local",
    "password": "...",
    "vddk_init_image": "quay.io/kubev2v/vddk:latest",
    "copyoffload": {
      "datastore_id": "datastore-123",
      "esxi_host": "esxi-01.example.com",
      "secondary_datastore_id": "datastore-456",
      "non_xcopy_datastore_id": "datastore-789",
      "rdm_lun_uuid": "naa.xxx",
      "esxi_clone_method": "ssh"
    }
  }
}
```

### RHV Configuration Example

```json
{
  "rhv-4": {
    "type": "ovirt",
    "api_url": "https://rhvm.example.com/ovirt-engine/api",
    "username": "admin@internal",
    "password": "..."
  }
}
```

### OpenStack Configuration Example

```json
{
  "openstack-17": {
    "type": "openstack",
    "api_url": "https://keystone.example.com:5000/v3",
    "username": "admin",
    "password": "...",
    "project_name": "mtv-testing",
    "user_domain_name": "Default",
    "region_name": "regionOne",
    "user_domain_id": "default",
    "project_domain_id": "default"
  }
}
```

> **Tip:** The `source_provider_insecure_skip_verify` key in `py_config` controls whether TLS certificate verification is skipped for the source provider connection.
