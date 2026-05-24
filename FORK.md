# raaghassi/cluster-api-provider-proxmox — DHCP fork

This is a soft fork of [ionos-cloud/cluster-api-provider-proxmox](https://github.com/ionos-cloud/cluster-api-provider-proxmox)
that adds DHCP-mode networking support to the v1alpha2 API. It exists
so the [aghassi cluster bring-up](https://github.com/raaghassi/cluster-aghassi-net)
can run Talos worker nodes on a per-cluster in-cluster VLAN (`172.16.41.0/24`
DEV-CTRL, `172.16.82.0/23` PROD-CTRL, etc.) under PVE-dnsmasq DHCP,
matching the design in `docs/cluster-realm-dns-dhcp.md` of that repo.

## Why a fork

Upstream v0.8.1 supports DHCP internally in the cloud-init renderer
(`pkg/cloudinit/network.go` has `dhcp4`/`dhcp6` template branches and
the `NetworkConfigData.DHCP4`/`DHCP6` bools propagate through
`pkg/types/network.go`). What's missing is the *user-facing* path:

1. **`ProxmoxClusterSpec` CEL gate** rejects clusters with no
   `ipv4Config`/`ipv6Config`. DHCP-only clusters have neither.
2. **Webhook `hasNoIPPoolConfig()`** repeats the same check at the
   admission layer.
3. **`NetworkDevice` CRD** has no `dhcp4`/`dhcp6` field for users to
   opt a NIC into DHCP mode.
4. **`getNetworkConfigDataForDevice`** in
   `internal/service/vmservice/bootstrap.go` doesn't copy any DHCP
   flag onto the `NetworkConfigData`; the renderer's DHCP support
   is unreachable from CRD input.

Upstream PR [#53](https://github.com/ionos-cloud/cluster-api-provider-proxmox/pull/53)
("Implement DHCP for machine network") has been open since Jan 2024
and is currently `mergeable: false`. Rather than wait, this fork
carries the small patch needed to wire the existing plumbing through.

## What the fork changes

Soft fork — all upstream commits remain. The fork branch `dhcp-support`
holds these additive patches on top of upstream `v0.8.1`:

| File | Change |
|---|---|
| `api/v1alpha2/proxmoxmachine_types.go` | Add `DHCP4 *bool` + `DHCP6 *bool` to `NetworkDevice` struct |
| `api/v1alpha2/proxmoxcluster_types.go` | Remove `XValidation` rule on `Spec` requiring `ipv4Config != null \|\| ipv6Config != null` |
| `api/v1alpha1/proxmoxcluster_types.go` | Remove same `XValidation` rule on the v1alpha1 storage version |
| `api/v1alpha1/proxmoxmachine_conversion.go` | Preserve `DHCP4`/`DHCP6` across v1alpha1 round-trip via the existing restore annotation |
| `internal/webhook/proxmoxcluster_webhook.go` | Drop `hasNoIPPoolConfig()` call from `ValidateCreate` |
| `internal/webhook/proxmoxclustertemplate_webhook.go` | Drop same `hasNoIPPoolConfig()` call from template webhook |
| `internal/service/vmservice/bootstrap.go` | In `getNetworkDevices`, copy `nic.DHCP4`/`nic.DHCP6` to `config.DHCP4`/`config.DHCP6` |

All other behaviour is unchanged. Clusters that continue to use
`ipv4Config` static IPAM work exactly as upstream. The fork only
*permits* DHCP — it doesn't force it.

## Versioning + image tags

Tagging convention: `vX.Y.Z-dhcp.N` where `vX.Y.Z` is the upstream
release the patches apply on top of, and `N` is the patch revision.

The `.github/workflows/dhcp-fork-image.yaml` workflow builds + pushes
to `ghcr.io/raaghassi/cluster-api-provider-proxmox`:

- Branch push to `dhcp-support` → `:dhcp-support` + `:dhcp-<sha>`
- Tag push matching `v*-dhcp.*` → `:vX.Y.Z-dhcp.N`

Upstream's `container-image.yaml` workflow is untouched (so upstream
sync doesn't conflict) and continues to handle `main`/`release-*`
branches + plain `v*` tags.

## Consuming the image

In the cluster-aghassi-net cluster-api-operator config:

```yaml
spec:
  fetchConfig:
    oci: ghcr.io/raaghassi/cluster-api-provider-proxmox:v0.8.1-dhcp.1
```

Or pin via the deployment image directly:

```yaml
spec:
  manager:
    image: ghcr.io/raaghassi/cluster-api-provider-proxmox:v0.8.1-dhcp.1
```

## Using DHCP mode

`ProxmoxCluster`: omit `ipv4Config` entirely if no NIC needs IPAM:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: ProxmoxCluster
spec:
  controlPlaneEndpoint:
    host: 172.16.40.70
    port: 6443
  # no ipv4Config block — DHCP-only deployment
  dnsServers: [1.1.1.1, 1.0.0.1]
  allowedNodes: [prox1, prox2, prox3]
```

`ProxmoxMachineTemplate`: opt the NIC into DHCP:

```yaml
spec:
  template:
    spec:
      network:
        networkDevices:
          - name: net0
            bridge: vmbr0
            vlan: 1041
            dhcp4: true
```

Mixed clusters (some NICs IPAM-bound, others DHCP) are supported —
the patches gate per-NIC, not per-cluster.

## Upstream sync

The `upstream-sync.yaml` workflow runs weekly on Mondays + on manual
dispatch. It opens a PR rebasing `dhcp-support` onto `upstream/main`
when there are new commits, or files an issue if the rebase hits
conflicts. Conflicts almost always sit in the seven files the fork
patches; resolve manually with:

```sh
git remote add upstream https://github.com/ionos-cloud/cluster-api-provider-proxmox
git fetch upstream
git checkout dhcp-support
git merge upstream/main
# resolve conflicts, keeping the DHCP-fork-side semantics
git push origin dhcp-support
```

Watch upstream PR #53 — when it merges, the fork is no longer
needed; retire by re-pointing the cluster-api-operator config back
at the upstream image and deleting `dhcp-support` after the next
upstream release.
