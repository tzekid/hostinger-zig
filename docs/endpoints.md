# Hostinger endpoint guide

[Back to the README](../README.md)

The paths below are relative to `https://developers.hostinger.com`. An **endpoint** is a particular HTTP method and path. `GET` reads information; `POST`, `PUT`, `PATCH`, and `DELETE` have endpoint-specific effects explained in the linked reference. A POST can also be a validation or availability check.

`{name}` marks a value you supply, such as an account, server, or domain ID. The tables list routes exposed by the current library, not every provider API. The official reference defines request bodies, permissions, service availability, and response fields. These tables describe code coverage; they are not a claim that every operation has been tested against a live account.

## Find an operation

- [VPS](#vps) — 30 operations
- [Server setup and firewalls](#server-setup-and-firewalls) — 22 operations
- [Docker Manager](#docker-manager) — 10 operations
- [DNS](#dns) — 8 operations
- [Domains](#domains) — 17 operations
- [Hosting and websites](#hosting-and-websites) — 22 operations
- [Billing](#billing) — 7 operations
- [Ecommerce, Horizons and Reach](#ecommerce-horizons-and-reach) — 13 operations

## Using the routes

For common reads, use `Client.getVirtualMachines`, `getVirtualMachineDetails`, `getVirtualMachineEndpoint`, or the appropriate `get...Endpoint` method. The helper column links to the route implementation and its argument types.

To restart a server, construct the route and explicitly send the request. This function changes the selected VPS when called:

```zig
pub fn requestRestart(
    client: hostinger.Client,
    io: std.Io,
    allocator: std.mem.Allocator,
    vm_id: []const u8,
) !hostinger.Response {
    const path = try hostinger.routes.vpsMutationPath(
        allocator, .restart, .{ .vm_id = vm_id },
    );
    defer allocator.free(path);
    const url = try std.fmt.allocPrint(allocator, "{s}{s}", .{
        client.base_url_override, path,
    });
    defer allocator.free(url);
    const response = try client.requestJson(io, allocator, .POST, url, null);
    errdefer response.deinit(allocator);
    const status = @intFromEnum(response.status);
    if (status < 200 or status >= 300) return error.HostingerApiError;
    return response;
}
```

`std` and `hostinger` are the imports used in the README example. The returned response belongs to the caller: use `defer response.deinit(allocator)` and read its JSON body for the created resource or action. The [restart reference](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/restart-virtual-machine) explains the returned action. Use `getActionDetails(io, allocator, vm_id, action_id)` to check whether it completed. This example requests a restart; it does not wait for the server to finish restarting.

To inspect the intended operation without changing a server:

```zig
const plan = try hostinger.routes.vpsMutationPlanJson(
    allocator, .restart, .{ .vm_id = vm_id },
);
defer allocator.free(plan);
```

A preview contains method, path, operation metadata, and the request schema reference. It does not construct or validate your JSON payload, authorize a change, or execute it. The `...Path` helpers allocate paths; the `...PlanJson` helpers allocate preview JSON. Both are freed with the caller's allocator.

For paginated collections, request each page with the corresponding `...Page` method. `models.paginationInfo(response.body)` provides `current_page`, `per_page`, `total`, and optional `total_pages`. `models.mergePaginatedBodies` combines already-fetched pages; it makes no network requests. See [Hostinger's pagination guide](https://docs.hostinger.com/api-reference/pagination) for the provider's response format.

## VPS

List virtual private servers, inspect a server, and read metrics. Start, stop, restart, recovery, backup, and snapshot changes may return a running action; use the action-details endpoint until it completes.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/vps/v1/virtual-machines` | [Get virtual machines](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/get-virtual-machines) | [`virtualMachinesUrl`](../src/routes.zig#L1501) |
| `POST /api/vps/v1/virtual-machines` | [Purchase new virtual machine](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/purchase-new-virtual-machine) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}` | [Get virtual machine details](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/get-virtual-machine-details) | [`virtualMachineDetailsUrl`](../src/routes.zig#L1505) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/actions` | [Get actions](https://docs.hostinger.com/api-reference/endpoints/vps/actions/get-actions) | [`endpointUrl`](../src/routes.zig#L1509) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/actions/{actionId}` | [Get action details](https://docs.hostinger.com/api-reference/endpoints/vps/actions/get-action-details) | [`actionDetailsUrl`](../src/routes.zig#L1520) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/backups` | [Get backups](https://docs.hostinger.com/api-reference/endpoints/vps/backups/get-backups) | [`endpointUrl`](../src/routes.zig#L1509) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/backups/{backupId}/restore` | [Restore backup](https://docs.hostinger.com/api-reference/endpoints/vps/backups/restore-backup) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `DELETE /api/vps/v1/virtual-machines/{virtualMachineId}/hostname` | [Reset hostname](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/reset-hostname) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `PUT /api/vps/v1/virtual-machines/{virtualMachineId}/hostname` | [Set hostname](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/set-hostname) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/metrics` | [Get metrics](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/get-metrics) | [`endpointUrl`](../src/routes.zig#L1509) |
| `DELETE /api/vps/v1/virtual-machines/{virtualMachineId}/monarx` | [Uninstall Monarx](https://docs.hostinger.com/api-reference/endpoints/vps/malware-scanner/uninstall-monarx) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/monarx` | [Get scan metrics](https://docs.hostinger.com/api-reference/endpoints/vps/malware-scanner/get-scan-metrics) | [`endpointUrl`](../src/routes.zig#L1509) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/monarx` | [Install Monarx](https://docs.hostinger.com/api-reference/endpoints/vps/malware-scanner/install-monarx) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `PUT /api/vps/v1/virtual-machines/{virtualMachineId}/nameservers` | [Set nameservers](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/set-nameservers) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `PUT /api/vps/v1/virtual-machines/{virtualMachineId}/panel-password` | [Set panel password](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/set-panel-password) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `DELETE /api/vps/v1/virtual-machines/{virtualMachineId}/ptr/{ipAddressId}` | [Delete PTR record](https://docs.hostinger.com/api-reference/endpoints/vps/ptr-records/delete-ptr-record) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/ptr/{ipAddressId}` | [Create PTR record](https://docs.hostinger.com/api-reference/endpoints/vps/ptr-records/create-ptr-record) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/public-keys` | [Get attached public keys](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/get-attached-public-keys) | [`endpointUrl`](../src/routes.zig#L1509) |
| `DELETE /api/vps/v1/virtual-machines/{virtualMachineId}/recovery` | [Stop recovery mode](https://docs.hostinger.com/api-reference/endpoints/vps/recovery/stop-recovery-mode) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/recovery` | [Start recovery mode](https://docs.hostinger.com/api-reference/endpoints/vps/recovery/start-recovery-mode) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/recreate` | [Recreate virtual machine](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/recreate-virtual-machine) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/restart` | [Restart virtual machine](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/restart-virtual-machine) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `PUT /api/vps/v1/virtual-machines/{virtualMachineId}/root-password` | [Set root password](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/set-root-password) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/setup` | [Setup purchased virtual machine](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/setup-purchased-virtual-machine) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `DELETE /api/vps/v1/virtual-machines/{virtualMachineId}/snapshot` | [Delete snapshot](https://docs.hostinger.com/api-reference/endpoints/vps/snapshots/delete-snapshot) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/snapshot` | [Get snapshot](https://docs.hostinger.com/api-reference/endpoints/vps/snapshots/get-snapshot) | [`endpointUrl`](../src/routes.zig#L1509) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/snapshot` | [Create snapshot](https://docs.hostinger.com/api-reference/endpoints/vps/snapshots/create-snapshot) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/snapshot/restore` | [Restore snapshot](https://docs.hostinger.com/api-reference/endpoints/vps/snapshots/restore-snapshot) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/start` | [Start virtual machine](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/start-virtual-machine) | [`vpsMutationPath`](../src/routes.zig#L1524) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/stop` | [Stop virtual machine](https://docs.hostinger.com/api-reference/endpoints/vps/virtual-machine/stop-virtual-machine) | [`vpsMutationPath`](../src/routes.zig#L1524) |

## Server setup and firewalls

Find available data centers and operating-system templates; inspect SSH keys, setup scripts, and firewalls before configuring a server. Recreating a machine or restoring data changes the server itself.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/vps/v1/data-centers` | [Get data center list](https://docs.hostinger.com/api-reference/endpoints/vps/data-centers/get-data-center-list) | [`vpsInventoryUrl`](../src/routes.zig#L2078) |
| `GET /api/vps/v1/firewall` | [Get firewall list](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/get-firewall-list) | [`vpsInventoryUrl`](../src/routes.zig#L2078) |
| `POST /api/vps/v1/firewall` | [Create new firewall](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/create-new-firewall) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `DELETE /api/vps/v1/firewall/{firewallId}` | [Delete firewall](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/delete-firewall) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `GET /api/vps/v1/firewall/{firewallId}` | [Get firewall details](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/get-firewall-details) | [`vpsInventoryDetailPath`](../src/routes.zig#L2092) |
| `POST /api/vps/v1/firewall/{firewallId}/activate/{virtualMachineId}` | [Activate firewall](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/activate-firewall) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `POST /api/vps/v1/firewall/{firewallId}/deactivate/{virtualMachineId}` | [Deactivate firewall](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/deactivate-firewall) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `POST /api/vps/v1/firewall/{firewallId}/rules` | [Create firewall rule](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/create-firewall-rule) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `DELETE /api/vps/v1/firewall/{firewallId}/rules/{ruleId}` | [Delete firewall rule](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/delete-firewall-rule) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `PUT /api/vps/v1/firewall/{firewallId}/rules/{ruleId}` | [Update firewall rule](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/update-firewall-rule) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `POST /api/vps/v1/firewall/{firewallId}/sync/{virtualMachineId}` | [Sync firewall](https://docs.hostinger.com/api-reference/endpoints/vps/firewall/sync-firewall) | [`firewallMutationPath`](../src/routes.zig#L2135) |
| `GET /api/vps/v1/post-install-scripts` | [Get post-install scripts](https://docs.hostinger.com/api-reference/endpoints/vps/post-install-scripts/get-post-install-scripts) | [`vpsInventoryUrl`](../src/routes.zig#L2078) |
| `POST /api/vps/v1/post-install-scripts` | [Create post-install script](https://docs.hostinger.com/api-reference/endpoints/vps/post-install-scripts/create-post-install-script) | [`vpsResourceMutationPath`](../src/routes.zig#L2096) |
| `DELETE /api/vps/v1/post-install-scripts/{postInstallScriptId}` | [Delete post-install script](https://docs.hostinger.com/api-reference/endpoints/vps/post-install-scripts/delete-post-install-script) | [`vpsResourceMutationPath`](../src/routes.zig#L2096) |
| `GET /api/vps/v1/post-install-scripts/{postInstallScriptId}` | [Get post-install script](https://docs.hostinger.com/api-reference/endpoints/vps/post-install-scripts/get-post-install-script) | [`vpsInventoryDetailPath`](../src/routes.zig#L2092) |
| `PUT /api/vps/v1/post-install-scripts/{postInstallScriptId}` | [Update post-install script](https://docs.hostinger.com/api-reference/endpoints/vps/post-install-scripts/update-post-install-script) | [`vpsResourceMutationPath`](../src/routes.zig#L2096) |
| `GET /api/vps/v1/public-keys` | [Get public keys](https://docs.hostinger.com/api-reference/endpoints/vps/public-keys/get-public-keys) | [`vpsInventoryUrl`](../src/routes.zig#L2078) |
| `POST /api/vps/v1/public-keys` | [Create public key](https://docs.hostinger.com/api-reference/endpoints/vps/public-keys/create-public-key) | [`vpsResourceMutationPath`](../src/routes.zig#L2096) |
| `POST /api/vps/v1/public-keys/attach/{virtualMachineId}` | [Attach public key](https://docs.hostinger.com/api-reference/endpoints/vps/public-keys/attach-public-key) | [`vpsResourceMutationPath`](../src/routes.zig#L2096) |
| `DELETE /api/vps/v1/public-keys/{publicKeyId}` | [Delete public key](https://docs.hostinger.com/api-reference/endpoints/vps/public-keys/delete-public-key) | [`vpsResourceMutationPath`](../src/routes.zig#L2096) |
| `GET /api/vps/v1/templates` | [Get templates](https://docs.hostinger.com/api-reference/endpoints/vps/os-templates/get-templates) | [`vpsInventoryUrl`](../src/routes.zig#L2078) |
| `GET /api/vps/v1/templates/{templateId}` | [Get template details](https://docs.hostinger.com/api-reference/endpoints/vps/os-templates/get-template-details) | [`vpsInventoryDetailPath`](../src/routes.zig#L2092) |

## Docker Manager

Read projects, containers, and logs managed by Hostinger on a VPS. These are Hostinger API routes; the library does not connect directly to a local Docker socket.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/docker` | [Get project list](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/get-project-list) | [`endpointUrl`](../src/routes.zig#L1509) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/docker` | [Create new project](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/create-new-project) | [`dockerMutationPath`](../src/routes.zig#L1576) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}` | [Get project contents](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/get-project-contents) | [`dockerEndpointPath`](../src/routes.zig#L1565) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/containers` | [Get project containers](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/get-project-containers) | [`dockerEndpointPath`](../src/routes.zig#L1565) |
| `DELETE /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/down` | [Delete project](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/delete-project) | [`dockerMutationPath`](../src/routes.zig#L1576) |
| `GET /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/logs` | [Get project logs](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/get-project-logs) | [`dockerEndpointPath`](../src/routes.zig#L1565) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/restart` | [Restart project](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/restart-project) | [`dockerMutationPath`](../src/routes.zig#L1576) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/start` | [Start project](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/start-project) | [`dockerMutationPath`](../src/routes.zig#L1576) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/stop` | [Stop project](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/stop-project) | [`dockerMutationPath`](../src/routes.zig#L1576) |
| `POST /api/vps/v1/virtual-machines/{virtualMachineId}/docker/{projectName}/update` | [Update project](https://docs.hostinger.com/api-reference/endpoints/vps/docker-manager/update-project) | [`dockerMutationPath`](../src/routes.zig#L1576) |

## DNS

Use a domain name to read its DNS zone. Validate the intended records before applying an update; restoring a DNS snapshot replaces the zone configuration.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/dns/v1/snapshots/{domain}` | [Get DNS snapshot list](https://docs.hostinger.com/api-reference/endpoints/dns/snapshot/get-dns-snapshot-list) | [`dnsEndpointPath`](../src/routes.zig#L1649) |
| `GET /api/dns/v1/snapshots/{domain}/{snapshotId}` | [Get DNS snapshot](https://docs.hostinger.com/api-reference/endpoints/dns/snapshot/get-dns-snapshot) | [`dnsEndpointPath`](../src/routes.zig#L1649) |
| `POST /api/dns/v1/snapshots/{domain}/{snapshotId}/restore` | [Restore DNS snapshot](https://docs.hostinger.com/api-reference/endpoints/dns/snapshot/restore-dns-snapshot) | [`dnsMutationPath`](../src/routes.zig#L1664) |
| `DELETE /api/dns/v1/zones/{domain}` | [Delete DNS records](https://docs.hostinger.com/api-reference/endpoints/dns/zone/delete-dns-records) | [`dnsMutationPath`](../src/routes.zig#L1664) |
| `GET /api/dns/v1/zones/{domain}` | [Get DNS records](https://docs.hostinger.com/api-reference/endpoints/dns/zone/get-dns-records) | [`dnsEndpointPath`](../src/routes.zig#L1649) |
| `PUT /api/dns/v1/zones/{domain}` | [Update DNS records](https://docs.hostinger.com/api-reference/endpoints/dns/zone/update-dns-records) | [`dnsMutationPath`](../src/routes.zig#L1664) |
| `POST /api/dns/v1/zones/{domain}/reset` | [Reset DNS records](https://docs.hostinger.com/api-reference/endpoints/dns/zone/reset-dns-records) | [`dnsMutationPath`](../src/routes.zig#L1664) |
| `POST /api/dns/v1/zones/{domain}/validate` | [Validate DNS records](https://docs.hostinger.com/api-reference/endpoints/dns/zone/validate-dns-records) | [`dnsMutationPath`](../src/routes.zig#L1664) |

## Domains

Read domain registrations, nameservers, forwarding, and WHOIS contact profiles. Availability checks are POST requests but do not purchase the domain; the purchase operation is separate.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `POST /api/domains/v1/availability` | [Check domain availability](https://docs.hostinger.com/api-reference/endpoints/domains/availability/check-domain-availability) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `POST /api/domains/v1/forwarding` | [Create domain forwarding](https://docs.hostinger.com/api-reference/endpoints/domains/forwarding/create-domain-forwarding) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `DELETE /api/domains/v1/forwarding/{domain}` | [Delete domain forwarding](https://docs.hostinger.com/api-reference/endpoints/domains/forwarding/delete-domain-forwarding) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `GET /api/domains/v1/forwarding/{domain}` | [Get domain forwarding](https://docs.hostinger.com/api-reference/endpoints/domains/forwarding/get-domain-forwarding) | [`domainEndpointPath`](../src/routes.zig#L1702) |
| `GET /api/domains/v1/portfolio` | [Get domain list](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/get-domain-list) | [`domainEndpointPath`](../src/routes.zig#L1702) |
| `POST /api/domains/v1/portfolio` | [Purchase new domain](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/purchase-new-domain) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `GET /api/domains/v1/portfolio/{domain}` | [Get domain details](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/get-domain-details) | [`domainEndpointPath`](../src/routes.zig#L1702) |
| `DELETE /api/domains/v1/portfolio/{domain}/domain-lock` | [Disable domain lock](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/disable-domain-lock) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `PUT /api/domains/v1/portfolio/{domain}/domain-lock` | [Enable domain lock](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/enable-domain-lock) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `PUT /api/domains/v1/portfolio/{domain}/nameservers` | [Update domain nameservers](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/update-domain-nameservers) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `DELETE /api/domains/v1/portfolio/{domain}/privacy-protection` | [Disable privacy protection](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/disable-privacy-protection) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `PUT /api/domains/v1/portfolio/{domain}/privacy-protection` | [Enable privacy protection](https://docs.hostinger.com/api-reference/endpoints/domains/portfolio/enable-privacy-protection) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `GET /api/domains/v1/whois` | [Get WHOIS profile list](https://docs.hostinger.com/api-reference/endpoints/domains/whois/get-whois-profile-list) | [`domainEndpointPath`](../src/routes.zig#L1702) |
| `POST /api/domains/v1/whois` | [Create WHOIS profile](https://docs.hostinger.com/api-reference/endpoints/domains/whois/create-whois-profile) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `DELETE /api/domains/v1/whois/{whoisId}` | [Delete WHOIS profile](https://docs.hostinger.com/api-reference/endpoints/domains/whois/delete-whois-profile) | [`domainMutationPath`](../src/routes.zig#L1738) |
| `GET /api/domains/v1/whois/{whoisId}` | [Get WHOIS profile](https://docs.hostinger.com/api-reference/endpoints/domains/whois/get-whois-profile) | [`domainEndpointPath`](../src/routes.zig#L1702) |
| `GET /api/domains/v1/whois/{whoisId}/usage` | [Get WHOIS profile usage](https://docs.hostinger.com/api-reference/endpoints/domains/whois/get-whois-profile-usage) | [`domainEndpointPath`](../src/routes.zig#L1702) |

## Hosting and websites

Inspect hosting accounts, orders, domains, websites, databases, WordPress installations, and supported Node.js settings. The library provides route construction for the listed changes; payload schemas live in the official reference.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/hosting/v1/accounts/{username}/databases` | [List account databases](https://docs.hostinger.com/api-reference/endpoints/hosting/databases/list-account-databases) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `POST /api/hosting/v1/accounts/{username}/databases` | [Create account database](https://docs.hostinger.com/api-reference/endpoints/hosting/databases/create-account-database) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `DELETE /api/hosting/v1/accounts/{username}/databases/{name}` | [Delete account database](https://docs.hostinger.com/api-reference/endpoints/hosting/databases/delete-account-database) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `PATCH /api/hosting/v1/accounts/{username}/databases/{name}/change-password` | [Change database password](https://docs.hostinger.com/api-reference/endpoints/hosting/databases/change-database-password) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `GET /api/hosting/v1/accounts/{username}/databases/{name}/phpmyadmin-link` | [Get phpMyAdmin link](https://docs.hostinger.com/api-reference/endpoints/hosting/databases/get-phpmyadmin-link) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `PATCH /api/hosting/v1/accounts/{username}/databases/{name}/repair` | [Repair database](https://docs.hostinger.com/api-reference/endpoints/hosting/databases/repair-database) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `GET /api/hosting/v1/accounts/{username}/websites/{domain}/nodejs/builds` | [List NodeJS builds](https://docs.hostinger.com/api-reference/endpoints/hosting/nodejs/list-nodejs-builds) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `GET /api/hosting/v1/accounts/{username}/websites/{domain}/nodejs/builds/{uuid}/logs` | [Get NodeJS build logs](https://docs.hostinger.com/api-reference/endpoints/hosting/nodejs/get-nodejs-build-logs) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `GET /api/hosting/v1/accounts/{username}/websites/{domain}/parked-domains` | [List website parked domains](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/list-website-parked-domains) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `POST /api/hosting/v1/accounts/{username}/websites/{domain}/parked-domains` | [Create website parked domain](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/create-website-parked-domain) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `DELETE /api/hosting/v1/accounts/{username}/websites/{domain}/parked-domains/{parkedDomain}` | [Delete website parked domain](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/delete-website-parked-domain) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `GET /api/hosting/v1/accounts/{username}/websites/{domain}/subdomains` | [List website subdomains](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/list-website-subdomains) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `POST /api/hosting/v1/accounts/{username}/websites/{domain}/subdomains` | [Create website subdomain](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/create-website-subdomain) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `DELETE /api/hosting/v1/accounts/{username}/websites/{domain}/subdomains/{subdomain}` | [Delete website subdomain](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/delete-website-subdomain) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `POST /api/hosting/v1/accounts/{username}/wordpress/installations` | [Install WordPress](https://docs.hostinger.com/api-reference/endpoints/wordpress/installations/install-wordpress) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `GET /api/hosting/v1/datacenters` | [List available datacenters](https://docs.hostinger.com/api-reference/endpoints/hosting/datacenters/list-available-datacenters) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `POST /api/hosting/v1/domains/free-subdomains` | [Generate a free subdomain](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/generate-a-free-subdomain) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `POST /api/hosting/v1/domains/verify-ownership` | [Verify domain ownership](https://docs.hostinger.com/api-reference/endpoints/hosting/domains/verify-domain-ownership) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `GET /api/hosting/v1/orders` | [List orders](https://docs.hostinger.com/api-reference/endpoints/hosting/orders/list-orders) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `GET /api/hosting/v1/websites` | [List websites](https://docs.hostinger.com/api-reference/endpoints/hosting/websites/list-websites) | [`hostingEndpointPath`](../src/routes.zig#L1797) |
| `POST /api/hosting/v1/websites` | [Create website](https://docs.hostinger.com/api-reference/endpoints/hosting/websites/create-website) | [`hostingMutationPath`](../src/routes.zig#L1854) |
| `GET /api/hosting/v1/wordpress/installations` | [List WordPress installations](https://docs.hostinger.com/api-reference/endpoints/wordpress/installations/list-wordpress-installations) | [`hostingEndpointPath`](../src/routes.zig#L1797) |

## Billing

Inspect catalog items, payment methods, and subscriptions. Purchase, renewal, cancellation, and payment-method changes are account operations; a dry-run plan does not perform them.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/billing/v1/catalog` | [Get catalog item list](https://docs.hostinger.com/api-reference/endpoints/billing/catalog/get-catalog-item-list) | [`billingEndpointUrl`](../src/routes.zig#L1603) |
| `GET /api/billing/v1/payment-methods` | [Get payment method list](https://docs.hostinger.com/api-reference/endpoints/billing/payment-methods/get-payment-method-list) | [`billingEndpointUrl`](../src/routes.zig#L1603) |
| `DELETE /api/billing/v1/payment-methods/{paymentMethodId}` | [Delete payment method](https://docs.hostinger.com/api-reference/endpoints/billing/payment-methods/delete-payment-method) | [`billingMutationPath`](../src/routes.zig#L1607) |
| `POST /api/billing/v1/payment-methods/{paymentMethodId}` | [Set default payment method](https://docs.hostinger.com/api-reference/endpoints/billing/payment-methods/set-default-payment-method) | [`billingMutationPath`](../src/routes.zig#L1607) |
| `GET /api/billing/v1/subscriptions` | [Get subscription list](https://docs.hostinger.com/api-reference/endpoints/billing/subscriptions/get-subscription-list) | [`billingEndpointUrl`](../src/routes.zig#L1603) |
| `DELETE /api/billing/v1/subscriptions/{subscriptionId}/auto-renewal/disable` | [Disable auto-renewal](https://docs.hostinger.com/api-reference/endpoints/billing/subscriptions/disable-auto-renewal) | [`billingMutationPath`](../src/routes.zig#L1607) |
| `PATCH /api/billing/v1/subscriptions/{subscriptionId}/auto-renewal/enable` | [Enable auto-renewal](https://docs.hostinger.com/api-reference/endpoints/billing/subscriptions/enable-auto-renewal) | [`billingMutationPath`](../src/routes.zig#L1607) |

## Ecommerce, Horizons and Reach

Read stores, Horizons-generated websites, and Reach contact-management resources. Write builders cover only the specific operations listed here.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /api/ecommerce/v1/stores` | [Get stores](https://docs.hostinger.com/api-reference/endpoints/ecommerce/stores/get-stores) | [`ecommerceEndpointUrl`](../src/routes.zig#L1941) |
| `POST /api/ecommerce/v1/stores` | [Create store](https://docs.hostinger.com/api-reference/endpoints/ecommerce/stores/create-store) | [`ecommerceMutationPath`](../src/routes.zig#L1951) |
| `POST /api/horizons/v1/websites` | [Create website](https://docs.hostinger.com/api-reference/endpoints/horizons/websites/create-website) | [`horizonsMutationPath`](../src/routes.zig#L1985) |
| `GET /api/horizons/v1/websites/{websiteId}` | [Get website](https://docs.hostinger.com/api-reference/endpoints/horizons/websites/get-website) | [`horizonsEndpointPath`](../src/routes.zig#L1977) |
| `GET /api/reach/v1/contacts` | [List contacts](https://docs.hostinger.com/api-reference/endpoints/reach/contacts/list-contacts) | [`reachEndpointPath`](../src/routes.zig#L2017) |
| `DELETE /api/reach/v1/contacts/{uuid}` | [Delete a contact](https://docs.hostinger.com/api-reference/endpoints/reach/contacts/delete-a-contact) | [`reachMutationPath`](../src/routes.zig#L2046) |
| `GET /api/reach/v1/profiles` | [List Profiles](https://docs.hostinger.com/api-reference/endpoints/reach/profiles/list-profiles) | [`reachEndpointPath`](../src/routes.zig#L2017) |
| `POST /api/reach/v1/profiles/{profileUuid}/contacts` | [Create new contacts](https://docs.hostinger.com/api-reference/endpoints/reach/contacts/create-new-contacts) | [`reachMutationPath`](../src/routes.zig#L2046) |
| `GET /api/reach/v1/profiles/{profileUuid}/segmentation/segments/{segmentUuid}/contacts` | [List profile segment contacts](https://docs.hostinger.com/api-reference/endpoints/reach/segments/list-profile-segment-contacts) | [`reachEndpointPath`](../src/routes.zig#L2017) |
| `GET /api/reach/v1/segmentation/segments` | [List segments](https://docs.hostinger.com/api-reference/endpoints/reach/segments/list-segments) | [`reachEndpointPath`](../src/routes.zig#L2017) |
| `POST /api/reach/v1/segmentation/segments` | [Create a new contact segment](https://docs.hostinger.com/api-reference/endpoints/reach/segments/create-a-new-contact-segment) | [`reachMutationPath`](../src/routes.zig#L2046) |
| `GET /api/reach/v1/segmentation/segments/{segmentUuid}` | [Get segment details](https://docs.hostinger.com/api-reference/endpoints/reach/segments/get-segment-details) | [`reachEndpointPath`](../src/routes.zig#L2017) |
| `GET /api/reach/v1/segmentation/segments/{segmentUuid}/contacts` | [List segment contacts](https://docs.hostinger.com/api-reference/endpoints/reach/segments/list-segment-contacts) | [`reachEndpointPath`](../src/routes.zig#L2017) |

## Node.js archive-upload path discrepancy

`HostingMutationEndpoint.create_nodejs_build_from_archive` builds:

`POST /api/hosting/v1/accounts/{username}/websites/{domain}/nodejs/builds/from-archive`

The current Hostinger specification instead documents:

`POST /api/hosting/v1/accounts/{username}/websites/{domain}/nodejs/builds/settings/from-archive`

This existing builder is not listed as a matching current endpoint above. Check the [official Node.js reference](https://docs.hostinger.com/api-reference/endpoints/hosting/nodejs) and its upload format before using it. A route preview is not proof that the provider accepts that path or payload.

## Maintaining this reference

Update this guide when adding or changing a route. Paths and methods were checked against the implementation and the provider's current OpenAPI specification on 5 October 2026. [Hostinger schema](https://github.com/hostinger/api/blob/main/openapi.json) is the upstream source; the linked operation pages are the reader-facing reference.
