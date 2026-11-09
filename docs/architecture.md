# IronWolf Architecture

## Overview

IronWolf is a Kubernetes operator that extends Metal3's Bare Metal Operator with advanced automation capabilities for enterprise bare metal infrastructure management.

It follows the operator pattern: custom resources define desired state, controllers reconcile actual state to match.

## System Architecture

```
 Users / CI/CD           Web Dashboard            Monitoring
    |                     (port 8888)                 ^
    |  kubectl / Helm         |                       | Prometheus scrape
    v                         v                       |
+-------------------------------------------------------------------------+
|                    Kubernetes API Server                                 |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  |                    IronWolf Manager                            |  |
|  |                                                                   |  |
|  |  +---------------------------+   +-----------------------------+  |  |
|  |  |       Controllers        |   |        Reconcilers          |  |  |
|  |  |                          |   |                             |  |  |
|  |  |  ClusterClaim        ----+-->|  BMC Reconciler             |  |  |
|  |  |  FirmwareProfile         |   |  (IPMI/Redfish/iLO/iDRAC)  |  |  |
|  |  |  MaintenanceWindow       |   |                             |  |  |
|  |  |  PowerPolicy             |   |  Ironic Reconciler          |  |  |
|  |  |  HardwarePool            |   |  (Inspect/Clean)            |  |  |
|  |  |  RemediationStrategy ----+-->|                             |  |  |
|  |  |  AlertPolicy             |   |  Cluster Reconciler         |  |  |
|  |  |  NetworkProfile          |   |  (Health Monitor)           |  |  |
|  |  |  SecurityPolicy          |   +-----------------------------+  |  |
|  |  |  EnergyPolicy            |                                    |  |
|  |  |  TelemetryPolicy         |   +-----------------------------+  |  |
|  |  |  HardwareBackup          |   |      Infrastructure         |  |  |
|  |  |  RestoreJob              |   |                             |  |  |
|  |  +----------+---------------+   |  Prometheus Metrics (:8080) |  |  |
|  |             |                   |  6 Admission Webhooks       |  |  |
|  |             v                   |  Leader Election            |  |  |
|  |  +---------------------------+  |  Health Probes              |  |  |
|  |  |       Strategies         |   +-----------------------------+  |  |
|  |  |                          |                                    |  |
|  |  |  Scheduling:             |   +-----------------------------+  |  |
|  |  |   - Binpack              |   |      Notifications          |  |  |
|  |  |   - Spread               |   |                             |  |  |
|  |  |   - TopologyAware        |   |  Webhook sender             |  |  |
|  |  |                          |   |  Slack integration           |  |  |
|  |  |  Remediation:            |   |  Email delivery              |  |  |
|  |  |   - Progressive          |   +-----------------------------+  |  |
|  |  |   - Immediate            |                                    |  |
|  |  |   - Maintenance          |                                    |  |
|  |  +---------------------------+                                   |  |
|  +-------------------------------------------------------------------+  |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  |                  Web Dashboard Server (:8888)                     |  |
|  |                                                                   |  |
|  |  +------------------+ +---------------+ +---------------------+  |  |
|  |  | REST API         | | WebSocket     | | React SPA           |  |  |
|  |  | - CRUD for CRDs  | | - Live events | | - 50+ pages         |  |  |
|  |  | - Host power     | | - State push  | | - Apple UI light/dark |  |  |
|  |  | - Host inspect   | |               | |   × dark/light      |  |  |
|  |  | - Topology       | +---------------+ | - Form modals       |  |  |
|  |  | - Overview stats |                   +---------------------+  |  |
|  |  +------------------+                                            |  |
|  |  +------------------+ +---------------+                          |  |
|  |  | API Key Auth     | | Audit Logger  |                          |  |
|  |  +------------------+ +---------------+                          |  |
|  +-------------------------------------------------------------------+  |
|                                                                         |
|  +---------------+ +----------------+ +-----------------------------+   |
|  | BareMetalHost | | 13 IronWolf CRs | | Secrets (BMC credentials)   |   |
|  | (Metal3 CRD)  | | (see table)    | |                             |   |
|  +------+--------+ +----------------+ +-----------------------------+   |
|         |                                                               |
+---------+---------------------------------------------------------------+
          |
          v
+--------------------+
|  Physical Hosts    |
|  (BMC / IPMI /     |
|   Redfish / iLO)   |
+--------------------+
```

## Components

### Controllers

| Controller | CRD | Reconcile Interval | Reconciler Wiring | Description |
|------------|-----|-------------------|-------------------|-------------|
| ClusterClaim | `clusterclaims.bmo.metal3.io` | On change + 30s retry | ClusterReconciler | Allocates hosts to clusters using scheduling strategies |
| FirmwareProfile | `firmwareprofiles.bmo.metal3.io` | On change + 60s retry | BMCReconciler | Fleet-wide BIOS/firmware configuration management. Uses finalizer `bmo.metal3.io/firmware-finalizer` to clean up `firmware.metal3.io/*` annotations on deletion. |
| MaintenanceWindow | `maintenancewindows.bmo.metal3.io` | On change + 30s active | -- | Scheduled maintenance with host draining |
| PowerPolicy | `powerpolicies.bmo.metal3.io` | On change + 5m poll | -- | Power state management (AlwaysOn/Off/Scheduled/OnDemand) |
| HardwarePool | `hardwarepools.bmo.metal3.io` | On change | -- | Resource pooling with quotas and priorities |
| RemediationStrategy | `remediationstrategies.bmo.metal3.io` | On change + 2m poll | BMCReconciler, IronicReconciler | Progressive failure detection and auto-remediation |
| AlertPolicy | `alertpolicies.bmo.metal3.io` | On change | -- | Alert condition evaluation with multi-channel notifications |
| NetworkProfile | `networkprofiles.bmo.metal3.io` | On change | -- | Switch, VLAN, BGP, and ACL network configuration. Uses finalizer `bmo.metal3.io/network-finalizer` to clean up `network.metal3.io/*` annotations on deletion. |
| SecurityPolicy | `securitypolicies.bmo.metal3.io` | On change | -- | Secure boot, TPM, compliance scanning, intrusion detection |
| EnergyPolicy | `energypolicies.bmo.metal3.io` | On change + 5m poll | -- | Power capping, energy scheduling, workload consolidation |
| TelemetryPolicy | `telemetrypolicies.bmo.metal3.io` | On change + collection interval | -- | Hardware telemetry collection and anomaly detection |
| HardwareBackup | `hardwarebackups.bmo.metal3.io` | On change + schedule | -- | Scheduled BIOS/BMC/firmware configuration backups |
| RestoreJob | `restorejobs.bmo.metal3.io` | On change | -- | Restore hardware configurations from backup snapshots |

### Reconcilers

Three reconcilers handle direct infrastructure communication and are wired into controllers that need them. Each reconciler implements a corresponding interface (`BMCReconcilerInterface`, `IronicReconcilerInterface`, `ClusterReconcilerInterface`) for testability and alternative implementations.

- **BMCReconciler**: Firmware lifecycle management via annotations. Detects BMC type (IPMI, Redfish, iLO, iDRAC) from address scheme using a validated allowlist. Manages firmware state machine (`pending` → `applying` → `done`/`error`/`timeout`), tracks `apply-started` timestamp for 30-minute timeout detection, and detects completion via `firmware.metal3.io/result` annotation set by external operators. Wired into FirmwareProfileReconciler and RemediationStrategyReconciler.
- **IronicReconciler**: Manages inspection and cleaning operations via annotations. Triggers operations by setting `inspect.metal3.io` / `clean.metal3.io` to `in-progress` with `*/started` timestamps. Polls for completion (HardwareDetails populated for inspection, result annotation for cleaning) and detects 30-minute timeouts. Wired into RemediationStrategyReconciler.
- **ClusterReconciler**: Monitors health of all hosts within a ClusterClaim, detecting error states. Wired into ClusterClaimReconciler.

### Admission Webhooks

17 validating webhooks enforce resource constraints at admission time, one per enterprise CRD with
a `v1` hub version (see `docs/guides/api-versioning.md`):

| Webhook | Validates |
|---------|-----------|
| ClusterClaimValidator | Control plane count 1-7, worker count >= 0, valid scheduling strategy, valid hardware requirements |
| FirmwareProfileValidator | Valid host selector, non-empty settings, valid firmware URL format |
| MaintenanceWindowValidator | Start time in future, valid duration, valid cron schedule, valid action types |
| PowerPolicyValidator | Valid power mode, schedule required for Scheduled mode, valid power cap |
| HardwarePoolValidator | Max hosts >= 1, valid priority range 0-100, valid quotas |
| RemediationStrategyValidator | Valid strategy type, ordered actions, valid retry config, valid escalation |
| AlertPolicyValidator | Valid severity (Critical/Warning/Info), at least one notification channel, positive cooldown |
| NetworkSwitchProfileValidator | Management address and credentials secret required, each port requires an ID |
| EnergyPolicyValidator | Positive power cap when capping is enabled, valid schedule action types, valid consolidation strategy |
| HardwareBackupValidator | Valid backup component types, at least one component, valid schedule interval (>= 1 minute), non-negative retention |
| HostReservationValidator | At least one of `matchLabels`/`matchExpressions`, image URL and format required together |
| HostDeployPolicyValidator | At least one namespace constraint (names, regex, or label selector) |
| HostUpdatePolicyValidator | `updateStrategy` must be `onPreparing` or `onReboot` |
| NetworkProfileValidator | VLAN IDs in range 1-4094 |
| RestoreJobValidator | At least one restore component, valid component types |
| SecurityPolicyValidator | At least one security check enabled, non-negative password rotation days |
| TelemetryPolicyValidator | Positive collection interval and retention period, at least one metric configured |

### Scheduling Strategies

| Strategy | Use Case | Algorithm |
|----------|----------|-----------|
| **Binpack** | Maximize utilization, minimize host count | Sort by CPU count descending, select top N |
| **Spread** | High availability across hosts | Round-robin selection |
| **TopologyAware** | Fault domain isolation | Distribute across racks/zones, round-robin per domain |

### Remediation Strategies

| Strategy | Behavior |
|----------|----------|
| **Immediate** | Execute first remediation action immediately on error detection |
| **Progressive** | Escalate through ordered actions (Reboot -> PowerCycle -> Inspect -> Decommission) with configurable retries and exponential backoff |
| **Maintenance** | Defer remediation to next maintenance window |

### Notifications Subsystem

The notifications package provides multi-channel alert delivery used by the AlertPolicy controller:

| Channel | Delivery | Configuration |
|---------|----------|---------------|
| **Webhook** | HTTP POST with JSON payload to configured URL | `channels[].type: Webhook`, `channels[].url` |
| **Slack** | Message via Slack incoming webhook | `channels[].type: Slack`, `channels[].url` |
| **Email** | SMTP delivery to configured addresses | `channels[].type: Email`, `channels[].emails[]` |

Email delivery requires SMTP configuration via environment variables: `SMTP_HOST`, `SMTP_PORT` (default 587), `SMTP_FROM`, `SMTP_USERNAME` (optional), `SMTP_PASSWORD` (optional). When username is set, SMTP AUTH with STARTTLS is used; otherwise plain SMTP for internal relays. STARTTLS is opportunistically used when the server advertises support. Email headers are sanitized to prevent header injection attacks.

Alert payloads include: condition type, severity, affected host(s), message, and timestamp. Cooldown periods prevent alert storms.

## Web Dashboard Architecture

The web dashboard is a separate binary (`ironwolf-web`) that provides a browser-based management interface.

### Backend (Go)

- **Server** (`web/internal/api/server.go`): HTTP server on port 8888 serving the REST API and static frontend assets.
- **Auth** (`web/internal/api/auth.go`): Role-based API key authentication (admin/operator/viewer) via `--api-key` flag. Supports `key1:admin,key2:viewer` format. WebSocket auth uses `Sec-WebSocket-Protocol` subprotocol (`auth.<token>`) instead of query parameters. Includes per-IP rate limiting (10 failed attempts/min, max tracked IPs configurable via `IRONWOLF_MAX_TRACKED_IPS`). Max WebSocket clients configurable via `IRONWOLF_MAX_WS_CLIENTS`.
- **Handlers**: Generic handler factory (`handlers_generic_crd.go`) for CRD list/get, plus specialized handlers:
  - `handlers_generic_crd.go` -- Generic list/get factory used by energy, network, security, telemetry, backup, restore handlers
  - `handlers_overview.go` -- Dashboard KPI stats and capacity
  - `handlers_hosts.go` -- Host listing, filtering, detail views
  - `handlers_host_actions.go` -- Provisioning, power, reboot, RAID, firmware, labels, bulk actions, import
  - `handlers_claims.go` -- ClusterClaim CRUD
  - `handlers_pools.go` -- HardwarePool CRUD
  - `handlers_power.go` -- PowerPolicy CRUD
  - `handlers_firmware.go` -- FirmwareProfile CRUD
  - `handlers_maintenance.go` -- MaintenanceWindow CRUD
  - `handlers_remediation.go` -- RemediationStrategy CRUD
  - `handlers_alerts.go` -- AlertPolicy CRUD with webhook testing
  - `handlers_topology.go` -- Rack/zone topology data
  - `handlers_events.go` -- Kubernetes event stream
  - `handlers_cluster.go` -- K8s node, workload, capacity, namespace info
  - `handlers_mutations.go` -- Create/update/patch/delete for all CRDs. The generic resource update endpoint blocks protected label/annotation prefixes to prevent accidental overwrites of system-managed metadata.
  - `handlers_secrets.go` -- Secret management (metadata only, never returns values)
  - `handlers_imagecatalog.go` -- OS image catalog
  - `handlers_metrics.go` -- Metrics history ring buffer (24h)
  - `handlers_capm3.go` -- Metal3Cluster/Machine/DataTemplate viewers
  - `handlers_ironic.go` -- Ironic service status
  - `handlers_host_firmware_settings.go` -- HostFirmwareSettings/Components/Schema
  - `handlers_audit.go` -- Audit log retrieval with filtering
- **WebSocket** (`web/internal/api/websocket.go`): Pushes live Kubernetes watch events to connected clients.
- **Audit** (`web/internal/api/audit.go`): Records all mutation operations with user, timestamp, resource, and action. The audit retrieval endpoint requires the admin role.
- **Response** (`web/internal/api/response.go`): Standardized JSON response formatting.
- **K8s Client** (`web/internal/k8s/`): Kubernetes client wrapper for all CRD and BareMetalHost operations.

### Frontend (React)

Located in `web/dashboard/`, built with Vite + React 19 + TypeScript + Tailwind CSS + Zustand + TanStack Query + Recharts.

41 pages across overview, infrastructure, orchestration, and operations categories. Key pages:

| Page | File | Description |
|------|------|-------------|
| Dashboard | `DashboardPage.tsx` | KPI cards, charts, capacity, health scores, telemetry, alerts |
| Hosts | `HostsPage.tsx` | Host inventory with filtering, bulk actions, CSV import |
| Host Detail | `HostDetailPage.tsx` | Full host details, provisioning, power, RAID, firmware |
| Host Compare | `HostComparePage.tsx` | Side-by-side host comparison |
| Claims | `ClaimsPage.tsx` | ClusterClaim management with claim wizard |
| Pools | `PoolsPage.tsx` | HardwarePool management with capacity bars |
| Power | `PowerPage.tsx` | PowerPolicy management with schedule visualization |
| Firmware | `FirmwarePage.tsx` | FirmwareProfile management with update wizard |
| Maintenance | `MaintenancePage.tsx` | MaintenanceWindow with table/calendar views |
| Remediation | `RemediationPage.tsx` | RemediationStrategy management |
| Alerts | `AlertsPage.tsx` | AlertPolicy management with severity levels |
| Energy | `EnergyPage.tsx` | EnergyPolicy with power savings and carbon tracking |
| Network | `NetworkPage.tsx` | NetworkProfile with VLAN configuration |
| Security | `SecurityPage.tsx` | SecurityPolicy with compliance tracking |
| Telemetry | `TelemetryPage.tsx` | TelemetryPolicy with anomaly detection |
| Backups | `BackupsPage.tsx` | HardwareBackup management |
| Restores | `RestoresPage.tsx` | RestoreJob tracking |
| Cluster | `ClusterPage.tsx` | K8s nodes, workloads, capacity, namespaces |
| Topology | `TopologyPage.tsx` | Visual rack/zone/region grid layout |
| Capacity Planner | `CapacityPlannerPage.tsx` | What-if analysis for cluster planning |
| Resource Graph | `ResourceGraphPage.tsx` | Resource relationship visualization |
| CAPM3 | `CAPM3Page.tsx` | Metal3Cluster/Machine/DataTemplate viewers |
| Audit | `AuditPage.tsx` | Searchable audit trail with date/user/action filtering |
| Settings | `SettingsPage.tsx` | Theme, API key, notifications, Ironic status |
| API Docs | `APIDocsPage.tsx` | Interactive API documentation browser |

## Data Flow

### ClusterClaim Lifecycle

```
Created --> Pending --> Partial --> Fulfilled
                |                       |
                v                       v
            Expired              (hosts claimed)
                |
                v
             Failed
```

1. User creates ClusterClaim with hardware requirements
2. Controller lists available BareMetalHosts in namespace
3. Filters by: no consumer, not decommissioned, no errors
4. Filters by: CPU count, RAM, disk, architecture
5. Applies scheduling strategy to select optimal hosts
6. Labels selected hosts with claim metadata and role (controlplane/worker)
7. Updates status to Fulfilled or requeues if insufficient hosts

### Firmware Update Flow

```
FirmwareProfile created
        |
        v
  Match hosts by selector
        |
        v
  Check current firmware state
        |
        +-- Already applied --> Skip
        |
        +-- Needs update --> Annotate host
                                  |
                                  v
                          BMC Reconciler picks up
                                  |
                                  v
                          Apply via Redfish/iLO
```

### AlertPolicy Flow

```
AlertPolicy created
        |
        v
  Watch hosts matching selector
        |
        v
  Evaluate conditions (HostError, HostOffline, etc.)
        |
        +-- No match --> requeue after interval
        |
        +-- Condition met --> Check cooldown
                                  |
                                  +-- In cooldown --> skip
                                  |
                                  +-- Fire alert --> Notifications subsystem
                                                        |
                                                        +-- Webhook POST
                                                        +-- Slack message
                                                        +-- Email delivery
```

### TelemetryPolicy Flow

```
TelemetryPolicy created
        |
        v
  Start collectors for matched hosts
        |
        v
  Collect metrics at configured interval
  (temperature, power, fan, disk_smart, memory_errors, network_errors)
        |
        +-- Export to configured exporters (Prometheus, Webhook, Slack)
        |
        +-- Run anomaly detection (statistical, ML, isolation_forest)
                  |
                  +-- Anomaly detected --> record + alert
```

### HardwareBackup / RestoreJob Flow

```
HardwareBackup created
        |
        v
  Match hosts by selector
        |
        v
  On schedule (full or incremental)
        |
        v
  Collect component configs (bios, bmc, firmware, raid, network, partitions)
        |
        v
  Compress + encrypt --> Store (S3 / local / NFS)
        |
        v
  Verify backup if enabled

RestoreJob created
        |
        v
  Locate backup by name + timestamp
        |
        v
  Restore components to target host
  (rolling or parallel strategy)
        |
        +-- Success --> run post-restore actions (reboot, verify, inspect)
        |
        +-- Failure + rollbackOnFailure --> RolledBack
```

### SecurityPolicy Flow

```
SecurityPolicy created
        |
        v
  Match hosts by selector
        |
        v
  Audit secure boot, TPM, BMC security
        |
        v
  Run compliance scans (CIS, NIST, DISA_STIG)
        |
        +-- Compliant --> report
        |
        +-- NonCompliant + autoRemediate --> auto-fix
        |
        +-- NonCompliant --> record violations

  Run intrusion detection sensors
        |
        +-- Detection --> execute actions (alert, lockdown, ip_block)
```

### EnergyPolicy Flow

```
EnergyPolicy created
        |
        v
  Match hosts by selector
        |
        v
  Apply power capping (static or dynamic profiles)
        |
        v
  Evaluate energy schedules
        |
        +-- Schedule rule matches --> on/off/suspend/hibernate
        |
        v
  Check consolidation thresholds
        |
        +-- Below CPU/memory threshold for minDuration --> power-off/suspend/migrate
        |
        v
  Carbon-aware: query carbon intensity API
        |
        +-- High carbon intensity --> reduce power
```

### NetworkProfile Flow

```
NetworkProfile created
        |
        v
  Match hosts by selector
        |
        v
  Configure switches (Arista/Cisco/Juniper/Mellanox)
  via eAPI/NETCONF/RESTCONF/SSH
        |
        v
  Provision VLANs (management, storage, tenant, data)
        |
        v
  Apply interface profiles (bonds, MTU, VLAN assignments)
        |
        v
  Configure BGP peers and advertised prefixes
        |
        v
  Apply ACL rules (permit/deny with logging)
```

## Security Model

- Controller runs as non-root (UID 65532)
- Read-only root filesystem
- All capabilities dropped
- Seccomp profile: RuntimeDefault
- RBAC: Minimal ClusterRole scoped to IronWolf CRDs + BareMetalHost read/update
- Secrets access: Read-only for BMC credentials
- Web dashboard: Role-based API key authentication (admin/operator/viewer), with per-IP rate limiting on failed auth attempts (configurable via `IRONWOLF_MAX_TRACKED_IPS` env var)
- Auth tokens: Header-only (`X-API-Key` or `Authorization: Bearer`); WebSocket auth uses `Sec-WebSocket-Protocol` subprotocol (`auth.<token>`) to avoid credential leakage in URL/proxy logs — always use TLS (`wss://`) in production
- TLS: Auto-generated self-signed certificate by default, with `--tls-cert`/`--tls-key` for custom certs
- SSRF protection: Centralized in `pkg/utils/network_utils.go` — blocks private/reserved IP ranges (RFC 1918, link-local, loopback, IPv4-mapped IPv6) at dial time, disables keep-alives to mitigate DNS rebinding, blocks redirects
- Security headers: Content-Security-Policy (CSP) and HTTP Strict-Transport-Security (HSTS) headers are set on all responses
- Boot mode validation: Boot mode values are validated against the allowed set (`UEFI`, `UEFISecureBoot`, `legacy`)
- Audit logging: All mutations through the web dashboard are logged with user, timestamp, resource, and action
- BMC security: Configurable HTTPS enforcement, minimum TLS version, allowed networks, password policies, certificate rotation (via SecurityPolicy)

## High Availability

- Leader election via Kubernetes lease (configurable)
- Pod anti-affinity spreads replicas across nodes
- Rolling update strategy with zero downtime (maxUnavailable: 0)
- Graceful shutdown with 30s termination period
- Health and readiness probes for load balancer integration

## Metrics

All metrics are exposed on `:8080/metrics` in Prometheus format:

| Metric | Type | Description |
|--------|------|-------------|
| `bmo_baremetal_hosts_total` | Gauge | Total hosts by status |
| `bmo_baremetal_host_power_state` | Gauge | Per-host power state |
| `bmo_cluster_claim_status` | Gauge | Claim status by phase |
| `bmo_power_state_transitions_total` | Counter | Power state changes |
| `bmo_power_consumption_watts` | Gauge | Per-host power draw |
| `bmo_remediation_attempts_total` | Counter | Remediation actions taken |
| `bmo_firmware_update_status` | Gauge | Firmware update progress |
| `bmo_maintenance_window_active` | Gauge | Active maintenance windows |
| `bmo_alert_policy_active_alerts` | Gauge | Active alerts per policy |
| `bmo_alerts_fired_total` | Counter | Total alerts fired |
| `bmo_security_compliant_hosts` | Gauge | Compliant hosts per security policy |
| `bmo_security_violations_total` | Counter | Security violations detected |
| `bmo_energy_savings_watts` | Gauge | Estimated power savings per energy policy |
| `bmo_energy_carbon_saved_kg` | Gauge | Carbon saved in kilograms |
| `bmo_telemetry_anomalies_total` | Counter | Anomalies detected by telemetry |
| `bmo_telemetry_threshold_violations` | Gauge | Active threshold violations |
| `bmo_backup_total` | Counter | Total backups completed |
| `bmo_backup_storage_bytes` | Gauge | Storage used by backups |
| `bmo_restore_jobs_total` | Counter | Total restore jobs by outcome |
| `bmo_network_configured_switches` | Gauge | Configured switches per profile |
| `bmo_network_active_vlans` | Gauge | Active VLANs per profile |

## Helm Chart

The Helm chart in `deploy/helm/ironwolf/` includes templates for:

| Template | Resource |
|----------|----------|
| `deployment.yaml` | Controller manager Deployment |
| `dashboard-deployment.yaml` | Web dashboard Deployment |
| `dashboard-service.yaml` | Web dashboard Service (port 8888) |
| `dashboard-ingress.yaml` | Web dashboard Ingress |
| `dashboard-secret.yaml` | Dashboard API key Secret (conditional) |
| `rbac.yaml` | ClusterRole and ClusterRoleBinding for the manager |
| `serviceaccount.yaml` | ServiceAccounts for controller and dashboard |
| `networkpolicy.yaml` | NetworkPolicies for controller and dashboard (conditional) |
| `_helpers.tpl` | Template helper functions |

CRDs are installed separately via `make install` or `kubectl apply -f config/base/crds/bases/` to avoid Helm CRD lifecycle issues.

### NetworkPolicy

When `networkPolicy.enabled: true` (default), the chart creates NetworkPolicies that:

- **Controller manager**: Allows inbound metrics (8080), health probes (8081), and webhook (configurable) traffic. Allows outbound DNS (53) and Kubernetes API server (443, 6443). API server egress is unrestricted by destination IP because the API server address varies by cluster.
- **Dashboard** (when enabled): Allows inbound HTTP/HTTPS on the dashboard port. Allows outbound DNS and Kubernetes API server.
