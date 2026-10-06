# Dell FleetOps iDRAC Manager v0.2

Self-hosted web UI for centralized Dell PowerEdge iDRAC inventory, firmware compliance and controlled Redfish firmware updates.

## v0.2 features
- Central iDRAC inventory and encrypted stored credentials
- Redfish hardware + firmware inventory
- Approved firmware baselines per PowerEdge model/component
- Compliance scans: current vs approved target
- Server groups
- Preflight health/power checks
- Rolling multi-server update batches with stop-on-failure
- Redfish TaskService URI capture/status support
- Maintenance-window data model/API for policy integration
- Job/batch history
- Docker deployment
- Simulation mode enabled by default
- Standalone demo dashboard (`docs/demo.html`) for previewing the UI without a backend

## Demo dashboard
[`docs/demo.html`](docs/demo.html) is a standalone, interactive preview of the FleetOps UI. Open it in any browser; it needs no backend, Docker or iDRAC access, and uses built-in sample data only.

It shows:
- Fleet summary: managed servers, health, firmware compliance %, servers needing updates, jobs in the last 24 h
- Compliance by component (iDRAC, BIOS, PERC, NIC), health per server group, recent activity
- Fleet inventory with group filters, search, per-server firmware detail (current → target), scan, add/remove and group assignment
- Simulated rolling updates with preflight checks (health, power) and stop-on-failure
- Approved firmware baselines (HTTPS URIs only), job history with Redfish task URIs, and maintenance windows on a 24-hour timeline

Suggested walkthrough: select `prd-db-02` and `edge-store-17` and run a BIOS rolling update, then add `dr-app-01` (health Critical) to see a preflight failure stop the batch. Changes are kept in memory and reset on page reload.

## Start
```bash
cp .env.example .env
# Replace SECRET_KEY before storing real credentials.
docker compose up -d --build
```
Open `http://SERVER:8080`.

## Safe evaluation
Leave `SIMULATION_MODE=true`. Add test servers in the UI; Redfish calls are simulated. Define an approved baseline, run a compliance scan, select servers, then test a rolling update.

## Onboarding Dell servers

### How FleetOps talks to servers
FleetOps is **agentless and IP-based**. It connects to each server's **iDRAC** (not the host OS) using the **DMTF Redfish REST API over HTTPS (TCP 443)**, authenticated with an iDRAC username and password.

| Protocol | Used? | Notes |
|---|---|---|
| Redfish over HTTPS (443) | **Yes** | All inventory, health, compliance and firmware update calls |
| SNMP | No | Not used for discovery, polling or traps |
| SSH / RACADM | No | Not used |
| OS agent / OpenManage Server Administrator | No | Nothing is installed on the server or its OS |

Redfish endpoints used:
- `/redfish/v1/Systems/System.Embedded.1`: model, service tag, BIOS version, health, power state
- `/redfish/v1/Managers/iDRAC.Embedded.1`: iDRAC firmware version
- `/redfish/v1/UpdateService/FirmwareInventory`: per-component firmware versions
- `/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate`: firmware install (iDRAC pulls the package from an HTTPS URI)
- `/redfish/v1/TaskService/Tasks/...`: update task status

### Prerequisites per server
1. **iDRAC network configured:** the iDRAC dedicated or shared LOM port has a static IP (or a stable DHCP reservation/DNS name) on your management network.
2. **Redfish enabled:** it is on by default on iDRAC9 and on iDRAC8 2.40.40.40 and later. Check it under *iDRAC Settings → Services → Redfish*. Older iDRAC7/8 firmware must be upgraded first.
3. **Service account:** create a dedicated local iDRAC user (or directory account) for FleetOps.
   - *Read Only* is enough for scans, firmware inventory, compliance and preflight.
   - *Administrator* (or an Operator role with Configure privileges) is required to run firmware updates.
4. **Network access:**
   - FleetOps backend → iDRAC on **TCP 443**.
   - iDRAC → your firmware repository on **TCP 443**, needed only for updates, because the iDRAC downloads the package itself.
5. **TLS (recommended):** install trusted certificates on each iDRAC and set `VERIFY_TLS=true`. With the default `VERIFY_TLS=false`, self-signed iDRAC certificates are accepted.

Quick connectivity test from the FleetOps host:
```bash
curl -k -u <idrac_user>:<password> https://<idrac_ip>/redfish/v1/Systems/System.Embedded.1
```
If this returns JSON, FleetOps can manage the server.

### Adding servers
In the UI, choose **Add server** and enter a display name, the **iDRAC IP or hostname**, and the iDRAC username and password. The password is stored encrypted with a key derived from `SECRET_KEY`. Changing `SECRET_KEY` later makes stored credentials unreadable, so you would need to re-enter them.

Or use the API:
```bash
curl -X POST http://SERVER:8080/api/servers \
  -H 'Content-Type: application/json' \
  -d '{"name":"prd-db-01","address":"10.10.20.11","username":"fleetops","password":"********"}'
```
Then run **Scan** (`POST /api/servers/{id}/scan`) to pull the model, service tag, BIOS/iDRAC versions, health and power state, and assign the server to a group.

> FleetOps has no automatic network discovery (no subnet sweep, SNMP or SLP). Each iDRAC is added by address. To onboard many servers, loop over a CSV and call `POST /api/servers`.

With `SIMULATION_MODE=true` no real iDRAC is contacted. Set it to `false` to onboard real hardware.

## Compliance and Dell update information

### Where the data comes from
- **Installed versions** come **live from each iDRAC** via Redfish `FirmwareInventory` whenever you run a compliance scan.
- **Target versions** come from **approved baselines that you define** in FleetOps (per PowerEdge model + component: target version + HTTPS package URI).

**FleetOps does not connect to Dell support (dell.com, the Dell catalog, TechDirect or SupportAssist).** It does not download catalogs, check warranty or fetch packages from Dell. This is intentional: an administrator decides which Dell release is approved before anything can be installed (see *Firmware baseline workflow* below).

### Getting update information from Dell
Use one of these Dell sources to decide what to approve:
1. **Dell Support site:** enter the server's **service tag** (shown in FleetOps after a scan) at dell.com/support → *Drivers & Downloads* to see the latest BIOS, iDRAC, PERC, NIC and other packages, with criticality and release notes.
2. **Dell Repository Manager (DRM):** build a repository for your PowerEdge models from the Dell enterprise catalog (`https://downloads.dell.com/catalog/Catalog.xml.gz`), then host the resulting Dell Update Packages (DUPs) on your internal HTTPS server.
3. **Dell Security Advisories (DSAs):** watch them to prioritize security-critical BIOS and iDRAC releases.

### Compliance workflow
1. Download the approved Dell DUPs (`.EXE` packages for PowerEdge) and host them on an internal HTTPS repository that every iDRAC can reach.
2. In **Baselines**, add an entry per model/component:
   - **Model:** exactly as reported by the scan, for example `PowerEdge R750`.
   - **Component:** exactly as shown in the server's firmware inventory/compliance results, for example `BIOS`.
   - **Target version:** the Dell release version, for example `1.13.2`.
   - **Image URI:** `https://repo.example.local/dell/BIOS_XXXXX_WN64_1.13.2.EXE`.
3. Run a **compliance scan** (`POST /api/servers/{id}/compliance`). Each component is reported as `Compliant`, `Update Available` or `No Baseline`.
4. Select non-compliant servers and start a **rolling update**. FleetOps runs preflight (health and power), sends the baseline's image URI to each iDRAC via Redfish SimpleUpdate, and records the Redfish task.

Compliance only covers components that have a baseline. Repeat steps 1–2 whenever Dell publishes a release you want to adopt.

## Production checklist
1. Set a long random `SECRET_KEY` and protect `.env`.
2. Set `SIMULATION_MODE=false` only after validation.
3. Prefer `VERIFY_TLS=true` with trusted iDRAC certificates.
4. Put FleetOps behind your enterprise reverse proxy/SSO and restrict it to the management network/VPN.
5. Use a dedicated least-privilege iDRAC service account where your update workflow permits it.
6. Host approved Dell DUP/firmware packages on a controlled HTTPS repository reachable by iDRAC.
7. Test baselines on non-production hardware before approving fleet rollout.
8. Back up the persistent `idrac_data` Docker volume.

## Core API
- `GET/POST /api/servers`
- `POST /api/servers/{id}/scan`
- `GET /api/servers/{id}/firmware`
- `POST /api/servers/{id}/compliance`
- `GET /api/servers/{id}/preflight`
- `GET/POST /api/baselines`
- `GET/POST /api/groups`
- `POST /api/rolling-update`
- `GET/POST /api/windows`
- `GET /api/jobs`

## Firmware baseline workflow
FleetOps intentionally separates *discovery* from *approval*. Import/enter a target version and a controlled HTTPS firmware URI, scan compliance, and then launch a rolling update. It does not automatically install an arbitrary newest package.

## Important
Firmware behavior varies by PowerEdge generation, iDRAC/Lifecycle Controller release and package type. Validate the Redfish SimpleUpdate flow against your exact Dell models before production rollout. Some updates may stage until a reboot or require a Dell job queue/reboot workflow; extend the worker for those model-specific cases.
