# Hardware certification

IronWolf validates platform combinations using `config/lab/certification-matrix.yaml`.

## Profiles

| ID | What it proves |
|----|----------------|
| `lab-libvirt-two-node` | Full claim → CAPM3 → target cluster on lab libvirt pool |
| `lab-smoke-cirros` | Fast OS provision smoke without Ginkgo |
| `kind-controller` | Controller + CRD paths on Kind (CI default) |
| `physical-dell-idrac` | Redfish smoke against Dell iDRAC (`IRONWOLF_BMC_*`) |
| `physical-hpe-ilo` | Redfish smoke against HPE iLO |
| `physical-supermicro` | Redfish smoke against Supermicro BMC |
| `restorejob-live` | RestoreJob dry-run against a live BareMetalHost |

## Run locally

```bash
# Full two-node (requires lab SSH)
IRONWOLF_LAB_HOST=<ephemeral-ip> IRONWOLF_LAB_USER=operator \
  # packaged operator tooling (provided with your license) --profile two-node

# Matrix runner (used by nightly workflow)
# packaged operator tooling (provided with your license) lab-libvirt-two-node

# Physical BMC SKU (secrets required)
export IRONWOLF_BMC_ADDRESS=https://bmc.example
export IRONWOLF_BMC_USER=root
export IRONWOLF_BMC_PASSWORD=secret
export IRONWOLF_BMC_INSECURE=1   # lab only
# packaged operator tooling (provided with your license) physical-dell-idrac

# RestoreJob dry-run on a cluster host
IRONWOLF_RESTORE_HOST=node-1 # packaged operator tooling (provided with your license) restorejob-live
```

## Nightly CI

`.github/workflows/hardware-cert-nightly.yaml` runs when `PROVISIONING_LAB_SSH_KEY` is configured.
Physical SKU jobs run when `IRONWOLF_BMC_ADDRESS` (repository secret) is set.

## Adding a BMC SKU

1. Add a row to `certification-matrix.yaml` with `bmc`, `provisioner`, and `image`.
2. Document BMC credentials and network in `config/lab/defaults.yaml`.
3. Run `# packaged operator tooling (provided with your license) <profile-id>` on the lab host.
