# hostinger-zig

Use Hostinger from a Zig application: list your servers, read their health and configuration, manage DNS, or build explicit requests for server and website operations.

This is an **unofficial, dependency-free client for the Hostinger API**. A developer adds it to a Zig program to send requests with your Hostinger account.

[Get started](#get-started) · [Endpoint guide](docs/endpoints.md) · [Official API documentation](https://docs.hostinger.com/api-reference/overview)

## What can you use it for?

- **Build a server inventory or health dashboard.** List VPS instances (virtual private servers), inspect their configuration, and read recent metrics.
- **Manage a server's lifecycle.** Prepare an explicit start, stop, restart, backup, snapshot, or recovery request; follow the returned action until it finishes before relying on the result.
- **Automate domain and hosting administration.** Read DNS records, domain details, subscriptions, websites, databases, and the supported ecommerce or contact-management resources.

The library covers these areas:

| Area | Examples | Reference |
| --- | --- | --- |
| VPS | Servers, metrics, actions, backups, snapshots, recovery, malware scanning | [Endpoints](docs/endpoints.md#vps) |
| Server setup | Data centers, operating-system templates, SSH keys, setup scripts, firewalls | [Endpoints](docs/endpoints.md#server-setup-and-firewalls) |
| Docker Manager | Projects, containers, logs, and explicit lifecycle requests | [Endpoints](docs/endpoints.md#docker-manager) |
| Domains and DNS | Domain portfolio, nameservers, forwarding, WHOIS contacts, DNS records and snapshots | [Domains](docs/endpoints.md#domains), [DNS](docs/endpoints.md#dns) |
| Hosting and websites | Hosting orders, domains, websites, databases, WordPress and Node.js settings | [Endpoints](docs/endpoints.md#hosting-and-websites) |
| Billing | Catalog items, payment methods, subscriptions | [Endpoints](docs/endpoints.md#billing) |
| Ecommerce, Horizons and Reach | Stores, generated websites, contacts, profiles and segments | [Endpoints](docs/endpoints.md#ecommerce-horizons-and-reach) |

**Read methods send requests; mutation helpers build routes and preview plans.** To change a resource, your application supplies the provider's JSON payload and explicitly calls `requestJson`. The [endpoint guide](docs/endpoints.md) lists HTTP methods, paths, useful Zig helpers, and official documentation for each operation. It also identifies a Node.js archive-upload route that differs from the current provider specification.

## Get started

Requires **Zig 0.17.0**. The package is pre-1.0 and has no other Zig dependencies. Linux is the validated target. The metrics helper uses a Linux clock; the full suite currently does not compile on macOS.

### 1. Add the dependency

From your Zig project's directory:

```sh
zig fetch --save=hostinger git+https://github.com/tzekid/hostinger-zig
```

This adds a dependency with a content hash to `build.zig.zon`. Commit that file so other builds use the same package. For a specific revision, append `#<commit>` to the repository URL.

In `build.zig`, after creating your executable, add its import:

```zig
const dependency = b.dependency("hostinger", .{
    .target = target,
    .optimize = optimize,
});
exe.root_module.addImport("hostinger", dependency.module("hostinger"));
```

Here, `target`, `optimize`, and `exe` are the values from your existing build.

### 2. Create an API token

[Create a token in hPanel](https://hpanel.hostinger.com/profile/api). Hostinger tokens use the owning user's permissions; an endpoint may also require a corresponding service on the account. See [Hostinger's authentication documentation](https://docs.hostinger.com/api-reference/overview).

The example below reads `HOSTINGER_API_TOKEN` from the environment. The library itself does not load environment variables or configuration files.

```sh
export HOSTINGER_API_TOKEN='your-api-token'
```

### 3. List your virtual private servers

This example makes a read-only request, checks the HTTP status, then prints server IDs from one response page.

```zig
const std = @import("std");
const hostinger = @import("hostinger");

pub fn main(init: std.process.Init) !void {
    const token = init.environ_map.get("HOSTINGER_API_TOKEN") orelse
        return error.MissingHostingerToken;
    const client = hostinger.Client.init(token);

    const response = try client.getVirtualMachines(init.io, init.gpa);
    defer response.deinit(init.gpa);
    const status = @intFromEnum(response.status);
    if (status < 200 or status >= 300) return error.HostingerApiError;

    const machines = try hostinger.models.parseVpsRows(init.gpa, response.body);
    defer machines.deinit(init.gpa);
    for (machines.items) |machine| {
        std.debug.print("{s}\n", .{machine.id});
    }
}
```

Use `client.getVirtualMachineDetails(io, allocator, vm_id)` for a specific server. `client.getVirtualMachineEndpoint(io, allocator, vm_id, .metrics)` constructs a request for the preceding 24 hours of metrics. See [the endpoint guide](docs/endpoints.md#using-the-routes) for request construction, mutation previews, and pagination.

## Request and response basics

- **Errors:** network and allocation failures are Zig errors. HTTP API failures are returned as `Response` values; check `status` before parsing the JSON. Hostinger's error body includes an error description and a correlation ID. The convenience row parsers can return an empty list for malformed or unexpected JSON, so an empty list alone does not establish success.
- **Pagination:** list helpers fetch one response page. The `...Page` methods request a specific page; `models.paginationInfo` reads pagination metadata, and `models.mergePaginatedBodies` merges bodies you have already fetched. The library does not fetch subsequent pages automatically.
- **Ownership:** clients borrow the token and configuration strings. Responses, parsed rows, allocated paths, and preview plans own their allocations. Use `deinit` or `allocator.free` with the same allocator that created them.
- **Transport:** buffered responses are capped at 12 MiB; the HTTP transport configures 30-second socket timeouts on Linux and macOS. Authenticated requests do not follow redirects automatically. Requests are not automatically retried; your application handles rate limits and retry policy.
- **Changes:** a successful start, stop, or restart request may return an action that is still running. Use `getActionDetails` to check its completion; a successful HTTP response is not proof that the server has reached its new state.

[`src/root.zig`](src/root.zig) is the public entry point. Use `Client` for requests, `routes` for paths and operation metadata, and `models` for the provided normalized response types. The library covers a subset of Hostinger's API, not every product or a complete set of typed payload schemas.

## Development

```sh
zig build test
```

The suite runs without API credentials.

This repository is the canonical source. Make library changes here; consumers such as [Cloudio](https://github.com/tzekid/cloudio) pin a revision independently. Keep endpoint changes and this guide in sync.

[MIT license](LICENSE) · [Changelog](CHANGELOG.md)
