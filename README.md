<h1 id="unifi-controller-api">
UniFi Controller API - Comprehensive Guide
</h1>

A comprehensive, up-to-date reference for developers working with Ubiquiti's UniFi APIs — covering both the **Official API** (Site Manager cloud + Network Integration API with `X-API-KEY`) and the **Classic/Internal Controller API** (cookie session).

## Table of Contents

- [Overview](#overview)
- [API Flavors](#api-flavors)
- [Core Concepts](#core-concepts)
- [Authentication](#authentication)
  - [Official API Key (X-API-KEY)](#official-api-key-x-api-key)
  - [Classic Login (Cookie Session)](#classic-login-cookie-session)
  - [UniFi OS vs Standalone Controller](#unifi-os-vs-standalone-controller)
  - [CSRF Token](#csrf-token)
  - [MFA / 2FA](#mfa--2fa)
- [Quick Start](#quick-start)
- [Official Site Manager API (Cloud)](#official-site-manager-api-cloud)
- [Official Network Integration API (Local)](#official-network-integration-api-local)
- [Classic API Endpoint Reference](#classic-api-endpoint-reference)
  - [Device Management](#device-management)
  - [Client Management](#client-management)
  - [Internet Access Control](#internet-access-control)
  - [Network Configuration](#network-configuration)
  - [WLAN Configuration](#wlan-configuration)
  - [Firewall & Security](#firewall--security)
  - [Port Forwarding](#port-forwarding)
  - [Routing & DynamicDNS](#routing--dynamicdns)
  - [Site Management](#site-management)
  - [Hotspot & Vouchers](#hotspot--vouchers)
  - [Events & Alarms](#events--alarms)
  - [Backups](#backups)
  - [Statistics & Monitoring](#statistics--monitoring)
  - [Admin Management](#admin-management)
- [Pagination and Filtering](#pagination-and-filtering)
- [Rate Limiting & Error Handling](#rate-limiting--error-handling)
- [WebSocket Events (Real-time)](#websocket-events-real-time)
- [Community SDKs & Tools](#community-sdks--tools)

## Overview

The UniFi API allows programmatic control over your entire UniFi network — devices, clients, WLANs, firewalls, vouchers, and more. Ubiquiti provides two officially supported APIs, alongside the long-standing internal controller API that community tooling has relied on for years.

## API Flavors

| Flavor | Scope | Auth | Base URL | Status |
| --- | --- | --- | --- | --- |
| **Site Manager API** | Cloud — monitor/manage deployments at scale | `X-API-KEY` | `https://api.ui.com/v1/` | Official, GA |
| **Network Integration API** | Local — per-application control on a console | `X-API-KEY` | `https://{console}/proxy/network/integration/v1/` | Official, v9.3+ |
| **Classic Controller API** | Local — full internal surface | Cookie session | `https://{controller}:8443/api/...` or `/proxy/network/api/...` on UniFi OS | Unofficial but stable |

> Prefer the Official APIs for new integrations. Fall back to the Classic API for endpoints not yet exposed officially (firewall rules, port forwards, detailed statistics, etc.).

## Core Concepts

- **Site Structure**: Most endpoints require a site identifier (`default` is the primary site).
- **Response Format**: JSON with standard structure `{ "meta": { "rc": "ok" }, "data": [...] }`. On error: `{ "meta": { "rc": "error", "msg": "api.err.Invalid" } }`.
- **MAC Addresses**: Use lowercase, typically without separators (e.g., `00112233aabb`).
- **IDs**: Most objects use a MongoDB-style `_id` field.

## Authentication

### Official API Key (X-API-KEY)

Available on **UniFi OS-based consoles** (UDM, UDR, UCG, UX, UDW, UCG-Ultra, UniFi OS Server). The key inherits the creator admin's permissions.

**Generating a key:**
- Cloud: <https://unifi.ui.com> → your profile → API Keys
- Local: UniFi Network → **Settings → Control Plane → Integrations**

```javascript
const response = await axios.get(
  "https://192.168.1.1/proxy/network/integration/v1/sites",
  {
    headers: { "X-API-KEY": process.env.UNIFI_API_KEY, Accept: "application/json" },
    httpsAgent: new https.Agent({ rejectUnauthorized: false }),
  }
);
```

Stateless — no login/logout flow required. Ideal for scripts, CI, and long-running services.

### Classic Login (Cookie Session)

| Operation      | Method | Endpoint      | Description                     |
| -------------- | ------ | ------------- | ------------------------------- |
| Login (legacy) | POST   | `/api/login`  | Standalone controller           |
| Login (UniFi OS) | POST | `/api/auth/login` | UDM / UCG / UniFi OS Server  |
| Verify Session | GET    | `/api/self`   | Check session validity          |
| Logout         | POST   | `/api/logout` / `/api/auth/logout` | End session       |

```json
{
  "username": "admin",
  "password": "your_password",
  "remember": true
}
```

### UniFi OS vs Standalone Controller

| Aspect | Standalone (software controller) | UniFi OS (UDM/UCG/UDR) |
| --- | --- | --- |
| Port | `8443` | `443` (or `11443` for UniFi OS Server) |
| Login | `/api/login` | `/api/auth/login` |
| API prefix | `/api/...` | `/proxy/network/api/...` |
| CSRF | Not required | Required on state-changing calls |

### CSRF Token

On UniFi OS, the login response includes a `X-CSRF-Token` header (or `TOKEN` cookie). Include this header on every `POST`/`PUT`/`DELETE`:

```javascript
api.defaults.headers.common["X-CSRF-Token"] = loginResp.headers["x-csrf-token"];
```

### MFA / 2FA

If the admin has 2FA enabled, the first login returns HTTP `499` with `meta.msg = "api.err.Ubic2faTokenRequired"`. Resubmit with:

```json
{ "username": "admin", "password": "pw", "token": "123456" }
```

## Quick Start

```javascript
// Official API (recommended)
const client = axios.create({
  baseURL: "https://192.168.1.1/proxy/network/integration/v1",
  headers: { "X-API-KEY": process.env.UNIFI_API_KEY },
  httpsAgent: new https.Agent({ rejectUnauthorized: false }),
});

const { data } = await client.get("/sites");
```

```javascript
// Classic API (cookie session, UniFi OS)
const jar = new CookieJar();
const api = wrapper(axios.create({
  baseURL: "https://192.168.1.1",
  jar, withCredentials: true,
  httpsAgent: new https.Agent({ rejectUnauthorized: false }),
}));

const login = await api.post("/api/auth/login", {
  username: "admin", password: "password", remember: true,
});
api.defaults.headers.common["X-CSRF-Token"] = login.headers["x-csrf-token"];
```

## Official Site Manager API (Cloud)

Base URL: `https://api.ui.com/v1/` · Auth: `X-API-KEY` · Rate limit: **10,000 req/min** (GA), `429` with `Retry-After` header on overflow.

| Operation | Method | Endpoint | Description |
| --- | --- | --- | --- |
| List Hosts | GET | `/hosts` | All UniFi OS hosts on the account |
| Host Details | GET | `/hosts/{id}` | Host metadata, status, uptime |
| List Sites | GET | `/sites` | All sites across hosts |
| List Devices | GET | `/devices` | Devices across every site |
| ISP Metrics | GET | `/isp-metrics` | ISP performance samples |
| Query ISP Metrics | POST | `/isp-metrics/query` | Filtered ISP metric query |
| List SD-WAN | GET | `/sdwan-configs` | SD-WAN configurations |
| SD-WAN Details | GET | `/sdwan-configs/{id}` | Specific SD-WAN config |
| SD-WAN Status | GET | `/sdwan-configs/{id}/status` | Deployment status |

> Currently read-mostly. Write scope (adopt, configure) is being rolled out through 2026.

## Official Network Integration API (Local)

Base URL: `https://{console}/proxy/network/integration/v1/` · Auth: `X-API-KEY` · Version: `v9.3.43`+ (check `/info`).

| Operation | Method | Endpoint | Description |
| --- | --- | --- | --- |
| App Info | GET | `/info` | Controller version & capabilities |
| List Sites | GET | `/sites` | Local sites |
| List Devices | GET | `/sites/{siteId}/devices` | Devices in site |
| Device Details | GET | `/sites/{siteId}/devices/{deviceId}` | Single device |
| Device Stats | GET | `/sites/{siteId}/devices/{deviceId}/statistics/latest` | Latest metrics |
| Device Action | POST | `/sites/{siteId}/devices/{deviceId}/actions` | e.g. `RESTART` |
| Port Action | POST | `/sites/{siteId}/devices/{deviceId}/interfaces/ports/{idx}/actions` | e.g. `POWER_CYCLE` |
| List Clients | GET | `/sites/{siteId}/clients` | Connected clients |
| Client Details | GET | `/sites/{siteId}/clients/{clientId}` | Specific client |
| Client Action | POST | `/sites/{siteId}/clients/{clientId}/actions` | `AUTHORIZE_GUEST_ACCESS`, etc. |
| List Vouchers | GET | `/sites/{siteId}/hotspot/vouchers` | Hotspot vouchers |
| Generate Vouchers | POST | `/sites/{siteId}/hotspot/vouchers` | Batch create |
| Delete Voucher | DELETE | `/sites/{siteId}/hotspot/vouchers/{id}` | Remove voucher |

```json
{ "action": "RESTART" }
```

## Classic API Endpoint Reference

All endpoints below are prefixed with `/api/s/{site}/` on a standalone controller, or `/proxy/network/api/s/{site}/` on UniFi OS.

### Device Management

| Operation        | Method | Endpoint                                | Description              |
| ---------------- | ------ | --------------------------------------- | ------------------------ |
| List All Devices | GET    | `/stat/device`                          | Full device payload      |
| Basic Device List | GET   | `/stat/device-basic`                    | Lightweight list         |
| Device Details   | GET    | `/stat/device/{mac}`                    | Specific device          |
| Restart / Adopt / Upgrade / Power-cycle / Spectrum-scan | POST | `/cmd/devmgr` | devmgr commands |
| Locate (LED)     | POST   | `/cmd/devmgr`                           | `set-locate` / `unset-locate` |

```json
{ "cmd": "restart", "mac": "00:11:22:33:44:55" }
{ "cmd": "adopt", "mac": "00:11:22:33:44:55" }
{ "cmd": "upgrade", "mac": "00:11:22:33:44:55" }
{ "cmd": "power-cycle", "mac": "00:11:22:33:44:55", "port_idx": 5 }
```

### Client Management

| Operation         | Method | Endpoint              | Description                        |
| ----------------- | ------ | --------------------- | ---------------------------------- |
| Active Clients    | GET    | `/stat/sta`           | Connected clients                  |
| All Clients       | GET    | `/stat/alluser`       | Including historical               |
| Client Details    | GET    | `/stat/user/{mac}`    | Specific client                    |
| Known Clients     | GET    | `/rest/user`          | Configured clients                 |
| Block / Unblock / Kick | POST | `/cmd/stamgr`       | `block-sta` / `unblock-sta` / `kick-sta` |
| Authorize Guest   | POST   | `/cmd/stamgr`         | `authorize-guest`                  |
| Unauthorize Guest | POST   | `/cmd/stamgr`         | `unauthorize-guest`                |
| Forget Client     | POST   | `/cmd/stamgr`         | `forget-sta`                       |

```json
{ "cmd": "authorize-guest", "mac": "aa:bb:cc:dd:ee:ff", "minutes": 60, "up": 1024, "down": 4096, "bytes": 1073741824 }
```

### Internet Access Control

| Operation        | Method | Endpoint                    | Description                   |
| ---------------- | ------ | --------------------------- | ----------------------------- |
| Update Client    | PUT    | `/rest/user/{client_id}`    | Set fixed IP, group, access   |
| Traffic Rules    | GET/POST/PUT/DELETE | `/rest/trafficrule` | L7/bandwidth rules     |
| Traffic Routes   | GET/POST/PUT/DELETE | `/rest/trafficroute` | Policy-based routing   |

```json
{
  "use_fixedip": true,
  "network_access": "deny",
  "fixedip": "192.168.1.100",
  "usergroup_id": "existing-group-id"
}
```

### Network Configuration

| Operation      | Method | Endpoint                        | Description                 |
| -------------- | ------ | ------------------------------- | --------------------------- |
| List Networks  | GET    | `/rest/networkconf`             | All LAN/VLAN networks       |
| Create Network | POST   | `/rest/networkconf`             | Add network                 |
| Update Network | PUT    | `/rest/networkconf/{id}`        | Modify                      |
| Delete Network | DELETE | `/rest/networkconf/{id}`        | Remove                      |

### WLAN Configuration

| Operation    | Method | Endpoint                     | Description           |
| ------------ | ------ | ---------------------------- | --------------------- |
| List WLANs   | GET    | `/rest/wlanconf`             | All SSIDs             |
| WLAN Details | GET    | `/rest/wlanconf/{id}`        | Specific WLAN         |
| Create WLAN  | POST   | `/rest/wlanconf`             | New SSID              |
| Update WLAN  | PUT    | `/rest/wlanconf/{id}`        | Modify SSID           |
| Delete WLAN  | DELETE | `/rest/wlanconf/{id}`        | Remove SSID           |
| WLAN Groups  | GET    | `/rest/wlangroup`            | AP group mapping      |

```json
{
  "name": "Guest Network",
  "x_passphrase": "guestpassword",
  "security": "wpapsk",
  "wpa_mode": "wpa2",
  "enabled": true,
  "is_guest": true,
  "vlan_enabled": false
}
```

### Firewall & Security

| Operation         | Method | Endpoint                           | Description               |
| ----------------- | ------ | ---------------------------------- | ------------------------- |
| List Rules        | GET    | `/rest/firewallrule`               | User-defined rules        |
| Update Rule       | PUT    | `/rest/firewallrule/{id}`          | Enable / edit             |
| List Groups       | GET    | `/rest/firewallgroup`              | Address/port groups       |
| Update Group      | PUT    | `/rest/firewallgroup/{id}`         | Edit group members        |
| IPS/IDS Settings  | GET/PUT | `/rest/setting/ips`               | Threat management         |

### Port Forwarding

| Operation       | Method | Endpoint                       | Description          |
| --------------- | ------ | ------------------------------ | -------------------- |
| List Forwards   | GET    | `/rest/portforward`            | All forward rules    |
| Create Forward  | POST   | `/rest/portforward`            | New rule             |
| Toggle Forward  | PUT    | `/rest/portforward/{rule-id}`  | Enable / disable     |
| Delete Forward  | DELETE | `/rest/portforward/{rule-id}`  | Remove               |
| Port Profiles   | GET/PUT | `/rest/portconf`              | Switch port profiles |

### Routing & DynamicDNS

| Operation         | Method | Endpoint                | Description                |
| ----------------- | ------ | ----------------------- | -------------------------- |
| Static Routes     | GET/POST/PUT/DELETE | `/rest/routing` | User-defined routes |
| DynamicDNS Config | GET/POST/PUT | `/rest/dynamicdns` | DDNS entries            |

### Site Management

| Operation   | Method | Endpoint                     | Description              |
| ----------- | ------ | ---------------------------- | ------------------------ |
| List Sites  | GET    | `/api/self/sites`            | Accessible sites         |
| Sites + Health | GET | `/api/stat/sites`            | Health / alert summary   |
| Site Health | GET    | `/stat/health`               | Per-subsystem health     |
| Create Site | POST   | `/api/s/default/cmd/sitemgr` | `add-site`               |
| Delete Site | POST   | `/api/s/default/cmd/sitemgr` | `delete-site`            |
| Move Device | POST   | `/api/s/default/cmd/sitemgr` | `move-device`            |

```json
{ "cmd": "add-site", "name": "New Office", "desc": "Branch office" }
```

### Hotspot & Vouchers

| Operation         | Method | Endpoint                  | Description              |
| ----------------- | ------ | ------------------------- | ------------------------ |
| List Vouchers     | GET    | `/stat/voucher`           | All vouchers             |
| Create Voucher    | POST   | `/cmd/hotspot`            | `create-voucher`         |
| Delete Voucher    | POST   | `/cmd/hotspot`            | `delete-voucher`         |
| Guest Operators   | GET    | `/rest/hotspotop`         | Hotspot operators        |
| Payments          | GET    | `/stat/payment`           | Guest payments           |

```json
{ "cmd": "create-voucher", "n": 10, "expire": 1440, "quota": 1, "note": "Lobby" }
```

### Events & Alarms

| Operation     | Method | Endpoint         | Description                        |
| ------------- | ------ | ---------------- | ---------------------------------- |
| List Events   | GET    | `/stat/event`    | Site events (capped at 3000)       |
| List Alarms   | GET    | `/stat/alarm`    | Active alarms                      |
| Archive Alarm | POST   | `/cmd/evtmgr`    | `archive-alarm`                    |

### Backups

| Operation       | Method | Endpoint         | Description        |
| --------------- | ------ | ---------------- | ------------------ |
| List Backups    | POST   | `/cmd/backup`    | `list-backups`     |
| Create Backup   | POST   | `/cmd/backup`    | `backup`           |
| Delete Backup   | POST   | `/cmd/backup`    | `delete-backup`    |
| Download Backup | GET    | `/dl/backup/{filename}` | Raw download |

### Statistics & Monitoring

| Operation        | Method | Endpoint                       | Description                      |
| ---------------- | ------ | ------------------------------ | -------------------------------- |
| DPI Stats        | GET    | `/stat/dpi`                    | Deep Packet Inspection           |
| Client DPI       | GET    | `/stat/stadpi`                 | Per-client DPI                   |
| Daily Report     | GET    | `/stat/report/daily.site`      | Daily metrics                    |
| Hourly Report    | GET    | `/stat/report/hourly.site`     | Hourly metrics                   |
| 5-min Report     | GET    | `/stat/report/5minutes.site`   | 5-minute granularity             |
| Gateway Stats    | GET    | `/stat/gateway`                | Gateway performance              |
| Rogue APs        | GET    | `/stat/rogueap`                | Detected rogue access points     |
| Known Rogues     | GET    | `/rest/rogueknown`             | Acknowledged rogues              |

### Admin Management

| Operation        | Method | Endpoint               | Description                     |
| ---------------- | ------ | ---------------------- | ------------------------------- |
| List Admins      | GET    | `/api/stat/admin`      | All admins & roles              |
| Invite Admin     | POST   | `/cmd/sitemgr`         | `invite-admin`                  |
| Revoke Admin     | POST   | `/cmd/sitemgr`         | `revoke-admin`                  |

## Pagination and Filtering

| Parameter | Description                                       | Example                                        |
| --------- | ------------------------------------------------- | ---------------------------------------------- |
| `_limit`  | Max items to return                               | `/stat/sta?_limit=50`                          |
| `_start`  | Offset                                            | `/stat/sta?_start=50`                          |
| `_sort`   | Sort field (prefix `-` for descending)            | `/stat/sta?_sort=-last_seen`                   |
| `mac`     | Filter by MAC                                     | `/stat/sta?mac=001122334455`                   |
| `ip`      | Filter by IP                                      | `/stat/sta?ip=192.168.1.100`                   |
| `within`  | Historical window in hours                        | `/stat/sta?within=24`                          |
| `attrs`   | Comma-separated attributes to include             | `/stat/sta?attrs=mac,ip,hostname`              |

```javascript
const response = await api.get("/api/s/default/stat/sta", {
  params: {
    _limit: 10,
    _sort: "-last_seen",
    attrs: "mac,hostname,ip,signal,tx_bytes,rx_bytes",
  },
});
```

## Rate Limiting & Error Handling

- **Official APIs** enforce per-minute quotas (10k/min for Site Manager GA). Overflow returns `429` with an RFC-compliant `Retry-After` header.
- **Classic API** has no documented quota but will return `401` on stale sessions — re-login on `meta.rc = "error"` with `api.err.LoginRequired`.
- **Common error codes** (`meta.msg`):
  - `api.err.Invalid` — bad request body
  - `api.err.LoginRequired` — session expired
  - `api.err.NoPermission` — admin role too low
  - `api.err.Ubic2faTokenRequired` — 2FA challenge

```javascript
try {
  await client.post(...);
} catch (err) {
  if (err.response?.status === 429) {
    const wait = Number(err.response.headers["retry-after"] ?? 1);
    await new Promise(r => setTimeout(r, wait * 1000));
  }
}
```

## WebSocket Events (Real-time)

The controller exposes a real-time feed:

- Standalone: `wss://{controller}:8443/wss/s/{site}/events`
- UniFi OS: `wss://{console}/proxy/network/wss/s/{site}/events`

Send the session cookie (or `X-API-KEY` where supported) on the upgrade request. Useful for live client join/leave, device state, alarm push, and voucher consumption without polling `/stat/event`.

## Community SDKs & Tools

- [Art-of-WiFi/UniFi-API-client](https://github.com/Art-of-WiFi/UniFi-API-client) — PHP client, most complete surface
- [Art-of-WiFi/UniFi-API-browser](https://github.com/Art-of-WiFi/UniFi-API-browser) — interactive data browser
- [aiounifi](https://github.com/Kane610/aiounifi) — async Python client (used by Home Assistant)
- [pyunifi](https://github.com/finish06/pyunifi) — Python sync client
- [node-unifi](https://github.com/jens-maus/node-unifi) — Node.js client
- [Pulumi UniFi provider](https://www.pulumi.com/registry/packages/unifi/) — IaC for networks/WLANs/firewalls

---

**References:**
[UniFi Developer Portal](https://developer.ui.com/) ·
[Site Manager API](https://developer.ui.com/site-manager/v1.0.0/gettingstarted) ·
[Network Integration API](https://developer.ui.com/network/v9.3.43/gettingstarted) ·
[Getting Started (Help Center)](https://help.ui.com/hc/en-us/articles/30076656117655-Getting-Started-with-the-Official-UniFi-API) ·
[Ubiquiti Community Wiki](https://ubntwiki.com/products/software/unifi-controller/api)
