# IronWolf Tutorial

> **Public docs:** Install from the commercial Helm package (`oci://ghcr.io/zyvorai/charts/ironwolf`). Product source is private.

A hands-on, step-by-step tutorial for deploying and using IronWolf to manage bare metal infrastructure at scale.

## Table of Contents

- [Part 1: Installation and Setup](#part-1-installation-and-setup)
- [Part 2: Managing Bare Metal Hosts](#part-2-managing-bare-metal-hosts)
- [Part 3: Claiming Hardware for Clusters](#part-3-claiming-hardware-for-clusters)
- [Part 4: Power Management](#part-4-power-management)
- [Part 5: Firmware Management](#part-5-firmware-management)
- [Part 6: Scheduled Maintenance](#part-6-scheduled-maintenance)
- [Part 7: Auto-Remediation](#part-7-auto-remediation)
- [Part 8: Alerting and Notifications](#part-8-alerting-and-notifications)
- [Part 9: Security Policies](#part-9-security-policies)
- [Part 10: Energy Management](#part-10-energy-management)
- [Part 11: Telemetry and Monitoring](#part-11-telemetry-and-monitoring)
- [Part 12: Network Configuration](#part-12-network-configuration)
- [Part 13: Backup and Restore](#part-13-backup-and-restore)
- [Part 14: Web Dashboard](#part-14-web-dashboard)
- [Part 15: CLI Tools](#part-15-cli-tools)
- [Part 16: Production Deployment](#part-16-production-deployment)
- [Part 17: API Reference](#part-17-api-reference)

---

## Part 1: Installation and Setup

### Prerequisites

- Kubernetes cluster v1.32+
- Metal3 Bare Metal Operator installed
- `kubectl` configured with cluster access
- Go 1.22+ (development only)
- Node.js 18+ and npm (dashboard development only)

### Option A: Local Development with Kind

The fastest way to get started is with a local Kind cluster:

```bash
# Charts and images ship with your commercial package (source is private).
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system --create-namespace \
  -f values-package.yaml
```

This installs CRDs, the controller, and the dashboard so you can explore immediately. For Kind-based lab setups, use the package’s lab profile when provided by Zyvor.

### Option B: Production values overlay

```bash
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system --create-namespace \
  -f values-production.yaml \
  -f values-package.yaml \
  --set image.tag=v1.0.0
```

Helm handles CRD installation, RBAC, namespace creation, and deployment. Environment value files configure resource limits automatically:

| Setting | Dev | Staging | Production |
|---------|-----|---------|------------|
| Replicas | 1 | 1 | 2 |
| Leader Election | No | Yes | Yes |
| Log Level | debug | info | info |
| Memory Limit | 256Mi | 512Mi | 1Gi |

### Option C: Helm Chart

```bash
# Install with defaults
helm install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system \
  --create-namespace

# Install with production settings
helm install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system \
  --create-namespace \
  --set image.repository=ghcr.io/zyvorai/ironwolf \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set replicaCount=2 \
  --set networkPolicy.enabled=true \
  --set dashboard.enabled=true \
  --set dashboard.auth.apiKey="my-secret-key"
```

The Helm chart includes: controller Deployment, dashboard Deployment, Services, Ingress, RBAC (ClusterRole/ClusterRoleBinding), ServiceAccounts, NetworkPolicies, and an optional dashboard API key Secret.

**Key Helm values:**

```yaml
replicaCount: 1
image:
  repository: ghcr.io/zyvorai/ironwolf
  pullPolicy: Always       # Use IfNotPresent with pinned tag for production
  tag: ""                  # Defaults to chart appVersion
leaderElection:
  enabled: true
metrics:
  bindAddress: ":8080"
healthProbe:
  bindAddress: ":8081"
webhook:
  port: 9443
networkPolicy:
  enabled: true            # Creates NetworkPolicies for controller and dashboard
resources:
  limits: { cpu: 500m, memory: 256Mi }
  requests: { cpu: 10m, memory: 128Mi }
dashboard:
  enabled: true
  port: 8888
  auth:
    enabled: true
    apiKey: ""             # "key1:admin,key2:viewer" for multi-user RBAC
```

### Option D: Kustomize

```bash
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf -n ironwolf-system --create-namespace
```

### Option E: Single Manifest (Quickest)

```bash
kubectl apply -f https://github.com/zyvorai/ironwolf/releases/latest/download/install.yaml
```

### Verify Installation

```bash
# Check the controller pod is running
kubectl get pods -n ironwolf-system

# Verify all 27 CRDs are installed
kubectl get crds | grep bmo.metal3.io

# Expected output:
# alertpolicies.bmo.metal3.io
# availabilitymonitors.bmo.metal3.io
# bmceventsubscriptions.bmo.metal3.io
# clusterclaims.bmo.metal3.io
# dataimages.bmo.metal3.io
# energypolicies.bmo.metal3.io
# firmwareprofiles.bmo.metal3.io
# firmwareschemas.bmo.metal3.io
# hardwarebackups.bmo.metal3.io
# hardwaredata.bmo.metal3.io
# hardwarepools.bmo.metal3.io
# hostdeploypolicies.bmo.metal3.io
# hostfirmwarecomponents.bmo.metal3.io
# hostfirmwaresettings.bmo.metal3.io
# hostipaddresses.bmo.metal3.io
# hostreservations.bmo.metal3.io
# hostupdatepolicies.bmo.metal3.io
# maintenancewindows.bmo.metal3.io
# networkprofiles.bmo.metal3.io
# networkswitchprofiles.bmo.metal3.io
# powerpolicies.bmo.metal3.io
# preprovisioningimages.bmo.metal3.io
# remediationstrategies.bmo.metal3.io
# restorejobs.bmo.metal3.io
# securitypolicies.bmo.metal3.io
# supporttickets.bmo.metal3.io
# telemetrypolicies.bmo.metal3.io

# Run post-install verification
kubectl get pods -n ironwolf-system

# Full health check
kubectl get pods -n ironwolf-system
```

---

## Part 2: Managing Bare Metal Hosts

IronWolf works with Metal3 BareMetalHost resources. Hosts are registered with their BMC (Baseboard Management Controller) credentials.

### Register a Host via the Web Dashboard

The dashboard provides a 4-step wizard for host registration:

1. **BMC Connection** -- Enter BMC address (e.g., `redfish://192.168.1.100/redfish/v1/Systems/1`), username, password
2. **Network & Boot** -- Set boot MAC address, boot mode (UEFI/UEFISecureBoot/legacy), automated cleaning mode
3. **Labels** -- Add labels for pool membership and scheduling
4. **Review** -- Confirm and create

### Register a Host via kubectl

```yaml
# host.yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: server-01
  namespace: default
  labels:
    environment: production
    rack: rack-a
    zone: us-east-1a
spec:
  bmc:
    address: redfish://192.168.1.100/redfish/v1/Systems/1
    credentialsName: server-01-bmc-credentials
  bootMACAddress: "AA:BB:CC:DD:EE:01"
  bootMode: UEFI              # Valid values: UEFI, UEFISecureBoot, legacy
  online: true
  automatedCleaningMode: metadata
```

```bash
# Create BMC credentials secret first
kubectl create secret generic server-01-bmc-credentials \
  --from-literal=username=admin \
  --from-literal=password='s3cur3p@ss'

# Register the host
kubectl apply -f host.yaml

# Watch it move through states: registering -> inspecting -> available
kubectl get baremetalhosts -w
```

### Register Hosts via the REST API

```bash
curl -X POST http://localhost:8888/api/v1/hosts \
  -H "X-API-Key: my-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "server-01",
    "namespace": "default",
    "bmcAddress": "redfish://192.168.1.100/redfish/v1/Systems/1",
    "bootMACAddress": "AA:BB:CC:DD:EE:01",
    "bootMode": "UEFI",
    "online": true,
    "bmcCredentials": {
      "username": "admin",
      "password": "s3cur3p@ss"
    },
    "labels": {
      "environment": "production",
      "rack": "rack-a"
    }
  }'
```

Valid `bootMode` values: `UEFI`, `UEFISecureBoot`, `legacy`.

### Bulk Import via CSV

The dashboard supports drag-and-drop CSV import. The CSV format:

```csv
name,namespace,bmcAddress,bootMACAddress,bootMode,username,password,online
server-01,default,redfish://192.168.1.100/redfish/v1/Systems/1,AA:BB:CC:DD:EE:01,UEFI,admin,pass1,true
server-02,default,redfish://192.168.1.101/redfish/v1/Systems/1,AA:BB:CC:DD:EE:02,UEFI,admin,pass2,true
```

Or via the API:

```bash
curl -X POST http://localhost:8888/api/v1/hosts/import \
  -H "X-API-Key: my-key" \
  -H "Content-Type: application/json" \
  -d '[
    {"name":"server-01", "namespace":"default", "bmcAddress":"redfish://192.168.1.100/redfish/v1/Systems/1", "bootMACAddress":"AA:BB:CC:DD:EE:01", "bmcCredentials":{"username":"admin","password":"pass1"}},
    {"name":"server-02", "namespace":"default", "bmcAddress":"redfish://192.168.1.101/redfish/v1/Systems/1", "bootMACAddress":"AA:BB:CC:DD:EE:02", "bmcCredentials":{"username":"admin","password":"pass2"}}
  ]'
```

### Supported BMC Protocols

| Protocol | Address Format | Firmware Updates |
|----------|---------------|------------------|
| IPMI | `ipmi://192.168.1.100` (port 623) | No |
| Redfish | `redfish://bmc.example.com` (port 443) | Yes |
| Redfish Virtual Media | `redfish-virtualmedia://bmc.example.com` | Yes |
| iLO4 | `ilo4://bmc.example.com` (port 443) | No |
| iLO5 | `ilo5://bmc.example.com` (port 443) | Yes |
| iDRAC | `idrac://bmc.example.com` (port 443) | Yes |
| iDRAC Virtual Media | `idrac-virtualmedia://bmc.example.com` | Yes |
| iRMC | `irmc://bmc.example.com` | No |

All BMC address schemes are validated against an allowlist at parse time. Unsupported schemes (ftp, ssh, file, etc.) are rejected.

### Host Provisioning

Provision a host with an OS image:

```bash
curl -X POST http://localhost:8888/api/v1/hosts/default/server-01/provision \
  -H "X-API-Key: my-key" \
  -H "Content-Type: application/json" \
  -d '{
    "imageURL": "https://images.example.com/ubuntu-22.04.qcow2",
    "checksum": "abc123def456",
    "checksumType": "sha256",
    "diskFormat": "qcow2",
    "bootMode": "UEFI"
  }'
```

The `imageURL` is validated to require `http://` or `https://` scheme. Supported image formats: raw, qcow2, vdi, vmdk, live-iso, and OCI (`oci://`). Valid `bootMode` values: `UEFI`, `UEFISecureBoot`, `legacy`.

### Host Lifecycle Operations

All operations are available via the dashboard, kubectl, and the REST API:

| Operation | API Endpoint | Method |
|-----------|-------------|--------|
| Power On/Off | `/api/v1/hosts/{ns}/{name}/power` | POST |
| Reboot (soft/hard) | `/api/v1/hosts/{ns}/{name}/reboot` | POST |
| Inspect | `/api/v1/hosts/{ns}/{name}/inspect` | POST |
| Provision | `/api/v1/hosts/{ns}/{name}/provision` | POST |
| Deprovision | `/api/v1/hosts/{ns}/{name}/deprovision` | POST |
| Edit | `/api/v1/hosts/{ns}/{name}` | PUT |
| Delete | `/api/v1/hosts/{ns}/{name}` | DELETE |
| RAID Config | `/api/v1/hosts/{ns}/{name}/raid` | PUT |
| Firmware Config | `/api/v1/hosts/{ns}/{name}/firmware` | PUT |
| Boot Mode | `/api/v1/hosts/{ns}/{name}/bootmode` | PUT |
| Labels | `/api/v1/hosts/{ns}/{name}/labels` | PUT |
| Annotations | `/api/v1/hosts/{ns}/{name}/annotations` | PUT |
| Credentials | `/api/v1/hosts/{ns}/{name}/credentials` | PUT |
| Custom Deploy | `/api/v1/hosts/{ns}/{name}/customdeploy` | POST |
| Detach/Attach | `/api/v1/hosts/{ns}/{name}/detach` | POST |
| Health Check | `/api/v1/hosts/{ns}/{name}/health` | GET |
| Inspect Control | `/api/v1/hosts/{ns}/{name}/inspect-control` | POST |
| Bulk Actions | `/api/v1/hosts/bulk` | POST |

### Bulk Actions

Apply an action to up to 100 hosts at once:

```bash
curl -X POST http://localhost:8888/api/v1/hosts/bulk \
  -H "X-API-Key: my-key" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "power-on",
    "hosts": [
      {"namespace": "default", "name": "server-01"},
      {"namespace": "default", "name": "server-02"},
      {"namespace": "default", "name": "server-03"}
    ]
  }'
```

Supported bulk actions: `power-on`, `power-off`, `reboot`, `inspect`, `labels`.

---

## Part 3: Claiming Hardware for Clusters

ClusterClaims let you allocate bare metal hosts to clusters with scheduling strategies.

### Step 1: Create a Hardware Pool

Pools define which hosts are available for allocation and enforce quotas:

```yaml
# pool.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: HardwarePool
metadata:
  name: production-pool
spec:
  hostSelector:
    matchLabels:
      environment: production
  maxHosts: 50
  priority: 90             # 0-100, higher = preferred
  quotas:
    maxCPUCores: 1600
    maxRAMGB: 6400
    maxDiskGB: 102400
    maxClaims: 10
```

```bash
kubectl apply -f pool.yaml
kubectl get hardwarepools    # short name: hwp
```

**Webhook validation rules:**
- `maxHosts` must be > 0
- `priority` must be 0-100
- All quota values must be non-negative
- Cannot reduce `maxHosts` below current `totalHosts` on update

### Step 2: Create a ClusterClaim

```yaml
# claim.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: ClusterClaim
metadata:
  name: my-cluster
spec:
  controlPlaneCount: 3
  workerCount: 10
  schedulingStrategy: topology-aware
  hardwareRequirements:
    minCPUCores: 32
    minRAMGB: 128
    minDiskGB: 1024
    storageType: NVMe          # SSD, NVMe, or HDD
    networkSpeed: 25Gbps       # 1Gbps, 10Gbps, 25Gbps, 100Gbps
    architecture: amd64        # amd64 or arm64
  labels:
    environment: production
    cluster: my-cluster
  timeout: 30m
```

```bash
kubectl apply -f claim.yaml
kubectl get clusterclaims -w   # short name: cc

# Check which hosts were claimed
kubectl get baremetalhosts -l bmo.metal3.io/claim-name=my-cluster
```

### Scheduling Strategies

| Strategy | Algorithm | Best For |
|----------|-----------|----------|
| `binpack` | Selects the most powerful hosts first | Maximize utilization, minimize host count |
| `spread` | Round-robin distribution across hosts | Even load distribution |
| `topology-aware` | Distributes across racks/zones, round-robin per fault domain | Fault tolerance and HA |

### Claim Lifecycle

```
Created --> Pending --> Partial --> Fulfilled
                |                       |
                v                       v
            Expired              (hosts claimed)
                |
                v
             Failed
```

**Webhook validation rules:**
- Control plane count should be odd (warning if even, for quorum)
- Total node count must be > 0
- Cannot reduce control plane count on a fulfilled claim

---

## Part 4: Power Management

PowerPolicies control host power states automatically.

### Power Modes

```yaml
# Always On - production workloads
apiVersion: bmo.metal3.io/v1beta1
kind: PowerPolicy
metadata:
  name: prod-always-on
spec:
  hostSelector:
    matchLabels:
      environment: production
  mode: AlwaysOn
  gracefulShutdown: true
---
# Scheduled - office hours only
apiVersion: bmo.metal3.io/v1beta1
kind: PowerPolicy
metadata:
  name: dev-scheduled
spec:
  hostSelector:
    matchLabels:
      environment: development
  mode: Scheduled
  powerOnSchedule: "0 8 * * 1-5"    # 8am weekdays
  powerOffSchedule: "0 20 * * 1-5"  # 8pm weekdays
---
# On Demand - spin up when needed
apiVersion: bmo.metal3.io/v1beta1
kind: PowerPolicy
metadata:
  name: staging-on-demand
spec:
  hostSelector:
    matchLabels:
      environment: staging
  mode: OnDemand
  idleTimeout: 2h
  powerCapWatts: 500
```

```bash
kubectl apply -f power-policies.yaml
kubectl get powerpolicies    # short name: pp
```

**Webhook validation rules:**
- `mode` must be one of: `AlwaysOn`, `AlwaysOff`, `Scheduled`, `OnDemand`
- `Scheduled` mode requires both `powerOnSchedule` and `powerOffSchedule`
- `OnDemand` mode without `idleTimeout` triggers a warning
- `powerCapWatts` must be >= 0

---

## Part 5: Firmware Management

FirmwareProfiles manage BIOS/firmware settings fleet-wide.

```yaml
# firmware.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: FirmwareProfile
metadata:
  name: dell-r740-firmware
spec:
  hostSelector:
    matchLabels:
      vendor: dell
      model: r740
  settings:
    firmwareVersion: "<ephemeral-ip>"
    firmwareURL: "https://firmware.example.com/dell-r740-2.80.bin"
    firmwareChecksum: "sha256:abc123def456"
    bios:
      SysProfile: PerfOptimized
      ProcVirtualization: Enabled
      BootMode: Uefi
      SecureBoot: Enabled
    bmc:
      NTPEnable: Enabled
      Timezone: UTC
  enforceOnDrift: true
  maxConcurrent: 2
```

```bash
kubectl apply -f firmware.yaml
kubectl get firmwareprofiles    # short name: fwp
```

**Webhook validation rules:**
- If `firmwareURL` is set, it must be a valid `http://` or `https://` URL
- If `firmwareURL` is set, `firmwareChecksum` is required
- Invalid URL format is rejected

**Firmware update flow:**
1. Controller matches hosts by selector
2. Compares current firmware state
3. Annotates hosts needing updates (`firmware.metal3.io/status: pending`)
4. BMCReconciler transitions status to `applying` and sets `firmware.metal3.io/apply-started` timestamp
5. External firmware operator sets `firmware.metal3.io/result` to `completed` or `failed`
6. BMCReconciler detects completion and sets status to `done` or `error`
7. If no result arrives within 30 minutes, status is set to `timeout`
8. Only Redfish, iLO5, and iDRAC BMC types support firmware updates

**Firmware annotations:**

| Annotation | Values | Description |
|-----------|--------|-------------|
| `firmware.metal3.io/status` | `pending`, `applying`, `done`, `error`, `timeout` | Current firmware update state |
| `firmware.metal3.io/apply-started` | RFC 3339 timestamp | When the update started (for timeout detection) |
| `firmware.metal3.io/result` | `completed`, `failed` | Set by external operator to signal completion |
| `firmware.metal3.io/last-reconciled` | RFC 3339 timestamp | Last time the BMCReconciler processed this host |

### View Firmware Settings via API

```bash
# Get current firmware settings for a host
curl http://localhost:8888/api/v1/hosts/default/server-01/firmwaresettings \
  -H "X-API-Key: my-key"

# Get firmware components
curl http://localhost:8888/api/v1/hosts/default/server-01/firmwarecomponents \
  -H "X-API-Key: my-key"

# Get firmware schema
curl http://localhost:8888/api/v1/hosts/default/server-01/firmwareschema \
  -H "X-API-Key: my-key"
```

---

## Part 6: Scheduled Maintenance

MaintenanceWindows schedule maintenance with host draining.

```yaml
# maintenance.yaml
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
  schedule: "0 2 * * 0"       # recurring: every Sunday at 2am
  actions:
  - type: Inspection
    timeout: 15m
  - type: FirmwareUpdate
    timeout: 45m
    parameters:
      force: "false"
  - type: Reboot
    timeout: 10m
  maxConcurrent: 3
  drainBeforeMaintenance: true
```

```bash
kubectl apply -f maintenance.yaml
kubectl get maintenancewindows    # short name: mw
```

**Webhook validation rules:**
- Duration must be at least 5 minutes
- At least one maintenance action must be defined
- Past start time triggers a warning (not an error)
- Cannot modify an active maintenance window

**Maintenance lifecycle:** `Scheduled` -> `Active` -> `Completed`

---

## Part 7: Auto-Remediation

RemediationStrategies detect failures and automatically remediate hosts.

```yaml
# remediation.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: RemediationStrategy
metadata:
  name: auto-heal
spec:
  hostSelector:
    matchLabels:
      environment: production
  strategy: Progressive
  maxConcurrent: 1
  retryConfig:
    maxRetries: 3
    retryDelay: 5m
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
    maxLevel: 3
    notificationChannels:
    - slack-ops
    - pagerduty
```

```bash
kubectl apply -f remediation.yaml
kubectl get remediationstrategies    # short name: rs
```

**Strategy types:**

| Strategy | Behavior |
|----------|----------|
| `Progressive` | Escalate through ordered actions with retries and backoff |
| `Immediate` | Execute first action immediately on error detection |
| `Maintenance` | Defer remediation to next maintenance window |

**Webhook validation rules:**
- At least one remediation action required
- Action `order` values must be unique (no duplicates)
- `maxConcurrent` must be >= 1
- Strategy must be `Progressive`, `Immediate`, or `Maintenance`
- If escalation is enabled, `maxLevel` must be >= 1

---

## Part 8: Alerting and Notifications

AlertPolicies evaluate conditions and send notifications through multiple channels.

```yaml
# alerts.yaml
apiVersion: bmo.metal3.io/v1beta1
kind: AlertPolicy
metadata:
  name: production-alerts
spec:
  hostSelector:
    matchLabels:
      environment: production
  severity: Critical
  enabled: true
  cooldown: 10m
  conditions:
  - type: HostError
  - type: HostOffline
  - type: ClaimFailed
  - type: RemediationFailed
  channels:
  - type: Webhook
    url: https://alerts.example.com/webhook/bmo
  - type: Slack
    url: https://hooks.slack.com/services/T.../B.../xxx
```

```bash
kubectl apply -f alerts.yaml
kubectl get alertpolicies    # short name: ap
```

**Available condition types:** `HostError`, `HostOffline`, `ClaimExpired`, `ClaimFailed`, `RemediationFailed`, `MaintenanceFailed`, `FirmwareError`, `PowerCapExceeded`

**Notification channels:** Webhook (HTTP POST), Slack (incoming webhook), Email (SMTP)

### Email Notification Setup

Email alerts require SMTP configuration via environment variables:

```bash
# Set SMTP environment variables before starting the controller
export SMTP_HOST=mail.example.com
export SMTP_PORT=587              # default: 587
export SMTP_FROM=alerts@example.com
export SMTP_USERNAME=smtp-user    # optional, for authenticated SMTP
export SMTP_PASSWORD=smtp-pass    # optional, for authenticated SMTP
```

Then add an Email channel to your AlertPolicy:

```yaml
channels:
- type: Email
  emails:
  - ops-team@example.com
  - oncall@example.com
```

When `SMTP_USERNAME` is set, the controller uses SMTP AUTH with STARTTLS. When omitted, it uses plain SMTP (common for internal mail relays that don't require authentication).

### Test a Webhook

```bash
curl -X POST http://localhost:8888/api/v1/alertpolicies/test-webhook \
  -H "X-API-Key: my-key" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://hooks.slack.com/services/T.../B.../xxx", "type": "slack"}'
```

The test endpoint validates URLs (HTTPS only) and blocks SSRF against private IP ranges.

---

## Part 9: Security Policies

SecurityPolicies enforce secure boot, TPM, compliance scanning, and intrusion detection.

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
    allowedNetworks: ["10.0.100.0/24"]
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
kubectl get securitypolicies    # short name: sp
```

### Auto-Remediation

When `autoRemediate: true` is set in the compliance section, the controller automatically annotates non-compliant hosts:

| Violation | Annotation Set | Effect |
|-----------|---------------|--------|
| BMC not using HTTPS | `security.metal3.io/bmc-https-enforce=true` | Signals external tooling to reconfigure BMC |
| Secure Boot not enabled | `security.metal3.io/remediation-secureboot=needed` | Flags host for operator attention (hardware feature) |
| TPM not present | `security.metal3.io/remediation-tpm=needed` | Flags host for operator attention (hardware feature) |

Remediation actions are recorded in `status.remediationActions` (capped at 50 entries) with host name, rule, action taken, and timestamp.

SecureBoot and TPM are hardware features that cannot be toggled via software annotations — the controller flags them for manual operator intervention.

---

## Part 10: Energy Management

EnergyPolicies implement power capping, carbon-aware scheduling, and workload consolidation.

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
      action: suspend           # on, off, suspend, hibernate
    - days: [Saturday, Sunday]
      startTime: "00:00"
      endTime: "23:59"
      action: off
  consolidation:
    enabled: true
    cpuThresholdPercent: 15
    memoryThresholdPercent: 20
    minDuration: 30m
    strategy: power-off         # power-off, suspend, migrate
    cooldownPeriod: 1h
  carbonAware:
    enabled: true
    apiEndpoint: https://carbon-api.example.com/intensity
    targetIntensity: 200
```

```bash
kubectl apply -f energy.yaml
kubectl get energypolicies    # short name: ep
```

---

## Part 11: Telemetry and Monitoring

TelemetryPolicies collect hardware metrics and detect anomalies.

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
    - type: network_errors
  exporters:
  - name: prometheus
    type: prometheus
    endpoint: http://prometheus:9090/api/v1/write
  - name: ops-webhook
    type: webhook
    endpoint: https://ops.example.com/telemetry
  anomalyDetection:
    enabled: true
    model: statistical          # statistical, ml, isolation_forest
    sensitivity: medium
  retention:
    rawMetrics: "7d"
    aggregatedMetrics: "90d"
    anomalies: "365d"
```

```bash
kubectl apply -f telemetry.yaml
kubectl get telemetrypolicies    # short name: tp
```

### Prometheus Metrics

IronWolf exposes metrics on `:8080/metrics`:

| Metric | Type | Description |
|--------|------|-------------|
| `bmo_baremetal_hosts_total` | Gauge | Total hosts by namespace and status |
| `bmo_baremetal_host_power_state` | Gauge | Per-host power state |
| `bmo_cluster_claim_status` | Gauge | Claim status by phase |
| `bmo_cluster_claim_duration_seconds` | Gauge | Time from claim creation to fulfillment |
| `bmo_remediation_attempts_total` | Counter | Remediation actions taken |
| `bmo_firmware_update_status` | Gauge | Firmware update progress |
| `bmo_alert_policy_active_alerts` | Gauge | Active alerts per policy |
| `bmo_security_compliant_hosts` | Gauge | Compliant hosts per security policy |
| `bmo_energy_savings_watts` | Gauge | Estimated power savings |
| `bmo_telemetry_anomalies_total` | Counter | Anomalies detected |
| `bmo_backup_total` | Counter | Backups completed |

---

## Part 12: Network Configuration

NetworkProfiles manage switch configuration, VLANs, BGP, and ACLs.

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
    vendor: arista               # arista, cisco, juniper, mellanox
    protocol: eapi               # eapi, netconf, restconf, ssh
    credentialsRef:
      name: switch-credentials
      namespace: ironwolf-system
  vlans:
  - id: 100
    name: management
    subnet: 10.0.100.0/24
    gateway: 10.0.100.1
    vlanType: management         # management, storage, tenant, data
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
    prefixes: ["10.0.100.0/24"]
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
kubectl get networkprofiles    # short name: np
```

---

## Part 13: Backup and Restore

### Schedule Backups

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
    fullBackup: "0 2 * * 0"          # weekly full
    incrementalBackup: "0 2 * * 1-6"  # daily incremental
    retention:
      fullBackups: 4
      incremental: 30
  storage:
    type: s3                         # s3, local, nfs
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

### Restore a Configuration

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
  strategy: rolling               # rolling or parallel
  rollbackOnFailure: true
  postRestore:
  - type: reboot
    waitForReady: true
  - type: verify
```

```bash
kubectl apply -f backup.yaml
kubectl get hardwarebackups    # short name: hwb

kubectl apply -f restore.yaml
kubectl get restorejobs -w     # short name: rj
```

### Backup Operator State

In addition to hardware backups, you can back up all CRD resources:

```bash
# Backup all IronWolf resources
# Backup CRDs/state with your platform backup tool or Velero

# Restore from backup
# Restore from your platform backup / Velero restore /path/to/backup.tar.gz

# Dry run restore
# Restore from your platform backup / Velero restore --backup /path/to/backup.tar.gz --dry-run
```

---

## Part 14: Web Dashboard

### Build and Run

```bash
# Build everything (frontend + backend)
make web

# Run locally
./bin/ironwolf-web --kubeconfig ~/.kube/config --static-dir web/dashboard/dist

# Open http://localhost:8888
```

### Development Mode

```bash
# Frontend dev server with hot reload
make web-dev

# Backend only
make web-server && ./bin/ironwolf-web --kubeconfig ~/.kube/config
```

### Authentication

```bash
# Single admin key
./bin/ironwolf-web --api-key my-secret-key

# Multi-user RBAC with role-based access
./bin/ironwolf-web --api-key "admin-key:admin,ops-key:operator,view-key:viewer"
```

**Roles:**

| Role | GET | POST/PUT/PATCH | DELETE hosts |
|------|-----|----------------|-------------|
| `admin` | Yes | Yes | Yes |
| `operator` | Yes | Yes | No |
| `viewer` | Yes | No | No |

Clients authenticate via `X-API-Key` header or `Authorization: Bearer <token>` header. WebSocket connections authenticate via `Sec-WebSocket-Protocol: auth.<token>` subprotocol (the dashboard handles this automatically).

### Dashboard Pages

| Category | Pages |
|----------|-------|
| **Overview** | Dashboard (KPIs, charts, capacity gauges) |
| **Infrastructure** | Hosts, Host Detail, Host Compare, Topology, Cluster (nodes/workloads/capacity/namespaces), CAPM3 (Metal3Cluster/Machine/DataTemplate) |
| **Orchestration** | Claims, Pools, Power, Firmware, Maintenance, Remediation |
| **Operations** | Alerts, Energy, Network, Security, Telemetry, Backups, Restores |
| **Tools** | Audit Log, Settings, API Docs, Capacity Planner, Resource Graph |

### Keyboard Shortcuts

Press `?` to see all shortcuts. Key shortcuts:

- `Cmd/Ctrl+K` -- Global search
- `?` -- Keyboard shortcuts help

### WebSocket Live Updates

The dashboard connects to `/api/v1/ws` for real-time event push. Connection status is shown in the header (green dot = connected).

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `IRONWOLF_MAX_TRACKED_IPS` | `10000` | Max IPs tracked by auth rate limiter |
| `IRONWOLF_MAX_WS_CLIENTS` | `100` | Max concurrent WebSocket connections |
| `GRAFANA_URL` | (none) | Grafana dashboard URL shown in the UI |

---

## Part 15: CLI Tools

### BMH Inspector

Inspect and report on BareMetalHost resources from the command line.

```bash
# Build
# CLI tools ship with the commercial package

# List all hosts
./bin/bmh-inspector list
# Output: NAME  NAMESPACE  STATUS  POWER  BMC TYPE  CLAIMED

# List hosts in a specific namespace
./bin/bmh-inspector list -n metal3-system

# Inspect a specific host (hardware details, claim info)
./bin/bmh-inspector inspect server-01

# Cluster hardware summary (totals, capacity)
./bin/bmh-inspector summary
```

Use `-n` / `--namespace` to target a specific namespace (defaults to the current kubeconfig context namespace).

**inspect** output includes: Name, Namespace, BMC address, BMC Type, Online status, CPU cores/arch, RAM, disk count and sizes, NIC count and MACs, claim info and role.

### Firmware Updater

Manage firmware updates for BareMetalHosts.

```bash
# Build
# CLI tools ship with the commercial package

# Show firmware status for all hosts
./bin/firmware-updater status
# Output: HOST  BMC TYPE  FW STATUS  PROFILE

# Show firmware status in a specific namespace
./bin/firmware-updater status -n metal3-system

# Check if a host supports firmware updates
./bin/firmware-updater check server-01
# Reports: BMC Type and whether firmware updates are supported

# Show details of a firmware profile
./bin/firmware-updater show dell-r740-firmware
```

Use `-n` / `--namespace` to target a specific namespace (defaults to the current kubeconfig context namespace).

Firmware updates are supported on Redfish, iLO5, and iDRAC BMC types only.

Both CLI tools use a 30-second timeout for Kubernetes API calls (configurable via `K8sDefaultTimeout` in `pkg/utils`).

---

## Part 16: Production Deployment

### Pre-Deployment Checklist

```bash
# Run automated pre-flight checks
kubectl get ns ironwolf-system && kubectl get crds | grep bmo.metal3.io

# Auto-fix common issues
kubectl get crds | grep bmo.metal3.io && kubectl get pods -n ironwolf-system
```

### Deploy

```bash
# packaged operator tooling (provided with your license) \
  --env production \
  --image ghcr.io/zyvorai/ironwolf:v1.0.0 \
  --replicas 2 \
  --wait
```

Or with Helm:

```bash
helm install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system \
  --create-namespace \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set replicaCount=2 \
  --set leaderElection.enabled=true \
  --set networkPolicy.enabled=true \
  --set dashboard.auth.apiKey="$(openssl rand -hex 32)"
```

### Post-Deployment Verification

```bash
# Run selftest
kubectl get pods -n ironwolf-system

# Or manually
kubectl get pods,deploy -n ironwolf-system

# Health check
kubectl get pods -n ironwolf-system

# JSON output for monitoring
kubectl get pods -n ironwolf-system -o wide
```

### Upgrades

```bash
# Rolling upgrade with auto-rollback on failure
# packaged operator tooling (provided with your license) --image ghcr.io/zyvorai/ironwolf:v1.1.0 --rollback-on-fail
```

### Manager Flags

```bash
./bin/manager \
  --metrics-bind-address :8080 \
  --health-probe-bind-address :8081 \
  --leader-elect \
  --dev-mode    # verbose, human-readable logs
```

### Security Hardening

The controller runs with:
- Non-root user (UID 65532)
- Read-only root filesystem
- All capabilities dropped
- Seccomp profile: RuntimeDefault
- Pod Security Standards: `restricted` (enforce, audit, warn)

SSRF protection is built in: all outbound webhook requests block private/reserved IP ranges (RFC 1918, link-local, loopback, IPv4-mapped IPv6), disable keep-alives to mitigate DNS rebinding, and block redirects.

### Troubleshooting

```bash
# Quick diagnostics
kubectl get pods -n ironwolf-system -o wide

# Controller logs
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf --tail=100

# Collect full debug bundle
make debug-bundle
# or
kubectl -n ironwolf-system logs deploy/ironwolf-controller-manager --tail=200

# Check all resources at a glance
for kind in cc fwp mw pp hwp rs ap np sp ep tp hwb rj; do
  echo "=== ${kind} ==="
  kubectl get "${kind}" --all-namespaces 2>/dev/null || echo "(not found)"
done
```

---

## Part 17: API Reference

### Authentication

All API requests require authentication when `--api-key` is configured:

```bash
# Via X-API-Key header
curl -H "X-API-Key: my-key" http://localhost:8888/api/v1/hosts

# Via Bearer token
curl -H "Authorization: Bearer my-key" http://localhost:8888/api/v1/hosts
```

### Health (No Auth Required)

```
GET /api/v1/health
```

### Overview

```
GET /api/v1/overview    -- Dashboard KPI stats
GET /api/v1/whoami      -- Current user role
```

### Resources (CRUD)

Every resource type supports the same pattern:

```
GET    /api/v1/{resource}                     -- List all
GET    /api/v1/{resource}/{ns}/{name}         -- Get one
POST   /api/v1/{resource}                     -- Create
PUT    /api/v1/{resource}/{ns}/{name}         -- Update
PATCH  /api/v1/{resource}/{ns}/{name}         -- Patch
DELETE /api/v1/{resource}/{ns}/{name}         -- Delete
```

**Resource types:** `hosts`, `clusterclaims`, `hardwarepools`, `powerpolicies`, `firmwareprofiles`, `maintenancewindows`, `remediationstrategies`, `alertpolicies`, `energypolicies`, `networkprofiles`, `securitypolicies`, `telemetrypolicies`, `hardwarebackups`, `restorejobs`

### Host Actions

```
POST   /api/v1/hosts                                -- Register host
POST   /api/v1/hosts/import                         -- Bulk import
POST   /api/v1/hosts/bulk                           -- Bulk actions
POST   /api/v1/hosts/{ns}/{name}/power              -- Power on/off
POST   /api/v1/hosts/{ns}/{name}/reboot             -- Reboot
POST   /api/v1/hosts/{ns}/{name}/inspect            -- Trigger inspection
POST   /api/v1/hosts/{ns}/{name}/provision          -- Provision with image
POST   /api/v1/hosts/{ns}/{name}/deprovision        -- Deprovision
POST   /api/v1/hosts/{ns}/{name}/customdeploy       -- Custom deploy
POST   /api/v1/hosts/{ns}/{name}/detach             -- Detach/attach
POST   /api/v1/hosts/{ns}/{name}/inspect-control    -- Inspect control
GET    /api/v1/hosts/{ns}/{name}/health             -- Health check
GET    /api/v1/hosts/{ns}/{name}/firmwaresettings   -- Firmware settings
GET    /api/v1/hosts/{ns}/{name}/firmwarecomponents -- Firmware components
GET    /api/v1/hosts/{ns}/{name}/firmwareschema     -- Firmware schema
PUT    /api/v1/hosts/{ns}/{name}                    -- Edit host
PUT    /api/v1/hosts/{ns}/{name}/raid               -- RAID config
PUT    /api/v1/hosts/{ns}/{name}/firmware            -- Firmware config
PUT    /api/v1/hosts/{ns}/{name}/bootmode           -- Boot mode
PUT    /api/v1/hosts/{ns}/{name}/labels             -- Labels
PUT    /api/v1/hosts/{ns}/{name}/annotations        -- Annotations
PUT    /api/v1/hosts/{ns}/{name}/credentials        -- Credentials
DELETE /api/v1/hosts/{ns}/{name}                    -- Delete host
DELETE /api/v1/hosts/{ns}/{name}/raid               -- Clear RAID
```

### Cluster Info

```
GET /api/v1/cluster/nodes       -- Kubernetes nodes
GET /api/v1/cluster/workloads   -- Deployments, StatefulSets, DaemonSets, Pods
GET /api/v1/cluster/capacity    -- CPU, memory, pod utilization
GET /api/v1/cluster/namespaces  -- Namespace list
```

### Other Endpoints

```
GET  /api/v1/topology           -- Resource topology tree
GET  /api/v1/events             -- All Kubernetes events
GET  /api/v1/events/{ns}/{name} -- Events for a resource
GET  /api/v1/audit              -- Audit log (requires admin role)
GET  /api/v1/metrics/history    -- Metrics history (24h ring buffer)
GET  /api/v1/secrets            -- List secrets (metadata only)
GET  /api/v1/secrets/{ns}/{name}/keys  -- Secret key names
POST /api/v1/secrets/{ns}       -- Create secret
GET  /api/v1/imagecatalog       -- OS image catalog
POST /api/v1/alertpolicies/test-webhook  -- Test webhook delivery
GET  /api/v1/capm3/clusters     -- Metal3Cluster list
GET  /api/v1/capm3/machines     -- Metal3Machine list
GET  /api/v1/capm3/datatemplates  -- Metal3DataTemplate list
GET  /api/v1/ironic/status      -- Ironic service status
GET  /api/v1/ws                 -- WebSocket (real-time events)
```

### WebSocket

Connect to `/api/v1/ws` for live updates. Authentication is via `Sec-WebSocket-Protocol` subprotocol:

```javascript
const ws = new WebSocket('wss://bmo.example.com/api/v1/ws', [
  'v1.bmo.metal3.io',      // protocol identifier
  'auth.YOUR_API_KEY'       // authentication
]);

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  console.log(msg.type, msg.data);
};
```

Messages are JSON with `{type: string, data: object}` format.

---

## Makefile Quick Reference

```bash
# Build
# Binaries ship with the commercial package          # Manager binary
# CLI tools ship with the commercial package    # CLI tools (bmh-inspector, firmware-updater)
# Binaries ship with the commercial package      # Everything (manager + tools + web)
# Binaries ship with the commercial package    # Cross-compile for linux/darwin amd64/arm64

# Web Dashboard
make web            # Build frontend + backend
make web-dev        # Frontend dev server with hot reload
make web-run        # Build and run locally
make web-server     # Build backend only

# Test
make test           # Unit tests with coverage
make test-web       # Web backend tests
make test-all       # All tests
make test-e2e       # E2E tests (requires cluster)
make test-coverage  # HTML coverage report

# Deploy
make install        # Install CRDs
make deploy         # Deploy to production
make deploy-dev     # Deploy in dev mode
make undeploy       # Remove controller

# Docker
make docker-build   # Build manager image
make docker-all     # Build all images

# Kind
make kind-setup     # Create Kind cluster with samples
make kind-reload    # Rebuild and reload into Kind
make kind-destroy   # Destroy Kind cluster

# Ops
kubectl get pods -n ironwolf-system       # Post-install verification
make preflight      # Pre-deployment checks
make health         # Full-stack health check
make debug-bundle   # Collect diagnostics
make backup         # Backup CRD resources
make restore        # Restore from backup

# Validate
make validate-crds     # Validate CRD manifests
make validate-helm     # Validate Helm chart
make validate-samples  # Validate sample manifests

# Generate
make manifests      # Regenerate CRD + RBAC manifests
make generate       # Regenerate DeepCopy implementations
```
