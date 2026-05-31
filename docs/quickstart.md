# Quickstart

This guide walks through pointing meadow at a remote retold-databeacon over an Ultravisor mesh: install, configure, authenticate against the coordinator, and relay CRUD to the remote beacon. Two paths are covered -- using the connection through [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/) (the usual case), and instantiating the connection directly.

---

## Prerequisites

- Node.js
- A reachable [Ultravisor](https://stevenvelozo.github.io/ultravisor/) coordinator, with credentials if it requires authentication
- A [retold-databeacon](https://fable-retold.github.io/retold-databeacon/) registered against that coordinator under a stable name, advertising the `MeadowProxy` capability
- The URL-safe connection hash of the remote `BeaconConnection` you want to reach

---

## Install

```bash
npm install meadow-connection-retold-databeacon fable
```

The paired provider `Meadow-Provider-RetoldDataBeacon` ships inside the `meadow` package, so install `meadow` as well if you intend to relay CRUD through a meadow entity.

---

## Path A: Through the Connection Manager (recommended)

The connection manager maps a `Type` string to the right connection module, sets up `fable.settings`, instantiates the connection, and connects it. You hand it a config object:

```javascript
const libFable = require('fable');
const libMeadowConnectionManager = require('meadow-connection-manager');

let _Fable = new libFable(
	{
		"Product": "BeaconExample",
		"ProductVersion": "1.0.0",
		"LogStreams": [ { "streamtype": "console" } ]
	});

_Fable.serviceManager.addAndInstantiateServiceType(
	'MeadowConnectionManager', libMeadowConnectionManager);

let tmpConnectionConfig =
	{
		Type: 'RetoldDataBeacon',
		UltravisorURL: 'https://ultravisor.noc.example',
		TargetBeaconName: 'customer-acme-prod',
		TargetConnectionHash: 'bookstore-mssql',
		UserName: 'engineer-alice',
		Password: 'hunter2',
		TimeoutMs: 30000,
		IDBeaconConnection: 1
	};

_Fable.MeadowConnectionManager.connect('acme-prod', tmpConnectionConfig,
	(pError, pConnection) =>
	{
		if (pError)
		{
			console.error('Connect failed:', pError.message);
			return;
		}
		console.log('Connected through beacon', tmpConnectionConfig.TargetBeaconName);
	});
```

The manager sets `fable.settings.RetoldDataBeacon` from the config, instantiates this module, and calls `connectAsync()` -- which authenticates against the coordinator's `Authenticate` endpoint and registers this connection on `fable.MeadowRetoldDataBeaconProvider`. From here, meadow entities set to the `RetoldDataBeacon` provider relay across the mesh.

> `TargetConnectionHash` is required. Omitting it makes `connectAsync()` fail with a "TargetConnectionHash is required" error. See [Configuration & Routing](configuration.md) for the full key list.

---

## Path B: Instantiate the Connection Directly

For a standalone process or a test, you can construct the connection without the manager. The constructor signature is the standard Fable service shape: `(pFable, pOptions, pServiceHash)`.

```javascript
const libFable = require('fable');
const libMeadowConnectionRetoldDataBeacon = require('meadow-connection-retold-databeacon');

let _Fable = new libFable(
	{
		"Product": "BeaconExample",
		"LogStreams": [ { "streamtype": "console" } ]
	});

let _Connection = new libMeadowConnectionRetoldDataBeacon(
	_Fable,
	{
		UltravisorURL: 'https://ultravisor.noc.example',
		TargetBeaconName: 'customer-acme-prod',
		TargetConnectionHash: 'bookstore-mssql',
		UserName: 'engineer-alice',
		Password: 'hunter2'
	},
	'acme-prod');
```

Config is read from the options object (the second argument), layered over `fable.settings.RetoldDataBeacon`. The Ultravisor client is built later, on connect -- not in the constructor.

---

## Authenticating

Unlike some connectors, this module does **not** auto-connect from the constructor. Call `connectAsync()` yourself:

```javascript
_Connection.connectAsync(
	(pError, pClient) =>
	{
		if (pError)
		{
			console.error('Connect failed:', pError.message);
			return;
		}

		console.log('Authenticated. connected =', _Connection.connected);
	});
```

`connectAsync()` validates the three required keys, builds the [fable-ultravisor-client](https://fable-retold.github.io/fable-ultravisor-client/), authenticates against the coordinator (`POST /1.0/Authenticate`), and on success registers the connection on `_Fable.MeadowRetoldDataBeaconProvider`. The synchronous `connect()` is a shim that fires `connectAsync()` with no callback -- use the async form when you need to wait or inspect the result.

---

## Relaying Meadow CRUD

Once connected, point a meadow entity at the `RetoldDataBeacon` provider. The provider builds the request path and HTTP verb from the FoxHound `MeadowEndpoints` dialect, prefixes it with the connection hash to form `/1.0/<TargetConnectionHash>/<Entity>`, and calls `dispatchRequest` to ship it across the mesh:

```javascript
const libMeadow = require('meadow');

let tmpBookMeadow = libMeadow.new(_Fable, 'Book')
	.setProvider('RetoldDataBeacon');

tmpBookMeadow.doReads(tmpBookMeadow.query,
	(pError, pQuery, pRecords) =>
	{
		if (pError) return console.error(pError);
		console.log(`Read ${pRecords.length} book(s) from the remote beacon.`);
	});
```

Behind the scenes each operation becomes a `MeadowProxy:Request` work item, routed by AffinityKey (`TargetBeaconName`) to the bound beacon, which runs the HTTP request against its own localhost REST API and returns the JSON response. See [Architecture](architecture.md) for the full flow.

---

## Dispatching a Request Directly

You can also call the transport yourself -- this is what the provider does internally. The response body comes back as a **string** to parse:

```javascript
_Connection.dispatchRequest(
	{ Method: 'GET', Path: '/1.0/bookstore-mssql/Book' },
	(pError, pResponseBody) =>
	{
		if (pError) return console.error(pError.message);
		let tmpRecords = JSON.parse(pResponseBody);
		console.log(`Read ${tmpRecords.length} book(s).`);
	});
```

---

## Introspecting the Remote Schema

The connection delegates introspection to the remote beacon's management capability:

```javascript
_Connection.listTables(
	(pError, pOutputs) =>
	{
		if (pError) return console.error(pError.message);
		console.log('Tables:', pOutputs.Tables);
	});

_Connection.introspectDatabaseSchema(
	(pError, pSchema) =>
	{
		if (pError) return console.error(pError.message);
		console.log('Full remote schema:', pSchema);
	});
```

If the remote beacon hosts more than one database, set `IDBeaconConnection` so introspection targets the right one. See [Configuration & Routing](configuration.md).

---

## Closing

```javascript
_Connection.close(
	() =>
	{
		// Clears connected state, drops the Ultravisor client, and
		// unregisters fable.MeadowRetoldDataBeaconProvider. Never throws.
		console.log('Closed.');
	});
```

---

## Summary

| Step | What It Does |
|------|-------------|
| Configure | Set `UltravisorURL`, `TargetBeaconName`, `TargetConnectionHash` (+ optional creds, timeout, `IDBeaconConnection`) |
| Authenticate | Call `connectAsync()` to log into the coordinator and register the transport |
| Relay CRUD | Set a meadow entity's provider to `'RetoldDataBeacon'`; requests dispatch as `MeadowProxy:Request` work items |
| Route | `TargetBeaconName` is the AffinityKey -- it pins traffic to the bound beacon |
| Introspect | `listTables()` / `introspectDatabaseSchema()` delegate to the remote management capability |
| Close | `close()` drops the client and unregisters the transport |
