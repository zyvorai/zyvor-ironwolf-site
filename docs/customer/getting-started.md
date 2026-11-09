# Getting Started with IronWolf

## What you need

| Requirement | Notes |
|-------------|--------|
| Kubernetes cluster | With Metal3 BMO / Ironic (or lab equivalent) |
| IronWolf operator + CRDs | Helm chart from your commercial package (`oci://ghcr.io/zyvorai/charts/ironwolf`) |
| Dashboard | Default **`:8888`** (TLS on by default) |
| Auth | API key via `IRONWOLF_API_KEY` (roles: admin / operator / viewer; optional `:namespace` tenant scope) |

## 1. Open the UI

- Local / port-forward: `https://localhost:8888` (self-signed cert by default — expect a browser warning unless you supply `--tls-cert` / `--tls-key`)
- Disable TLS only for lab: `--no-tls` / `IRONWOLF_NO_TLS=true`
- Health: open the dashboard or call the API behind the same port

## 2. Sign in

| Mode | What you do |
|------|-------------|
| API key | Enter the key (`X-API-Key` / Bearer). Format `key:role` or `key:role:namespace` for tenant viewers. |
| Open (no key) | Implicit **admin** when no key is configured — **harden before production** |
| Lab default | Deploy scripts may seed a lab API key — **rotate immediately** |

After a successful login the UI leaves `/login` and lands on the dashboard (staying on `/login` is a bug — hard-reload if you still see 404).

## 3. Orient yourself

### Operator (Ops)

1. **Dashboard** (`/`) — fleet counts and shortcuts (hero + tiles).
2. **Hosts** (`/hosts`) — BareMetalHost inventory and lifecycle (product-tile grid by default).
3. **Claims / Pools** — allocate metal into target clusters.
4. **Firmware / Policies / Operations** — day-2 lifecycle (includes **Provision Logs**).
5. Spotlight / command palette (⌘K or ⌘Space) for any page by name.
6. **Settings → Appearance** — light / dark / system mode.

### Customer (Portal)

1. Click **Portal** in the chrome (viewers land here automatically).
2. **Bare Metal** — dedicated servers (power, install, traffic, networking, remote access).
3. **Cart / Usage / Storage** — order SKUs, bandwidth & invoices, object buckets.
4. **Monitoring / Support / System Status** — uptime checks, tickets, platform posts.
5. **Account** — SSH vault, notify prefs, favorites / recently viewed servers.

## 4. First workflows

### A. Register and inspect a host (Ops)

**Hosts** → register / import → wait for inspection → review **Hardware Data**.

### B. Claim hosts into a cluster (Ops)

**Claims** → create wizard: set counts and strategy, optionally **pick hosts and roles** (including `gpu-worker`) → wait for **Fulfilled** → claim detail **Bootstrap** (K8s version + CNI) → **Apply & watch** → download kubeconfig from the claim or **Target Clusters**.

Minimal two-node path: 1 control plane + 1 worker, pick both hosts, Bootstrap with Cilium. See [workflows](workflows.md) and [Claims](pages/orchestration/claims.md).

### C. Manage a dedicated server (Portal)

**Bare Metal** → open a host → **Traffic** / **Installation** / **Remote Access**.

### D. Open a support ticket (Portal)

**Support** → New ticket, or deep-link `/support?new=1&title=…`.

### E. Order from the catalog (Portal)

**Cart** → add SKUs → **Checkout — create order ticket** (sales follow-up; no payment in-portal).

## Next steps

- [Using the Dashboard](using-the-dashboard.md)
- [Admin basics](admin-basics.md)
- [Page guides](pages/README.md)
- [Common workflows](workflows.md)

## Operate from the console (UX)

1. Open this route from the nav or command palette and wait for live API data.
2. Use filters/search when present; drill into a row for detail.
3. For mutating actions: confirm role gates and impact before applying.
4. **Empty / fail:** Check service health, auth, and that required CRDs/backends for this domain are installed.
5. **Success:** Live data loads; created/updated objects appear without error toasts.

