# IronWolf — Customer Documentation

**IronWolf** extends Metal3 Bare Metal Operator with fleet-wide firmware, claims, pools, remediation, energy policies, and a full React dashboard — including a **Customer Portal** mode for dedicated-server tenants.

| You want to… | Open |
|--------------|------|
| Install and log in | [Getting Started](getting-started.md) |
| Learn Ops vs Portal | [Using the Dashboard](using-the-dashboard.md) |
| Change light / dark look | [Appearance](pages/settings/settings-appearance.md) |
| Follow a page, step by step | [Page-by-page guides](pages/README.md) |
| Look up any screen by route | [Complete page index](PAGE_INDEX.md) |
| Deploy, auth, ports | [Admin basics](admin-basics.md) |
| Multi-page jobs | [Common workflows](workflows.md) |
| Capability map | [Feature Guide](../ironwolf-customer-feature-guide.md) |
| Portal ops (ConfigMaps, Prometheus, branding, tenant keys) | [guides/customer-portal-ops.md](../guides/customer-portal-ops.md) |

## Printable PDFs

```bash
node scripts/customer-docs/build-customer-pdfs.mjs
```

Output lands in [`pdf/`](pdf/):

| PDF | Contents |
|-----|----------|
| `IronWolf-Customer-README.pdf` | This overview |
| `IronWolf-Getting-Started.pdf` | Access, login, dashboard basics, workflows |
| `IronWolf-Page-by-Page.pdf` | Complete page manual |
| `IronWolf-Admin-Basics.pdf` | Deploy, auth, ports |

## Product at a glance

```text
  Dashboard   →  :8888  (TLS on by default; React SPA · Ops | Portal)
  Operator    →  24+ CRDs · ClusterClaim · FirmwareProfile · portal CRDs
  Foundation  →  Metal3 BMO · Ironic · BareMetalHost
```

## Support surfaces (quick map)

| Need | Typical path |
|------|----------------|
| Home / health | `/` |
| Hosts (Ops) / Bare Metal (Portal) | `/hosts` |
| Cart / Usage / Storage | `/cart`, `/usage`, `/storage` |
| Monitoring / Support / Status | `/monitoring`, `/support`, `/system-status` |
| Account | `/account` |
| Claims / pools | `/claims`, `/pools` |
| Firmware | `/firmware` |
| Maintenance / remediation | `/maintenance`, `/remediation` |
| Image Factory | `/image-factory/recipes` |
| Appearance (light / dark) | `/settings/appearance` |
| Settings | `/settings/general` |

---

*ZyvorAI Labs · [zyvor.dev](https://zyvor.dev) · IronWolf*
