# Meadow Connection Retold DataBeacon

A [meadow](https://fable-retold.github.io/meadow/) connection that relays CRUD and introspection through an [Ultravisor](https://stevenvelozo.github.io/ultravisor/) mesh to a remote [retold-databeacon](https://fable-retold.github.io/retold-databeacon/) agent. It pairs with the in-meadow `Meadow-Provider-RetoldDataBeacon` provider: the provider turns a meadow query into an HTTP request descriptor, and this module is the connection-manager-shaped wrapper that owns the Ultravisor client, the configuration shape, and the `dispatchRequest` transport that carries the descriptor across the mesh.

The result is that a process which already speaks meadow -- a beacon, a CLI, an engineer's laptop -- can pick `Type: 'RetoldDataBeacon'` and read and write against a customer database that lives behind a remote databeacon, with no VPN and no vendor SQL client.

## When to Use It

You have a customer database reachable only by a [retold-databeacon](https://fable-retold.github.io/retold-databeacon/) on the customer's side, and a coordinator (Ultravisor) both sides can reach. This connection lets a meadow consumer dispatch CRUD and introspection to that beacon as if it were a local data source. The canonical v1 case is an engineer's laptop talking to one customer beacon at a time.

## What It Is (and Isn't)

This module is a **relay wrapper**, not a database driver. It does not build the meadow CRUD requests itself -- that work lives in the meadow core provider `Meadow-Provider-RetoldDataBeacon`, which uses the FoxHound `MeadowEndpoints` dialect to produce a `{ Method, Path, Body }` descriptor. This module supplies three things around that provider:

- **The Ultravisor client** -- it owns a [fable-ultravisor-client](https://fable-retold.github.io/fable-ultravisor-client/) handle, authenticates against the coordinator at connect time, and registers itself as the singleton transport on `fable.MeadowRetoldDataBeaconProvider`.
- **Configuration shape** -- it resolves the coordinator URL, the target beacon name, the remote connection hash, credentials, and the per-request timeout from constructor options layered over `fable.settings.RetoldDataBeacon`.
- **Dispatch transport** -- `dispatchRequest({ Method, Path, Body }, cb)` wraps the descriptor as a `MeadowProxy:Request` work item, dispatches it through the coordinator with `TargetBeaconName` as the AffinityKey, and unwraps the response.

See [Architecture](architecture.md) for the full request flow.

## Install

```bash
npm install meadow-connection-retold-databeacon
```

The runtime dependencies are `fable-serviceproviderbase` (the service base class) and [fable-ultravisor-client](https://fable-retold.github.io/fable-ultravisor-client/) (the coordinator transport). The paired provider `Meadow-Provider-RetoldDataBeacon` ships inside the `meadow` package, not here.

## Quick Look

Through [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/), a connection is a config object with a `Type`:

```json
{
	"Type": "RetoldDataBeacon",
	"UltravisorURL": "https://ultravisor.noc.example",
	"TargetBeaconName": "customer-acme-prod",
	"TargetConnectionHash": "bookstore-mssql",
	"UserName": "engineer-alice",
	"Password": "hunter2",
	"TimeoutMs": 30000,
	"IDBeaconConnection": 1
}
```

The manager maps `Type: 'RetoldDataBeacon'` to this module, sets `fable.settings.RetoldDataBeacon` from the config, instantiates the connection, and calls `connectAsync()`. From then on, meadow entities set to the `RetoldDataBeacon` provider relay their CRUD across the mesh.

For a standalone walkthrough that instantiates the connection directly, see the [Quickstart](quickstart.md).

> `TargetConnectionHash` is required by the connection (`connectAsync()` errors without it). It is the URL-safe hash of the remote connection name, and the paired provider uses it to build the beacon's hash-namespaced route `/1.0/<hash>/<Entity>`. See [Configuration & Routing](configuration.md).

## Routing

`TargetBeaconName` becomes the Ultravisor `AffinityKey` on every dispatch. The coordinator routes the work item to the beacon bound to that affinity slot -- typically deterministic when each customer has exactly one databeacon registered. See [Architecture](architecture.md) for how affinity routing reaches the bound beacon.

## Learn More

- [Quickstart](quickstart.md) -- Instantiate the connection, authenticate, and relay CRUD, step by step
- [Architecture](architecture.md) -- The mesh relay design: `MeadowProxy:Request` to the bound beacon to localhost REST, and AffinityKey routing
- [Configuration & Routing](configuration.md) -- Every config key, AffinityKey routing, and the connection-hash route
- [API Reference](api.md) -- Methods and properties on the connection class
- [Meadow](https://fable-retold.github.io/meadow/) -- The data access layer, host of `Meadow-Provider-RetoldDataBeacon`
- [Retold DataBeacon](https://fable-retold.github.io/retold-databeacon/) -- The remote agent this connection talks to
