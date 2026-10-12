<p align="center">
  <a href="https://ui.com/" target="blank">
    <img src="logo.png" width="240" alt="UniFi Logo" />
  </a>
</p>

<p align="center">A comprehensive developer reference and integration guide for Ubiquiti's <b>UniFi Controller APIs</b> - covering Site Manager Cloud, Cloud Connector Proxy, Local Network Integration, and Classic/Internal APIs.</p>

<p align="center">
  <a href="https://developer.ui.com/"><img src="https://img.shields.io/badge/unifi--api-v10.1.68-0055ff?logo=ubiquiti" alt="UniFi API Version" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-CC--BY--4.0-lightgrey" alt="License: CC BY 4.0" /></a>
</p>

## Overview

The UniFi API allows programmatic control over your entire UniFi network - devices, clients, WLANs, firewalls, vouchers, and more. Ubiquiti provides officially supported Cloud APIs, a secure Cloud Connector Proxy, and local REST endpoints, alongside the legacy internal controller API.

---

## API Flavors

Depending on your architecture, select the appropriate Integration API:

| Flavor                      | Scope                                                   | Auth           | Base URL / Path                                               | Status                       |
| --------------------------- | ------------------------------------------------------- | -------------- | ------------------------------------------------------------- | ---------------------------- |
| **Site Manager API**        | Cloud - monitor/manage deployments at scale             | `X-API-KEY`    | `https://api.ui.com/v1/`                                      | Official, GA                 |
| **Connector Proxy**         | Cloud-to-Local - secure remote access to local consoles | `X-API-KEY`    | `https://api.ui.com/v1/connector/consoles/{consoleId}/proxy/` | Official, firmware >= 5.0.3  |
| **Network Integration API** | Local - per-application control on a local console      | `X-API-KEY`    | `https://{console}/proxy/network/integration/`                | Official, Network App >= 9.3 |
| **Classic Controller API**  | Local - full internal legacy interface                  | Cookie session | `https://{controller}:8443/api/` or `/proxy/network/api/`     | Unofficial but stable        |

> Prefer the **Official Network Integration API** (either directly or routed via the **Connector Proxy**) for new applications. Use the **Classic API** only for features not yet exposed officially (e.g. firewall rules, port forwards, routing).

---

## Cloud Connector Proxy (Proxied Control)

Firmware version **5.0.3+** introduces the UniFi Connector Proxy. This allows your cloud applications to execute local API actions remotely by proxying the request through Ubiquiti's cloud endpoint to the physical console without exposing ports or establishing VPNs.

### Proxy Routing Example (Axios)

Simply replace the base URL and supply the `consoleId`:

```javascript
const client = axios.create({
  baseURL:
    "https://api.ui.com/v1/connector/consoles/YOUR_CONSOLE_ID/proxy/network/integration/v1",
  headers: {
    "X-API-KEY": process.env.UNIFI_API_KEY,
    Accept: "application/json",
  },
});

// Retrieves local network sites securely via the Cloud Proxy
const { data } = await client.get("/sites");
```

---

## Core Concepts

- **Site Structure**: Most endpoints require a site identifier (`default` is the primary site).
- **Response Format**: JSON with standard structure `{ "meta": { "rc": "ok" }, "data": [...] }`. On error: `{ "meta": { "rc": "error", "msg": "api.err.Invalid" } }`.
- **MAC Addresses**: Use lowercase, typically without separators (e.g., `00112233aabb`).
- **IDs**: Most objects use a MongoDB-style `_id` field.

---

## Authentication

### Official API Key (X-API-KEY)

Available on **UniFi OS-based consoles** (UDM, UDR, UCG, UX, UDW, UCG-Ultra, UniFi OS Server). The key inherits the creator admin's permissions.

**Generating a key:**

- Cloud: [unifi.ui.com](https://unifi.ui.com) → your profile → API Keys
- Local: UniFi Network → **Settings → Control Plane → Integrations**

Stateless - no login/logout flow required. Ideal for scripts, CI, and long-running services.

### Classic Login (Cookie Session)

| Operation        | Method | Endpoint                           | Description                 |
| ---------------- | ------ | ---------------------------------- | --------------------------- |
| Login (legacy)   | POST   | `/api/login`                       | Standalone controller       |
| Login (UniFi OS) | POST   | `/api/auth/login`                  | UDM / UCG / UniFi OS Server |
| Verify Session   | GET    | `/api/self`                        | Check session validity      |
| Logout           | POST   | `/api/logout` / `/api/auth/logout` | End session                 |

```json
{
  "username": "admin",
  "password": "your_password",
  "remember": true
}
```

### UniFi OS vs Standalone Controller (Classic)

| Aspect     | Standalone (software controller) | UniFi OS (UDM/UCG/UDR)                 |
| ---------- | -------------------------------- | -------------------------------------- |
| Port       | `8443`                           | `443` (or `11443` for UniFi OS Server) |
| Login      | `/api/login`                     | `/api/auth/login`                      |
| API prefix | `/api/...`                       | `/proxy/network/api/...`               |
| CSRF       | Not required                     | Required on state-changing calls       |

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

---

## Official Site Manager API (Cloud)

Base URL: `https://api.ui.com/v1/` · Auth: `X-API-KEY` · Rate limit: **10,000 req/min** (GA), `429` with `Retry-After` header on overflow.

| Operation         | Method | Endpoint                     | Description                       |
| ----------------- | ------ | ---------------------------- | --------------------------------- |
| List Hosts        | GET    | `/hosts`                     | All UniFi OS hosts on the account |
| Host Details      | GET    | `/hosts/{id}`                | Host metadata, status, uptime     |
| List Sites        | GET    | `/sites`                     | All sites across hosts            |
| List Devices      | GET    | `/devices`                   | Devices across every site         |
| ISP Metrics       | GET    | `/isp-metrics`               | ISP performance samples           |
| Query ISP Metrics | POST   | `/isp-metrics/query`         | Filtered ISP metric query         |
| List SD-WAN       | GET    | `/sdwan-configs`             | SD-WAN configurations             |
| SD-WAN Details    | GET    | `/sdwan-configs/{id}`        | Specific SD-WAN config            |
| SD-WAN Status     | GET    | `/sdwan-configs/{id}/status` | Deployment status                 |

---

## Official Network Integration API (Local & Proxied)

API paths are relative to `/v1` locally or proxied through `/connector/consoles/{id}/proxy/network/integration/v1`.

| Operation           | Method | Endpoint                                                            | Description                                              |
| ------------------- | ------ | ------------------------------------------------------------------- | -------------------------------------------------------- |
| App Info            | GET    | `/info`                                                             | Controller version (`applicationVersion`) & capabilities |
| List Sites          | GET    | `/sites`                                                            | Paginated list of local sites                            |
| List Devices        | GET    | `/sites/{siteId}/devices`                                           | Devices associated with the site                         |
| Device Details      | GET    | `/sites/{siteId}/devices/{deviceId}`                                | Single device info                                       |
| Device Stats        | GET    | `/sites/{siteId}/devices/{deviceId}/statistics/latest`              | Real-time statistics                                     |
| Adopt Device        | POST   | `/sites/{siteId}/devices`                                           | Programmatic device adoption                             |
| Device Action       | POST   | `/sites/{siteId}/devices/{deviceId}/actions`                        | Trigger actions (e.g. `RESTART`)                         |
| Port Action         | POST   | `/sites/{siteId}/devices/{deviceId}/interfaces/ports/{idx}/actions` | Power-cycle switch ports                                 |
| List Clients        | GET    | `/sites/{siteId}/clients`                                           | Connected clients                                        |
| Client Details      | GET    | `/sites/{siteId}/clients/{clientId}`                                | Specific client                                          |
| Client Action       | POST   | `/sites/{siteId}/clients/{clientId}/actions`                        | `AUTHORIZE_GUEST_ACCESS`, etc.                           |
| WiFi Broadcast Info | GET    | `/sites/{siteId}/wifi/broadcasts/{id}`                              | SSID, MLO, & client filtering policies                   |
| List Vouchers       | GET    | `/sites/{siteId}/hotspot/vouchers`                                  | Hotspot vouchers                                         |
| Generate Vouchers   | POST   | `/sites/{siteId}/hotspot/vouchers`                                  | Batch create vouchers                                    |
| Delete Voucher      | DELETE | `/sites/{siteId}/hotspot/vouchers/{id}`                             | Remove voucher                                           |

---

## Classic API Endpoint Reference

All endpoints below are prefixed with `/api/s/{site}/` on a standalone controller, or `/proxy/network/api/s/{site}/` on UniFi OS.

### Device Management

| Operation         | Method | Endpoint             | Description                               |
| ----------------- | ------ | -------------------- | ----------------------------------------- |
| List All Devices  | GET    | `/stat/device`       | Full device payload                       |
| Basic Device List | GET    | `/stat/device-basic` | Lightweight list                          |
| Device Details    | GET    | `/stat/device/{mac}` | Specific device                           |
| Device Commands   | POST   | `/cmd/devmgr`        | e.g. adopt, restart, upgrade, power-cycle |
| Locate (LED)      | POST   | `/cmd/devmgr`        | `set-locate` / `unset-locate`             |

```json
{ "cmd": "restart", "mac": "00:11:22:33:44:55" }
```

### Client Management

| Operation              | Method | Endpoint           | Description                              |
| ---------------------- | ------ | ------------------ | ---------------------------------------- |
| Active Clients         | GET    | `/stat/sta`        | Connected clients                        |
| All Clients            | GET    | `/stat/alluser`    | Historical client logs                   |
| Client Details         | GET    | `/stat/user/{mac}` | Specific client                          |
| Known Clients          | GET    | `/rest/user`       | Configured clients                       |
| Block / Unblock / Kick | POST   | `/cmd/stamgr`      | `block-sta` / `unblock-sta` / `kick-sta` |
| Authorize Guest        | POST   | `/cmd/stamgr`      | `authorize-guest`                        |
| Unauthorize Guest      | POST   | `/cmd/stamgr`      | `unauthorize-guest`                      |
| Forget Client          | POST   | `/cmd/stamgr`      | `forget-sta`                             |

```json
{ "cmd": "authorize-guest", "mac": "aa:bb:cc:dd:ee:ff", "minutes": 60 }
```

### Internet Access Control

| Operation      | Method              | Endpoint                 | Description                 |
| -------------- | ------------------- | ------------------------ | --------------------------- |
| Update Client  | PUT                 | `/rest/user/{client_id}` | Set fixed IP, group, access |
| Traffic Rules  | GET/POST/PUT/DELETE | `/rest/trafficrule`      | L7/bandwidth rules          |
| Traffic Routes | GET/POST/PUT/DELETE | `/rest/trafficroute`     | Policy-based routing        |

### Network Configuration

| Operation      | Method | Endpoint                 | Description           |
| -------------- | ------ | ------------------------ | --------------------- |
| List Networks  | GET    | `/rest/networkconf`      | All LAN/VLAN networks |
| Create Network | POST   | `/rest/networkconf`      | Add network           |
| Update Network | PUT    | `/rest/networkconf/{id}` | Modify                |
| Delete Network | DELETE | `/rest/networkconf/{id}` | Remove                |

### WLAN Configuration

| Operation    | Method | Endpoint              | Description   |
| ------------ | ------ | --------------------- | ------------- |
| List WLANs   | GET    | `/rest/wlanconf`      | All SSIDs     |
| WLAN Details | GET    | `/rest/wlanconf/{id}` | Specific WLAN |
| Create WLAN  | POST   | `/rest/wlanconf`      | New SSID      |
| Update WLAN  | PUT    | `/rest/wlanconf/{id}` | Modify SSID   |
| Delete WLAN  | DELETE | `/rest/wlanconf/{id}` | Remove SSID   |

### Firewall & Security

| Operation        | Method  | Endpoint                   | Description         |
| ---------------- | ------- | -------------------------- | ------------------- |
| List Rules       | GET     | `/rest/firewallrule`       | User-defined rules  |
| Update Rule      | PUT     | `/rest/firewallrule/{id}`  | Enable / edit       |
| List Groups      | GET     | `/rest/firewallgroup`      | Address/port groups |
| Update Group     | PUT     | `/rest/firewallgroup/{id}` | Edit group members  |
| IPS/IDS Settings | GET/PUT | `/rest/setting/ips`        | Threat management   |

### Port Forwarding

| Operation      | Method  | Endpoint                      | Description          |
| -------------- | ------- | ----------------------------- | -------------------- |
| List Forwards  | GET     | `/rest/portforward`           | All forward rules    |
| Create Forward | POST    | `/rest/portforward`           | New rule             |
| Toggle Forward | PUT     | `/rest/portforward/{rule-id}` | Enable / disable     |
| Delete Forward | DELETE  | `/rest/portforward/{rule-id}` | Remove               |
| Port Profiles  | GET/PUT | `/rest/portconf`              | Switch port profiles |

### Routing & DynamicDNS

| Operation         | Method              | Endpoint           | Description         |
| ----------------- | ------------------- | ------------------ | ------------------- |
| Static Routes     | GET/POST/PUT/DELETE | `/rest/routing`    | User-defined routes |
| DynamicDNS Config | GET/POST/PUT        | `/rest/dynamicdns` | DDNS entries        |

### Site Management

| Operation      | Method | Endpoint                     | Description            |
| -------------- | ------ | ---------------------------- | ---------------------- |
| List Sites     | GET    | `/api/self/sites`            | Accessible sites       |
| Sites + Health | GET    | `/api/stat/sites`            | Health / alert summary |
| Site Health    | GET    | `/stat/health`               | Per-subsystem health   |
| Create Site    | POST   | `/api/s/default/cmd/sitemgr` | `add-site`             |
| Delete Site    | POST   | `/api/s/default/cmd/sitemgr` | `delete-site`          |

### Hotspot & Vouchers

| Operation       | Method | Endpoint          | Description       |
| --------------- | ------ | ----------------- | ----------------- |
| List Vouchers   | GET    | `/stat/voucher`   | All vouchers      |
| Create Voucher  | POST   | `/cmd/hotspot`    | `create-voucher`  |
| Delete Voucher  | POST   | `/cmd/hotspot`    | `delete-voucher`  |
| Guest Operators | GET    | `/rest/hotspotop` | Hotspot operators |
| Payments        | GET    | `/stat/payment`   | Guest payments    |

### Events & Alarms

| Operation     | Method | Endpoint      | Description                  |
| ------------- | ------ | ------------- | ---------------------------- |
| List Events   | GET    | `/stat/event` | Site events (capped at 3000) |
| List Alarms   | GET    | `/stat/alarm` | Active alarms                |
| Archive Alarm | POST   | `/cmd/evtmgr` | `archive-alarm`              |

### Backups

| Operation       | Method | Endpoint                | Description     |
| --------------- | ------ | ----------------------- | --------------- |
| List Backups    | POST   | `/cmd/backup`           | `list-backups`  |
| Create Backup   | POST   | `/cmd/backup`           | `backup`        |
| Delete Backup   | POST   | `/cmd/backup`           | `delete-backup` |
| Download Backup | GET    | `/dl/backup/{filename}` | Raw download    |

### Statistics & Monitoring

| Operation     | Method | Endpoint                     | Description                  |
| ------------- | ------ | ---------------------------- | ---------------------------- |
| DPI Stats     | GET    | `/stat/dpi`                  | Deep Packet Inspection       |
| Client DPI    | GET    | `/stat/stadpi`               | Per-client DPI               |
| Daily Report  | GET    | `/stat/report/daily.site`    | Daily metrics                |
| Hourly Report | GET    | `/stat/report/hourly.site`   | Hourly metrics               |
| 5-min Report  | GET    | `/stat/report/5minutes.site` | 5-minute granularity         |
| Gateway Stats | GET    | `/stat/gateway`              | Gateway performance          |
| Rogue APs     | GET    | `/stat/rogueap`              | Detected rogue access points |

### Admin Management

| Operation    | Method | Endpoint          | Description        |
| ------------ | ------ | ----------------- | ------------------ |
| List Admins  | GET    | `/api/stat/admin` | All admins & roles |
| Invite Admin | POST   | `/cmd/sitemgr`    | `invite-admin`     |
| Revoke Admin | POST   | `/cmd/sitemgr`    | `revoke-admin`     |

---

## Pagination and Filtering

| Parameter | Description                            | Example                           |
| --------- | -------------------------------------- | --------------------------------- |
| `_limit`  | Max items to return                    | `/stat/sta?_limit=50`             |
| `_start`  | Offset                                 | `/stat/sta?_start=50`             |
| `_sort`   | Sort field (prefix `-` for descending) | `/stat/sta?_sort=-last_seen`      |
| `mac`     | Filter by MAC                          | `/stat/sta?mac=001122334455`      |
| `ip`      | Filter by IP                           | `/stat/sta?ip=192.168.1.100`      |
| `within`  | Historical window in hours             | `/stat/sta?within=24`             |
| `attrs`   | Comma-separated attributes to include  | `/stat/sta?attrs=mac,ip,hostname` |

```javascript
const response = await api.get("/api/s/default/stat/sta", {
  params: {
    _limit: 10,
    _sort: "-last_seen",
    attrs: "mac,hostname,ip,signal,tx_bytes,rx_bytes",
  },
});
```

---

## Rate Limiting & Error Handling

- **Official APIs** enforce per-minute quotas (10k/min for Site Manager GA). Overflow returns `429` with an RFC-compliant `Retry-After` header.
- **Classic API** has no documented quota but will return `401` on stale sessions - re-login on `meta.rc = "error"` with `api.err.LoginRequired`.
- **Common error codes** (`meta.msg`):
  - `api.err.Invalid` - bad request body
  - `api.err.LoginRequired` - session expired
  - `api.err.NoPermission` - admin role too low
  - `api.err.Ubic2faTokenRequired` - 2FA challenge

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

---

## WebSocket Events (Real-time)

The controller exposes a real-time feed:

- Standalone: `wss://{controller}:8443/wss/s/{site}/events`
- UniFi OS: `wss://{console}/proxy/network/wss/s/{site}/events`

Send the session cookie (or `X-API-KEY` where supported) on the upgrade request. Useful for live client join/leave, device state, alarm push, and voucher consumption without polling `/stat/event`.
