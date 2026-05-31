# API Reference

The connection class `MeadowConnectionRetoldDataBeacon` extends `FableServiceProviderBase`. This page documents its public methods and properties as implemented in `source/Meadow-Connection-RetoldDataBeacon.js`.

The class is the default export of the package:

```javascript
const libMeadowConnectionRetoldDataBeacon = require('meadow-connection-retold-databeacon');
```

---

## Constructor

```javascript
new MeadowConnectionRetoldDataBeacon(pFable, pManifest, pServiceHash)
```

Standard Fable service constructor. The second argument doubles as the options object (`this.options`): each config key is read from the options object if present, otherwise from `fable.settings.RetoldDataBeacon`, with a built-in default for `TimeoutMs`.

The constructor:

- Sets `serviceType` to `'MeadowConnectionRetoldDataBeacon'`.
- Sets `connected` to `false`.
- Resolves `UltravisorURL`, `TargetBeaconName`, `TargetConnectionHash`, `UserName`, `Password`, `TimeoutMs` (default `30000`), and `IDBeaconConnection` into private fields.
- Leaves the Ultravisor client `null` -- it is built lazily on connect so reconnects start clean.

When instantiated through [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/), the manager supplies the config via `fable.settings.RetoldDataBeacon`.

---

## Properties

| Property | Type | Description |
|----------|------|-------------|
| `serviceType` | string | Always `'MeadowConnectionRetoldDataBeacon'` |
| `connected` | boolean | `true` after a successful connect; `false` initially and after `close()` |
| `_UltravisorURL` | string | Resolved coordinator base URL |
| `_TargetBeaconName` | string | Resolved beacon name; used as the AffinityKey on every dispatch |
| `_TargetConnectionHash` | string | Resolved URL-safe remote connection hash; read by the provider to build `/1.0/<hash>/<Entity>` |
| `_UserName` | string | Resolved coordinator username; also sent as `RemoteUser` on each dispatch |
| `_TimeoutMs` | number | Resolved per-request timeout (default `30000`) |
| `_Client` | object \| null | The `fable-ultravisor-client` handle, built on connect; `null` before connect and after `close()` |

> The config fields are prefixed with `_` in the source. They are documented here because the paired provider reads `_TargetConnectionHash` off the instance, but treat them as internal state rather than a stable public API.

---

## Lifecycle Methods

### `connect()`

Synchronous connect compatibility shim. The standard meadow connection interface has a sync `connect()`; here the work is asynchronous, so this calls `connectAsync()` with no callback and returns immediately. Use `connectAsync()` when you need to wait or inspect the result.

---

### `connectAsync(fCallback)`

Authenticate against the Ultravisor coordinator and register as the transport.

```javascript
_Connection.connectAsync(
	(pError, pClient) =>
	{
		if (pError) return console.error(pError.message);
		console.log('connected =', _Connection.connected);
	});
```

Behavior:

1. Validates the three required keys -- `UltravisorURL`, `TargetBeaconName`, `TargetConnectionHash`. A missing key logs an error and calls back with an `Error` naming it.
2. If already connected with a live client, logs a warning, re-registers `fable.MeadowRetoldDataBeaconProvider = this`, and calls back with the existing client.
3. Otherwise constructs a `fable-ultravisor-client` with `{ UltravisorURL, UserName, Password }` and calls its `authenticate()`.
4. On auth success: sets `connected = true`, registers `fable.MeadowRetoldDataBeaconProvider = this`, and calls back with `(null, client)`. On auth failure: logs and calls back with the error.

The callback signature is `(pError, pClient)` -- the second argument is the Ultravisor client handle.

---

### `close(fCallback)`

Tear down the connection. Sets `connected = false`, drops the client (`_Client = null`), and -- if `fable.MeadowRetoldDataBeaconProvider` still points at this instance -- deletes that registration. Always calls back with `(null)`.

```javascript
_Connection.close(
	() => console.log('Closed.'));
```

---

## Dispatch

### `dispatchRequest(pRequest, fCallback)`

Relay an HTTP request descriptor to the remote databeacon through the coordinator. This is the method the meadow provider calls on every CRUD operation.

```javascript
_Connection.dispatchRequest(
	{ Method: 'GET', Path: '/1.0/bookstore-mssql/Book' },
	(pError, pResponseBody) =>
	{
		if (pError) return console.error(pError.message);
		let tmpRecords = JSON.parse(pResponseBody);
	});
```

- **`pRequest`** -- `{ Method, Path, Body }`. `Method` defaults to `'GET'`, `Path` and `Body` to `''`.
- **`fCallback`** -- `(pError, pResponseBodyString)`.

It wraps the descriptor as a `MeadowProxy:Request` work item with `AffinityKey = TargetBeaconName`, `TimeoutMs = _TimeoutMs`, and `Settings.RemoteUser = UserName`, then dispatches it.

| Condition | Result |
|-----------|--------|
| Not connected (no client) | `(Error: not connected)` |
| Dispatch transport error | `(pError)` |
| `Outputs.Status` is a number `>= 400` | `(Error: Remote databeacon returned HTTP N: <body snippet>)` |
| Success | `(null, Outputs.Body)` -- the raw response **string**; caller parses it |

The response is read from `result.Outputs` (falling back to a bare `result` object if `Outputs` is absent), tolerating minor changes to the coordinator's envelope shape. The returned `Body` is a string -- `JSON.parse` it yourself; the meadow provider does this for you.

---

## Introspection Methods

These delegate to the remote beacon via `_dispatchAction`, sending `IDBeaconConnection` (resolved by `_defaultConnectionID()`) and the `TargetBeaconName` AffinityKey.

### `listTables(fCallback)`

Dispatches a `DataBeaconAccess:ListTables` work item. Calls back with `(pError, pOutputs)` where `pOutputs` is the beacon handler's `Outputs` envelope (e.g. `{ Tables: [ ... ] }`).

```javascript
_Connection.listTables(
	(pError, pOutputs) =>
	{
		if (pError) return console.error(pError.message);
		console.log(pOutputs.Tables);
	});
```

---

### `introspectDatabaseSchema(fCallback)`

Dispatches a `DataBeaconManagement:Introspect` work item. Calls back with `(pError, pOutputs)` -- the full introspection result for the connection.

---

### `introspectTableSchema(pTableName, fCallback)`

Dispatches the same `DataBeaconManagement:Introspect` work item (the remote beacon only supports "introspect all" today), then filters the result's `Tables` array for the entry whose `TableName` matches `pTableName`. Calls back with `(null, tmpMatch)` -- the matching table object, or `null` if not found.

```javascript
_Connection.introspectTableSchema('Book',
	(pError, pTable) =>
	{
		if (pError) return console.error(pError.message);
		console.log(pTable); // the Book table descriptor, or null
	});
```

---

## Schema Parity Stubs

Remote databases handle their own DDL, so these exist only for interface parity with the SQL drivers:

| Method | Behavior |
|--------|----------|
| `createTable(pSchema, fCallback)` | Calls back with an `Error` -- not supported on remote connections |
| `createTables(pSchema, fCallback)` | Calls back with an `Error` -- not supported on remote connections |
| `generateCreateTableStatement()` | Returns an empty string `''` |
| `generateDropTableStatement()` | Returns an empty string `''` |

---

## Internal Helpers

These are not part of the public surface but explain the introspection behavior:

- **`_dispatchAction(pCapability, pAction, pSettings, fCallback)`** -- dispatches an arbitrary `{ Capability, Action, Settings }` work item with the `TargetBeaconName` AffinityKey and the configured timeout, and calls back with `(pError, pOutputs)` (the unwrapped `Outputs` envelope). Errors with "not connected" if called before connect.
- **`_defaultConnectionID()`** -- resolves the remote `BeaconConnection` ID for introspection: `options.IDBeaconConnection`, else `fable.settings.RetoldDataBeacon.IDBeaconConnection`, else `1`.

---

## Notes on the Relay

The meadow CRUD operations (Create / Read / Update / Delete / Count) are **not** methods on this class -- they are implemented in the meadow core provider `Meadow-Provider-RetoldDataBeacon`, which finds this connection through `fable.MeadowRetoldDataBeaconProvider`, builds the `{ Method, Path, Body }` descriptor (path `/1.0/<TargetConnectionHash>/<Entity>`), and calls `dispatchRequest`. See [Architecture](architecture.md) for that split.
