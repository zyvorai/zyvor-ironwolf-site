# Operations Guide

## Deployment

### Quick Deploy

```bash
# Deploy to dev environment
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf -n ironwolf-system --create-namespace

# Deploy to production with 2 replicas
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf -n ironwolf-system --create-namespace -f values-production.yaml

# Dry run to preview manifests
./scripts/deploy.sh --env staging --dry-run
```

### Environment Profiles

| Setting | Dev | Staging | Production |
|---------|-----|---------|------------|
| Replicas | 1 | 1 | 2 |
| Leader Election | No | Yes | Yes |
| Log Level | debug | info | info |
| CPU Request | 10m | 100m | 250m |
| CPU Limit | 500m | 1000m | 2000m |
| Memory Request | 64Mi | 128Mi | 256Mi |
| Memory Limit | 256Mi | 512Mi | 1Gi |

### Deployment Methods

**Deploy script (recommended):**
```bash
helm upgrade --install ironwolf oci://ghcr.io/zyvorai/charts/ironwolf -n ironwolf-system --create-namespace -f values-production.yaml
```

**Helm (product package):**
```bash
helm upgrade --install ironwolf deploy/helm/ironwolf \
  --namespace ironwolf-system --create-namespace \
  -f deploy/helm/ironwolf/values-production.yaml \
  -f deploy/helm/ironwolf/values-package.yaml

# Optional classic install without packaging seeds:
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system \
  --create-namespace \
  --set image.repository=ghcr.io/zyvorai/ironwolf \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set replicaCount=2

# With dashboard API key and NetworkPolicy
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system \
  --create-namespace \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set networkPolicy.enabled=true \
  --set dashboard.auth.apiKey="my-secret-key"
```

> **Note:** The chart defaults to `pullPolicy: Always` for development convenience.
> For production, use `IfNotPresent` with a pinned image tag.

**Kustomize:**
```bash
kubectl apply -k deploy/kustomize/overlays/
```

**Single manifest (from release):**
```bash
kubectl apply -f https://github.com/zyvorai/ironwolf/releases/latest/download/install.yaml
```

**Local development:**
```bash
./scripts/deploy-local.sh
```

### Pre-Deployment Checks

Run the preflight check before deploying to verify cluster readiness:

```bash
# Check cluster readiness
make preflight

# Check with auto-fix for common issues
make preflight-fix

# Or use the script directly
./scripts/preflight-check.sh
./scripts/preflight-check.sh --fix
```

### Post-Deployment Verification

Run selftest after deployment to verify everything is working:

```bash
# Full selftest
make selftest

# Quick selftest (skip cluster checks)
make selftest-quick

# Or use the script directly
./scripts/selftest.sh
./scripts/selftest.sh --quick
```

## Upgrades

### Rolling Upgrade

```bash
# Upgrade with auto-rollback on failure
./scripts/upgrade.sh \
  --image ghcr.io/zyvorai/ironwolf:v1.1.0 \
  --update-crds \
  --rollback-on-fail

# Manual upgrade
kubectl set image deployment/ironwolf-controller-manager \
  manager=ghcr.io/zyvorai/ironwolf:v1.1.0 \
  -n ironwolf-system
```

### Rollback

```bash
# Immediate rollback to previous version
kubectl rollout undo deployment/ironwolf-controller-manager \
  -n ironwolf-system

# Rollback to specific revision
kubectl rollout undo deployment/ironwolf-controller-manager \
  -n ironwolf-system --to-revision=3

# Check rollout history
kubectl rollout history deployment/ironwolf-controller-manager \
  -n ironwolf-system
```

## Health Monitoring

### Health Check Script

```bash
# Full health report
make health

# JSON output (for monitoring integration)
make health-json

# Or use the script directly
./scripts/health-check.sh
./scripts/health-check.sh --json
./scripts/health-check.sh --exit-code
```

### Manual Checks

```bash
# Pod status
kubectl get pods -n ironwolf-system -o wide

# Controller logs
kubectl logs -n ironwolf-system \
  -l app.kubernetes.io/name=ironwolf -f

# Events
kubectl get events -n ironwolf-system \
  --sort-by='.lastTimestamp' --watch

# Metrics
kubectl port-forward -n ironwolf-system \
  svc/ironwolf-metrics 8080:8080
curl localhost:8080/metrics

# All managed resources at a glance
for kind in cc fwp mw pp hwp rs ap np sp ep tp hwb rj; do
  echo "=== ${kind} ==="
  kubectl get "${kind}" --all-namespaces 2>/dev/null || echo "(not found)"
done
```

### Prometheus Alerts (Examples)

```yaml
groups:
- name: ironwolf
  rules:
  - alert: IronWolfControllerDown
    expr: absent(up{job="ironwolf-metrics"})
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "IronWolf controller is down"

  - alert: IronWolfHighRestartCount
    expr: increase(kube_pod_container_status_restarts_total{container="manager",namespace="ironwolf-system"}[1h]) > 3
    labels:
      severity: warning
    annotations:
      summary: "IronWolf controller restarting frequently"

  - alert: IronWolfClaimStuck
    expr: bmo_cluster_claim_status{phase="Partial"} > 0
    for: 30m
    labels:
      severity: warning
    annotations:
      summary: "ClusterClaim {{ $labels.name }} stuck in Partial state"

  - alert: IronWolfHostUnhealthy
    expr: bmo_host_health_status == 0
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "BareMetalHost {{ $labels.host }} is unhealthy"

  - alert: IronWolfSecurityNonCompliant
    expr: bmo_security_compliant_hosts / bmo_security_compliant_hosts + bmo_security_violations_total < 0.95
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Security compliance below 95%"

  - alert: IronWolfBackupFailed
    expr: bmo_backup_total{status="Failed"} > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Hardware backup failed"

  - alert: IronWolfHighEnergyUsage
    expr: bmo_energy_savings_watts < 0
    for: 1h
    labels:
      severity: info
    annotations:
      summary: "Energy policy not achieving savings"
```

## Backup and Restore

### Backup All CRD Resources

Use the built-in backup script to create a compressed archive of all CRD resources:

```bash
# Create a backup archive
make backup

# Or use the script directly
./scripts/backup-state.sh
```

This creates a timestamped archive containing all 20 backable CRD resource types across all
namespaces (the 7 remaining `bmo.metal3.io` CRD definitions — PreprovisioningImage, DataImage,
HostFirmwareSettings, HostFirmwareComponents, BMCEventSubscription, HardwareData, FirmwareSchema —
are deprecated duplicates of upstream `metal3.io/v1alpha1` types and are not installed; see
`docs/guides/bmo-upstream-parity.md`).

### Restore CRD Resources

```bash
# Restore from a backup archive
./scripts/restore-state.sh /path/to/backup.tar.gz
```

### Manual Backup

```bash
for kind in clusterclaims firmwareprofiles maintenancewindows \
            powerpolicies hardwarepools remediationstrategies \
            alertpolicies networkprofiles securitypolicies \
            energypolicies telemetrypolicies hardwarebackups \
            restorejobs; do
  kubectl get "${kind}.bmo.metal3.io" --all-namespaces -o yaml \
    > "backup-${kind}.yaml"
done
```

### Manual Restore

```bash
for f in backup-*.yaml; do
  kubectl apply -f "$f"
done
```

### Hardware Configuration Backup

In addition to CRD resource backup, use HardwareBackup resources to back up actual hardware configurations (BIOS, BMC, firmware, RAID, network, partitions). See the [API Reference](api-reference.md#hardwarebackup) for HardwareBackup and RestoreJob CRD details.

## Debug Bundle Collection

Collect a comprehensive debug bundle for troubleshooting:

```bash
make debug-bundle

# Or use the script directly
./scripts/collect-debug-bundle.sh
```

The debug bundle includes:
- Controller logs
- Pod descriptions and status
- All CRD resources
- Kubernetes events
- RBAC configuration
- CRD status conditions
- Webhook configurations
- Node and cluster information

Share the resulting archive when opening issues or requesting support.

## Selftest and Preflight

### Selftest

Post-installation verification that checks:
- Controller pod is running and ready
- All 39 CRD manifests (27 `bmo.metal3.io` + upstream `metal3.io`/`image.ironwolf.io`) are present and installed
- Webhook endpoints are reachable
- Metrics endpoint responds
- RBAC permissions are correct

```bash
make selftest          # full selftest
make selftest-quick    # skip cluster checks
```

### Preflight Check

Pre-deployment cluster readiness check that verifies:
- Kubernetes version compatibility
- Metal3 BareMetalOperator is installed
- Required RBAC permissions
- Namespace existence
- Resource availability

```bash
make preflight         # check readiness
make preflight-fix     # check and auto-fix
```

## Teardown

```bash
# Interactive teardown (prompts for confirmation)
./scripts/teardown.sh

# Force teardown including CRDs
./scripts/teardown.sh --force

# Keep CRDs and data, only remove controller
./scripts/teardown.sh --keep-crds --force

# Complete uninstall
./scripts/uninstall.sh
```

## CI/CD Pipeline

### CI Script

Run the full CI pipeline locally:

```bash
make ci           # full pipeline: fmt + vet + lint + test + build
make ci-quick     # quick: fmt + vet + test + build (no lint)

# Or use the script directly
./scripts/ci.sh
```

### Release

Prepare release artifacts:

```bash
make release

# Or use the script directly
./scripts/release.sh
```

This performs:
- Cross-compilation for linux/darwin amd64/arm64
- Docker image build and push
- Manifest generation
- Changelog generation

### Generate Install Manifest

Generate a single-file install manifest for release distribution:

```bash
make generate-install
# Output: dist/install.yaml
```

## Web Dashboard Operations

### Build

```bash
# Build complete web dashboard (frontend + backend)
make web

# Build frontend only
make web-build

# Build backend only
make web-server
```

### Run

```bash
# Build and run locally
make web-run

# Run with API key authentication
./bin/ironwolf-web --kubeconfig ~/.kube/config --static-dir web/dashboard/dist --api-key my-secret-key

# Development mode with hot reload
make web-dev
```

### Docker Deployment

```bash
# Build web dashboard docker image
make docker-build-web

# Push web dashboard docker image
make docker-push-web
```

### Helm Deployment

The Helm chart includes dashboard-specific templates:

- `dashboard-deployment.yaml` -- Deploys the web dashboard as a Kubernetes Deployment
- `dashboard-service.yaml` -- Exposes the dashboard on port 8888
- `dashboard-ingress.yaml` -- Optional ingress for external access
- `dashboard-secret.yaml` -- API key Secret (when `dashboard.auth.apiKey` is set)
- `networkpolicy.yaml` -- NetworkPolicies for controller and dashboard (when `networkPolicy.enabled: true`)
- `rbac.yaml` -- ClusterRole and ClusterRoleBinding for the manager
- `serviceaccount.yaml` -- ServiceAccounts for controller and dashboard

```bash
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system \
  --create-namespace \
  --set dashboard.enabled=true \
  --set dashboard.apiKey=my-secret-key
```

### Authentication Setup

For production deployments, always set an API key:

```bash
# Via command line flag
./bin/ironwolf-web --api-key $(openssl rand -hex 32)

# Via Helm values
helm install ironwolf deploy/helm/ironwolf/ \
  --set dashboard.apiKey=$(openssl rand -hex 32)
```

Clients must include the `X-API-Key` or `Authorization: Bearer` header in all API requests when authentication is enabled. WebSocket connections authenticate via the `Sec-WebSocket-Protocol` header using `auth.<token>` subprotocol.

### Environment Variables

The web dashboard supports the following environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `IRONWOLF_MAX_TRACKED_IPS` | `10000` | Maximum IPs tracked by the auth rate limiter (read once at startup) |
| `IRONWOLF_MAX_WS_CLIENTS` | `100` | Maximum concurrent WebSocket connections (read once at startup) |
| `GRAFANA_URL` | (none) | Grafana dashboard URL shown in the web UI |
| `SMTP_HOST` | (none) | SMTP server hostname for email notifications |
| `SMTP_PORT` | `587` | SMTP server port |
| `SMTP_FROM` | (none) | Sender email address for alert emails |
| `SMTP_USERNAME` | (none) | SMTP authentication username (optional; enables STARTTLS auth) |
| `SMTP_PASSWORD` | (none) | SMTP authentication password (optional) |

> **SMTP note:** When `SMTP_USERNAME` and `SMTP_PASSWORD` are configured, the
> server automatically upgrades the connection with STARTTLS if the remote
> server advertises support for it. Email headers (From, To, Subject) are
> sanitized to prevent header-injection attacks.

## CLI Tools

### bmh-inspector

The `bmh-inspector` CLI inspects and lists BareMetalHost resources. Use the
`-n` / `--namespace` flag to target a specific namespace (defaults to the
current kubeconfig namespace).

```bash
# List all hosts in a namespace
bmh-inspector list -n metal3-system

# Inspect a specific host
bmh-inspector inspect my-host -n metal3-system
```

### firmware-updater

The `firmware-updater` CLI shows firmware profile details and checks host
firmware status. Use the `-n` / `--namespace` flag to target a specific
namespace.

```bash
# Check firmware status of a host
firmware-updater status -n metal3-system

# Show details of a firmware profile
firmware-updater show my-profile -n metal3-system

# Check firmware compliance for a host
firmware-updater check my-host -n metal3-system
```

## API Security

### Audit Endpoint

The `/api/v1/audit` endpoint returns the in-memory audit log of mutations
performed through the dashboard API. This endpoint now requires the **admin
role** -- requests without an admin-level API key receive `403 Forbidden`.

### Web Security Headers

The web dashboard automatically sets the following security headers on every
HTTP response:

- **Content-Security-Policy (CSP)** -- restricts resource origins to mitigate
  XSS attacks.
- **Strict-Transport-Security (HSTS)** -- instructs browsers to use HTTPS for
  future requests.

No additional configuration is required; the headers are applied by default.

## Local Development

### Kind Cluster

```bash
# Full local setup (Kind + build + deploy + samples)
make kind-setup

# Or with the script directly
./scripts/setup-kind.sh --with-samples

# With custom worker count
./scripts/setup-kind.sh --workers 3 --with-samples

# Destroy and recreate
make kind-destroy
make kind-setup
```

### Rebuild Loop

```bash
# After code changes, rebuild and reload into Kind
make kind-reload

# This runs: build + docker-build + kind load + rollout restart
```

### Install Development Dependencies

```bash
make install-deps

# Or use the script directly
./scripts/install-deps.sh
```

## Scripts Reference

All scripts are located in the `scripts/` directory.

| Script | Description | Key Flags |
|--------|-------------|-----------|
| `deploy.sh` | Deploy to dev/staging/production | `--env`, `--image`, `--dry-run` |
| `deploy-local.sh` | Deploy to local development environment | -- |
| `upgrade.sh` | Rolling upgrade with auto-rollback | `--image`, `--update-crds`, `--rollback-on-fail` |
| `teardown.sh` | Clean removal with confirmation | `--force`, `--keep-crds` |
| `uninstall.sh` | Complete uninstall | -- |
| `health-check.sh` | Comprehensive health report | `--json`, `--exit-code` |
| `selftest.sh` | Post-installation verification | `--quick` |
| `preflight-check.sh` | Pre-deployment cluster readiness | `--fix` |
| `setup-kind.sh` | Local dev cluster with Kind | `--with-samples`, `--workers N`, `--destroy` |
| `build.sh` | Build binaries and container images | `all`, `--push` |
| `install-deps.sh` | Install development dependencies | -- |
| `backup-state.sh` | Backup all CRD resources to archive | -- |
| `restore-state.sh` | Restore CRD resources from archive | `<archive-path>` |
| `collect-debug-bundle.sh` | Collect debug bundle | -- |
| `ci.sh` | Run full CI pipeline | -- |
| `release.sh` | Prepare release artifacts | -- |

## Makefile Target Reference

Run `make help` to see all 64 targets grouped by category.

Key categories:

| Category | Targets |
|----------|---------|
| General | `help`, `version` |
| Development | `manifests`, `generate`, `fmt`, `vet`, `lint`, `lint-fix`, `run` |
| Testing | `test`, `test-e2e`, `test-web`, `test-all`, `test-coverage` |
| Build | `build`, `build-tools`, `build-all`, `build-cross` |
| Container Images | `docker-build`, `docker-push`, `docker-buildx`, `docker-build-web`, `docker-push-web`, `docker-all` |
| Web Dashboard | `web-install`, `web-build`, `web-dev`, `web-server`, `web-clean`, `web`, `web-run` |
| Deployment | `install`, `uninstall`, `deploy`, `deploy-dev`, `undeploy` |
| Lifecycle/Ops | `selftest`, `selftest-quick`, `preflight`, `preflight-fix`, `health`, `health-json`, `debug-bundle`, `backup`, `restore` |
| CI/CD | `ci`, `ci-quick`, `release`, `generate-install` |
| Kind | `kind-setup`, `kind-destroy`, `kind-reload` |
| Cleanup | `clean`, `clean-web`, `clean-all` |
| Validation | `validate-crds`, `validate-samples`, `validate-helm` |
| Dependencies | `kustomize`, `controller-gen`, `envtest`, `golangci-lint`, `install-deps` |
