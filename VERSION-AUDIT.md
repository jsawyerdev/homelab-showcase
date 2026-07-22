# Homelab Version Audit and Recurring SOP

This document is both the version audit dated **2026-07-22** and the standard
operating procedure for repeating the audit every two to four weeks.

The audit is read-only. It does not authorize upgrades, reboots, chart changes,
database migrations, firmware changes, or deletion. Use a separate, explicitly
approved maintenance window to apply accepted changes.

## Audit metadata

| Field | Value |
|---|---|
| Checked | 2026-07-22, Europe/Paris |
| Public report repository | `homelab-showcase` |
| Deployment source of truth | `homelab-k8s-migration` GitLab `master` plus live Argo CD state |
| Live checks | Kubernetes, Helm, Argo CD, Talos node metadata, Proxmox, OPNsense, OpenMediaVault |
| Online checks | Official release pages, official GitHub releases, Helm repositories, OCI registries |
| Excluded from automatic action | Major upgrades, stateful migrations, network edge, storage, firmware |
| Redaction | No internal addresses, credentials, private hostnames, or secret values |

## Executive result

The Kubernetes cluster was healthy at audit time: all three nodes were Ready,
all 17 Argo CD applications were Synced/Healthy, no pod was in a failed phase,
and all 13 attached Longhorn volumes were healthy. One volume was detached with
`unknown` robustness; that state alone is not an update blocker, but its PVC and
backup ownership should be confirmed before cleanup.

Updates are available. They should be split into separate change windows.

### Patch queue

These are compatible patch, chart, digest, or operator-tool updates. They still
require normal backup, validation, Git review, rollout observation, and rollback
preparation.

| Component | Deployed or installed | Latest checked | Action |
|---|---:|---:|---|
| Talos Linux | 1.13.6 | [1.13.7](https://github.com/siderolabs/talos/releases/tag/v1.13.7) | Upgrade one node at a time after updating `talosctl` and rebuilding the existing Factory schematic at 1.13.7. The patch also advances the bundled kernel, containerd, and CoreDNS. |
| Cilium chart/app | 1.19.5 | [1.19.6](https://github.com/cilium/cilium/releases/tag/v1.19.6) | Dedicated CNI patch window; verify LB-IPAM, L2 announcements, DNS, and ingress after rollout. |
| Argo CD chart | 10.1.3 | [10.1.4](https://github.com/argoproj/argo-helm/releases) | Chart-only patch; Argo CD application remains 3.4.5. |
| Semaphore | 2.18.27 | [2.18.28](https://github.com/semaphoreui/semaphore/releases/tag/v2.18.28) | Patch image in GitOps after PostgreSQL backup. |
| Grafana | 13.1.0 | [13.1.1](https://github.com/grafana/grafana/releases/tag/v13.1.1) | Patch image and digest; validate InfluxDB data source and dashboards. |
| Telegraf | 1.39.1 | [1.39.2](https://github.com/influxdata/telegraf/releases/tag/v1.39.2) | Patch all three references together and validate ping, SNMP, Kubernetes, and DNS series. |
| InfluxDB 2.x image digest | 2.9.1 at older pinned digest | 2.9.1 tag has a newer digest | Controlled digest refresh only; verify backup, startup, token auth, retention, and dashboard queries. This is not the 3.x migration. |
| Debian helper image digest | Debian 13 at older pinned digest | Debian 13 tag has a newer digest | Refresh the helper/init image digest and re-run the collector path. |
| UniFi LinuxServer image | 10.4.57-ls136 | 10.4.57-ls138 | Refresh the pinned digest after a `.unf` backup; upstream application version is unchanged. |
| Proxmox VE packages | manager 9.2.4, kernel 7.0.14-4 | manager 9.2.5, kernel 7.0.14-6 | 31 packages are available, including QEMU, firmware, firewall, Corosync libraries, backup client, and security packages. Use the safe-reboot runbook. |
| OpenMediaVault | 8.5.1 | 8.5.4 | Eight packages are available. Update outside a backup job and verify SFTP backup access afterward. |
| `talosctl` workstation client | 1.11.5 | 1.13.7 | Update before the Talos server patch; keep client and server on the same minor. |
| `kubectl` workstation client | 1.36.1 | [1.36.2](https://kubernetes.io/releases/download/) | Align with the server patch. |
| OpenTofu workstation | 1.12.3 | [1.12.5](https://github.com/opentofu/opentofu/releases/tag/v1.12.5) | Patch locally and pin the CI image rather than using `latest`. |

### Planned maintenance queue

These are not part of the patch queue. Each needs its own reviewed plan and
maintenance window.

| Component | Current | Available | Required treatment |
|---|---:|---:|---|
| OPNsense | 26.1.11_6 | [26.7.1](https://docs.opnsense.org/releases/CE_26.7.html) | Major firewall/OS upgrade. Export `config.xml`, confirm console access, review source NAT and legacy firewall-page migration notes, then validate DHCP, DNS, WireGuard, NAT, and WAN failback. Version 26.7.1 includes security fixes. |
| GitLab CE | 19.1.2-ce.0 | [19.2.0](https://about.gitlab.com/whats-new/19-2/) | One-minor stateful upgrade. Take GitLab and DR backups, read required stops, upgrade GitLab first, then validate repositories, CI, registry references, and Argo access. |
| GitLab Runner | 19.1.1 | 19.2.0 | Upgrade only after GitLab 19.2 is healthy; run a real Kubernetes-executor pipeline and MinIO cache test. |
| InfluxDB | 2.9.1 | [3.10.0](https://github.com/influxdata/influxdb/releases/tag/v3.10.0) | Keep 2.9.1. Treat 2.x to 3.x as a data/API migration, not an image bump. |
| MongoDB major | runtime 7.0.37 | 8.x/8.3 available | Keep the supported UniFi track until a tested UniFi/Mongo migration plan exists. Do not infer compatibility from the container tag. |
| PostgreSQL major | runtime 16.14 | newer majors available | Keep major 16 unless a separate database migration is justified. PostgreSQL 16.14 is the current 16.x patch. |

### Reproducibility fixes

These do not necessarily change running behavior, but they close future
uncontrolled-update paths.

1. Replace the CI images `ghcr.io/opentofu/opentofu:latest` and
   `ghcr.io/yannh/kubeconform:latest-alpine` with exact tags and digests. The
   checked targets are OpenTofu 1.12.5 and kubeconform 0.7.0.
2. Move the dashboard validation image from Python 3.13.5 to
   `python:3.13.14-alpine3.23` and pin its digest. The previously assumed
   Alpine 3.22 tag is not published for Python 3.13.14.
3. Replace `python:3.12-slim` in the DNS sync Dockerfile with an exact 3.12 patch
   and digest when that local image is next rebuilt.
4. Replace the unbounded OpenTofu/provider constraints with tested ranges. The
   lock files currently hold `bpg/proxmox` 0.111.1 and
   `siderolabs/talos` 0.11.0, both current stable versions. Do not select the
   provider's published `0.12.0-alpha.*` tags as stable updates.
5. Change `postgres:16-alpine` to `postgres:16.14-alpine@sha256:...` and
   `mongo:7.0` to `mongo:7.0.37@sha256:...`. The running containers are already
   on those patch versions, but the manifests do not make that state reproducible.

## Full stack status

### Hypervisor, operating systems, and storage

| Component | Observed | Latest or update signal | Status |
|---|---:|---:|---|
| Proxmox VE | 9.2.4 | 9.2.5 in configured repository | Update available; maintenance and reboot required because a newer PVE kernel is included. |
| Proxmox kernel | 7.0.14-4-pve | 7.0.14-6 package | Update available. Do not combine with Talos, Cilium, or storage changes. |
| OpenMediaVault | 8.5.1-1 | 8.5.4-1 package | Patch available; eight total packages. |
| TrueNAS | Not reachable by configured DNS name | Not verified | Restore management-name resolution or use the documented management address, then run the TrueNAS update check. |
| Longhorn | 1.12.0 | [1.12.0](https://github.com/longhorn/longhorn/releases/tag/v1.12.0) | Current. Do not upgrade alongside Talos or Cilium. |
| MinIO server | RELEASE.2025-09-07 | Docker Hub `latest` resolves to the same release | Current on the still-published community container track. A later source release exists but its matching Docker Hub tag was not published. |
| MinIO client | RELEASE.2025-08-13 | Docker Hub `latest` resolves to the same release | Current. |

The Proxmox package audit also found security and maintenance updates for
`fwupd`, Debian kernel tools, Python libraries, NTFS tools, PVE firmware,
`pve-firewall`, `pve-container`, Corosync libraries, Proxmox Backup Client, and
QEMU Server. Inspect the final package transaction before accepting it.
The package list also contains third-party `linux-xanmod` kernels even though the
host is running the PVE kernel. Confirm why they are installed and ensure the PVE
kernel remains the boot default; do not remove them as part of the routine patch
window without a separate dependency check.

### Cluster operating system and control plane

| Component | Observed | Latest checked | Status |
|---|---:|---:|---|
| Talos Linux | 1.13.6 | 1.13.7 | Patch available. |
| Kubernetes | 1.36.2 | [1.36.2](https://kubernetes.io/releases/) | Current. |
| containerd | 2.2.5, Talos-bundled | 2.2.6 in Talos 1.13.7 | Update only through the Talos patch. |
| CoreDNS | 1.14.2, Talos-bundled | 1.14.4 in Talos 1.13.7; 1.14.6 upstream | Update only through the Talos-supported bundle; do not force the standalone upstream version. |
| Cilium | 1.19.5 | 1.19.6 | Patch available. |
| Traefik chart/app | 41.0.2 / 3.7.6 | chart 41.0.2 / app 3.7.8 | The chart is current but has not adopted the app patch. Monitor; do not override the chart image ad hoc. |
| Argo CD chart/app | 10.1.3 / 3.4.5 | 10.1.4 / 3.4.5 | Chart patch available; app current. |
| Argo CD Redis/Dex dependencies | Redis 8.2.3 / Dex 2.45.1 | latest chart keeps 8.2.3 / 2.45.1 | Current on the chart-supported path. Do not override them independently. |
| metrics-server chart/app | 3.13.1 / 0.8.1 | chart 3.13.1; app 0.9.0 upstream | Chart current. Wait for the official chart to adopt 0.9.0 rather than overriding it. |
| Sealed Secrets | 0.38.4 | [0.38.4](https://github.com/bitnami-labs/sealed-secrets/releases/tag/v0.38.4) | Current. |
| Distribution registry | 3.1.1 | [3.1.1](https://github.com/distribution/distribution/releases/tag/v3.1.1) | Current. |

### Cluster applications and data services

| Component | Observed | Latest checked | Status |
|---|---:|---:|---|
| GitLab CE | 19.1.2-ce.0 | 19.2.0-ce.0 | Planned minor upgrade. |
| GitLab Runner | 19.1.1 | 19.2.0 | Upgrade after GitLab. |
| Homepage | 1.13.2 | [1.13.2](https://github.com/gethomepage/homepage/releases/tag/v1.13.2) | Current. |
| Backrest | 1.14.1 | [1.14.1](https://github.com/garethgeorge/backrest/releases/tag/v1.14.1) | Current. |
| borgmatic | 2.1.6 | [2.1.6](https://github.com/borgmatic-collective/borgmatic/releases/tag/2.1.6) | Current. |
| Grafana | 13.1.0 | 13.1.1 | Patch available. |
| InfluxDB | 2.9.1 | 2.9.1 on the 2.x track | Version current; controlled digest refresh available. 3.x is a migration. |
| Telegraf | 1.39.1 | 1.39.2 | Patch available. |
| Semaphore | 2.18.27 | 2.18.28 | Patch available. |
| PostgreSQL | live 16.14, manifest `16-alpine` | [16.14](https://www.postgresql.org/docs/16/release-16-14.html) | Runtime current; pin exact patch and digest. |
| MongoDB | live 7.0.37, manifest `7.0` | [7.0.37](https://www.mongodb.com/docs/manual/release-notes/7.0/) | Runtime current; pin exact patch and digest. 7.0.38 was marked upcoming, not stable, at audit time. |
| UniFi Network Application | 10.4.57-ls136 | 10.4.57-ls138 | Same app version; digest rebuild available. |
| Whoami | 1.11.0 | [1.11.0](https://github.com/traefik/whoami/releases/tag/v1.11.0) | Current. |
| BusyBox helpers | 1.38.0 | 1.38.0 | Current. |
| curl helpers | 8.21.0 | [8.21.0](https://github.com/curl/curl/releases/tag/curl-8_21_0) | Current. |

Locally built workloads use commit-like, dated, or application-specific tags and
have no public upstream version to compare. Audit them by checking that the live
image matches the GitLab `master` manifest, that the tag maps to a known source
commit, and that its Dockerfile base image is supported. The local working tree
must not be treated as deployed state when it is on a feature branch or contains
uncommitted changes.

### Network, edge, hardware, and firmware

| Component | Observed | Status |
|---|---:|---|
| OPNsense | 26.1.11_6 | Planned upgrade to 26.7.1. |
| Technitium DNS/DHCP | Version not exposed by the available read-only path | Check the dashboard update page and record installed/latest versions. Do not update before exporting the configuration and verifying secondary recovery. |
| UniFi AP firmware | Not queried | Check both APs in the controller; stage one AP at a time and verify adoption and client roaming. |
| Cisco 2960-X firmware | Not queried | Compare the exact model and boot image with Cisco's supported release and advisory pages. Console and config backup are required before change. |
| HPE iLO, system ROM, Smart Array, disks, NICs | Not queried | Quarterly firmware and health audit. Record iLO version, system ROM, P840/H240ar firmware, cache battery state, disk predictive failures, and NIC firmware. |
| Observium VM | Not queried | Confirm whether it remains operationally required; if retained, check application and guest OS packages. |
| TrueNAS | Unreachable by configured management name | Restore management access and run the built-in update check. Do not infer a target train. |

## Recommended update sequence

Do not combine these into one change.

1. Pin the CI tool images and database patch tags. This removes uncontrolled
   drift without changing application major versions.
2. Update workstation clients: `talosctl` 1.13.7, `kubectl` 1.36.2, and OpenTofu
   1.12.5.
3. Patch Proxmox packages in a dedicated host window and use the existing
   `scripts/proxmox-safe-reboot.sh` procedure. Verify all Talos VMs, storage,
   networking, and hardware sensors afterward.
4. Patch OpenMediaVault and verify the backup repository path.
5. Apply application patches separately: Semaphore, Grafana, Telegraf, then the
   UniFi digest rebuild. Observe one soak period between stateful changes.
6. Patch Argo CD chart, then Cilium. Keep Cilium isolated because it owns cluster
   networking and service load balancer announcements.
7. Upgrade Talos 1.13.6 to 1.13.7 one node at a time while preserving etcd quorum
   and Longhorn replica availability.
8. Schedule GitLab 19.2 plus Runner 19.2 as a stateful application window.
9. Schedule OPNsense 26.7.1 separately with local console access and a tested
   exported configuration.
10. Resolve and audit the currently unverifiable TrueNAS, Technitium, switch,
    AP, Observium, and HPE firmware layers.

## Recurring read-only SOP

Run this every two to four weeks and before any maintenance window.

### 1. Establish the source of truth

1. Read the workspace `AGENTS.md` and any nearer instructions.
2. Use a clean worktree of `homelab-k8s-migration` at the live GitLab `master`.
   Do not discard or overwrite another operator's dirty working tree.
3. Record UTC time, local time zone, current commit, remote `master`, and Argo CD
   revision. If they differ, resolve which revision is actually deployed before
   comparing versions.
4. Treat `homelab-showcase` as public output. Never copy internal addresses,
   credentials, MAC addresses, private hostnames, sealed-secret contents, tokens,
   or unredacted screenshots into this report.

### 2. Capture live cluster health and versions

From the deployment repository:

```bash
export KUBECONFIG="$PWD/kubeconfig"

kubectl get nodes -o wide
kubectl get pods -A --field-selector='status.phase!=Running,status.phase!=Succeeded'
kubectl get applications.argoproj.io -n argocd
kubectl -n longhorn-system get volumes.longhorn.io
helm list -A

kubectl get nodes -o json | jq -r '
  .items[] |
  [.status.nodeInfo.osImage,
   .status.nodeInfo.kubeletVersion,
   .status.nodeInfo.containerRuntimeVersion,
   .status.nodeInfo.kernelVersion] | @tsv'

kubectl get pods -A -o json | jq -r '
  .items[] | .metadata.namespace as $namespace |
  .spec.containers[] | [$namespace, .image] | @tsv' | sort -u
```

Stop the audit and report a blocker before proposing routine upgrades if a node
is NotReady, a pod is unexpectedly failed, an Argo application is not
Synced/Healthy, etcd is unhealthy, or an attached Longhorn volume is degraded.

### 3. Capture repository pins

```bash
rg -n --hidden \
  -g '!**/.git/**' \
  -g '!**/.terraform/**' \
  -g '!docs/**' \
  -g '!**/*sealed*.yaml' \
  '(^|[[:space:]])(image:|FROM |version[[:space:]]*=|targetRevision:)' .

tofu -version
kubectl version
talosctl version --client
helm version
```

Also inspect `.terraform.lock.hcl`, Helm releases, init containers, Jobs,
CronJobs, CI images, Dockerfile bases, and controller-injected images. A simple
search of Deployment containers is not complete coverage.

For floating database tags, check the real running binary:

```bash
kubectl -n lan-ops exec deploy/postgres -- postgres --version
kubectl -n unifi exec deploy/unifi-db -- mongod --version
```

### 4. Refresh official online release data

Use only primary upstream sources. Record both the version and the publication
date. Reject drafts, pre-releases, release candidates, beta tags, and alpha tags
unless the deployment intentionally follows that channel.

For GitHub-hosted projects:

```bash
gh api repos/OWNER/REPOSITORY/releases/latest \
  --jq '{tag: .tag_name, published: .published_at, url: .html_url}'
```

For Helm-managed software:

```bash
helm repo update
helm search repo cilium/cilium --versions
helm search repo traefik/traefik --versions
helm search repo longhorn/longhorn --versions
helm search repo argo/argo-cd --versions
helm search repo metrics-server/metrics-server --versions
```

Record chart version and application version separately. If the upstream
application is newer but the latest official chart has not adopted it, mark it
`monitor` instead of overriding the chart image.

For OCI images and digest rebuilds:

```bash
skopeo inspect --override-os linux --override-arch amd64 \
  docker://REGISTRY/IMAGE:TAG |
  jq '{Digest, Created, Labels}'
```

Use the project's official release and upgrade pages for GitLab, OPNsense,
PostgreSQL, MongoDB, Proxmox, OpenMediaVault, TrueNAS, and hardware firmware.
Do not use a search-result snippet, arbitrary blog, or container `latest` tag as
the sole authority for a stateful or infrastructure upgrade.

### 5. Check hosts without changing them

Read package state from the existing package cache. Do not run package upgrades
or reboots during the audit.

```bash
ssh proxmox 'pveversion; uname -r; apt list --upgradable 2>/dev/null'
ssh opnsense "pkg query '%n %v' opnsense; freebsd-version"
ssh omv 'dpkg-query -W openmediavault; uname -r; apt list --upgradable 2>/dev/null'
```

For TrueNAS, use `midclt call system.version` and the built-in update check once
management access works. For Technitium, UniFi device firmware, Cisco, HPE iLO,
and storage controllers, use their authenticated read-only management views.

### 6. Classify every result

Use exactly one status per component:

- `current`: deployed stable release matches the supported target.
- `patch`: compatible patch or digest refresh is available.
- `planned`: minor/major, stateful, network, storage, or OS migration.
- `monitor`: upstream application is newer but its supported packaging path is not.
- `pin`: runtime is current but the manifest or CI uses a mutable tag or unbounded constraint.
- `local`: self-built image; verify source commit, base image, and live manifest.
- `unverified`: access or authoritative version data is unavailable.
- `blocked`: health, backup, compatibility, or recovery prerequisite failed.

Never label a component current merely because its tag contains `latest`, a
major-only tag, or a digest. Resolve the actual application version.

### 7. Produce the next report

Update these sections in place:

1. Audit metadata and date.
2. Executive result and cluster health.
3. Patch queue.
4. Planned maintenance queue.
5. Reproducibility fixes.
6. Full stack status, including every previously unverified layer.
7. Recommended sequence.
8. Audit history.

For each proposed update, include current version, latest stable version,
official source, risk class, backup prerequisite, validation, and rollback or
restore boundary. Do not apply the update unless the user separately asks for
implementation.

## Agent handoff prompt

Use this prompt for the next audit:

```text
Repeat the homelab version audit using homelab-showcase/VERSION-AUDIT.md.
Read all applicable AGENTS.md and CLAUDE.md files first. Treat the live Argo CD
revision and the homelab-k8s-migration GitLab master branch as deployment truth;
do not assume the current local branch is deployed. Run the SOP read-only across
Kubernetes, Helm, Talos, Proxmox, OPNsense, OpenMediaVault, TrueNAS, Technitium,
UniFi, storage, CI tooling, Terraform/OpenTofu providers, all cluster images,
init containers, CronJobs, and Dockerfile bases. Check the latest stable versions
online using only official upstream release pages, official GitHub releases,
official Helm repositories, and OCI registry metadata. Exclude alpha, beta, RC,
and draft releases. Separate chart versions from application versions and
separate patch updates from major or stateful migrations. Preserve all unrelated
working-tree changes. Do not upgrade, reboot, delete, or change external systems.
Update VERSION-AUDIT.md with a dated, public-safe report, exact official links,
validation evidence, residual unknowns, and a risk-ordered maintenance sequence.
Never include internal addresses, private hostnames, credentials, secret data,
MAC addresses, or unredacted operational output.
```

## Audit history

| Date | Result |
|---|---|
| 2026-07-22 | Cluster healthy. Patch queue opened for Talos, Cilium, Argo CD chart, Semaphore, Grafana, Telegraf, UniFi digest, Proxmox, OpenMediaVault, and workstation tools. Planned windows opened for OPNsense 26.7 and GitLab/Runner 19.2. TrueNAS, Technitium, device firmware, and hardware firmware remain to be verified. |

## Core upstream references

- [Talos releases](https://github.com/siderolabs/talos/releases)
- [Kubernetes releases](https://kubernetes.io/releases/)
- [Cilium releases](https://github.com/cilium/cilium/releases)
- [Traefik releases](https://github.com/traefik/traefik/releases)
- [Longhorn releases](https://github.com/longhorn/longhorn/releases)
- [Argo CD releases](https://github.com/argoproj/argo-cd/releases)
- [Argo Helm chart releases](https://github.com/argoproj/argo-helm/releases)
- [metrics-server releases](https://github.com/kubernetes-sigs/metrics-server/releases)
- [GitLab releases](https://about.gitlab.com/releases/)
- [GitLab upgrade paths](https://docs.gitlab.com/update/upgrade_paths/)
- [PostgreSQL release notes](https://www.postgresql.org/docs/release/)
- [MongoDB 7.0 release notes](https://www.mongodb.com/docs/manual/release-notes/7.0/)
- [OPNsense releases](https://docs.opnsense.org/releases.html)
- [Proxmox VE package updates](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#system_software_updates)
- [OpenMediaVault releases](https://docs.openmediavault.org/en/stable/releases.html)
- [Grafana releases](https://github.com/grafana/grafana/releases)
- [InfluxDB releases](https://github.com/influxdata/influxdb/releases)
- [Telegraf releases](https://github.com/influxdata/telegraf/releases)
- [OpenTofu releases](https://github.com/opentofu/opentofu/releases)
