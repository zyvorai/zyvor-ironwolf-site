# API Reference

## Group: bmo.metal3.io/v1beta1

All IronWolf custom resources belong to the `bmo.metal3.io` API group at version `v1beta1`.

---

### ClusterClaim

Manages automated hardware allocation for clusters.

**Resource:** `clusterclaims.bmo.metal3.io` | **Short name:** `cc`

**Webhook:** Validating (ClusterClaimValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.controlPlaneCount` | int | Yes | Number of control plane nodes (1-7) |
| `spec.workerCount` | int | Yes | Number of worker nodes (0+) |
| `spec.schedulingStrategy` | string | Yes | `binpack`, `spread`, or `topology-aware` (used when `selectedHosts` is empty) |
| `spec.hardwareRequirements` | object | Yes | Hardware requirements per host |
| `spec.hardwareRequirements.minCPUCores` | int | Yes | Minimum CPU cores per host (min: 2) |
| `spec.hardwareRequirements.minRAMGB` | int | Yes | Minimum RAM in GB per host (min: 4) |
| `spec.hardwareRequirements.minDiskGB` | int | Yes | Minimum disk in GB per host (min: 20) |
| `spec.hardwareRequirements.storageType` | string | No | `SSD`, `NVMe`, or `HDD` |
| `spec.hardwareRequirements.networkSpeed` | string | No | `1Gbps`, `10Gbps`, `25Gbps`, or `100Gbps` |
| `spec.hardwareRequirements.architecture` | string | No | `amd64` or `arm64` |
| `spec.hardwareRequirements.gpuRequired` | bool | No | Require at least one host with GPU (`bmo.metal3.io/gpu=true` or inspect data) |
| `spec.selectedHosts` | []object | No | Pin BareMetalHosts and roles; when set, auto-scheduling is skipped |
| `spec.selectedHosts[].name` | string | Yes* | BareMetalHost name (*required per entry) |
| `spec.selectedHosts[].namespace` | string | No | Defaults to the claim namespace |
| `spec.selectedHosts[].role` | string | Yes* | `controlplane`, `worker`, or `gpu-worker` |
| `spec.installGPUOperator` | bool | No | Request NVIDIA GPU Operator install after bootstrap when GPU workers are present |
| `spec.preferredImageURL` | string | No | Optional OS image URL for bootstrap (e.g. GPU golden image) |
| `spec.labels` | map[string]string | No | Labels to apply to claimed hosts |
| `spec.annotations` | map[string]string | No | Annotations to apply to claimed hosts |
| `spec.timeout` | duration | No | Timeout for claim fulfillment |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Pending`, `Partial`, `Fulfilled`, `Failed`, or `Expired` |
| `status.claimedHosts` | []string | List of claimed host names |
| `status.assignedHosts` | []string | All assigned host names |
| `status.controlPlaneHosts` | []string | Control-plane host names |
| `status.workerHosts` | []string | Worker host names |
| `status.gpuWorkerHosts` | []string | Hosts claimed with role `gpu-worker` |
| `status.bootstrap` | object | Post-fulfillment CAPM3 / lab bootstrap progress |
| `status.bootstrap.phase` | string | `Pending`, `Associated`, `OSProvisioning`, `Provisioned`, or `Ready` |
| `status.bootstrap.gpuOperatorInstalled` | bool | NVIDIA GPU Operator install succeeded |
| `status.bootstrap.gpuOperatorMessage` | string | Human-readable GPU Operator status |
| `status.lastTransitionTime` | time | Timestamp of last phase transition |
| `status.reason` | string | Machine-readable reason for current phase |
| `status.assignedHostCount` | int | Number of hosts currently assigned to this claim |
| `status.message` | string | Human-readable message |

**Validation:** `MinCPUCores`, `MinRAMGB`, and `MinDiskGB` each have a Minimum of 1. GPU workers require GPU-capable hosts. Counts should match `selectedHosts` role totals when that list is set.

---

### FirmwareProfile

Manages firmware configurations across host fleets.

**Resource:** `firmwareprofiles.bmo.metal3.io` | **Short name:** `fwp`

**Webhook:** Validating (FirmwareProfileValidator)

**Finalizer:** `bmo.metal3.io/firmware-finalizer` -- cleans up `firmware.metal3.io/*` annotations on matched hosts when the FirmwareProfile is deleted.

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching hosts |
| `spec.settings` | object | Yes | Firmware settings to apply |
| `spec.settings.bios` | map[string]string | No | BIOS settings key-value pairs |
| `spec.settings.bmc` | map[string]string | No | BMC settings key-value pairs |
| `spec.settings.firmwareVersion` | string | No | Target firmware version |
| `spec.settings.firmwareURL` | string | No | URL for firmware image |
| `spec.settings.firmwareChecksum` | string | No | SHA256 checksum of firmware image |
| `spec.enforceOnDrift` | bool | No | Re-apply on configuration drift (default: true) |
| `spec.maxConcurrent` | int | No | Max concurrent updates (default: 1, min: 1) |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.matchingHosts` | int | Number of hosts matching the selector |
| `status.appliedHosts` | int | Number of hosts with firmware applied |
| `status.pendingHosts` | int | Number of hosts pending update |
| `status.errorHosts` | int | Number of hosts with errors |
| `status.hostStatuses` | []object | Per-host status (name, status, message) |
| `status.lastReconcileTime` | time | Last reconciliation timestamp |

---

### MaintenanceWindow

Orchestrates scheduled maintenance operations.

**Resource:** `maintenancewindows.bmo.metal3.io` | **Short name:** `mw`

**Webhook:** Validating (MaintenanceWindowValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching hosts |
| `spec.startTime` | time | Yes | Start of maintenance window |
| `spec.duration` | duration | Yes | Window duration |
| `spec.schedule` | string | No | Cron schedule for recurring windows |
| `spec.actions` | []object | Yes | Actions to perform during maintenance (MinItems=1) |
| `spec.actions[].type` | string | Yes | `Reboot`, `FirmwareUpdate`, `BIOSConfig`, `Inspection`, or `PowerCycle` |
| `spec.actions[].parameters` | map[string]string | No | Parameters for the action |
| `spec.actions[].timeout` | duration | No | Timeout for this action |
| `spec.maxConcurrent` | int | No | Max concurrent hosts (default: 1, min: 1) |
| `spec.drainBeforeMaintenance` | bool | No | Drain workloads first (default: true) |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Scheduled`, `Active`, `Completed`, `Failed`, or `Cancelled` |
| `status.completedHosts` | int | Number of hosts completed |
| `status.inProgressHosts` | int | Number of hosts in progress |
| `status.failedHosts` | int | Number of hosts failed |
| `status.hostStatuses` | []object | Per-host status (name, currentAction, status, message) |
| `status.nextRun` | time | Next scheduled run (for recurring windows) |

---

### PowerPolicy

Implements power management policies.

**Resource:** `powerpolicies.bmo.metal3.io` | **Short name:** `pp`

**Webhook:** Validating (PowerPolicyValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching hosts |
| `spec.mode` | string | Yes | `AlwaysOn`, `AlwaysOff`, `Scheduled`, or `OnDemand` |
| `spec.powerOnSchedule` | string | No | Cron for power on (Scheduled mode) |
| `spec.powerOffSchedule` | string | No | Cron for power off (Scheduled mode) |
| `spec.idleTimeout` | duration | No | Idle timeout (OnDemand mode) |
| `spec.gracefulShutdown` | bool | No | Gracefully shut down workloads before power off (default: true) |
| `spec.powerCapWatts` | int | No | Power cap in watts (0 = no cap, min: 0) |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.managedHosts` | int | Number of managed hosts |
| `status.poweredOnHosts` | int | Hosts currently powered on |
| `status.poweredOffHosts` | int | Hosts currently powered off |
| `status.estimatedSavingsWatts` | int | Estimated power savings in watts |
| `status.hostStatuses` | []object | Per-host status (name, powerState, lastTransition, powerConsumptionWatts) |

---

### HardwarePool

Manages resource pools with quotas.

**Resource:** `hardwarepools.bmo.metal3.io` | **Short name:** `hwp`

**Webhook:** Validating (HardwarePoolValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching hosts |
| `spec.maxHosts` | int | Yes | Maximum hosts in pool (min: 1) |
| `spec.quotas` | object | No | Resource quotas |
| `spec.quotas.maxCPUCores` | int | No | Maximum total CPU cores |
| `spec.quotas.maxRAMGB` | int | No | Maximum total RAM in GB |
| `spec.quotas.maxDiskGB` | int | No | Maximum total disk in GB |
| `spec.quotas.maxClaims` | int | No | Maximum number of claims against this pool |
| `spec.priority` | int | No | Pool priority 0-100 (default: 50) |
| `spec.allowOverProvisioning` | bool | No | Allow over-provisioning (default: false) |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Ready` or `OverCapacity` |
| `status.totalHosts` | int | Total hosts in the pool |
| `status.availableHosts` | int | Available (unclaimed) hosts |
| `status.claimedHosts` | int | Claimed hosts |
| `status.totalCPUCores` | int | Total CPU cores available |
| `status.totalRAMGB` | int | Total RAM in GB available |
| `status.totalDiskGB` | int | Total disk in GB available |
| `status.activeClaims` | int | Active claims against this pool |

---

### RemediationStrategy

Handles progressive failure remediation.

**Resource:** `remediationstrategies.bmo.metal3.io` | **Short name:** `rs`

**Webhook:** Validating (RemediationStrategyValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching hosts |
| `spec.strategy` | string | Yes | `Progressive`, `Immediate`, or `Maintenance` |
| `spec.maxConcurrent` | int | No | Max concurrent remediations (default: 1, min: 1) |
| `spec.retryConfig` | object | No | Retry configuration |
| `spec.retryConfig.maxRetries` | int | No | Maximum number of retries (default: 3, min: 0) |
| `spec.retryConfig.retryDelay` | duration | No | Delay between retries |
| `spec.retryConfig.exponentialBackoff` | bool | No | Use exponential backoff (default: true) |
| `spec.actions` | []object | Yes | Ordered remediation actions (MinItems=1) |
| `spec.actions[].order` | int | Yes | Order in the remediation sequence (min: 1) |
| `spec.actions[].type` | string | Yes | `Reboot`, `PowerCycle`, `Reprovision`, `Inspect`, or `Decommission` |
| `spec.actions[].timeout` | duration | No | Timeout for this action |
| `spec.actions[].successCondition` | string | No | Condition to check after action |
| `spec.escalation` | object | No | Escalation policy |
| `spec.escalation.enabled` | bool | No | Whether to escalate on failure (default: true) |
| `spec.escalation.maxLevel` | int | No | Maximum escalation level (default: 3, min: 1) |
| `spec.escalation.notificationChannels` | []string | No | Notification channels for escalation |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.monitoredHosts` | int | Number of hosts being monitored |
| `status.remediatingHosts` | int | Hosts currently being remediated |
| `status.remediatedHosts` | int | Hosts successfully remediated |
| `status.failedHosts` | int | Hosts that failed remediation |
| `status.hostStatuses` | []object | Per-host status (name, status, currentAction, retryCount, escalationLevel, message) |

---

### AlertPolicy

Defines alert conditions with multi-channel notifications.

**Resource:** `alertpolicies.bmo.metal3.io` | **Short name:** `ap`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for resources to monitor |
| `spec.conditions` | []object | Yes | Conditions that trigger alerts (MinItems=1) |
| `spec.conditions[].type` | string | Yes | `HostError`, `HostOffline`, `ClaimExpired`, `ClaimFailed`, `RemediationFailed`, `MaintenanceFailed`, `FirmwareError`, or `PowerCapExceeded` |
| `spec.conditions[].threshold` | string | No | Threshold (e.g., number of hosts, duration) |
| `spec.channels` | []object | Yes | Notification channels (MinItems=1) |
| `spec.channels[].type` | string | Yes | `Webhook`, `Slack`, or `Email` |
| `spec.channels[].url` | string | No | Webhook or Slack webhook URL |
| `spec.channels[].emails` | []string | No | Email addresses (for Email type) |
| `spec.severity` | string | Yes | `Critical`, `Warning`, or `Info` |
| `spec.enabled` | bool | No | Whether alerts are enabled (default: true) |
| `spec.cooldown` | duration | No | Cooldown between repeated alerts |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.activeAlerts` | int | Number of active alerts |
| `status.lastAlertTime` | time | Last alert fired timestamp |
| `status.totalAlertsFired` | int | Total alerts fired |
| `status.recentAlerts` | []object | Recent alert history (condition, message, timestamp, severity, resolved) |

---

### NetworkProfile

Manages switch, VLAN, BGP, and ACL network configurations for bare metal hosts.

**Resource:** `networkprofiles.bmo.metal3.io` | **Short name:** `np`

**Webhook:** None

**Finalizer:** `bmo.metal3.io/network-finalizer` -- cleans up `network.metal3.io/*` annotations on matched hosts when the NetworkProfile is deleted.

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching BareMetalHosts |
| `spec.switches` | []object | No | Switch configurations |
| `spec.switches[].name` | string | Yes | Name of the switch |
| `spec.switches[].managementIP` | string | Yes | Management IP address |
| `spec.switches[].vendor` | string | Yes | `arista`, `cisco`, `juniper`, or `mellanox` |
| `spec.switches[].model` | string | No | Switch model |
| `spec.switches[].protocol` | string | Yes | `eapi`, `netconf`, `restconf`, or `ssh` |
| `spec.switches[].credentialsRef` | SecretReference | No | Reference to credentials secret |
| `spec.vlans` | []object | No | VLAN configurations |
| `spec.vlans[].id` | int | Yes | VLAN ID (range: 1-4094) |
| `spec.vlans[].name` | string | Yes | VLAN name |
| `spec.vlans[].subnet` | string | No | Subnet in CIDR notation |
| `spec.vlans[].gateway` | string | No | Gateway IP address |
| `spec.vlans[].vlanType` | string | No | `management`, `storage`, `tenant`, or `data` |
| `spec.interfaceProfiles` | []object | No | Interface profiles for host network configuration |
| `spec.interfaceProfiles[].name` | string | Yes | Profile name |
| `spec.interfaceProfiles[].matchLabels` | map[string]string | No | Labels to match hosts for this profile |
| `spec.interfaceProfiles[].interfaces` | []object | Yes | Interface configurations |
| `spec.interfaceProfiles[].interfaces[].name` | string | Yes | Interface name |
| `spec.interfaceProfiles[].interfaces[].type` | string | Yes | `member`, `bond`, or `standalone` |
| `spec.interfaceProfiles[].interfaces[].bond` | string | No | Bond name (if type is member) |
| `spec.interfaceProfiles[].interfaces[].bondMode` | string | No | `802.3ad`, `active-backup`, or `balance-rr` |
| `spec.interfaceProfiles[].interfaces[].mtu` | int | No | MTU size |
| `spec.interfaceProfiles[].interfaces[].vlanIds` | []int | No | VLAN IDs assigned to this interface |
| `spec.bgp` | object | No | BGP configuration |
| `spec.bgp.enabled` | bool | Yes | Whether BGP is enabled |
| `spec.bgp.asNumber` | int | Yes | Autonomous system number |
| `spec.bgp.routerID` | string | No | Router ID |
| `spec.bgp.peers` | []object | No | BGP peers (address, asNumber) |
| `spec.bgp.prefixes` | []string | No | Advertised prefixes |
| `spec.acls` | []object | No | Access control lists |
| `spec.acls[].name` | string | Yes | ACL name |
| `spec.acls[].rules` | []object | Yes | ACL rules |
| `spec.acls[].rules[].action` | string | Yes | `permit` or `deny` |
| `spec.acls[].rules[].source` | string | Yes | Source address or CIDR |
| `spec.acls[].rules[].destination` | string | Yes | Destination address or CIDR |
| `spec.acls[].rules[].port` | int | No | Port number |
| `spec.acls[].rules[].protocol` | string | No | `tcp`, `udp`, `icmp`, or `any` |
| `spec.acls[].rules[].log` | bool | No | Whether to log matches |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Pending`, `Configuring`, `Active`, or `Error` |
| `status.configuredSwitches` | int | Number of configured switches |
| `status.configuredHosts` | int | Number of configured hosts |
| `status.activeVLANs` | int | Number of active VLANs |
| `status.lastReconcileTime` | time | Last reconciliation timestamp |
| `status.message` | string | Status message |

---

### SecurityPolicy

Enforces security configurations including secure boot, TPM, BMC hardening, compliance scanning, and intrusion detection.

**Resource:** `securitypolicies.bmo.metal3.io` | **Short name:** `sp`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching BareMetalHosts |
| `spec.secureBoot` | object | No | Secure boot configuration |
| `spec.secureBoot.enforced` | bool | Yes | Whether secure boot is enforced |
| `spec.secureBoot.remediation` | string | No | `block`, `warn`, or `auto-fix` |
| `spec.secureBoot.auditSchedule` | string | No | Audit schedule in cron format |
| `spec.tpm` | object | No | TPM configuration |
| `spec.tpm.required` | bool | Yes | Whether TPM is required |
| `spec.tpm.attestation` | object | No | Attestation configuration |
| `spec.tpm.attestation.enabled` | bool | Yes | Whether attestation is enabled |
| `spec.tpm.attestation.server` | string | No | Attestation server URL |
| `spec.tpm.attestation.refreshInterval` | duration | No | Refresh interval for attestation |
| `spec.bmcSecurity` | object | No | BMC security configuration |
| `spec.bmcSecurity.httpsRequired` | bool | No | Whether HTTPS is required for BMC access |
| `spec.bmcSecurity.tlsVersion` | string | No | Minimum TLS version: `1.2` or `1.3` |
| `spec.bmcSecurity.allowedNetworks` | []string | No | Allowed networks for BMC access in CIDR notation |
| `spec.bmcSecurity.passwordPolicy` | object | No | Password policy configuration |
| `spec.bmcSecurity.passwordPolicy.minLength` | int | No | Minimum password length |
| `spec.bmcSecurity.passwordPolicy.complexity` | string | No | `weak`, `medium`, or `strong` |
| `spec.bmcSecurity.passwordPolicy.expiryDays` | int | No | Password expiry in days |
| `spec.bmcSecurity.certificateRotation` | string | No | Certificate rotation interval as duration string |
| `spec.compliance` | object | No | Compliance configuration |
| `spec.compliance.standards` | []string | No | Compliance standards (e.g., `CIS`, `NIST`, `DISA_STIG`) |
| `spec.compliance.scanSchedule` | string | No | Scan schedule in cron format |
| `spec.compliance.autoRemediate` | bool | No | Whether to auto-remediate violations |
| `spec.compliance.reportFormats` | []string | No | Report output formats |
| `spec.intrusionDetection` | object | No | Intrusion detection configuration |
| `spec.intrusionDetection.enabled` | bool | Yes | Whether intrusion detection is enabled |
| `spec.intrusionDetection.sensors` | []object | No | IDS sensors |
| `spec.intrusionDetection.sensors[].type` | string | Yes | `port_scan`, `brute_force`, or `hardware_tamper` |
| `spec.intrusionDetection.sensors[].threshold` | int | No | Detection threshold |
| `spec.intrusionDetection.actions` | []object | No | IDS actions on detection |
| `spec.intrusionDetection.actions[].type` | string | Yes | `alert`, `lockdown`, or `ip_block` |
| `spec.intrusionDetection.actions[].duration` | string | No | Duration for the action |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Pending`, `Compliant`, `NonCompliant`, `Scanning`, or `Error` |
| `status.monitoredHosts` | int | Number of monitored hosts |
| `status.compliantHosts` | int | Number of compliant hosts |
| `status.nonCompliantHosts` | int | Number of non-compliant hosts |
| `status.lastScanTime` | time | Last scan timestamp |
| `status.violations` | []object | Security violations (host, rule, severity, message, detectedAt) |
| `status.remediationActions` | []object | Auto-remediation actions taken (host, rule, action, timestamp). Capped at 50 entries. |

---

### EnergyPolicy

Manages power capping, energy scheduling, workload consolidation, and carbon-aware operations.

**Resource:** `energypolicies.bmo.metal3.io` | **Short name:** `ep`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching BareMetalHosts |
| `spec.powerCapping` | object | No | Power capping configuration |
| `spec.powerCapping.enabled` | bool | Yes | Whether power capping is enabled |
| `spec.powerCapping.defaultCapWatts` | int | No | Default power cap in watts |
| `spec.powerCapping.profiles` | []object | No | Power cap profiles for different host groups |
| `spec.powerCapping.profiles[].name` | string | Yes | Profile name |
| `spec.powerCapping.profiles[].capWatts` | int | Yes | Power cap in watts |
| `spec.powerCapping.profiles[].matchLabels` | map[string]string | No | Labels to match hosts for this profile |
| `spec.powerCapping.dynamicCapping` | object | No | Dynamic capping configuration |
| `spec.powerCapping.dynamicCapping.enabled` | bool | Yes | Whether dynamic capping is enabled |
| `spec.powerCapping.dynamicCapping.algorithm` | string | No | `workload-aware`, `time-based`, or `carbon-aware` |
| `spec.powerCapping.dynamicCapping.minCapWatts` | int | No | Minimum power cap in watts |
| `spec.powerCapping.dynamicCapping.maxCapWatts` | int | No | Maximum power cap in watts |
| `spec.schedules` | []object | No | Energy schedules |
| `spec.schedules[].name` | string | Yes | Schedule name |
| `spec.schedules[].timezone` | string | No | Timezone for the schedule |
| `spec.schedules[].rules` | []object | Yes | Schedule rules |
| `spec.schedules[].rules[].days` | []string | No | Days of the week |
| `spec.schedules[].rules[].startTime` | string | Yes | Start time in HH:MM format |
| `spec.schedules[].rules[].endTime` | string | Yes | End time in HH:MM format |
| `spec.schedules[].rules[].action` | string | Yes | `on`, `off`, `suspend`, or `hibernate` |
| `spec.schedules[].selector` | LabelSelector | No | Optional selector to override the policy-level selector |
| `spec.consolidation` | object | No | Workload consolidation configuration |
| `spec.consolidation.enabled` | bool | Yes | Whether consolidation is enabled |
| `spec.consolidation.cpuThresholdPercent` | int | No | CPU utilization threshold percentage |
| `spec.consolidation.memoryThresholdPercent` | int | No | Memory utilization threshold percentage |
| `spec.consolidation.minDuration` | duration | No | Minimum duration below threshold before consolidation |
| `spec.consolidation.strategy` | string | No | `power-off`, `suspend`, or `migrate` |
| `spec.consolidation.cooldownPeriod` | duration | No | Cooldown period between consolidation actions |
| `spec.carbonAware` | object | No | Carbon-aware scheduling configuration |
| `spec.carbonAware.enabled` | bool | Yes | Whether carbon-aware scheduling is enabled |
| `spec.carbonAware.apiEndpoint` | string | No | Carbon intensity API endpoint |
| `spec.carbonAware.targetIntensity` | int | No | Target carbon intensity in gCO2eq/kWh |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.managedHosts` | int | Number of managed hosts |
| `status.powerCappedHosts` | int | Number of power-capped hosts |
| `status.suspendedHosts` | int | Number of suspended hosts |
| `status.estimatedSavingsWatts` | int | Estimated power savings in watts |
| `status.estimatedCostSavings` | string | Estimated cost savings |
| `status.carbonSavedKg` | float64 | Carbon saved in kilograms |
| `status.lastReconcileTime` | time | Last reconciliation timestamp |

---

### TelemetryPolicy

Configures hardware telemetry collection, export, and anomaly detection.

**Resource:** `telemetrypolicies.bmo.metal3.io` | **Short name:** `tp`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching BareMetalHosts |
| `spec.collection` | object | Yes | Collection configuration |
| `spec.collection.interval` | duration | Yes | Interval between collections (default: 30s) |
| `spec.collection.timeout` | duration | No | Timeout for collection operations |
| `spec.collection.metrics` | []object | Yes | Metrics to collect |
| `spec.collection.metrics[].type` | string | Yes | `temperature`, `power`, `fan`, `disk_smart`, `memory_errors`, or `network_errors` |
| `spec.collection.metrics[].sources` | []string | No | Sources to collect from |
| `spec.collection.metrics[].thresholds` | object | No | Threshold values for alerting |
| `spec.collection.metrics[].thresholds.warning` | string | No | Warning threshold value |
| `spec.collection.metrics[].thresholds.critical` | string | No | Critical threshold value |
| `spec.exporters` | []object | No | Exporters for telemetry data |
| `spec.exporters[].name` | string | Yes | Name of the exporter |
| `spec.exporters[].type` | string | Yes | `prometheus`, `webhook`, or `slack` |
| `spec.exporters[].endpoint` | string | No | Endpoint URL for the exporter |
| `spec.exporters[].interval` | duration | No | Export interval |
| `spec.exporters[].onlyOnThreshold` | bool | No | Only export when thresholds are exceeded |
| `spec.anomalyDetection` | object | No | Anomaly detection configuration |
| `spec.anomalyDetection.enabled` | bool | Yes | Whether anomaly detection is enabled |
| `spec.anomalyDetection.model` | string | No | `statistical`, `ml`, or `isolation_forest` |
| `spec.anomalyDetection.trainingWindow` | duration | No | Training window for the model |
| `spec.anomalyDetection.sensitivity` | string | No | `low`, `medium`, or `high` |
| `spec.retention` | object | No | Data retention configuration |
| `spec.retention.rawMetrics` | string | No | Retention period for raw metrics |
| `spec.retention.aggregatedMetrics` | string | No | Retention period for aggregated metrics |
| `spec.retention.anomalies` | string | No | Retention period for anomaly data |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.monitoredHosts` | int | Number of hosts being monitored |
| `status.activeCollectors` | int | Number of active collectors |
| `status.lastCollectionTime` | time | Last time telemetry was collected |
| `status.anomaliesDetected` | int | Number of anomalies detected |
| `status.thresholdViolations` | int | Number of threshold violations |
| `status.exporterStatuses` | []object | Status of each exporter (name, status, lastExportTime, error) |

---

### HardwareBackup

Manages scheduled backups of hardware configurations (BIOS, BMC, firmware, RAID, network, partitions).

**Resource:** `hardwarebackups.bmo.metal3.io` | **Short name:** `hwb`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.hostSelector` | LabelSelector | Yes | Selector for matching BareMetalHosts |
| `spec.components` | []object | Yes | Components to back up (MinItems=1) |
| `spec.components[].type` | string | Yes | `bios`, `bmc`, `firmware`, `raid`, `network`, or `partitions` |
| `spec.components[].settings` | string | No | Which settings to back up (`all` or specific) |
| `spec.schedule` | object | Yes | Backup schedule configuration |
| `spec.schedule.fullBackup` | string | No | Cron expression for full backups |
| `spec.schedule.incrementalBackup` | string | No | Cron expression for incremental backups |
| `spec.schedule.retention` | object | No | Retention policy |
| `spec.schedule.retention.fullBackups` | int | No | Number of full backups to retain |
| `spec.schedule.retention.incremental` | int | No | Number of incremental backups to retain |
| `spec.storage` | object | Yes | Storage configuration for backups |
| `spec.storage.type` | string | Yes | `s3`, `local`, or `nfs` |
| `spec.storage.endpoint` | string | No | Storage endpoint URL |
| `spec.storage.bucket` | string | No | S3 bucket name |
| `spec.storage.path` | string | No | Storage path |
| `spec.storage.encryption` | object | No | Encryption settings |
| `spec.storage.encryption.enabled` | bool | Yes | Whether encryption is enabled |
| `spec.storage.encryption.kmsKeyId` | string | No | KMS key ID for encryption |
| `spec.storage.compression` | string | No | `gzip`, `zstd`, or `none` |
| `spec.verification` | object | No | Verification settings |
| `spec.verification.enabled` | bool | Yes | Whether verification is enabled |
| `spec.verification.validateAfterBackup` | bool | No | Whether to validate immediately after backup |
| `spec.verification.restoreTestSchedule` | string | No | Cron expression for restore test schedule |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Idle`, `Running`, `Completed`, or `Failed` |
| `status.lastBackupTime` | time | Last backup timestamp |
| `status.lastBackupId` | string | Last backup identifier |
| `status.backedUpHosts` | int | Number of hosts backed up |
| `status.totalBackups` | int | Total number of backups |
| `status.storageUsedBytes` | int64 | Storage used in bytes |
| `status.lastVerificationTime` | time | Last verification timestamp |
| `status.verificationStatus` | string | `Passed`, `Failed`, or `NotRun` |

---

### RestoreJob

Restores hardware configurations from HardwareBackup snapshots to target hosts.

**Resource:** `restorejobs.bmo.metal3.io` | **Short name:** `rj`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.backupName` | string | Yes | Name of the HardwareBackup to restore from |
| `spec.backupTimestamp` | time | No | Specific backup timestamp to restore (latest if not specified) |
| `spec.targetHost` | string | Yes | Target host to restore to |
| `spec.targetNamespace` | string | No | Target namespace (defaults to current namespace) |
| `spec.components` | []BackupComponent | Yes | Components to restore. Array of objects with a `type` field (enum: `bios`, `bmc`, `firmware`, `raid`, `network`, `partitions`) |
| `spec.strategy` | string | No | `rolling` or `parallel` |
| `spec.rollbackOnFailure` | bool | No | Whether to rollback on failure |
| `spec.dryRun` | bool | No | Whether this is a dry run |
| `spec.postRestore` | []object | No | Actions to perform after restore |
| `spec.postRestore[].type` | string | Yes | `reboot`, `verify`, or `inspect` |
| `spec.postRestore[].waitForReady` | bool | No | Whether to wait for the host to be ready |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Pending`, `Running`, `Completed`, `Failed`, or `RolledBack` |
| `status.startTime` | time | Time the restore started |
| `status.completionTime` | time | Time the restore completed |
| `status.restoredComponents` | []string | Components that were successfully restored |
| `status.failedComponents` | []string | Components that failed to restore |
| `status.message` | string | Human-readable status message |

---

### HostIPAddress

Per-host IP inventory for the Customer Portal (synced from BareMetalHost NIC inspection).

**Resource:** `hostipaddresses.bmo.metal3.io` | **Short name:** `hip`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.address` | string | Yes | IPv4 or IPv6 address |
| `spec.family` | string | No | `IPv4` or `IPv6` |
| `spec.hostRef` | object | Yes | BareMetalHost reference (`name`, optional `namespace`) |
| `spec.reverseDNS` | string | No | Reverse DNS hostname |
| `spec.role` | string | No | `primary`, `mgmt`, or `additional` (default `primary`) |
| `spec.reserved` | bool | No | Reserved / unassigned pool address |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Active`, `Blocked`, or `Reserved` |
| `status.message` | string | Human-readable message |

---

### AvailabilityMonitor

HTTP/ICMP availability checks probed by the operator for Portal Monitoring.

**Resource:** `availabilitymonitors.bmo.metal3.io` | **Short name:** `amon`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.target` | string | Yes | IP address or URL to probe |
| `spec.type` | string | Yes | `icmp` or `http` (default `http`) |
| `spec.interval` | string | No | Probe interval (default `60s`) |
| `spec.hostRef` | object | No | Optional BareMetalHost association |
| `spec.displayName` | string | No | UI display name |
| `spec.enabled` | bool | No | When false, probes are skipped (default true) |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Up`, `Down`, or `Unknown` |
| `status.lastCheckTime` | time | Last probe timestamp |
| `status.latencyMs` | int64 | Last successful latency in milliseconds |
| `status.message` | string | Last check message |

---

### SupportTicket

Service-desk and sales-order tickets for Portal tenants (including Cart checkout).

**Resource:** `supporttickets.bmo.metal3.io` | **Short name:** `stkt`

**Webhook:** None

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.title` | string | Yes | Ticket title |
| `spec.body` | string | Yes | Initial description |
| `spec.priority` | string | Yes | `Low`, `Normal`, `High`, or `Urgent` (default `Normal`) |
| `spec.author` | string | No | Author display name / email |
| `spec.hostRef` | object | No | Optional related BareMetalHost |
| `spec.messages` | []object | No | Conversation thread (`author`, `body`, `createdAt`) |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `BeingProcessed`, `AwaitingReply`, `CustomerResponded`, `Resolved`, or `Closed` |
| `status.lastUpdated` | time | Last update timestamp |

---

### NetworkSwitchProfile

Manages a Top-of-Rack (ToR) switch configuration via the Ironic Networking Service. Renamed from
`BareMetalSwitch` to avoid colliding with upstream baremetal-operator's own `BareMetalSwitch` type
(Ironic switch-config generation) — see `docs/guides/bmo-upstream-parity.md`.

**Resource:** `networkswitchprofiles.bmo.metal3.io` | **Short name:** `nsp`

**Webhook:** Validating (NetworkSwitchProfileValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.managementAddress` | string | Yes | Management address of the switch |
| `spec.credentialsSecretRef` | object | Yes | Reference to a Secret containing switch credentials |
| `spec.switchModel` | string | No | Switch model identifier |
| `spec.ports` | []object | No | Port configurations |
| `spec.ports[].portId` | string | Yes | Port identifier |
| `spec.ports[].vlan` | int | No | Access VLAN ID (1-4094) |
| `spec.ports[].trunkVlans` | []int | No | Trunk VLAN IDs |
| `spec.ports[].mtu` | int | No | MTU size (68-9216) |
| `spec.ports[].description` | string | No | Port description |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Registering`, `Ready`, or `Error` |
| `status.configuredPorts` | int | Number of configured ports |
| `status.lastConfigTime` | time | Last time the switch was configured |
| `status.error` | string | Error message, if any |

---

### HostReservation

Cross-namespace claiming of BareMetalHosts with label-based host selection for multi-tenant
allocation — a per-host claim, distinct from `ClusterClaim`'s cluster-level scheduling. Renamed
from `HostClaim` to avoid colliding with upstream baremetal-operator's own `HostClaim`
CAPI/CAPM3 controller — see `docs/guides/bmo-upstream-parity.md`.

**Resource:** `hostreservations.bmo.metal3.io` | **Short name:** `hr`

**Webhook:** Validating (HostReservationValidator)

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.matchLabels` | map[string]string | No | Label selector for matching BareMetalHosts |
| `spec.matchExpressions` | []LabelSelectorRequirement | No | Label selector expressions for matching BareMetalHosts |
| `spec.inNamespace` | string | No | Namespace where the target BareMetalHost resides |
| `spec.image` | object | No | Image to provision on the claimed host (`url`, `checksum`, `checksumType`, `format`, `authSecretName`) |
| `spec.userDataSecretRef` | object | No | Secret containing cloud-init user data |
| `spec.networkDataSecretRef` | object | No | Secret containing network data |
| `spec.metaDataSecretRef` | object | No | Secret containing metadata |
| `spec.customDeploy` | object | No | Custom deploy method (`method`) instead of standard image provisioning |
| `spec.failureDomain` | string | No | Failure domain to constrain host selection |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `status.phase` | string | `Pending`, `Bound`, `Released`, or `Failed` |
| `status.boundHost` | string | Name of the bound BareMetalHost (`namespace/name`) |
| `status.hardwareDataRef` | string | Reference to the HardwareData for the bound host |

---

## Dashboard REST — Fleet operations

All paths are under `/api/v1` on the dashboard binary (default `:8888`). Auth: `X-API-Key` / `Authorization: Bearer`. OpenAPI: [`docs/openapi/ironwolf-dashboard-api.yaml`](openapi/ironwolf-dashboard-api.yaml).

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/fleet/health` | Scored fleet grade (hosts, firmware, approvals) |
| GET | `/package/readiness` | Install / product-package checklist |
| POST | `/clusterclaims/dry-run` | Scheduling what-if for a claim shape |
| POST | `/capacity/plan` | Server-side capacity plan (same scoring) |
| POST | `/hosts/{ns}/{name}/quarantine` | Quarantine or clear quarantine |
| GET | `/firmware/compliance` | Fleet firmware compliance (`?format=csv`) |
| GET/POST | `/approvals` | List / create destructive-action approvals |
| POST | `/approvals/{id}/approve` | Approve (admin) |
| POST | `/approvals/{id}/reject` | Reject (admin) |
| POST | `/notify-prefs/test` | Test Slack/webhook from saved prefs |
| GET | `/metrics/history` | Sparklines history (`range=1h\|6h\|24h`) |
| GET | `/ws` | WebSocket (overview, host-state, approval, bootstrap-job) |

## Dashboard REST — Customer Portal

All paths are under `/api/v1` on the dashboard binary (default `:8888`). Auth: `X-API-Key` / `Authorization: Bearer`. Tenant-scoped keys (`key:role:namespace`) limit list/mutate to that namespace. OpenAPI: `docs/openapi/ironwolf-dashboard-api.yaml`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/usage/summary` | Bandwidth usage summary across hosts |
| GET/POST/PUT | `/invoices` | List / publish / replace tenant invoices |
| GET/PUT | `/cart` | Get or replace shopping cart |
| GET | `/cart/catalog` | Order catalog (ConfigMap or builtin SKUs) |
| POST | `/cart/checkout` | Checkout → create sales `SupportTicket` (no payment) |
| GET | `/storage/buckets` | List object storage buckets |
| GET | `/storage/buckets/{bucket}/objects` | List objects in a bucket |
| GET/PUT | `/ssh-keys` | Tenant SSH public key vault |
| GET/PUT | `/notify-prefs` | Tenant notification preferences |
| POST | `/notify-prefs/test` | Send test Slack/webhook from saved prefs |
| GET/PUT | `/system-status` | Platform status components (PUT admin/operator) |
| GET | `/security/scans` | External / Hadrian / bulletin scan findings |
| GET/PUT | `/config/branding` | Tenant/system branding |
| PUT | `/hosts/{ns}/{name}/alias` | Set BareMetalHost alias label |
| POST | `/hosts/{ns}/{name}/install` | Install/reinstall OS with cloud-init password |
| GET | `/hosts/{ns}/{name}/traffic` | Host bandwidth time series |
| GET | `/hosts/{ns}/{name}/vpn-profile` | Download OpenVPN profile |
| GET | `/hostipaddresses` | List `HostIPAddress` resources |
| GET/POST | `/availabilitymonitors` | List / create monitors |
| GET/POST | `/supporttickets` | List / create support tickets |

Operator env and ConfigMap details: `docs/guides/customer-portal-ops.md`.
