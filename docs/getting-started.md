# Getting Started with IronWolf

## Prerequisites

- Kubernetes cluster v1.32+
- Metal3 Bare Metal Operator installed (`clusterctl init --infrastructure metal3`)
- `kubectl` configured with cluster access
- Go 1.22+ (for development only)
- Node.js 18+ and npm (for web dashboard development only)
- An IronWolf trial or commercial licence key — the controller manager refuses to start without one (see [Licensing](#licensing) below)

## Installation

### Option 1: Single Manifest (Quickest)

```bash
kubectl apply -f https://github.com/zyvorai/ironwolf/releases/latest/download/install.yaml
```

### Option 2: Helm product package (recommended)

Charts and images ship with your commercial package (OCI / tarball). Source is private — this site is docs only.

```bash
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system --create-namespace
```

The Helm chart includes the controller manager, web dashboard deployment, service, ingress, RBAC (ClusterRole/ClusterRoleBinding), ServiceAccounts, NetworkPolicies (enabled by default), and an optional dashboard API key Secret.

#### Product package overlay

For a polished out-of-box install (branding ConfigMap, notify prefs, starter AlertPolicy, Grafana/PrometheusRule hooks):

```bash
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system --create-namespace \
  -f values-production.yaml \
  -f values-package.yaml \
  --set dashboard.observability.grafanaUrl=http://HOST:30301
```

Then open the dashboard and check `GET /api/v1/package/readiness`, or see [helm-values.md](helm-values.md#product-package-overlay).

### Verify Installation

```bash
# Check pods
kubectl get pods -n ironwolf-system

# Check IronWolf CRDs
kubectl get crds | grep bmo.metal3.io
```

Expected CRDs:

```
alertpolicies.bmo.metal3.io
availabilitymonitors.bmo.metal3.io
bmceventsubscriptions.bmo.metal3.io
clusterclaims.bmo.metal3.io
dataimages.bmo.metal3.io
energypolicies.bmo.metal3.io
firmwareprofiles.bmo.metal3.io
firmwareschemas.bmo.metal3.io
hardwarebackups.bmo.metal3.io
hardwaredata.bmo.metal3.io
hardwarepools.bmo.metal3.io
hostdeploypolicies.bmo.metal3.io
hostfirmwarecomponents.bmo.metal3.io
hostfirmwaresettings.bmo.metal3.io
hostipaddresses.bmo.metal3.io
hostreservations.bmo.metal3.io
hostupdatepolicies.bmo.metal3.io
maintenancewindows.bmo.metal3.io
networkprofiles.bmo.metal3.io
networkswitchprofiles.bmo.metal3.io
powerpolicies.bmo.metal3.io
preprovisioningimages.bmo.metal3.io
remediationstrategies.bmo.metal3.io
restorejobs.bmo.metal3.io
securitypolicies.bmo.metal3.io
supporttickets.bmo.metal3.io
telemetrypolicies.bmo.metal3.io
```

## Licensing

The controller manager checks for a licence key before it starts — with none set it exits immediately:

```
IronWolf licence error: No IronWolf licence key found.
Request a 30-day trial at: https://zyvor.dev/contact?intent=trial&product=ironwolf
```

`manager --version` and `manager --help` work without a licence, so you can confirm what you
downloaded before requesting a key.

Set the key once you have one:

```bash
export IRONWOLF_LICENSE_KEY="<your-key>"
# or
mkdir -p ~/.config/ironwolf && echo "<your-key>" > ~/.config/ironwolf/license.key
```

In Kubernetes, create a Secret and point the Helm chart at it (preferred over the inline
`operator.licenseKey` value):

```bash
kubectl create secret generic ironwolf-license -n ironwolf-system \
  --from-literal=license-key="<your-key>"

helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf -n ironwolf-system \
  --set operator.licenseKeySecret=ironwolf-license
```

`# packaged operator tooling (provided with your license) HOST USER --license-key '<your-key>'` (or `IRONWOLF_LICENSE_KEY` in
the environment) does the same for a remote deploy. See `docs/helm-values.md` for the full
`operator.licenseKey*` reference.

## Your First ClusterClaim

### Step 1: Create a Hardware Pool

Define which hosts are available for allocation:

```yaml
# pool.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: HardwarePool
metadata:
  name: general-pool
spec:
  hostSelector:
    matchLabels:
      environment: available
  maxHosts: 20
  priority: 50
```

```bash
kubectl apply -f pool.yaml
kubectl get hardwarepools
```

### Step 2: Claim Hardware for a Cluster

```yaml
# claim.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: ClusterClaim
metadata:
  name: my-first-cluster
spec:
  controlPlaneCount: 1
  workerCount: 2
  schedulingStrategy: spread
  hardwareRequirements:
    minCPUCores: 4
    minRAMGB: 8
    minDiskGB: 100
  labels:
    cluster: my-first-cluster
  timeout: 15m
```

```bash
kubectl apply -f claim.yaml
kubectl get clusterclaims -w
```

### Step 3: Monitor the Claim

```bash
# Watch claim status
kubectl get cc my-first-cluster -o yaml

# See which hosts were claimed
kubectl get baremetalhosts -l bmo.metal3.io/claim-name=my-first-cluster
```

## Next Steps

### Set Up Alerting

```yaml
# alert-policy.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: AlertPolicy
metadata:
  name: prod-alerts
spec:
  hostSelector:
    matchLabels:
      environment: production
  severity: Critical
  conditions:
  - type: HostError
  - type: HostOffline
  - type: RemediationFailed
  channels:
  - type: Slack
    url: https://hooks.slack.com/services/T.../B.../xxx
  - type: Webhook
    url: https://alerts.example.com/bmo
  cooldown: 15m
```

```bash
kubectl apply -f alert-policy.yaml
kubectl get alertpolicies
```

Available condition types: `HostError`, `HostOffline`, `ClaimExpired`, `ClaimFailed`, `RemediationFailed`, `MaintenanceFailed`, `FirmwareError`, `PowerCapExceeded`.

**Email notifications:** To use the Email channel, set SMTP environment variables on the controller:

```bash
# Required
export SMTP_HOST=mail.example.com
export SMTP_FROM=alerts@example.com

# Optional (for authenticated SMTP)
export SMTP_USERNAME=smtp-user
export SMTP_PASSWORD=smtp-pass
export SMTP_PORT=587  # default
```

Then add an email channel: `channels: [{type: Email, emails: [ops@example.com]}]`

### Configure Telemetry Collection

```yaml
# telemetry.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: TelemetryPolicy
metadata:
  name: hw-monitoring
spec:
  hostSelector:
    matchLabels:
      environment: production
  collection:
    interval: 30s
    metrics:
    - type: temperature
      thresholds:
        warning: "75"
        critical: "90"
    - type: power
    - type: fan
    - type: disk_smart
    - type: memory_errors
  exporters:
  - name: prometheus
    type: prometheus
    endpoint: http://prometheus:9090/api/v1/write
  anomalyDetection:
    enabled: true
    model: statistical
    sensitivity: medium
  retention:
    rawMetrics: "7d"
    aggregatedMetrics: "90d"
    anomalies: "365d"
```

```bash
kubectl apply -f telemetry.yaml
kubectl get telemetrypolicies
```

### Enforce Security Policies

```yaml
# security.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: SecurityPolicy
metadata:
  name: baseline
spec:
  hostSelector:
    matchLabels:
      environment: production
  secureBoot:
    enforced: true
    remediation: warn
    auditSchedule: "0 6 * * *"
  tpm:
    required: true
    attestation:
      enabled: true
      server: https://attestation.internal
      refreshInterval: 1h
  bmcSecurity:
    httpsRequired: true
    tlsVersion: "1.3"
    allowedNetworks:
    - 10.0.100.0/24
    passwordPolicy:
      minLength: 16
      complexity: strong
      expiryDays: 90
    certificateRotation: "720h"
  compliance:
    standards: [CIS, NIST]
    scanSchedule: "0 2 * * 0"
    autoRemediate: false
    reportFormats: [json, html]
  intrusionDetection:
    enabled: true
    sensors:
    - type: brute_force
      threshold: 5
    - type: port_scan
      threshold: 100
    actions:
    - type: alert
    - type: ip_block
      duration: "24h"
```

```bash
kubectl apply -f security.yaml
kubectl get securitypolicies
```

### Set Up Energy Management

```yaml
# energy.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: EnergyPolicy
metadata:
  name: dev-energy
spec:
  hostSelector:
    matchLabels:
      environment: development
  powerCapping:
    enabled: true
    defaultCapWatts: 500
    dynamicCapping:
      enabled: true
      algorithm: workload-aware
      minCapWatts: 200
      maxCapWatts: 800
  schedules:
  - name: off-hours
    timezone: America/New_York
    rules:
    - days: [Monday, Tuesday, Wednesday, Thursday, Friday]
      startTime: "20:00"
      endTime: "06:00"
      action: suspend
    - days: [Saturday, Sunday]
      startTime: "00:00"
      endTime: "23:59"
      action: off
  consolidation:
    enabled: true
    cpuThresholdPercent: 15
    memoryThresholdPercent: 20
    minDuration: 30m
    strategy: power-off
    cooldownPeriod: 1h
  carbonAware:
    enabled: true
    apiEndpoint: https://carbon-api.example.com/intensity
    targetIntensity: 200
```

```bash
kubectl apply -f energy.yaml
kubectl get energypolicies
```

### Configure Network Profiles

```yaml
# network.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: NetworkProfile
metadata:
  name: datacenter-east
spec:
  hostSelector:
    matchLabels:
      datacenter: east
  switches:
  - name: tor-east-1
    managementIP: 10.0.1.1
    vendor: arista
    protocol: eapi
    credentialsRef:
      name: switch-credentials
      namespace: ironwolf-system
  vlans:
  - id: 100
    name: management
    subnet: 10.0.100.0/24
    gateway: 10.0.100.1
    vlanType: management
  - id: 200
    name: storage
    subnet: 10.0.200.0/24
    gateway: 10.0.200.1
    vlanType: storage
  interfaceProfiles:
  - name: standard-server
    interfaces:
    - name: eth0
      type: bond
      bondMode: 802.3ad
      mtu: 9000
      vlanIds: [100, 200]
  bgp:
    enabled: true
    asNumber: 65001
    peers:
    - address: 10.0.1.1
      asNumber: 65000
    prefixes:
    - 10.0.100.0/24
  acls:
  - name: management-acl
    rules:
    - action: permit
      source: 10.0.0.0/8
      destination: 10.0.100.0/24
      protocol: tcp
      port: 443
    - action: deny
      source: 0.0.0.0/0
      destination: 10.0.100.0/24
      protocol: any
      log: true
```

```bash
kubectl apply -f network.yaml
kubectl get networkprofiles
```

### Set Up Hardware Backups

```yaml
# backup.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: HardwareBackup
metadata:
  name: nightly-config
spec:
  hostSelector:
    matchLabels:
      environment: production
  components:
  - type: bios
    settings: all
  - type: bmc
    settings: all
  - type: firmware
  - type: raid
  schedule:
    fullBackup: "0 2 * * 0"
    incrementalBackup: "0 2 * * 1-6"
    retention:
      fullBackups: 4
      incremental: 30
  storage:
    type: s3
    endpoint: https://s3.example.com
    bucket: bmo-backups
    path: /hardware-configs
    encryption:
      enabled: true
      kmsKeyId: arn:aws:kms:us-east-1:123456:key/abcd-1234
    compression: zstd
  verification:
    enabled: true
    validateAfterBackup: true
    restoreTestSchedule: "0 3 1 * *"
```

```bash
kubectl apply -f backup.yaml
kubectl get hardwarebackups
```

### Restore a Host Configuration

```yaml
# restore.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: RestoreJob
metadata:
  name: restore-host-42
spec:
  backupName: nightly-config
  targetHost: host-42
  components:
  - type: bios
  - type: bmc
  strategy: rolling
  rollbackOnFailure: true
  postRestore:
  - type: reboot
    waitForReady: true
  - type: verify
```

```bash
kubectl apply -f restore.yaml
kubectl get restorejobs -w
```

### Set Up Power Management

```yaml
apiVersion: bmo.metal3.io/v1beta1
kind: PowerPolicy
metadata:
  name: dev-on-demand
spec:
  hostSelector:
    matchLabels:
      environment: development
  mode: OnDemand
  idleTimeout: 2h
```

### Configure Firmware Profiles

```yaml
apiVersion: bmo.metal3.io/v1beta1
kind: FirmwareProfile
metadata:
  name: standard-bios
spec:
  hostSelector:
    matchLabels:
      vendor: dell
  settings:
    bios:
      BootMode: Uefi
      SecureBoot: Enabled
    firmwareVersion: "<ephemeral-ip>"
  maxConcurrent: 2
```

### Schedule Maintenance

```yaml
apiVersion: bmo.metal3.io/v1beta1
kind: MaintenanceWindow
metadata:
  name: weekly-patching
spec:
  hostSelector:
    matchLabels:
      environment: production
  startTime: "2024-03-17T02:00:00Z"
  duration: 4h
  schedule: "0 2 * * 0"
  actions:
  - type: Inspection
    timeout: 15m
  - type: Reboot
    timeout: 10m
  maxConcurrent: 3
  drainBeforeMaintenance: true
```

### Set Up Remediation

```yaml
apiVersion: bmo.metal3.io/v1beta1
kind: RemediationStrategy
metadata:
  name: auto-heal
spec:
  hostSelector:
    matchLabels:
      environment: production
  strategy: Progressive
  retryConfig:
    maxRetries: 3
    exponentialBackoff: true
  actions:
  - order: 1
    type: Reboot
    timeout: 10m
  - order: 2
    type: PowerCycle
    timeout: 15m
  - order: 3
    type: Inspect
    timeout: 30m
  escalation:
    enabled: true
    notificationChannels: [slack-ops]
```

## Web Dashboard

### Build and Run

```bash
# Build frontend and backend
make web

# Run with kubeconfig
./bin/ironwolf-web --kubeconfig ~/.kube/config --static-dir web/dashboard/dist

# Run with API key authentication (single admin key)
./bin/ironwolf-web --kubeconfig ~/.kube/config --static-dir web/dashboard/dist --api-key my-secret-key

# Run with multi-user RBAC (key:role pairs)
./bin/ironwolf-web --kubeconfig ~/.kube/config --static-dir web/dashboard/dist \
  --api-key "admin-key:admin,ops-key:operator,readonly-key:viewer"

# Open http://localhost:8888
```

### Development Mode

```bash
# Start frontend dev server with hot reload (runs on a separate port)
make web-dev

# Start backend separately
make web-server
./bin/ironwolf-web --kubeconfig ~/.kube/config
```

### Dashboard Pages

| Page | URL Path | Description |
|------|----------|-------------|
| Dashboard | `/` | KPI cards, capacity bars, host status breakdown |
| Hosts | `/hosts` | Host inventory with filtering and bulk actions |
| Host Detail | `/hosts/:name` | Single host details, power control, inspection |
| Claims | `/claims` | ClusterClaim management with create/delete modals |
| Pools | `/pools` | HardwarePool management |
| Power | `/power` | PowerPolicy management |
| Firmware | `/firmware` | FirmwareProfile management |
| Maintenance | `/maintenance` | MaintenanceWindow management |
| Remediation | `/remediation` | RemediationStrategy management |
| Topology | `/topology` | Visual rack/zone grid layout |
| Audit | `/audit` | Searchable audit trail of all mutations |

### API Endpoints

The REST API provides CRUD operations for all CRDs and host management:

- `GET /api/overview` -- Dashboard stats
- `GET /api/hosts` -- List hosts
- `POST /api/hosts/:name/power` -- Toggle host power
- `POST /api/hosts/:name/inspect` -- Trigger inspection
- `GET /api/claims` -- List ClusterClaims
- `POST /api/claims` -- Create ClusterClaim
- `DELETE /api/claims/:name` -- Delete ClusterClaim
- `GET /api/pools` -- List HardwarePools
- `POST /api/pools` -- Create HardwarePool
- `DELETE /api/pools/:name` -- Delete HardwarePool
- `GET /api/topology` -- Topology data
- `GET /api/events` -- Event stream
- `GET /api/audit` -- Audit log (requires admin role)
- `GET /ws` -- WebSocket for live updates (auth via `Sec-WebSocket-Protocol: auth.<token>` subprotocol)

## CLI Tools

### BMH Inspector

```bash
# Build
# CLI tools ship with the commercial package

# List all hosts
./bin/bmh-inspector list

# List hosts in a specific namespace
./bin/bmh-inspector list -n metal3-system

# Inspect a specific host
./bin/bmh-inspector inspect my-host-01

# Cluster summary
./bin/bmh-inspector summary
```

Use `-n` / `--namespace` to target a specific namespace (defaults to the current kubeconfig context namespace).

### Firmware Updater

```bash
# Build
# CLI tools ship with the commercial package

# Check firmware status
./bin/firmware-updater status

# Check firmware status in a specific namespace
./bin/firmware-updater status -n metal3-system

# Check if a host supports firmware updates
./bin/firmware-updater check my-host-01

# Show details of a firmware profile
./bin/firmware-updater show dell-r740-firmware
```

Use `-n` / `--namespace` to target a specific namespace (defaults to the current kubeconfig context namespace).

## Further Reading

- [Provisioning Guide](provisioning-guide.md) -- End-to-end: Ironic setup, network, register, inspect, provision, deprovision
- [Tutorial](tutorial.md) -- Hands-on guide covering all 27 CRDs, host lifecycle, dashboard, CLI tools, and full API reference
- [Architecture](architecture.md) -- System design, 27 controllers, reconciler wiring, data flows
- [API Reference](api-reference.md) -- Complete field reference for all 27 CRDs + upstream Metal3 types
- [OS Image Guide](os-image-guide.md) -- OS selection, image formats, golden image pipeline, disk encryption
- [Helm Chart Reference](helm-values.md) -- Complete Helm values reference with examples
- [Development Guide](development.md) -- Build from source, add a CRD, test, debug, release
- [Production Checklist](production-checklist.md) -- HA, security, TLS, monitoring, upgrades, multi-tenancy
- [Operations Guide](operations-guide.md) -- Deploy, upgrade, backup, all scripts and Makefile targets
- [Troubleshooting](troubleshooting.md) -- Diagnostics for all CRDs, debug bundle, selftest
