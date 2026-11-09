# Helm Chart Reference

Chart location: `deploy/helm/ironwolf/`

## Values Quick Reference

All defaults in a single copyable block:

```yaml
replicaCount: 1

image:
  repository: ghcr.io/zyvorai/ironwolf
  pullPolicy: Always
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  name: "ironwolf-controller-manager"

leaderElection:
  enabled: true

metrics:
  bindAddress: ":8080"

healthProbe:
  bindAddress: ":8081"

webhook:
  port: 9443

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 10m
    memory: 128Mi

nodeSelector: {}
tolerations: []
affinity: {}

pdb:
  enabled: false
  minAvailable: 1

networkPolicy:
  enabled: true

serviceMonitor:
  enabled: false
  interval: 30s
  labels: {}

grafana:
  dashboards:
    enabled: false
    labels: {}
    annotations: {}

prometheusRule:
  enabled: false
  additionalLabels: {}

dashboard:
  enabled: true
  image:
    repository: ghcr.io/zyvorai/ironwolf-web
    pullPolicy: Always
    tag: ""
  replicaCount: 1
  port: 8888
  auth:
    enabled: true
    apiKey: ""
  service:
    type: ClusterIP
    port: 8888
  ingress:
    enabled: false
    className: ""
    annotations: {}
    hosts:
      - host: ironwolf.local
        paths:
          - path: /
            pathType: Prefix
    tls: []
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 5
    targetCPUUtilizationPercentage: 80
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 10m
      memory: 64Mi
```

## Installation

```bash
# Default install
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system \
  --create-namespace

# Production install
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system \
  --create-namespace \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set replicaCount=2 \
  --set pdb.enabled=true \
  --set dashboard.auth.apiKey="your-secret-key" \
  --set dashboard.ingress.enabled=true \
  --set dashboard.ingress.hosts[0].host=ironwolf.example.com
```

CRDs are installed separately (Helm does not manage CRD lifecycle):

```bash
kubectl apply -f config/base/crds/bases/
# OR
make install
```

## Values Reference

### Controller Manager

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `replicaCount` | int | `1` | Manager replicas. Use `2` for HA with leader election. |
| `image.repository` | string | `ghcr.io/zyvorai/ironwolf` | Manager container image |
| `image.pullPolicy` | string | `Always` | `Always`, `IfNotPresent`, or `Never`. Use `IfNotPresent` with a pinned tag in production. |
| `image.tag` | string | `""` (chart appVersion) | Image tag. Empty defaults to `Chart.appVersion`. |
| `imagePullSecrets` | list | `[]` | Secrets for private registries |
| `nameOverride` | string | `""` | Override chart name |
| `fullnameOverride` | string | `""` | Override full release name |

### Service Account

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `serviceAccount.create` | bool | `true` | Create ServiceAccount |
| `serviceAccount.name` | string | `ironwolf-controller-manager` | ServiceAccount name |

### Leader Election

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `leaderElection.enabled` | bool | `true` | Enable leader election. Required when `replicaCount > 1`. |

### Metrics and Health

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `metrics.bindAddress` | string | `:8080` | Prometheus metrics listen address |
| `metrics.secure` | bool | `false` | HTTPS metrics with Kubernetes authn/authz (production: `:8443`) |
| `metrics.service.enabled` | bool | `true` | Create a ClusterIP Service for scraping |
| `metrics.service.port` | int | `8080` | Service / container metrics port |
| `metrics.reader.enabled` | bool | `false` | Bind `metrics-reader` ClusterRole to Prometheus SA (requires `metrics.secure`) |
| `metrics.reader.serviceAccount.name` | string | `""` | Prometheus ServiceAccount name (e.g. `kube-prometheus-prometheus`) |
| `metrics.reader.serviceAccount.namespace` | string | `""` | Prometheus namespace (e.g. `monitoring`) |
| `healthProbe.bindAddress` | string | `:8081` | Health/readiness probe endpoint |
| `webhook.enabled` | bool | `false` | Start the admission/conversion webhook TLS server and create the `<release>-webhook-service` Service + cert-manager `Certificate`/`Issuer` (`certificate.yaml`, `webhook-service.yaml`). Required for both validating webhooks and CRD conversion webhooks (see `operator.conversionWebhooks.enabled` below) — neither works without this. |
| `webhook.port` | int | `9443` | Admission webhook server port |

### Operator runtime

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `operator.licenseKey` | string | `""` | → `IRONWOLF_LICENSE_KEY` (required for manager start unless secret is set) |
| `operator.licenseKeySecret` | string | `""` | Secret containing the license key |
| `operator.licenseKeySecretKey` | string | `license-key` | Key within the license secret |
| `operator.notifications.slackWebhook` | string | `""` | → `IRONWOLF_SLACK_WEBHOOK` (SupportTicket alerts) |
| `operator.notifications.supportEmails` | string | `""` | → `IRONWOLF_SUPPORT_NOTIFY_EMAILS` |
| `operator.notifications.supportWebhook` | string | `""` | → `IRONWOLF_SUPPORT_WEBHOOK` |
| `operator.notifications.smtpHost` | string | `""` | → `SMTP_HOST` |
| `operator.notifications.smtpPort` | string | `587` | → `SMTP_PORT` |
| `operator.notifications.smtpFrom` | string | `""` | → `SMTP_FROM` |
| `operator.notifications.smtpUsername` | string | `""` | → `SMTP_USERNAME` |
| `operator.notifications.smtpPasswordSecret` | string | `""` | Secret name for `SMTP_PASSWORD` |
| `operator.notifications.smtpPasswordKey` | string | `password` | Secret key for the SMTP password |
| `operator.provisioner` | string | `ironic` | `demo` or `ironic` → `IRONWOLF_PROVISIONER` |
| `operator.watchNamespace` | string | `""` | Limit reconciler cache; maps to `WATCH_NAMESPACE` and `--watch-namespace` |
| `operator.buildPreprovImage` | bool | `false` | Enable PreprovisioningImage controller |
| `operator.controllerConcurrency` | int | `3` | → `IRONWOLF_CONTROLLER_CONCURRENCY` |
| `operator.kubeAPI.qps` | float | `20` | Kubernetes client QPS (`--kube-api-qps`) |
| `operator.kubeAPI.burst` | int | `30` | Kubernetes client burst (`--kube-api-burst`) |
| `operator.labHost` | string | `""` (unset in `values.yaml`) | → `IRONWOLF_LAB_HOST` |
| `operator.allowPrivateProvisioningURLs` | bool | `false` (unset in `values.yaml`) | → `IRONWOLF_ALLOW_PRIVATE_PROVISIONING_URLS`; needed for lab/private-network Ironic endpoints |
| `operator.ironic.endpoint` | string | `""` | Ironic API URL → `IRONIC_ENDPOINT` |
| `operator.ironic.port` | int | `6385` | Ironic API port — only used to open the matching NetworkPolicy egress rule |
| `operator.ironic.insecure` | bool | `false` | Skip TLS verification → `IRONIC_INSECURE` (self-signed lab certs) |
| `operator.ironic.authTokenSecret` | string | `""` | Secret name for `IRONIC_AUTH_TOKEN` |
| `operator.ironic.authTokenKey` | string | `token` | Secret key for auth token |
| `operator.ironicNetwork.url` | string | `""` | Deprecated switch webhook URL → `IRONWOLF_IRONIC_NETWORK_URL` |
| `operator.openstack.authUrl` | string | `""` | Keystone URL → `OS_AUTH_URL` (Neutron switch apply) |
| `operator.openstack.username` | string | `""` | → `OS_USERNAME` |
| `operator.openstack.passwordSecret` | string | `""` | Secret for `OS_PASSWORD` |
| `operator.openstack.passwordKey` | string | `password` | Secret key for OpenStack password |
| `operator.openstack.projectName` | string | `""` | → `OS_PROJECT_NAME` |
| `operator.openstack.domainName` | string | `Default` | → `OS_USER_DOMAIN_NAME` |
| `operator.openstack.region` | string | `""` | → `OS_REGION_NAME` |
| `operator.attestation.enabled` | bool | `false` | → `IRONWOLF_ENABLE_ATTESTATION` (TPM quote verification) |
| `operator.attestation.secret` | string | `""` | HMAC secret for TPM quote verification → `IRONWOLF_ATTESTATION_SECRET` Secret |
| `operator.luksEscrow.enabled` | bool | `false` | → `IRONWOLF_ENABLE_LUKS_ESCROW` |
| `operator.luksEscrow.vaultAddr` | string | `""` | → `IRONWOLF_VAULT_ADDR` |
| `operator.luksEscrow.vaultToken` | string | `""` | → `IRONWOLF_VAULT_TOKEN` Secret |
| `operator.ebpfTrace.enabled` | bool | `false` | → `IRONWOLF_ENABLE_EBPF_TRACE` (provisioning trace controller) |
| `operator.conversionWebhooks.enabled` | bool | `false` | → `IRONWOLF_ENABLE_CONVERSION_WEBHOOKS`. Requires `webhook.enabled: true` and the CRD `spec.conversion` wiring from `make add-crd-conversion-webhook` (chained into `make manifests`) — see `docs/guides/api-versioning.md`. `values-production.yaml` sets this to `true`. |
| `operator.networkDriver` | string | `annotation` | `annotation`, `ansible`, or `noop` |

### Resources

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `resources.requests.cpu` | string | `10m` | CPU request |
| `resources.requests.memory` | string | `128Mi` | Memory request |
| `resources.limits.cpu` | string | `500m` | CPU limit |
| `resources.limits.memory` | string | `256Mi` | Memory limit |

### Scheduling

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `nodeSelector` | object | `{}` | Node selector for manager pods |
| `tolerations` | list | `[]` | Tolerations for manager pods |
| `affinity` | object | `{}` | Affinity rules for manager pods |

### Pod Disruption Budget

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `pdb.enabled` | bool | `false` | Create a PodDisruptionBudget for the controller manager. Disabled by default because the default `replicaCount` is `1` and a PDB with `minAvailable: 1` on a single replica blocks voluntary evictions (node drains, cluster upgrades). Enable this when running `replicaCount >= 2`. |
| `pdb.minAvailable` | int | `1` | Minimum number of pods that must remain available during voluntary disruptions. |

### Network Policy

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `networkPolicy.enabled` | bool | `true` | Create NetworkPolicies for controller and dashboard. Allows inbound metrics (8080), health (8081), webhook (9443), and outbound DNS (53) + API server (443, 6443). |

### ServiceMonitor

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `serviceMonitor.enabled` | bool | `false` | Create a Prometheus ServiceMonitor resource for automatic metrics scraping. Requires the prometheus-operator CRDs to be installed in the cluster. |
| `serviceMonitor.interval` | string | `30s` | Scrape interval for the ServiceMonitor. |
| `serviceMonitor.labels` | object | `{}` | Additional labels to add to the ServiceMonitor. Use this to match your Prometheus instance's `serviceMonitorSelector` (e.g., `release: prometheus`). |

### Grafana Dashboards

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `grafana.dashboards.enabled` | bool | `false` | Provision Grafana dashboards as ConfigMaps. When enabled, dashboard JSON files from `deploy/helm/ironwolf/dashboards/` are created as ConfigMaps that the Grafana sidecar can auto-discover. |
| `grafana.dashboards.labels` | object | `{}` | Labels to add to the dashboard ConfigMaps. Typically set to `grafana_dashboard: "1"` so the Grafana sidecar picks them up. |
| `grafana.dashboards.annotations` | object | `{}` | Annotations to add to the dashboard ConfigMaps (e.g., to specify the Grafana folder). |

### PrometheusRule

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `prometheusRule.enabled` | bool | `false` | Create a PrometheusRule resource containing alerting rules for IronWolf. Requires the prometheus-operator CRDs to be installed in the cluster. |
| `prometheusRule.additionalLabels` | object | `{}` | Additional labels to add to the PrometheusRule. Use this to match your Prometheus instance's `ruleSelector` (e.g., `release: prometheus`). |

### Dashboard

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.enabled` | bool | `true` | Deploy the web dashboard |
| `dashboard.image.repository` | string | `ghcr.io/zyvorai/ironwolf-web` | Dashboard container image |
| `dashboard.image.pullPolicy` | string | `Always` | Image pull policy |
| `dashboard.image.tag` | string | `""` (chart appVersion) | Image tag |
| `dashboard.replicaCount` | int | `1` | Dashboard replicas |
| `dashboard.port` | int | `8888` | Dashboard listen port |

### Dashboard Auth

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.auth.enabled` | bool | `false` | Enable API key authentication |
| `dashboard.auth.apiKey` | string | `""` | API key. Supports `key:role[:namespace]` pairs: `"admin-key:admin,ops-key:operator,viewer-key:viewer:customer-ns"`. Created as a Secret. Tenant namespace scopes list/mutate to that ns. Viewers land in Customer Portal UI. |
| `dashboard.requireApprovals` | bool | `false` | → `IRONWOLF_REQUIRE_APPROVALS` (two-person review for destructive host actions) |
| `dashboard.jobStore` | string | `""` | → `IRONWOLF_JOB_STORE` (default `/var/lib/ironwolf/lab-jobs.json` when persistence enabled) |
| `dashboard.auditStore` | string | `""` | → `IRONWOLF_AUDIT_STORE` |
| `dashboard.approvalStore` | string | `""` | → `IRONWOLF_APPROVAL_STORE` |
| `dashboard.persistence.enabled` | bool | `true` | Mount writable volume at `/var/lib/ironwolf` (needed with `readOnlyRootFilesystem`) |
| `dashboard.persistence.existingClaim` | string | `""` | Use an existing PVC (prefer RWX + Redis when `replicaCount` > 1) |
| `dashboard.redisUrl` | string | `""` | → `IRONWOLF_REDIS_URL` (shared bootstrap jobs + WebSocket pub/sub) |
| `dashboard.redisUrlSecret` | string | `""` | Secret holding Redis URL (key `dashboard.redisUrlSecretKey`, default `url`) |

### Dashboard Observability

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.observability.grafanaUrl` | string | `""` | Grafana base URL surfaced in the dashboard's Observability page → `IRONWOLF_GRAFANA_URL` |
| `dashboard.observability.prometheusUrl` | string | `""` | Prometheus base URL for the host Traffic tab, e.g. `http://prometheus.monitoring.svc:9090` |
| `dashboard.observability.trafficDemo` | bool | `false` | Synthesize inbound/outbound traffic series when Prometheus is unavailable (lab only) |

### Dashboard Customer Portal

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.portal.icmpEnabled` | bool | `false` | Enable ICMP ping checks from the portal |
| `dashboard.portal.signupEnabled` | bool | `true` | Public self-registration at `/register` |
| `dashboard.portal.publicUrl` | string | `""` | Absolute URL used in invite/approve emails, e.g. `https://portal.example.com` |
| `dashboard.portal.hadrianUrl` / `hadrianToken` / `hadrianTokenSecret` / `hadrianTokenSecretKey` | string | `""` / `""` / `""` / `token` | Hadrian security-scan integration endpoint + auth |
| `dashboard.portal.atlasUrl` / `atlasToken` / `atlasTokenSecret` / `atlasTokenSecretKey` | string | `""` / `""` / `""` / `token` | Atlas object-storage control-plane endpoint + auth (customer Object Storage projects) |
| `dashboard.portal.rdnsProvider` | string | `""` | Reverse-DNS provider: `noop`, `powerdns`, or `cloudflare` |
| `dashboard.portal.powerdnsUrl` / `powerdnsApiKey` / `powerdnsZone` | string | `""` | PowerDNS rDNS provider config |
| `dashboard.portal.cloudflareToken` / `cloudflareZoneId` | string | `""` | Cloudflare rDNS provider config |
| `dashboard.portal.s3.*` | object | see `values.yaml` | S3-compatible endpoint/bucket/credentials for customer downloads (`existingSecret` + `existingSecretKeys` to reference a pre-created Secret instead of inline keys) |

### Dashboard Zeus Assistant (LLM)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.llm.enabled` | bool | `false` | Enable the Zeus ops assistant (OpenAI-compatible LLM) on the dashboard backend |
| `dashboard.llm.apiKey` | string | `""` | LLM API key (ignored if `existingSecret` is set) |
| `dashboard.llm.baseUrl` | string | `https://openrouter.ai/api/v1` | OpenAI-compatible API base URL |
| `dashboard.llm.model` | string | `openrouter/free,openai/gpt-oss-120b:free,meta-llama/llama-3.3-70b-instruct:free,qwen/qwen3-4b:free` | Comma-separated fallback model chain — the backend tries the next model on 429/rate-limit |
| `dashboard.llm.existingSecret` / `existingSecretKeys.apiKey` | string | `""` / `api-key` | Reference a pre-created Secret for the API key instead of `dashboard.llm.apiKey` |

### Dashboard Service

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.service.type` | string | `ClusterIP` | Service type: `ClusterIP`, `NodePort`, or `LoadBalancer` |
| `dashboard.service.port` | int | `8888` | Service port |

### Dashboard Ingress

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.ingress.enabled` | bool | `false` | Create Ingress resource |
| `dashboard.ingress.className` | string | `""` | Ingress class name |
| `dashboard.ingress.annotations` | object | `{}` | Ingress annotations (e.g., cert-manager, auth) |
| `dashboard.ingress.hosts` | list | `[{host: ironwolf.local, paths: [{path: /, pathType: Prefix}]}]` | Ingress host rules |
| `dashboard.ingress.tls` | list | `[]` | TLS configuration for Ingress |

### Dashboard Autoscaling

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.autoscaling.enabled` | bool | `false` | Enable HorizontalPodAutoscaler for the dashboard. When enabled, `dashboard.replicaCount` is ignored and the HPA manages replica count based on CPU utilization. |
| `dashboard.autoscaling.minReplicas` | int | `1` | Minimum number of dashboard replicas the HPA will scale down to. |
| `dashboard.autoscaling.maxReplicas` | int | `5` | Maximum number of dashboard replicas the HPA will scale up to. |
| `dashboard.autoscaling.targetCPUUtilizationPercentage` | int | `80` | Target average CPU utilization (as a percentage of requested CPU) that triggers scaling. |

### Dashboard Resources

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dashboard.resources.requests.cpu` | string | `10m` | Dashboard CPU request |
| `dashboard.resources.requests.memory` | string | `64Mi` | Dashboard memory request |
| `dashboard.resources.limits.cpu` | string | `200m` | Dashboard CPU limit |
| `dashboard.resources.limits.memory` | string | `256Mi` | Dashboard memory limit |

## Example Configurations

### Minimal Dev

```bash
helm install ironwolf deploy/helm/ironwolf/ \
  --set leaderElection.enabled=false \
  --set dashboard.auth.enabled=false \
  --set networkPolicy.enabled=false
```

### Production HA

```bash
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system --create-namespace \
  --set replicaCount=2 \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set pdb.enabled=true \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=256Mi \
  --set resources.limits.cpu=2000m \
  --set resources.limits.memory=1Gi \
  --set serviceMonitor.enabled=true \
  --set prometheusRule.enabled=true \
  --set grafana.dashboards.enabled=true \
  --set grafana.dashboards.labels.grafana_dashboard=1 \
  --set dashboard.auth.apiKey="admin-key:admin,ops-key:operator" \
  --set dashboard.image.tag=v1.0.0 \
  --set dashboard.image.pullPolicy=IfNotPresent \
  --set dashboard.ingress.enabled=true \
  --set dashboard.ingress.className=nginx \
  --set dashboard.ingress.hosts[0].host=ironwolf.example.com \
  --set dashboard.ingress.hosts[0].paths[0].path=/ \
  --set dashboard.ingress.hosts[0].paths[0].pathType=Prefix \
  --set dashboard.ingress.tls[0].secretName=ironwolf-tls \
  --set dashboard.ingress.tls[0].hosts[0]=ironwolf.example.com
```

### Production HA with Autoscaling Dashboard

```bash
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system --create-namespace \
  --set replicaCount=2 \
  --set image.tag=v1.0.0 \
  --set image.pullPolicy=IfNotPresent \
  --set pdb.enabled=true \
  --set dashboard.autoscaling.enabled=true \
  --set dashboard.autoscaling.minReplicas=2 \
  --set dashboard.autoscaling.maxReplicas=10 \
  --set dashboard.autoscaling.targetCPUUtilizationPercentage=70 \
  --set dashboard.auth.apiKey="admin-key:admin,ops-key:operator"
```

### Dashboard Only (External Operator)

```bash
helm install ironwolf deploy/helm/ironwolf/ \
  --set replicaCount=0 \
  --set dashboard.enabled=true \
  --set dashboard.auth.apiKey="my-key"
```

### Monitoring Stack

```bash
helm install ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system --create-namespace \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.labels.release=prometheus \
  --set prometheusRule.enabled=true \
  --set prometheusRule.additionalLabels.release=prometheus \
  --set grafana.dashboards.enabled=true \
  --set grafana.dashboards.labels.grafana_dashboard=1
```

## Product package overlay

Opinionated out-of-box packaging lives in `deploy/helm/ironwolf/values-package.yaml`.
Chain it after production (or observability) values:

```bash
helm upgrade --install ironwolf deploy/helm/ironwolf \
  --namespace ironwolf-system --create-namespace \
  -f deploy/helm/ironwolf/values-production.yaml \
  -f deploy/helm/ironwolf/values-package.yaml \
  --set dashboard.observability.grafanaUrl=http://HOST:30301
```

| Key | Default (values.yaml) | Effect when enabled |
|-----|------------------------|---------------------|
| `productPackage.branding.enabled` | `false` | Seeds ConfigMap `ironwolf-branding` (`branding.json`) |
| `productPackage.branding.productName` | `IronWolf` | Portal / dashboard product name |
| `productPackage.branding.primaryColor` | `#38bdf8` | Accent color |
| `productPackage.notifyPrefs.enabled` | `false` | Seeds ConfigMap `ironwolf-notify-prefs` (`prefs.json`) |
| `productPackage.notifyPrefs.slackWebhook` | `""` | Optional https Slack webhook |
| `productPackage.starterAlertPolicy.enabled` | `false` | Creates starter `AlertPolicy` (`ironwolf-starter`) |
| `productPackage.starterAlertPolicy.channels` | `[]` | Optional Slack/Webhook channels on the starter policy |

`values-package.yaml` also turns on Grafana dashboard ConfigMaps, PrometheusRule, and ServiceMonitor.
After install, check readiness: `GET /api/v1/package/readiness`.

## Upgrading

```bash
helm upgrade ironwolf deploy/helm/ironwolf/ \
  --namespace ironwolf-system \
  --set image.tag=v1.1.0 \
  --set dashboard.image.tag=v1.1.0
```

## Uninstalling

```bash
helm uninstall ironwolf --namespace ironwolf-system

# CRDs must be removed separately
kubectl delete -f config/base/crds/bases/
```
