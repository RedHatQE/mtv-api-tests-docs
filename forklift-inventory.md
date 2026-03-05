# Forklift Inventory API

The Forklift Inventory API layer provides a unified interface for querying the MTV (Migration Toolkit for Virtualization) inventory service. It abstracts provider-specific differences behind a common base class, allowing tests to discover VMs, resolve storage mappings, and build network mappings regardless of the source provider type.

**Source:** `libs/forklift_inventory.py`

## Architecture Overview

```
ForkliftInventory (abstract base)
├── OvirtForkliftInventory        (RHV / oVirt)
├── OpenstackForliftinventory     (OpenStack)
├── VsphereForkliftInventory      (VMware vSphere)
├── OvaForkliftInventory          (OVA files)
└── OpenshiftForkliftInventory    (OpenShift virtualization)
```

Each subclass connects to the same Forklift inventory REST API exposed via an OpenShift Route (`forklift-inventory`), but queries provider-specific endpoints (datastores vs. storage domains vs. volume types) and interprets the response structures unique to each virtualization platform.

## Base Class: `ForkliftInventory`

The abstract base class handles provider discovery, HTTP communication, VM lookup, and synchronization waiting. All provider-specific subclasses inherit these capabilities.

### Constructor

```python
class ForkliftInventory(abc.ABC):
    def __init__(
        self,
        client: DynamicClient,
        provider_name: str,
        mtv_namespace: str,
        provider_type: str,
    ) -> None:
```

| Parameter       | Type            | Description                                              |
|-----------------|-----------------|----------------------------------------------------------|
| `client`        | `DynamicClient` | Kubernetes dynamic client for API communication          |
| `provider_name` | `str`           | Name of the MTV Provider CR in the namespace             |
| `mtv_namespace` | `str`           | Namespace where MTV operator is deployed                 |
| `provider_type` | `str`           | Provider type constant from `Provider.ProviderType`      |

During initialization, the constructor:

1. Creates a `Route` object pointing to the `forklift-inventory` service
2. Resolves the Forklift-assigned provider ID by polling the inventory API (up to 180 seconds)
3. Builds internal URL path templates for subsequent queries

```python
self.route = Route(client=self.client, name="forklift-inventory", namespace=mtv_namespace)
self.provider_id = self._provider_id
self.provider_url_path = f"{self.provider_type}/{self.provider_id}"
self.vms_path = f"{self.provider_url_path}/vms"
```

### Core Methods

#### `_request(url_path)`

Executes HTTP GET requests against the Forklift inventory API.

```python
def _request(self, url_path: str = "") -> Any:
    return self.route.api_request(
        method="GET",
        url=f"https://{self.route.host}",
        action=f"providers{f'/{url_path}' if url_path else ''}",
    )
```

All public methods and properties build on `_request()` by passing the appropriate URL path segment.

#### `get_data()`

Returns the complete provider metadata from the inventory.

```python
def get_data(self) -> dict[str, Any]:
    return self._request(url_path=self.provider_url_path)
```

#### `get_vm(name)`

Retrieves detailed information for a specific VM by name. Iterates through the provider's VM list, matches by name, then fetches the full VM object by ID.

```python
def get_vm(self, name: str) -> dict[str, Any]:
    for _vm in self.vms:
        if _vm["name"] == name:
            return self._request(url_path=f"{self.vms_path}/{_vm['id']}")

    raise ValueError(f"VM {name} not found. Available VMs: {self.vms_names}")
```

> **Note:** The error message includes all available VM names to aid debugging when a VM is not found.

### Properties

| Property    | Return Type             | Description                                   |
|-------------|-------------------------|-----------------------------------------------|
| `vms`       | `list[dict[str, Any]]`  | All VMs for the provider (summary records)    |
| `vms_names` | `list[str]`             | Just the VM names (convenience accessor)      |
| `networks`  | `list[dict[str, Any]]`  | All networks for the provider                 |
| `storages`  | `list[dict[str, Any]]`  | Storage resources (abstract, subclass-defined) |

### Abstract Methods

Every subclass must implement these two methods:

```python
@abc.abstractmethod
def vms_storages_mappings(self, vms: list[str]) -> list[dict[str, str]]:
    """Return storage source entries for the given VM names."""

@abc.abstractmethod
def vms_networks_mappings(self, vms: list[str]) -> list[dict[str, str]]:
    """Return network source entries for the given VM names."""
```

These methods produce the `source` entries used when constructing `StorageMap` and `NetworkMap` custom resources for migration plans.

## Provider Discovery

The `_provider_id` property polls the inventory API until the provider appears. This handles the delay between creating a Provider CR and Forklift completing its initial sync.

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
        LOGGER.error(f"Timed out waiting for provider {self.provider_name} to appear in inventory.")
        raise
```

> **Warning:** If the provider is not found within 180 seconds, a `TimeoutExpiredError` is raised. This usually indicates the Provider CR was not created successfully or the inventory service is unreachable.

## Waiting for Provider Sync

### `wait_for_vm()`

After cloning a VM, the Forklift inventory must sync before the new VM is visible. The `wait_for_vm()` method polls the inventory until the VM appears and all dependent resources are ready.

```python
def wait_for_vm(self, name: str, timeout: int = 300, sleep: int = 10) -> dict[str, Any]:
```

| Parameter | Type  | Default | Description                            |
|-----------|-------|---------|----------------------------------------|
| `name`    | `str` | —       | VM name to wait for                    |
| `timeout` | `int` | 300     | Maximum wait time in seconds           |
| `sleep`   | `int` | 10      | Interval between polling attempts      |

**Basic behavior:** Polls `get_vm(name)` until the VM is found, returning the full VM dictionary.

**OpenStack-specific behavior:** For OpenStack providers, `wait_for_vm()` additionally waits for:
- Attached volumes to be synced and queryable
- VM network addresses to appear in the inventory

This extra synchronization is required because OpenStack volume and network data is synced separately from VM metadata.

**Usage in the test framework** (from `conftest.py`):

```python
# Wait for cloned VM to appear in Forklift inventory before proceeding
# OVA is excluded because it doesn't clone VMs (uses pre-existing files)
if source_provider.type != Provider.ProviderType.OVA:
    source_provider_inventory.wait_for_vm(name=vm["name"], timeout=300)
```

**Error messages** provide context for debugging:

- VM not found: `"VM '{name}' did not appear in Forklift inventory after {timeout}s. Available VMs: {vms_names}"`
- VM found but resources missing (OpenStack): `"VM '{name}' found in Forklift inventory but attached volumes or networks did not sync after {timeout}s."`

### OpenStack Volume Synchronization

The `_check_openstack_volumes_synced()` method verifies that every attached volume listed on an OpenStack VM is individually queryable through the inventory API.

```python
def _check_openstack_volumes_synced(self, vm: dict[str, Any], vm_name: str) -> bool:
    attached_volumes = vm.get("attachedVolumes", [])
    if not attached_volumes:
        return False

    for attached_volume in attached_volumes:
        volume_id = attached_volume.get("ID")
        if not volume_id:
            LOGGER.warning(f"VM '{vm_name}' has attached volume without ID - incomplete sync")
            return False

        try:
            self._request(url_path=f"{self.provider_url_path}/volumes/{volume_id}")
        except (ValueError, ConnectionError, TimeoutError) as e:
            LOGGER.warning(
                f"VM '{vm_name}' found but volume '{volume_id}' not yet queryable in inventory: {e}"
            )
            return False

    return True
```

> **Tip:** Volume sync issues are typically transient. If `wait_for_vm()` times out on volumes, increase the timeout or check that the OpenStack provider credentials have read access to the Cinder volume API.

### OpenStack Network Synchronization

The `_check_openstack_networks_synced()` method ensures all networks referenced in the VM's `addresses` field are present in the Forklift inventory's network list.

```python
def _check_openstack_networks_synced(self, vm: dict[str, Any], vm_name: str) -> bool:
    vm_addresses = vm.get("addresses", {})
    if not vm_addresses:
        return False

    vm_network_names = set(vm_addresses.keys())
    try:
        inventory_network_names = {net["name"] for net in self.networks}
    except (ValueError, ConnectionError, TimeoutError) as e:
        LOGGER.warning(f"VM '{vm_name}' found but networks not yet queryable in inventory: {e}")
        return False

    missing_networks = vm_network_names - inventory_network_names
    if missing_networks:
        LOGGER.debug(
            f"VM '{vm_name}' found but networks {missing_networks} not yet in inventory. "
            f"Available networks: {inventory_network_names}"
        )
        return False

    return True
```

## Provider-Specific Implementations

### VMware vSphere — `VsphereForkliftInventory`

Queries vSphere-specific inventory endpoints: datastores for storage, network objects for networking.

```python
class VsphereForkliftInventory(ForkliftInventory):
    def __init__(self, client: DynamicClient, provider_name: str, namespace: str) -> None:
        self.provider_type = Provider.ProviderType.VSPHERE
        super().__init__(
            client=client, provider_name=provider_name,
            mtv_namespace=namespace, provider_type=self.provider_type,
        )
```

| Aspect | Endpoint | Mapping Key |
|--------|----------|-------------|
| Storages | `{provider}/datastores` | `{"name": "<datastore_name>"}` |
| Networks | `{provider}/networks` | `{"name": "<network_name>"}` |

**Storage mapping logic:** Extracts the datastore ID from each VM disk's `datastore.id` field, resolves it against the datastores list to produce the datastore name.

```python
for _disk in _vm.get("disks", []):
    if _storage_id := _disk.get("datastore", {}).get("id"):
        if _storage_name_match := [_stg["name"] for _stg in _storages if _storage_id == _stg["id"]]:
            _mappings.append({"name": _storage_name_match[0]})
```

**Network mapping logic:** Resolves network IDs from the VM's `networks` list against the inventory networks list.

### RHV / oVirt — `OvirtForkliftInventory`

Queries oVirt-specific endpoints: storage domains for storage, NIC profiles for networking.

```python
class OvirtForkliftInventory(ForkliftInventory):
    def __init__(self, client: DynamicClient, provider_name: str, namespace: str) -> None:
        self.provider_type = Provider.ProviderType.RHV
        super().__init__(
            client=client, provider_name=provider_name,
            mtv_namespace=namespace, provider_type=self.provider_type,
        )
```

| Aspect | Endpoint | Mapping Key |
|--------|----------|-------------|
| Storages | `{provider}/storagedomains` | `{"name": "<storage_domain_name>"}` |
| Networks | `{provider}/nicprofiles`, `{provider}/networks` | `{"name": "<network_path>"}` |

**Storage mapping logic:** Follows a multi-step lookup:
1. Get disk attachment IDs from the VM
2. Query each disk to find its `storageDomain` reference
3. Resolve the storage domain ID to a name

**Network mapping logic:** Follows NIC profile references:
1. Get NIC profile IDs from the VM's NICs
2. Query the NIC profile to find its network reference
3. Resolve the network ID to a network `path` (not `name`)

> **Note:** RHV network mappings use the network `path` field rather than `name`, which includes the data center hierarchy path.

### OpenStack — `OpenstackForliftinventory`

Queries OpenStack-specific endpoints: volumes for storage, networks with address-based resolution for networking.

```python
class OpenstackForliftinventory(ForkliftInventory):
    def __init__(self, client: DynamicClient, provider_name: str, namespace: str) -> None:
        self.provider_type = Provider.ProviderType.OPENSTACK
        super().__init__(
            client=client, provider_name=provider_name,
            mtv_namespace=namespace, provider_type=self.provider_type,
        )
```

| Aspect | Endpoint | Mapping Key |
|--------|----------|-------------|
| Storages | `{provider}/volumes` | `{"name": "<volume_type>"}` |
| Networks | `{provider}/networks` | `{"id": "<network_id>", "name": "<network_name>"}` |

**Storage mapping logic:** Queries attached volumes, retrieves each volume's details, and extracts the `volumeType` field. Duplicates are filtered.

```python
for attached_volume in _vm.get("attachedVolumes", []):
    volume_id = attached_volume.get("ID")
    if not volume_id:
        continue
    volume_info = self._request(url_path=f"{self.provider_url_path}/volumes/{volume_id}")
    volume_type = volume_info.get("volumeType")
    if volume_type and not any(m.get("name") == volume_type for m in _mappings):
        _mappings.append({"name": volume_type})
```

**Network mapping logic:** Uses the VM's `addresses` dictionary keys (network names) and resolves them against the inventory networks list to get both `id` and `name`.

> **Note:** OpenStack network mappings include both `id` and `name` fields, unlike other providers that use only `name`. This is because OpenStack network mappings require the network ID for the `NetworkMap` source entry.

### OVA — `OvaForkliftInventory`

Queries OVA-specific endpoints with a unique storage matching strategy.

```python
class OvaForkliftInventory(ForkliftInventory):
    def __init__(self, client: DynamicClient, provider_name: str, namespace: str) -> None:
        self.provider_type = Provider.ProviderType.OVA
        super().__init__(
            client=client, provider_name=provider_name,
            mtv_namespace=namespace, provider_type=self.provider_type,
        )
```

| Aspect | Endpoint | Mapping Key |
|--------|----------|-------------|
| Storages | `{provider}/storages` | `{"id": "<storage_id>"}` |
| Networks | `{provider}/networks` | `{"name": "<network_name>"}` |

**Storage mapping logic:** Uses substring matching — if a VM name appears within a storage entry's name, that storage is included. This is unique to OVA where storage names embed the VM name.

```python
for _storage in _storages:
    if [vm for vm in vms if vm in _storage["name"]]:
        _mappings.append({"id": _storage["id"]})
```

> **Note:** OVA storage mappings use `id` rather than `name` as the mapping key. OVA network mappings use uppercase `"ID"` for network IDs from the VM object (`_network.get("ID")`), reflecting the OVA inventory API response format.

### OpenShift — `OpenshiftForkliftInventory`

Queries OpenShift-native resources: storage classes and VM spec networks (pod/multus).

```python
class OpenshiftForkliftInventory(ForkliftInventory):
    def __init__(self, client: DynamicClient, provider_name: str, namespace: str) -> None:
        self.provider_type = Provider.ProviderType.OPENSHIFT
        super().__init__(
            client=client, provider_name=provider_name,
            mtv_namespace=namespace, provider_type=self.provider_type,
        )
```

| Aspect | Endpoint | Mapping Key |
|--------|----------|-------------|
| Storages | `{provider}/storageclasses` | `{"name": "<storage_class_name>"}` |
| Networks | Derived from VM spec | `{"name": "<multus_network>"}` or `{"type": "pod"}` |

**Storage mapping logic:** Reads the VM's `spec.template.spec.volumes`, finds `dataVolume` entries, queries the corresponding `PersistentVolumeClaim`, and extracts the `storageClassName`.

```python
for _volume in _vm["object"]["spec"]["template"]["spec"]["volumes"]:
    if data_volume := _volume.get("dataVolume"):
        _pvc = PersistentVolumeClaim(
            name=data_volume["name"], namespace=_namespace,
            client=self.client, ensure_exists=True,
        )
        _storage_class = _pvc.instance.spec.storageClassName
        _mappings.append({"name": _storage_class})
```

**Network mapping logic:** Inspects the VM's `spec.template.spec.networks` list and produces either multus or pod network mappings:

```python
for _network in _vm["object"]["spec"]["template"]["spec"]["networks"]:
    if _multus := _network.get("multus"):
        _network_map = {"name": _multus.get("networkName")}
    elif isinstance(_network.get("pod"), dict):
        _network_map = {"type": "pod"}
```

## Mapping Output Formats Summary

Each provider returns mappings in a specific format that becomes the `source` entry in `StorageMap` and `NetworkMap` custom resources:

### Storage Mappings

| Provider  | Format                             | Example                               |
|-----------|------------------------------------|---------------------------------------|
| vSphere   | `{"name": "<datastore>"}`          | `{"name": "vsanDatastore"}`           |
| RHV       | `{"name": "<storage_domain>"}`     | `{"name": "hosted_storage"}`          |
| OpenStack | `{"name": "<volume_type>"}`        | `{"name": "__DEFAULT__"}`             |
| OVA       | `{"id": "<storage_id>"}`           | `{"id": "ova-storage-001"}`           |
| OpenShift | `{"name": "<storage_class>"}`      | `{"name": "ocs-storagecluster-ceph"}` |

### Network Mappings

| Provider  | Format                                         | Example                                     |
|-----------|-------------------------------------------------|----------------------------------------------|
| vSphere   | `{"name": "<network>"}`                         | `{"name": "VM Network"}`                     |
| RHV       | `{"name": "<network_path>"}`                    | `{"name": "/DC/cluster/network"}`             |
| OpenStack | `{"id": "<id>", "name": "<network>"}`           | `{"id": "abc-123", "name": "provider-net"}`  |
| OVA       | `{"name": "<network>"}`                         | `{"name": "nat"}`                             |
| OpenShift | `{"name": "<multus>"}` or `{"type": "pod"}`     | `{"type": "pod"}`                             |

## Fixture Integration

The `source_provider_inventory` session-scoped fixture in `conftest.py` automatically instantiates the correct subclass based on the source provider type:

```python
@pytest.fixture(scope="session")
def source_provider_inventory(
    ocp_admin_client: DynamicClient, mtv_namespace: str, source_provider: BaseProvider
) -> ForkliftInventory:
    if not source_provider.ocp_resource:
        raise ValueError("source_provider.ocp_resource is not set")

    providers = {
        Provider.ProviderType.OVA: OvaForkliftInventory,
        Provider.ProviderType.RHV: OvirtForkliftInventory,
        Provider.ProviderType.VSPHERE: VsphereForkliftInventory,
        Provider.ProviderType.OPENSHIFT: OpenshiftForkliftInventory,
        Provider.ProviderType.OPENSTACK: OpenstackForliftinventory,
    }
    provider_instance = providers.get(source_provider.type)

    if not provider_instance:
        raise ValueError(f"Provider {source_provider.type} not implemented")

    return provider_instance(
        client=ocp_admin_client,
        namespace=mtv_namespace,
        provider_name=source_provider.ocp_resource.name,
    )
```

> **Tip:** Since this is a session-scoped fixture, the provider ID resolution and initial API connection happen once per test session. All test classes within the session share the same inventory instance.

## Usage in Migration Workflows

### Creating a StorageMap

The `get_storage_migration_map()` function in `utilities/mtv_migration.py` uses the inventory to build storage mappings:

```python
storage_migration_map = source_provider_inventory.vms_storages_mappings(vms=vms)
for storage in storage_migration_map:
    storage_map_list.append({
        "destination": {"storageClass": target_storage_class},
        "source": storage,
    })
```

### Creating a NetworkMap

The `gen_network_map_list()` function in `utilities/utils.py` maps source networks to destination pod or multus networks:

```python
def gen_network_map_list(
    source_provider_inventory: ForkliftInventory,
    target_namespace: str,
    vms: list[str],
    multus_network_name: dict[str, str],
    pod_only: bool = False,
) -> list[dict[str, dict[str, str]]]:
    network_map_list: list[dict[str, dict[str, str]]] = []
    _destination_pod: dict[str, str] = {"type": "pod"}
    multus_counter = 1

    for index, network in enumerate(source_provider_inventory.vms_networks_mappings(vms=vms)):
        if pod_only or index == 0:
            _destination = _destination_pod
        else:
            multus_network_name_str = multus_network_name["name"]
            multus_namespace = multus_network_name["namespace"]
            nad_name = f"{multus_network_name_str}-{multus_counter}"
            _destination = {
                "name": nad_name,
                "namespace": multus_namespace,
                "type": "multus",
            }
            multus_counter += 1

        network_map_list.append({
            "destination": _destination,
            "source": network,
        })
    return network_map_list
```

The first source network is always mapped to a pod network. Additional networks are mapped to multus `NetworkAttachmentDefinition` resources with auto-generated names.

### Populating VM IDs in Plans

The `populate_vm_ids()` function in `utilities/utils.py` resolves VM names to Forklift inventory IDs, which are required when creating Plan CRs:

```python
def populate_vm_ids(plan: dict[str, Any], inventory: ForkliftInventory) -> None:
    if not isinstance(plan, dict) or not isinstance(plan.get("virtual_machines"), list):
        raise ValueError("plan must contain 'virtual_machines' list")

    for vm in plan["virtual_machines"]:
        vm_name = vm["name"]
        vm_data = inventory.get_vm(vm_name)
        vm["id"] = vm_data["id"]
```

This is called during `test_create_plan` before constructing the Plan CR:

```python
def test_create_plan(self, prepared_plan, ..., source_provider_inventory):
    """Create MTV Plan CR resource."""
    populate_vm_ids(prepared_plan, source_provider_inventory)
    self.__class__.plan_resource = create_plan_resource(...)
```

## Error Handling

The inventory API raises specific exceptions at each layer:

| Exception              | When                                                    | Source Method               |
|------------------------|---------------------------------------------------------|-----------------------------|
| `TimeoutExpiredError`  | Provider not found in inventory after 180s              | `_provider_id`              |
| `TimeoutExpiredError`  | VM not found or resources not synced after timeout      | `wait_for_vm()`             |
| `ValueError`           | VM not found by name                                    | `get_vm()`                  |
| `ValueError`           | No storage mappings found for given VMs                 | `vms_storages_mappings()`   |
| `ValueError`           | No network mappings found for given VMs                 | `vms_networks_mappings()`   |
| `ValueError`           | Provider type not implemented                           | `source_provider_inventory` fixture |
