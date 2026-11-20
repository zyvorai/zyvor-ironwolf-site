# Using the Dashboard

IronWolf’s React dashboard serves **two modes** in one product: **Ops** (operator control plane) and **Portal** (tenant / dedicated-server experience). Switch with **Ops | Portal** in the chrome. Viewers scoped to a tenant namespace are forced into Portal and cannot switch.

Separately, **Settings → Appearance** chooses **light** or **dark** mode for the Apple Store / Zeus paper UI (see [Appearance](pages/settings/settings-appearance.md)).

## Surfaces

| Element | Purpose |
|---------|---------|
| **Ops \| Portal** | Toggle operator vs customer UI |
| **Appearance** | Light / dark / system mode (Settings only) |
| **Dashboard** | Home — fleet health (Ops) or services / attention / recent servers (Portal); homepage-style hero + tiles |
| **Overview / Infrastructure / Orchestration / Firmware / Policies / Operations** | Ops nav (global nav + More) |
| **Bare Metal, Cart, Usage, Storage, Monitoring, Security, IP, Support, Status, Account** | Portal nav |
| **Spotlight / command palette (⌘K or ⌘Space)** | Jump to any labeled page |
| **Detail routes** | `/hosts/:ns/:name`, tickets, monitors, and other resources |
| **Settings** | General, appearance, notifications, security, provisioning, AI, advanced |
| **Provision Logs** | Saved host provision console sessions (`/provision-logs`) |

## Browse vs act

Inventory and observability pages are safe to explore. Provision, claim, firmware apply, remediation, and cart checkout require an appropriate role and should respect maintenance windows. Portal checkout creates a sales **SupportTicket** — it does not charge cards.

## Approvals

When `dashboard.requireApprovals` is on, destructive host actions (power, deprovision, delete, reboot) create a pending request. Operators review them at **`/approvals`** (approve / reject). Slack or webhook notify can fire when prefs are configured.

## Host quarantine

On **host detail** (`/hosts/:ns/:name`), quarantine blocks further mutations on that host until cleared. Fleet health on the home page surfaces quarantined count.

## Capacity planner / claim dry-run

**Orchestration → Capacity Planner** (`/planner`) runs a what-if capacity plan. Cluster claim create also supports scheduling dry-run so you can see feasibility and candidate hosts before applying.

## Portal deep links (bookmarkable)

| Flow | Example |
|------|---------|
| Support ticket | `/support?ticket=ns/name` |
| New ticket draft | `/support?new=1&title=…&body=…` |
| Monitor add | `/monitoring?add=1&target=<ephemeral-ip>` |
| Cart SKU | `/cart?add=BM-EPYC-64` |
| Host tab | `/hosts/ns/name?tab=Traffic&range=1w` |
| Usage / invoices | `/usage?range=1w&inv=open` |
| Storage bucket | `/storage?bucket=name` |

Ops details for ConfigMaps, Prometheus, and tenant keys: [Customer portal ops](../guides/customer-portal-ops.md) (repo guide).

## Related

- [Getting Started](getting-started.md)
- [Appearance](pages/settings/settings-appearance.md)
- [Page-by-page guides](pages/README.md)
- [Common workflows](workflows.md)

## Operate from the console (UX)

1. Open this route from the nav or command palette and wait for live API data.
2. Use filters/search when present; drill into a row for detail.
3. For mutating actions: confirm role gates and impact before applying.
4. **Empty / fail:** Check service health, auth, and that required CRDs/backends for this domain are installed.
5. **Success:** Live data loads; created/updated objects appear without error toasts.
