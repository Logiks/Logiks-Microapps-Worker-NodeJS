# Logiks Microapp Worker

A **Microapp Worker** is an independent, containerisable Node.js service that bundles everything needed to deliver a fully functional application once it connects to a properly configured **MicroAppServer**. Workers can be hot‑plugged into and removed from the AppServer at runtime, enabling rapid development and deployment of services **without restarting the full service stack**.

Each worker exposes a `plugins/` folder where plugins (microapps) are dropped in and hot‑loaded into the running process. This decouples the enterprise platform from individual development requirements — teams can ship features as self‑contained plugins.

## How It Works

The worker is built on [Moleculer](https://moleculer.services/) and joins the cluster over a shared transporter (Redis, NATS, etc.). On boot it:

1. **Loads helpers and controllers** from [helpers/](helpers/) and [controllers/](controllers/) into the global scope.
2. **Starts a Plugin broker** ([helpers/boostrap.js](helpers/boostrap.js)) using `nodeID = worker-<NODE_ID>-<host>-<pid>`, and loads local services from [services/](services/).
3. **Catalogs and activates plugins** ([helpers/plugins.js](helpers/plugins.js)) — reads each plugin's `logiks.json`, optionally installs plugin dependencies, registers routes/actions, and exposes source/www endpoints.
4. **Registers with the main broker** by calling `system.registerWorker` (authenticated with `CLUSTER_TOKEN`), then sends a heartbeat every 10s via `system.workerHeartbeat`.
5. **Handles lifecycle events** — auto re‑registers on transporter reconnect and performs a graceful drain (`system.drainWorker`) on `SIGINT`/`SIGTERM` for safe rolling restarts.

Workers support **blue/green deployment** via the `WORKER_COLOR` flag, so traffic can be shifted between worker colors without downtime.

## Prerequisites

- **Node.js** 18+ and npm
- A running **transporter** reachable by both the worker and the AppServer — e.g. Redis (`redis://127.0.0.1:6379`) or NATS (`nats://localhost:4222`)
- A running **MicroAppServer** configured with the same transporter and serializer and a matching `CLUSTER_TOKEN`
- *(optional)* [PM2](https://pm2.keymetrics.io/) for production process management

## Getting Started

### Installation

```bash
git clone git@github.com:Logiks/Logiks-Microapps-Boilerplate-NodeJS.git
cd Logiks-Microapps-Boilerplate-NodeJS/
npm install
```

### Configuration

Copy the sample environment file and edit it so the worker points at the **same transporter and serializer** the MicroAppServer is connected, using the **same `CLUSTER_TOKEN`**:

```bash
cp env_sample .env
```

| Variable | Description | Example |
| --- | --- | --- |
| `TRANSPORTER` | Connection string for the cluster transporter. Must match the AppServer. | `redis://127.0.0.1:6379` |
| `TRANSPORTER_SERIALIZER` | Connection payload serializer for the transporter. Must match the AppServer. (`json`, `notepack`, `msgpack`) | json |
| `NODE_ID` | Identifier for this worker; combined with host and pid to form the `nodeID`. | `demo1223` |
| `CLUSTER_TOKEN` | Shared secret used to authenticate this worker with the main broker. | `super-secret-cluster-token` |
| `PLUGIN_LOG_LEVEL` | Plugin log level (`error`, `warn`, `info`, `debug`, `trace`). | `error` |
| `ENABLE_DBMIGRATION` | When `true`, runs plugin DB migrations after connecting. | `false` |
| `ENABLE_PLUGINS_INSTALL_DEPS` | When `true`, installs each plugin's declared npm dependencies on boot. | `true` |
| `NAMESPACE` | *(optional)* Plugin namespace. Defaults to `default`. | `default` |
| `WORKER_COLOR` | *(optional)* Blue/green deployment color. Defaults to `blue`. | `blue` |

### Running

Development:

```bash
npm start
```

Production (via PM2 — see [ecosystem.config.js](ecosystem.config.js)):

```bash
npm run deploy              # npm install && pm2 reload ecosystem.config.js
npm run deploy_production   # same, with --env production
```

## Project Structure

```
.
├── index.js              # Entry point — loads helpers/controllers, starts broker, loads plugins
├── logiks.json           # Worker app manifest (name, version, policies, dependencies)
├── ecosystem.config.js   # PM2 process configuration
├── env_sample            # Sample environment configuration
├── helpers/              # Globally available helpers (boostrap, plugins, baseapp, commons, jitcompiler)
├── controllers/          # Controllers auto-loaded and initialized at boot
├── services/             # Local Moleculer services (*.service.js), including admin services
├── plugins/              # Hot-loaded microapps (each with its own logiks.json)
└── www/                  # Static assets
```

## Plugins

Each subfolder of [plugins/](plugins/) is a microapp. A plugin is recognised only if it contains a `logiks.json` config file. Folders prefixed with `z_`, `x_`, `temp_` (and `node_modules`) are skipped.

A typical plugin contains:

| File / Folder | Purpose |
| --- | --- |
| `logiks.json` | Plugin manifest — name, policies, and npm `dependencies` to install. |
| `routes.json` | Declares HTTP routes mapped to controller methods (`CONTROLLER.method`) or static data. |
| `api.js` | Plugin controller — exported functions callable from routes and across the cluster. |
| `www/` | Static files served via the plugin's `www` endpoint. |
| `forms/`, `pages/`, `dashboards/`, `reports/`, `menus/`, `dbschema/`, `i18n/` | Plugin resources, exposed through the generated `source` endpoint. |
| `api/` | Other helper files and required js files to be included into `api.js` |

Routes declared in `routes.json` are registered under `/api/services/<plugin>/<path>`. Example:

```json
{
  "enabled": true,
  "routes": {
    "/": { "method": "POST", "data": "PROFILE.test1", "format": "json" },
    "/:actionid": { "method": "GET", "data": "PROFILE.test2", "format": "json" }
  }
}
```

When `ENABLE_PLUGINS_INSTALL_DEPS=true`, dependencies declared across all plugins' `logiks.json` files are aggregated and installed into `plugins/` on boot, so each plugin can use any npm package it needs.

## License

See [LICENSE](LICENSE).
