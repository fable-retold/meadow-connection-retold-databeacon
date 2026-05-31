# Configuration & Routing

This page documents every configuration key the connection reads, where configuration comes from, how `TargetBeaconName` drives AffinityKey routing, and how the connection hash shapes the remote route.

---

## Where Configuration Comes From

The constructor resolves its config by layering two sources, with constructor options taking precedence:

1. **`fable.settings.RetoldDataBeacon`** -- the conventional settings bag. The connection manager sets this from a stored connection's config.
2. **Constructor options** -- the second constructor argument (`this.options`); each key overrides the matching settings value when present.

There is one built-in default: `TimeoutMs` falls back to `30000` (30 seconds) when neither source supplies it. The other keys default to an empty string, and the three required keys are validated at connect time.

> Unlike the SQL connectors, this module does **not** project its resolved config back onto `fable.settings`. The paired provider finds the live transport through `fable.MeadowRetoldDataBeaconProvider` (set on connect), and reads `_TargetConnectionHash` directly off the connection instance.

---

## Connection Keys

| Key | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `UltravisorURL` | string | Yes | `''` | Base URL of the Ultravisor coordinator |
| `TargetBeaconName` | string | Yes | `''` | Stable name of the customer-side beacon; used as the AffinityKey on every dispatch |
| `TargetConnectionHash` | string | Yes | `''` | URL-safe hash of the remote connection name; the provider builds `/1.0/<hash>/<Entity>` from it |
| `UserName` | string | No | `''` | Coordinator auth username; also sent as `RemoteUser` on each dispatch |
| `Password` | string | No | `''` | Coordinator auth password |
| `TimeoutMs` | number | No | `30000` | Per-request timeout, set on each dispatched work item |
| `IDBeaconConnection` | number | No | `1` | Remote `BeaconConnection` ID used by introspection calls (see below) |

All three of `UltravisorURL`, `TargetBeaconName`, and `TargetConnectionHash` are validated by `connectAsync()`. If any is missing, connect logs an error and calls back with an `Error` naming the missing key; `connected` stays `false`.

> The module's root README configuration example predates the `TargetConnectionHash` requirement and omits it. The source requires `TargetConnectionHash` -- always include it. This documentation reflects the source.

### Example

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

The keys are the same whether you nest them under a stored connection record consumed by [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/) or pass them as flat constructor options.

---

## AffinityKey Routing

`TargetBeaconName` is the single most important routing key. The connection sets it as the Ultravisor `AffinityKey` on every work item it dispatches -- both `MeadowProxy:Request` items and introspection items.

- The coordinator routes work items by AffinityKey, binding that key to a registered beacon.
- A stable `TargetBeaconName` means every request for a given customer reaches the same beacon and therefore the same database.
- When each customer has exactly one databeacon registered, the routing is effectively deterministic.

Choose a `TargetBeaconName` that matches the stable name the customer-side beacon registers under with the coordinator. See [Architecture](architecture.md) for the routing diagram.

---

## The Connection Hash and the Remote Route

`TargetConnectionHash` is the URL-safe hash of the remote connection name on the customer-side beacon. A single beacon can host several `BeaconConnection`s (one per customer database), and the hash selects which one a request targets.

The paired provider (`Meadow-Provider-RetoldDataBeacon` in the `meadow` package) reads `_TargetConnectionHash` off this connection and builds the request path as:

```
/1.0/<TargetConnectionHash>/<Entity>
```

So with `TargetConnectionHash: 'bookstore-mssql'`, a read of the `Book` entity dispatches a descriptor whose `Path` is `/1.0/bookstore-mssql/Book`. That path is what the remote beacon's `MeadowProxy` handler runs against its own localhost REST API.

---

## Introspection Connection ID

The introspection methods (`listTables`, `introspectTableSchema`, `introspectDatabaseSchema`) need to tell the remote beacon **which** of its hosted connections to introspect. They send an `IDBeaconConnection` in the work item settings, resolved by `_defaultConnectionID()`:

1. `options.IDBeaconConnection` (constructor options), else
2. `fable.settings.RetoldDataBeacon.IDBeaconConnection`, else
3. the fallback `1`.

If your remote beacon hosts multiple databases, set `IDBeaconConnection` to the right remote `BeaconConnection` ID; otherwise introspection defaults to connection `1`.

---

## Credentials and the Session

`UserName` and `Password` are passed to the [fable-ultravisor-client](https://fable-retold.github.io/fable-ultravisor-client/) and used for the `POST /1.0/Authenticate` call during connect. On success the client captures a session cookie and attaches it to every dispatch. Both are optional from this module's point of view -- whether they are required depends on the coordinator's authentication configuration. `UserName` is additionally sent as the `RemoteUser` field on each `MeadowProxy:Request` so the remote beacon can attribute the request.

---

## Timeout

`TimeoutMs` (default `30000`) is set on every dispatched work item as the per-request timeout. The Ultravisor client also uses it to bound the underlying HTTP socket, so a stuck remote does not hang the caller forever. Raise it for slow remote queries; lower it to fail fast.

---

## No DDL / Schema Configuration

There are no DDL or schema-creation options. The remote database owns its own schema, so:

- `createTable()` / `createTables()` call back with a "not supported on remote connections" error.
- `generateCreateTableStatement()` / `generateDropTableStatement()` return an empty string.

To discover what the remote exposes, use the introspection methods (`listTables`, `introspectDatabaseSchema`), which delegate to the remote beacon's management capability. See [API Reference](api.md).

---

## No Form Schema (Yet)

[meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/) ships a pure-data form schema for most connection types so a UI can render a connection form. `RetoldDataBeacon` does **not** have one yet -- the manager has no `FORM_SCHEMA_PATHS` entry for it, and `getProviderFormSchema('RetoldDataBeacon')` returns `null`. A connection UI must build the config form by hand until a schema is added to this module.

---

## Related Pages

- [Quickstart](quickstart.md) -- Configure and connect end to end
- [Architecture](architecture.md) -- The dispatch flow and AffinityKey routing
- [API Reference](api.md) -- The methods and properties on the connection class
