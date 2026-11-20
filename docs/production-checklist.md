# Production Checklist

Operator runbook (ops gates for a live cluster) — not a list of unfinished product code. Product package status: `GET /api/v1/package/readiness`; Helm seeds in [`values-package.yaml`](../values-package.yaml) (see [helm-values.md](helm-values.md)).

Everything to verify before running IronWolf in production.

## Pre-Deployment

### Cluster Requirements

- [ ] Kubernetes 1.32+ with RBAC enabled
- [ ] Metal3 Bare Metal Operator installed and healthy
- [ ] Ironic deployed with TLS and basic-auth enabled
- [ ] cert-manager installed (for webhook and Ironic TLS)
- [ ] Provisioning network configured (DHCP, PXE, TFTP)
- [ ] BMC management network reachable from Ironic pod

### Licensing

- [ ] Commercial (not trial) licence key obtained from `sales@zyvor.dev`
- [ ] Key stored as a Secret, not `operator.licenseKey` inline in values (`operator.licenseKeySecret`) — every replica reads the same key
  ```bash
  kubectl create secret generic ironwolf-license -n ironwolf-system \
    --from-literal=license-key="<your-key>"
  ```
- [ ] Expiry date tracked — the manager logs a warning once `IRONWOLF_LICENSE_KEY` has 5 days or fewer remaining

### Image Configuration

- [ ] Use pinned image tags (not `latest`)
  ```yaml
  image:
    tag: "v1.0.0"
    pullPolicy: IfNotPresent
  ```
- [ ] Images pulled and verified in a staging environment first
- [ ] Private registry credentials configured if needed (`imagePullSecrets`)

## Deployment

### Product package

- [ ] Install with `-f values-production.yaml -f values-package.yaml` (or equivalent `--set productPackage.*.enabled`)
- [ ] `GET /api/v1/package/readiness` shows required checks green
- [ ] Branding ConfigMap `ironwolf-branding` present (or set in Settings)
- [ ] Notify prefs configured; optional `POST /api/v1/notify-prefs/test`
- [ ] Optional: `dashboard.requireApprovals: true` + Slack webhook for approval notify

### High Availability

- [ ] `replicaCount: 2` with leader election enabled
- [ ] Pod anti-affinity to spread across nodes:
  ```yaml
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: ironwolf
          topologyKey: kubernetes.io/hostname
  ```
- [ ] Pod disruption budget enabled. PDB is **disabled by default** because the default `replicaCount` is 1 and a PDB with `minAvailable: 1` on a single replica blocks voluntary evictions (node drains, cluster upgrades). For production with `replicaCount >= 2`, enable it explicitly:
  ```yaml
  replicaCount: 2
  pdb:
    enabled: true
    minAvailable: 1
  ```
- [ ] Resource requests and limits set appropriately:
  ```yaml
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 2000m
      memory: 1Gi
  ```

### Security

- [ ] NetworkPolicies enabled (`networkPolicy.enabled: true`)
- [ ] Dashboard API key set and rotated (`dashboard.auth.apiKey`)
- [ ] Multi-user RBAC configured (`"admin-key:admin,ops-key:operator,viewer-key:viewer"`)
- [ ] Dashboard HA: `replicaCount >= 2` requires `dashboard.redisUrl` (jobs + WebSocket fan-out); sticky sessions alone are not enough for approvals/jobs
- [ ] Dashboard persistence enabled (`dashboard.persistence`) so audit/job/approval JSON can write under `readOnlyRootFilesystem`
- [ ] Optional: `dashboard.requireApprovals: true` for two-person destructive actions
- [ ] Dashboard behind Ingress with TLS termination
- [ ] No default/weak BMC credentials in Secrets
- [ ] SMTP credentials stored securely (Kubernetes Secrets, not env vars in manifests)
- [ ] Webhook TLS certificates managed by cert-manager
- [ ] RBAC ClusterRole uses least-privilege (default role is already scoped)

### TLS

- [ ] Ironic API using TLS with valid certificates
- [ ] Dashboard Ingress with TLS:
  ```yaml
  dashboard:
    ingress:
      enabled: true
      tls:
      - secretName: ironwolf-tls
        hosts:
        - ironwolf.example.com
  ```
- [ ] Webhook server using TLS (auto-configured by controller-runtime)
- [ ] Internal communication encrypted (metrics endpoint with TLS if exposed externally)

### Webhook Configuration

IronWolf deploys 25 validating webhooks for its CRD types. Webhook configuration is critical for production stability.

- [ ] **`webhook.enabled: true` set**, and the `<release>-webhook-service` Service actually exists:
  ```bash
  kubectl get svc -n ironwolf-system | grep webhook-service
  ```
  This Service (`templates/webhook-service.yaml`) is what every validating
  webhook's `clientConfig.service` and the CRD conversion webhook (below) call — without it every
  admission/conversion request fails (or is silently skipped under `failurePolicy: Ignore`).

- [ ] **failurePolicy reviewed per webhook.** IronWolf uses two failure policies depending on the resource's criticality:
  - **`Fail`** for core resources where invalid data could cause infrastructure damage: ClusterClaim, FirmwareProfile, HardwarePool, MaintenanceWindow, PowerPolicy, RemediationStrategy, SecurityPolicy. If the webhook is unreachable, the API server rejects the request -- this is safer for resources that directly control hardware.
  - **`Ignore`** for advisory resources where a missed validation is less risky than blocking the API: AlertPolicy, EnergyPolicy, HardwareBackup, NetworkProfile, RestoreJob, TelemetryPolicy. If the webhook is unreachable, the request proceeds without validation.
  - When deploying in environments with strict availability requirements, verify that all `Fail`-policy webhooks have healthy backing pods. A crashed webhook pod with `failurePolicy: Fail` will block all CREATE/UPDATE operations on those resource types cluster-wide.
- [ ] **timeoutSeconds is set to 5s** (the default for all IronWolf webhooks). Considerations:
  - Verify the webhook service pod is colocated or has low-latency network access to the API server.
  - Monitor API server webhook metrics: `apiserver_admission_webhook_rejection_count` and `apiserver_admission_webhook_fail_open_count`.
  - Increase to 10s only if you observe timeout-related rejections, as higher values increase API latency for affected resources.
  - Never set above 30s; the API server has a hard limit and will reject the configuration.
- [ ] **cert-manager annotation present** on the ValidatingWebhookConfiguration:
  ```yaml
  metadata:
    annotations:
      cert-manager.io/inject-ca-from: ironwolf-system/ironwolf-serving-cert
  ```
  Verify that the Certificate and Issuer resources exist in `ironwolf-system` and that the certificate is not expired:
  ```bash
  kubectl get certificate -n ironwolf-system
  kubectl get secret ironwolf-serving-cert -n ironwolf-system -o jsonpath='{.data.tls\.crt}' \
    | base64 -d | openssl x509 -noout -dates
  ```
- [ ] **Webhook service is reachable** from the API server. In clusters with restrictive NetworkPolicies, ensure the API server can reach port 9443 on the webhook service pods. The default NetworkPolicy allows inbound traffic on 9443.

### CRD Conversion Webhook

Every `bmo.metal3.io` CRD is stored as `v1` (an opaque `json.RawMessage` hub type) with `v1beta1`
still served for reads/writes — see `docs/guides/api-versioning.md`. **If the conversion webhook
isn't reachable, writes silently succeed but drop the entire spec/status.** This is not a
validation failure that shows up as an error; a `kubectl apply` returns success and `kubectl get`
shows an empty `spec: {}`.

- [ ] `operator.conversionWebhooks.enabled: true` set (`values-production.yaml` sets this) — maps
  to `IRONWOLF_ENABLE_CONVERSION_WEBHOOKS`, requires `webhook.enabled: true` too
- [ ] Every CRD has `spec.conversion.strategy: Webhook` with a populated `caBundle` (cert-manager
  injects this via the `cert-manager.io/inject-ca-from` annotation on the CRD itself, separate
  from the webhook configuration's annotation above):
  ```bash
  kubectl get crd clusterclaims.bmo.metal3.io -o jsonpath='{.spec.conversion}'
  ```
  If `spec.conversion` is missing or `caBundle` is empty, run `make add-crd-conversion-webhook`
  (chained into `make manifests`) and re-apply the CRDs — `controller-gen` does not generate this
  block on its own.
- [ ] **Round-trip verified** after any CRD or webhook change:
  ```bash
  kubectl create -f - <<'EOF'
  apiVersion: bmo.metal3.io/v1beta1
  kind: ClusterClaim
  metadata: {name: conversion-smoke-test, namespace: default}
  spec: {controlPlaneCount: 1, workerCount: 2, schedulingStrategy: binpack}
  EOF
  kubectl get clusterclaim conversion-smoke-test -o jsonpath='{.spec.controlPlaneCount}'  # must print 1, not empty
  kubectl delete clusterclaim conversion-smoke-test
  ```

### SSRF Protection

IronWolf includes built-in SSRF (Server-Side Request Forgery) protection for outbound HTTP calls (notification webhooks, external integrations). This prevents attackers from using IronWolf as a proxy to reach internal services.

- [ ] **Understand what is blocked.** The SSRF-safe HTTP client (`pkg/utils/network_utils.go`) blocks connections to any IP in private or reserved ranges:
  - IPv4: `0.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16`, `100.64.0.0/10` (CGNAT)
  - IPv6: `::1/128` (loopback), `fc00::/7` (unique local), `fe80::/10` (link-local)
  - IPv4-mapped IPv6 addresses (e.g., `::ffff:10.0.0.1`) are normalized to IPv4 before checking, so they are also blocked.
- [ ] **DNS rebinding mitigation.** The transport uses `DisableKeepAlives: true` and resolves DNS at dial time, verifying every resolved IP before connecting. This prevents attacks where a DNS record initially points to a public IP and then changes to a private one on subsequent connections.
- [ ] **No redirect following.** The HTTP client returns `http.ErrUseLastResponse` on any redirect, preventing open-redirect-based SSRF bypass where an attacker uses a public URL that 302-redirects to an internal service.
- [ ] **Connection timeout.** The dial timeout is 5 seconds and the overall HTTP client timeout is 10 seconds, preventing slowloris-style resource exhaustion.
- [ ] **Review notification webhook URLs.** If AlertPolicy or other notification resources reference webhook URLs, verify those URLs point to external services. Webhook URLs targeting internal cluster services (e.g., `http://internal-service.namespace.svc.cluster.local`) will be blocked by SSRF protection because they resolve to pod/cluster IPs in private ranges. If you need to reach in-cluster services for notifications, deploy a sidecar proxy or use a dedicated egress gateway with a public-facing IP.

## Monitoring

### Prometheus

IronWolf exposes metrics on `:8080/metrics`. Enable the ServiceMonitor for automatic scraping:

```yaml
serviceMonitor:
  enabled: true
  interval: 30s
  labels:
    release: prometheus  # Match your Prometheus serviceMonitorSelector
```

Alternatively, configure scraping manually:

```yaml
# ServiceMonitor (if using prometheus-operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ironwolf
  namespace: ironwolf-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ironwolf
  endpoints:
  - port: metrics
    interval: 30s
```

Key metrics to monitor (all use the `bmo_` prefix):

| Metric | Type | Description |
|--------|------|-------------|
| `bmo_baremetal_hosts` | Gauge | Current number of bare metal hosts by namespace and status (`online`, `offline`, `error`) |
| `bmo_baremetal_host_power_state` | Gauge | Power state of each bare metal host (`on`, `off`) with host and namespace labels |
| `bmo_cluster_claim_status` | Gauge | Status of cluster claims by namespace, name, and phase |
| `bmo_cluster_claim_duration_seconds` | Gauge | Duration of cluster claims in seconds (time from creation to last transition) |
| `bmo_host_health_status` | Gauge | Health status of hosts (1 = healthy, 0 = unhealthy) |
| `bmo_remediation_attempts_total` | Counter | Total remediation attempts |
| `bmo_maintenance_window_active` | Gauge | Whether a maintenance window is currently active (1 = active, 0 = inactive) |
| `bmo_firmware_update_status` | Gauge | Firmware update status by host |
| `bmo_power_consumption_watts` | Gauge | Power consumption in watts per host |

**Important:** All IronWolf metrics use the `bmo_` prefix (not `ironwolf_`). This aligns with the Metal3 Bare Metal Operator naming convention. If you have existing dashboards or alerts referencing `ironwolf_*` metrics, they must be updated to use `bmo_*`.

### Alerting Rules

Enable the built-in PrometheusRule via the Helm chart:

```yaml
prometheusRule:
  enabled: true
  additionalLabels:
    release: prometheus  # Match your Prometheus ruleSelector
```

Or define custom rules. All metric references must use the `bmo_` prefix:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ironwolf-alerts
  namespace: ironwolf-system
spec:
  groups:
  - name: ironwolf.rules
    rules:
    # Controller availability
    - alert: IronWolfControllerDown
      expr: up{job="ironwolf"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "IronWolf controller manager is down"
        description: "The IronWolf controller manager has been unreachable for 5 minutes."

    # Host health
    - alert: BareMetalHostInErrorState
      expr: bmo_baremetal_hosts{status="error"} > 0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Bare metal host(s) in error state"
        description: "{{ $value }} host(s) in namespace {{ $labels.namespace }} have been in error state for 10 minutes."

    - alert: BareMetalHostHealthDegraded
      expr: bmo_host_health_status == 0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Bare metal host health check failing"
        description: "Host health status has been unhealthy for 10 minutes."

    # Cluster claims
    - alert: ClusterClaimStuckPending
      expr: bmo_cluster_claim_status{phase="Pending"} == 1
      for: 30m
      labels:
        severity: warning
      annotations:
        summary: "ClusterClaim stuck in Pending"
        description: "ClusterClaim {{ $labels.name }} in {{ $labels.namespace }} has been Pending for 30 minutes."

    - alert: ClusterClaimSlowFulfillment
      expr: bmo_cluster_claim_duration_seconds > 3600
      for: 0m
      labels:
        severity: info
      annotations:
        summary: "ClusterClaim took over 1 hour to fulfill"
        description: "ClusterClaim {{ $labels.name }} in {{ $labels.namespace }} took {{ $value | humanizeDuration }} to complete."

    # Power state
    - alert: BareMetalHostPowerStateUnknown
      expr: bmo_baremetal_host_power_state{state="unknown"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Bare metal host power state unknown"
        description: "Host {{ $labels.host }} in {{ $labels.namespace }} has been in unknown power state for 10 minutes."

    # Remediation
    - alert: HighRemediationRate
      expr: rate(bmo_remediation_attempts_total[15m]) > 0.1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High remediation rate detected"
        description: "Remediation attempts are occurring at a high rate, which may indicate a systemic hardware or network issue."

    # Maintenance
    - alert: MaintenanceWindowActive
      expr: bmo_maintenance_window_active == 1
      for: 0m
      labels:
        severity: info
      annotations:
        summary: "Maintenance window is currently active"
        description: "A maintenance window is active. Automated operations may be deferred."

    # Power consumption
    - alert: HighPowerConsumption
      expr: bmo_power_consumption_watts > 1000
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "High power consumption detected"
        description: "Host {{ $labels.host }} is consuming {{ $value }}W for over 15 minutes."

    # Firmware
    - alert: FirmwareUpdateStuck
      expr: bmo_firmware_update_status{status="in_progress"} == 1
      for: 60m
      labels:
        severity: warning
      annotations:
        summary: "Firmware update stuck"
        description: "A firmware update has been in progress for over 1 hour."
```

### Grafana

Enable Grafana dashboard provisioning via the Helm chart:

```yaml
grafana:
  dashboards:
    enabled: true
    labels:
      grafana_dashboard: "1"   # Required for Grafana sidecar discovery
    annotations:
      grafana_folder: "Infrastructure"  # Optional: target Grafana folder
```

Set the `GRAFANA_URL` environment variable on the dashboard pod to link to your Grafana instance:

```yaml
env:
- name: GRAFANA_URL
  value: "https://grafana.example.com/d/ironwolf"
```

### Log Aggregation

Controller logs are structured JSON (via zap). Ship to your log aggregation system:

```bash
# Useful log queries
kubectl logs -n ironwolf-system -l app.kubernetes.io/name=ironwolf \
  | jq 'select(.level == "error")'

# Watch for reconciliation errors
kubectl logs -n ironwolf-system -l app.kubernetes.io/name=ironwolf \
  | jq 'select(.msg | test("reconcile.*error"; "i"))'
```

## Operational Procedures

### Upgrade Procedure

```bash
# 1. Check current version
helm list -n ironwolf-system

# 2. Review release notes for breaking changes

# 3. Backup CRD resources
make backup
# OR
# Backup CRDs/state with your platform backup tool or Velero

# 4. Update CRDs first (Helm doesn't manage CRD upgrades)
kubectl apply -f config/base/crds/bases/

# 5. Upgrade Helm release
helm upgrade ironwolf oci://ghcr.io/zyvorai/charts/ironwolf \
  --namespace ironwolf-system \
  --set image.tag=v1.1.0 \
  --set dashboard.image.tag=v1.1.0

# 6. Verify
kubectl get pods -n ironwolf-system
kubectl get pods -n ironwolf-system
```

### Rollback

```bash
helm rollback ironwolf --namespace ironwolf-system
```

### Backup and Restore

```bash
# Backup all CRD resources
# Backup CRDs/state with your platform backup tool or Velero
# Creates: backup-<timestamp>.tar.gz

# Restore from backup
# Restore from your platform backup / Velero restore /path/to/backup-<timestamp>.tar.gz
```

Schedule regular backups via CronJob or the `HardwareBackup` CRD.

### Scaling

The controller manager is single-leader -- scaling `replicaCount` beyond 2 adds standby replicas but doesn't increase throughput. For large fleets:

- Increase concurrent reconciles via `IRONWOLF_CONCURRENCY` env var (default: `min(NumCPU, 8)`)
- Increase memory limits if managing hundreds of BareMetalHosts
- Consider namespace-scoped deployments for multi-tenant isolation

For the dashboard, enable autoscaling if you expect variable load:

```yaml
dashboard:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
```

### Multi-Tenancy

To restrict IronWolf to a single namespace:

```yaml
# Set WATCH_NAMESPACE on the controller
env:
- name: WATCH_NAMESPACE
  value: "team-a"
```

Deploy separate IronWolf instances per namespace for full isolation.

## Post-Deployment Verification

```bash
# Full health check
kubectl get pods -n ironwolf-system

# Post-install self-test
kubectl get pods -n ironwolf-system

# Verify all CRDs
kubectl get crds | grep -E 'bmo\.metal3\.io|image\.ironwolf\.io' | wc -l   # Should match config/base/crds/bases (27)

# Verify controller is running
kubectl get pods -n ironwolf-system

# Verify webhooks are registered and have valid CA bundles
kubectl get validatingwebhookconfigurations | grep ironwolf
kubectl get validatingwebhookconfigurations ironwolf-validating-webhook-configuration \
  -o jsonpath='{.webhooks[*].name}' | tr ' ' '\n'

# Verify metrics endpoint is serving
kubectl port-forward -n ironwolf-system svc/ironwolf-metrics 8080:8080 &
curl -s http://localhost:8080/metrics | grep bmo_

# Verify dashboard
curl -sk https://<dashboard>/api/v1/health

# Test a sample resource
kubectl apply -f config/samples/clusterclaim_sample.yaml
kubectl get cc -w
```

## Checklist Summary

| Category | Items | Status |
|----------|-------|--------|
| Cluster prerequisites | K8s 1.32+, Metal3, Ironic, cert-manager | |
| Images | Pinned tags, pullPolicy: IfNotPresent | |
| HA | 2 replicas, anti-affinity, leader election, PDB enabled | |
| Security | NetworkPolicy, API keys, TLS, RBAC, no default creds | |
| Webhooks | failurePolicy reviewed, TLS certs valid, timeouts verified | |
| SSRF protection | Notification URLs reviewed, private IP ranges blocked | |
| Monitoring | ServiceMonitor, PrometheusRule, Grafana dashboards (all `bmo_` prefix) | |
| Logging | Structured logs shipped to aggregator | |
| Backups | Scheduled via HardwareBackup CRD or cron | |
| Upgrade plan | CRD-first upgrade, rollback tested | |
