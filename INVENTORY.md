# Software inventory

Everything the homelab runs or is built with, grouped by layer, with upstream links.
Versions are the exact tags pinned in the GitOps manifests at the time of writing; the
repository is the source of truth.

Image digests (`@sha256:...`) are pinned in-repo for several components (Grafana,
InfluxDB, Telegraf, Homepage, UniFi, OpenSpeedTest, borgmatic, sealed-secrets, debian base)
— omitted here for readability.

## Hypervisor & infrastructure-as-code

| Software | What it does here | Link |
|---|---|---|
| Proxmox VE 9 | Hypervisor on the ML350 Gen9; all VMs and storage tiers | https://www.proxmox.com/en/proxmox-virtual-environment |
| OpenTofu | Declares the Talos VMs, ISO, and Proxmox jump host | https://opentofu.org/ |
| bpg/proxmox provider | Terraform/OpenTofu provider used for VM definitions | https://github.com/bpg/terraform-provider-proxmox |
| siderolabs/talos provider | Machine config + bootstrap as code | https://github.com/siderolabs/terraform-provider-talos |
| QEMU guest agent | Baked into the Talos image for Proxmox integration | https://pve.proxmox.com/wiki/Qemu-guest-agent |

## Kubernetes platform

| Software | What it does here | Link |
|---|---|---|
| Talos Linux v1.13.10 | Immutable, API-only Kubernetes OS — no SSH, no shell | https://www.talos.dev/ |
| Talos Image Factory | Builds the boot image with system extensions (iscsi-tools, util-linux-tools, qemu-guest-agent) | https://factory.talos.dev/ |
| Kubernetes v1.36.4 | The cluster itself (3× combined control-plane/worker, etcd HA) | https://kubernetes.io/ |
| Cilium 1.20.1 | CNI, kube-proxy replacement, LB-IPAM + L2 announcements, Hubble relay + UI for flow observability | https://cilium.io/ |
| Traefik v3.7.13 (chart 41.5.0) | Ingress controller for the internal wildcard zone | https://traefik.io/traefik/ |
| Longhorn 1.12.1 | Replicated block storage / CSI, default StorageClass (2 replicas, least-effort auto-balance) | https://longhorn.io/ |
| Argo CD v3.5.2 (chart 10.8.2) | GitOps — ApplicationSet generates an app per manifest directory | https://argo-cd.readthedocs.io/ |
| Sealed Secrets 0.39.1 | Encrypted secrets committed to git | https://github.com/bitnami-labs/sealed-secrets |
| metrics-server (chart 3.14.0, app 0.9.0) | `kubectl top` / resource metrics | https://github.com/kubernetes-sigs/metrics-server |
| Distribution (registry 3.1.1) | In-cluster image registry fed by CI | https://github.com/distribution/distribution |
| kaniko | In-cluster image builds from GitLab CI (no Docker daemon) | https://github.com/GoogleContainerTools/kaniko |
| Helm | Installs the pinned platform charts | https://helm.sh/ |
| kubeconform | Manifest validation in CI | https://github.com/yannh/kubeconform |

## DevOps & source control

| Software | What it does here | Link |
|---|---|---|
| GitLab CE 19.3.1 | Self-hosted source control + CI, running on the cluster | https://about.gitlab.com/install/ce-or-ee/ |
| GitLab Runner 19.3.1 | Kubernetes-executor runner, also on-cluster | https://docs.gitlab.com/runner/ |
| Silo | S3-compatible object storage: runner cache + Longhorn backup target. Maintained fork of MinIO, adopted after `minio/minio` was archived upstream and left a CVE permanently unpatched | https://github.com/pgsty/silo |
| Semaphore | Ansible automation UI (+ PostgreSQL) | https://semaphoreui.com/ |
| restic / Backrest | Backups with a web UI, NAS SFTP repository | https://restic.net/ / https://github.com/garethgeorge/backrest |

## Network & edge

| Software | What it does here | Link |
|---|---|---|
| OPNsense | Router / firewall / WAN edge, Unbound resolver | https://opnsense.org/ |
| Technitium DNS Server | Sole LAN DHCP + DNS authority; `.lan` wildcard → Traefik | https://technitium.com/dns/ |
| Unbound | Recursive resolution at the edge | https://nlnetlabs.nl/projects/unbound/about/ |
| WireGuard | Remote access VPN with LAN DNS over the tunnel | https://www.wireguard.com/ |
| UniFi Network Application | Wi-Fi controller for U6/U7 Pro APs (linuxserver.io image + MongoDB) | https://ui.com/download/releases/network-server |
| Squid | Forward proxy for LAN devices | https://www.squid-cache.org/ |
| OpenSpeedTest | Browser-based LAN/WAN speed test | https://github.com/openspeedtest/Speed-Test |
| Observium | Legacy SNMP network monitoring VM (kept, pruned) | https://www.observium.org/ |

## Storage & NAS

| Software | What it does here | Link |
|---|---|---|
| OpenMediaVault | NAS VM — same-host backup staging | https://www.openmediavault.org/ |
| TrueNAS | Separate physical box (ZFS) — off-box backup target | https://www.truenas.com/ |

## Observability

| Software | What it does here | Link |
|---|---|---|
| Grafana | Dashboards: latency, host power/thermals, cluster evidence | https://grafana.com/oss/grafana/ |
| InfluxDB | Time-series store for the telemetry stack | https://www.influxdata.com/ |
| Telegraf | Ping/SNMP collection agent | https://www.influxdata.com/time-series-platform/telegraf/ |

## Self-built apps (deployed via the same GitOps flow)

| App | What it does |
|---|---|
| gitscout | GitHub API scanner + dashboard (DuckDB); flags exposed credentials in accessible repos. Replaced apikey-monitor. |
| oil-dashboard | Oil/gas market intelligence dashboard — scrapers, scenario engine, FastAPI + APScheduler |
| freshrss | Self-hosted RSS/Atom reader; curated news corpus for a downstream summarization pipeline |
| market-stress | Market stress data collector + dashboard |
| MaxMind Search | GeoLite2 IP geolocation lookup web app |
| text-cleaner | Stateless text cleanup utility |
| lan-ops / dns-sync | LAN operations stack; CronJob syncing DNS records (Python) |
| whoami | Pilot/smoke-test app behind the LoadBalancer ([traefik/whoami](https://github.com/traefik/whoami)) |

Retired: **apikey-monitor** (archived — superseded by gitscout) and **Ollama Hunter**
(dashboard over an exposed-Ollama-endpoint scanner; removed).

## Operator tooling

| Software | Link |
|---|---|
| talosctl | https://www.talos.dev/latest/introduction/getting-started/ |
| kubectl | https://kubernetes.io/docs/reference/kubectl/ |
| Ansible | https://www.ansible.com/ |
| cluster-janitor | Custom Python script that finds orphaned cluster resources (PVCs, Secrets) unreferenced by any live workload |
| Python 3.13 + Ruff / Black / Mypy / Pytest | https://docs.astral.sh/ruff/ · https://black.readthedocs.io/ |
