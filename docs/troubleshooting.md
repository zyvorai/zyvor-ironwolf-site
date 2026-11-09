# Troubleshooting Guide

## Quick Diagnostics

```bash
# One-liner status check
./scripts/health-check.sh --json | python3 -m json.tool

# Controller logs (last 100 lines)
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf --tail=100

# All events sorted by time
kubectl get events -n ironwolf-system --sort-by='.lastTimestamp'

# All CRD resources at a glance
for kind in cc fwp mw pp hwp rs ap np sp ep tp hwb rj; do
  echo "=== ${kind} ==="
  kubectl get "${kind}" --all-namespaces 2>/dev/null || echo "(CRD not found)"
done

# Post-install verification
make selftest

# Collect debug bundle for deeper analysis
make debug-bundle
```

## Common Issues

### Controller Pod Not Starting

**Symptoms:** Pod stuck in `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff`

```bash
# Check pod status
kubectl describe pod -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf

# Check events
kubectl get events -n ironwolf-system \
  --field-selector reason=Failed
```

| Cause | Fix |
|-------|-----|
| ImagePullBackOff | Verify image exists: `docker pull <image>` |
| CrashLoopBackOff | Check logs: `kubectl logs -n ironwolf-system -l app.kubernetes.io/name=ironwolf --previous` |
| `IronWolf licence error: No IronWolf licence key found` in logs | Create the licence Secret and point the chart at it: `kubectl create secret generic ironwolf-license -n ironwolf-system --from-literal=license-key="<key>"` then `helm upgrade ironwolf ... --set operator.licenseKeySecret=ironwolf-license`. See [Licensing](getting-started.md#licensing). |
| Insufficient resources | Reduce resource requests or scale up nodes |
| RBAC errors | Verify ClusterRole/ClusterRoleBinding: `kubectl get clusterrolebinding ironwolf-manager-rolebinding` |

### CRDs Not Working

**Symptoms:** `error: the server doesn't have a resource type "clusterclaims"`

```bash
# Check all 27 CRDs are installed
kubectl get crds | grep bmo.metal3.io

# Expected output: 27 CRDs
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

# Re-install CRDs
kubectl apply -f config/base/crds/bases/

# Check CRD conditions
kubectl get crd clusterclaims.bmo.metal3.io -o yaml | grep -A5 conditions
```

### ClusterClaim Stuck in Partial

**Symptoms:** ClusterClaim stays in `Partial` phase

```bash
# Check claim status
kubectl get clusterclaim <name> -o yaml

# Check available hosts
kubectl get baremetalhosts -o wide

# Check hardware match
kubectl get baremetalhosts -o json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for h in data.get('items', []):
    hw = h.get('status', {}).get('hardwareDetails', {})
    if hw:
        cpu = hw.get('cpu', {}).get('count', 0)
        ram = hw.get('ramMebibytes', 0) // 1024
        print(f\"{h['metadata']['name']}: CPU={cpu}, RAM={ram}GB, Consumer={h['spec'].get('consumerRef', 'none')}\")
"
```

**Common causes:**
- Not enough available (unclaimed) hosts
- Hardware requirements too strict (reduce minCPUCores/minRAMGB/minDiskGB)
- Hosts have errors: `kubectl get bmh -o jsonpath='{range .items[*]}{.metadata.name}: {.status.errorMessage}{"\n"}{end}'`
- Hosts lack hardware details (need inspection first)

### FirmwareProfile Not Applying

**Symptoms:** FirmwareProfile shows matching hosts but pendingHosts stays non-zero

```bash
# Check profile status
kubectl get firmwareprofile <name> -o yaml

# Check individual host firmware status
kubectl get fwp <name> -o jsonpath='{.status.hostStatuses}' | python3 -m json.tool

# Check BMC connectivity for affected hosts
kubectl get baremetalhosts -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.bmc.address}{"\n"}{end}'
```

**Common causes:**
- BMC unreachable (check network connectivity to BMC addresses)
- Invalid firmware URL or checksum
- Host vendor not supported by BMC reconciler
- maxConcurrent limit reached (check status.pendingHosts vs spec.maxConcurrent)

### MaintenanceWindow Not Activating

**Symptoms:** MaintenanceWindow stays in `Scheduled` phase past startTime

```bash
# Check window status
kubectl get maintenancewindow <name> -o yaml

# Check if startTime has passed
kubectl get mw <name> -o jsonpath='{.spec.startTime}'

# Check controller logs for maintenance window reconciler
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf --tail=200 | grep -i maintenance
```

**Common causes:**
- startTime in the future
- No hosts match the hostSelector
- drainBeforeMaintenance is true but drain is failing
- Controller not running or not the leader

### PowerPolicy Not Enforcing

**Symptoms:** Hosts not transitioning to expected power state

```bash
# Check policy status
kubectl get powerpolicy <name> -o yaml

# Check per-host power status
kubectl get pp <name> -o jsonpath='{.status.hostStatuses}' | python3 -m json.tool

# For Scheduled mode, verify cron expressions
kubectl get pp <name> -o jsonpath='{.spec.powerOnSchedule} / {.spec.powerOffSchedule}'
```

**Common causes:**
- For Scheduled mode: missing powerOnSchedule or powerOffSchedule
- For OnDemand mode: idleTimeout not set
- BMC communication failure
- Host in error state prevents power operations

### AlertPolicy Not Firing

**Symptoms:** Conditions are met but no notifications sent

```bash
# Check policy status
kubectl get alertpolicy <name> -o yaml

# Check if alerts are enabled
kubectl get ap <name> -o jsonpath='{.spec.enabled}'

# Check cooldown period
kubectl get ap <name> -o jsonpath='{.spec.cooldown}'
kubectl get ap <name> -o jsonpath='{.status.lastAlertTime}'

# Check notification channel reachability
curl -X POST <webhook-url> -d '{"test": true}'
```

**Common causes:**
- `spec.enabled` is false
- Alert is in cooldown period (check lastAlertTime + cooldown)
- Notification channel URL unreachable
- No hosts match the hostSelector
- Condition type does not match any actual host state

### Email Notifications Not Sending

**Symptoms:** AlertPolicy fires (status shows activeAlerts > 0) but email channel produces errors in controller logs

```bash
# Check controller logs for SMTP errors
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf --tail=100 | grep -i smtp

# Verify SMTP environment variables are set on the controller pod
kubectl exec -n ironwolf-system \
  deploy/ironwolf-controller-manager -- env | grep SMTP
```

**Common causes:**

| Cause | Fix |
|-------|-----|
| `SMTP_HOST` not set | Set the env var on the controller deployment |
| `SMTP_FROM` not set | Set the sender address env var |
| SMTP server unreachable | Verify network connectivity to SMTP_HOST:SMTP_PORT (default 587) |
| Authentication failed | Check `SMTP_USERNAME` and `SMTP_PASSWORD`; omit both for internal relays without auth |
| TLS handshake failure | Ensure SMTP server supports STARTTLS on the configured port |
| Empty `emails[]` in channel | At least one recipient address is required |

### NetworkProfile in Error State

**Symptoms:** NetworkProfile phase is `Error`

```bash
# Check profile status and error message
kubectl get networkprofile <name> -o yaml

# Check switch connectivity
kubectl get np <name> -o jsonpath='{.status.message}'

# Verify switch credentials secret exists
kubectl get secret -n <namespace> <credentials-ref-name>
```

**Common causes:**
- Switch management IP unreachable
- Invalid credentials in referenced secret
- Switch vendor/protocol mismatch
- VLAN ID conflicts
- BGP peer unreachable or AS number mismatch

### SecurityPolicy Showing NonCompliant

**Symptoms:** SecurityPolicy phase is `NonCompliant` with violations

```bash
# Check violations
kubectl get securitypolicy <name> -o jsonpath='{.status.violations}' | python3 -m json.tool

# Check compliance status per host
kubectl get sp <name> -o yaml

# Check specific violation details
kubectl get sp <name> -o jsonpath='{range .status.violations[*]}{.host}: {.rule} ({.severity}) - {.message}{"\n"}{end}'
```

**Common causes:**
- Secure boot not enabled on host BIOS
- TPM not present or not enabled
- BMC using HTTP instead of HTTPS
- TLS version below minimum requirement
- Password policy not met
- Compliance scan found CIS/NIST violations

**Resolution steps:**
1. For `autoRemediate: true`, the controller will attempt to fix violations automatically
2. For manual remediation, address each violation listed in `status.violations`
3. Re-run compliance scan: delete and recreate the SecurityPolicy, or wait for next scheduled scan

### EnergyPolicy Not Saving Power

**Symptoms:** estimatedSavingsWatts is 0 or hosts not being suspended

```bash
# Check policy status
kubectl get energypolicy <name> -o yaml

# Check if schedules are active
kubectl get ep <name> -o jsonpath='{.spec.schedules}' | python3 -m json.tool

# Check consolidation thresholds vs actual utilization
kubectl get ep <name> -o jsonpath='{.spec.consolidation}'
```

**Common causes:**
- Schedule timezone mismatch (ensure timezone string is valid)
- Consolidation thresholds too low (hosts never drop below threshold)
- Carbon intensity API endpoint unreachable
- Dynamic capping minCapWatts equals maxCapWatts
- Hosts are claimed by ClusterClaims (energy policies may not override claim ownership)

### TelemetryPolicy Not Collecting

**Symptoms:** monitoredHosts is 0 or lastCollectionTime not updating

```bash
# Check policy status
kubectl get telemetrypolicy <name> -o yaml

# Check exporter statuses
kubectl get tp <name> -o jsonpath='{.status.exporterStatuses}' | python3 -m json.tool

# Check for collection errors in logs
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf --tail=200 | grep -i telemetry
```

**Common causes:**
- No hosts match the selector
- BMC unreachable for IPMI/Redfish telemetry queries
- Exporter endpoint unreachable or misconfigured
- Collection interval too short causing timeouts
- Anomaly detection model not yet trained (check trainingWindow)

### HardwareBackup Failing

**Symptoms:** HardwareBackup phase is `Failed`

```bash
# Check backup status
kubectl get hardwarebackup <name> -o yaml

# Check last backup attempt
kubectl get hwb <name> -o jsonpath='{.status.phase}: {.status.lastBackupTime}'

# Check verification status
kubectl get hwb <name> -o jsonpath='{.status.verificationStatus}'
```

**Common causes:**
- Storage endpoint unreachable (S3/NFS)
- S3 bucket does not exist or permissions insufficient
- KMS key ID invalid for encryption
- BMC unreachable for config extraction
- Disk space exhausted on local storage
- Backup retention policy deleting backups faster than creating them

### RestoreJob Stuck or Failed

**Symptoms:** RestoreJob phase is `Failed` or stays in `Running`

```bash
# Check restore job status
kubectl get restorejob <name> -o yaml

# Check which components failed
kubectl get rj <name> -o jsonpath='{.status.failedComponents}'

# Check status message
kubectl get rj <name> -o jsonpath='{.status.message}'
```

**Common causes:**
- Referenced HardwareBackup does not exist
- Backup data corrupted or storage unreachable
- Target host is not in a state that allows configuration changes
- Component type mismatch (backup contains different components than requested)
- rollbackOnFailure is true and some components fail, causing rollback to `RolledBack` phase

### Boolean Fields No Longer Omitted When False

**Symptoms:** After upgrading, policies that previously appeared to ignore
`enforceOnDrift`, `gracefulShutdown`, or `drainBeforeMaintenance` now enforce
those settings.

These boolean fields are no longer omitted from the spec when their value is
`false`. In previous versions, an unset (nil) field and an explicit `false`
were treated identically -- often as "not configured". The current version
properly distinguishes between the two states, so an explicit `false` is now
respected.

**Resolution:** Review your CRD manifests and ensure the boolean values match
your intent. If you relied on omitting the field to mean "disabled", the
behavior is unchanged. If you explicitly set `false`, it will now be honored.

### 400 Bad Request on Boot Mode Change

**Symptoms:** Updating a host's boot mode returns `400 Bad Request` from the
admission webhook.

The valid boot mode values are:

- `UEFI`
- `UEFISecureBoot`
- `legacy`

Any other value is rejected. Check for typos (e.g., `uefi` instead of `UEFI`,
or `Secure` instead of `UEFISecureBoot`).

### 403 Forbidden on /api/v1/audit

**Symptoms:** `GET /api/v1/audit` returns `403 Forbidden`.

The audit endpoint now requires an API key with the **admin** role. Generate an
admin-level key and pass it via the `X-API-Key` or `Authorization: Bearer`
header.

```bash
curl -H "X-API-Key: <admin-key>" http://localhost:8888/api/v1/audit
```

### FirmwareProfile or NetworkProfile Deletion Leaves Stale Annotations

**Symptoms:** After deleting a FirmwareProfile or NetworkProfile, host
annotations referencing the deleted resource remain.

This is no longer expected behavior. The controller now installs finalizers on
FirmwareProfile and NetworkProfile resources and automatically cleans up
annotations from all associated hosts before the resource is removed. If stale
annotations persist, check the controller logs for errors during finalizer
processing:

```bash
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf --tail=200 | grep -i finalizer
```

### SMTP STARTTLS Warnings

**Symptoms:** Controller logs show a warning about STARTTLS when sending email
notifications, but emails are still delivered.

When `SMTP_USERNAME` and `SMTP_PASSWORD` are configured, the controller
attempts to upgrade the connection with STARTTLS. If the SMTP server does not
advertise STARTTLS support, a warning is logged but the connection proceeds
without TLS. This is expected for internal relay servers that do not require
encryption.

If credentials are configured and you require encrypted transport, ensure the
SMTP server supports STARTTLS on the configured port (default 587).

### Leader Election Issues

**Symptoms:** Multiple controllers running, or none active

```bash
# Check lease
kubectl get lease ironwolf-leader -n ironwolf-system -o yaml

# Check which pod holds the lease
kubectl get lease ironwolf-leader -n ironwolf-system \
  -o jsonpath='{.spec.holderIdentity}'
```

### Webhook Failures

**Symptoms:** `Internal error occurred: failed calling webhook`

```bash
# Check webhook configurations
kubectl get validatingwebhookconfigurations | grep bmo
kubectl get mutatingwebhookconfigurations | grep bmo

# Check webhook service
kubectl get svc -n ironwolf-system | grep webhook

# Check TLS certificates
kubectl get secret -n ironwolf-system | grep webhook
```

Every `bmo.metal3.io` enterprise CRD with a `v1` hub version has a validating webhook — 17 in
total (`kubectl get validatingwebhookconfigurations ironwolf-validating-webhook-configuration -o
jsonpath='{.webhooks[*].name}'`). See `docs/production-checklist.md` for which use
`failurePolicy: Fail` vs `Ignore`. The `HostIPAddress`, `AvailabilityMonitor`, and `SupportTicket`
Customer Portal CRDs do not have admission webhooks.

**A different, silent failure mode:** if `spec.conversion` on a CRD is missing or its webhook is
unreachable while `operator.conversionWebhooks.enabled: true`, writes to that CRD don't error at
all — `kubectl apply`/`create` succeeds but the stored object's `spec`/`status` come back empty.
This is not "webhook failures" in the loud sense above; see the **CRD Conversion Webhook** section
of `docs/production-checklist.md` for the round-trip check that catches it.

### High Memory Usage

```bash
# Check current memory
kubectl top pods -n ironwolf-system

# Check for memory leaks in logs
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf | grep -i "memory\|oom"

# Increase limits if needed
kubectl patch deployment ironwolf-controller-manager \
  -n ironwolf-system \
  --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"512Mi"}]'
```

## Web Dashboard Troubleshooting

### Dashboard Not Loading

**Symptoms:** Browser shows blank page or connection refused on port 8888

```bash
# Check if ironwolf-web is running
ps aux | grep ironwolf-web

# Check if port 8888 is bound
ss -tlnp | grep 8888

# Check dashboard logs
# If running as a pod:
kubectl logs -n ironwolf-system -l app=ironwolf-web

# Verify static files exist
ls web/dashboard/dist/index.html
```

**Common causes:**
- `--static-dir` flag points to wrong directory (must contain `dist/index.html`)
- Frontend not built (run `make web-build` first)
- Port 8888 already in use by another process
- kubeconfig invalid or cluster unreachable

### API Key Authentication Errors

**Symptoms:** API returns 401 Unauthorized

```bash
# Test with API key header
curl -H "X-API-Key: <your-key>" http://localhost:8888/api/overview

# Check if API key is set
# The --api-key flag must match what the client sends in X-API-Key header
```

### WebSocket Not Connecting

**Symptoms:** Dashboard shows stale data, no live updates

```bash
# Test WebSocket endpoint (auth is sent via Sec-WebSocket-Protocol subprotocol)
curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: test" \
  -H "Sec-WebSocket-Protocol: v1.bmo.metal3.io, auth.YOUR_API_KEY" \
  http://localhost:8888/api/v1/ws

# Check browser console for WebSocket errors
# Common: Mixed content (HTTPS page, WS endpoint) - use WSS
```

**Common causes:**
- Proxy or load balancer not configured for WebSocket upgrade
- API key not sent via `Sec-WebSocket-Protocol` subprotocol (the dashboard handles this automatically; custom clients must send `auth.<token>` as a subprotocol)
- Proxy stripping `Sec-WebSocket-Protocol` header — ensure proxy forwards all WebSocket headers
- Firewall blocking WebSocket connections
- `IRONWOLF_MAX_WS_CLIENTS` limit reached (default: 100) — check server logs for "too many WebSocket connections"

### Audit Log Not Recording

**Symptoms:** Audit page is empty despite mutations

```bash
# Check audit endpoint
curl http://localhost:8888/api/audit

# Audit logs are in-memory by default
# Restarting ironwolf-web clears the audit log
```

## Debug Mode

### Enable Verbose Logging

```bash
# Restart with debug logging
kubectl set env deployment/ironwolf-controller-manager \
  -n ironwolf-system LOG_LEVEL=debug

# Watch debug logs
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf -f
```

### Collect Debug Bundle

```bash
# Collect comprehensive debug information
make debug-bundle

# Or use the script directly
./scripts/collect-debug-bundle.sh
```

The bundle includes logs, pod descriptions, CRD resources, events, RBAC configs, webhook configs, and node information. Share this archive when reporting issues.

### Interpret Selftest Output

```bash
make selftest
```

Selftest checks (and what failures mean):

| Check | Failure Meaning |
|-------|-----------------|
| Controller pod running | Deployment not created, image pull failed, or crash loop |
| CRDs installed (13) | Missing `kubectl apply -f config/base/crds/bases/` |
| CRDs established | CRD schema validation errors in YAML |
| Webhooks reachable | Webhook service or TLS certificate issues |
| Metrics endpoint | Metrics server not started (check port 8080) |
| RBAC permissions | ClusterRole or ClusterRoleBinding missing |

### Port-Forward for Metrics

```bash
kubectl port-forward -n ironwolf-system \
  svc/ironwolf-metrics 8080:8080 &

# Check specific metrics
curl -s localhost:8080/metrics | grep bmo_

# Check host counts
curl -s localhost:8080/metrics | grep bmo_baremetal_hosts_total

# Check claim status
curl -s localhost:8080/metrics | grep bmo_cluster_claim_status

# Check security compliance
curl -s localhost:8080/metrics | grep bmo_security

# Check energy savings
curl -s localhost:8080/metrics | grep bmo_energy

# Check telemetry anomalies
curl -s localhost:8080/metrics | grep bmo_telemetry

# Check backup status
curl -s localhost:8080/metrics | grep bmo_backup
```

### Inspect Controller State

```bash
# List all managed resources with short names
for kind in cc fwp mw pp hwp rs ap np sp ep tp hwb rj; do
  echo "=== ${kind} ==="
  kubectl get "${kind}" --all-namespaces 2>/dev/null || echo "(CRD not found)"
done

# Check finalizers (if deletion is stuck)
kubectl get clusterclaim <name> -o jsonpath='{.metadata.finalizers}'

# Remove stuck finalizer (use with caution)
kubectl patch clusterclaim <name> \
  --type=json \
  -p='[{"op":"remove","path":"/metadata/finalizers"}]'
```

## Getting Help

1. Check this troubleshooting guide
2. Run `make selftest` and review the output
3. Run `make debug-bundle` and save the archive
4. Run `./scripts/health-check.sh --json` and save the output
5. Check controller logs: `kubectl logs -n ironwolf-system -l app.kubernetes.io/name=ironwolf --tail=500 > bmo-debug.log`
6. Open an issue with:
   - Debug bundle archive
   - Health check JSON output
   - Selftest output
   - Controller logs
   - Description of expected vs actual behavior
