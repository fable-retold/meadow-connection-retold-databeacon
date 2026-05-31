# Meadow Connection Retold DataBeacon

> A remote retold-databeacon as a local meadow data source, over an Ultravisor mesh

A meadow connection that relays CRUD and introspection to a remote
[retold-databeacon](https://fable-retold.github.io/retold-databeacon/) agent
through an [Ultravisor](https://stevenvelozo.github.io/ultravisor/) coordinator.
Reach a customer database transparently -- no VPN, no vendor SQL client.

- **Relay, not driver** -- wraps the in-meadow `Meadow-Provider-RetoldDataBeacon`; this module owns the Ultravisor client, the config shape, and the dispatch transport
- **Mesh transport** -- each request is a `MeadowProxy:Request` work item dispatched through the coordinator to the bound beacon
- **AffinityKey routing** -- `TargetBeaconName` is the AffinityKey on every dispatch, binding the work to the customer-side beacon
- **Introspection passthrough** -- `listTables` and `introspectDatabaseSchema` delegate to the remote beacon's management capability
- **Type dispatch** -- selected via `Type: 'RetoldDataBeacon'` through meadow-connection-manager

[Get Started](README.md)
[Quickstart](quickstart.md)
[Architecture](architecture.md)
[Configuration](configuration.md)
[API Reference](api.md)
[GitHub](https://github.com/fable-retold/meadow-connection-retold-databeacon)
