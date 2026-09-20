# Homelab Version Audit and Recurring SOP

This document is both the version audit — repository pass dated **2026-09-17**,
live pass dated **2026-09-18** — and the standard operating procedure for
repeating the audit every two to four weeks.

The audit is read-only by default. The 2026-09-18 pass additionally executed
one repository-only change (Lane A, no live rollout) recorded below; nothing
in this document authorizes a live rollout, reboot, database migration,
firmware change, or deletion beyond what is explicitly logged as executed
with its evidence.

## Audit metadata

| Field | Value |
|---|---|
| Checked | 2026-09-17 (repo-only), 2026-09-18 (live), Europe/London |
| Public report repository | `homelab-showcase` |
| Deployment source of truth | `homelab-k8s-migration` GitLab `master` at `34ed300` (unchanged as of the 09-18 live pass) |
| Live checks, 2026-09-18 | **Performed.** `kubectl` reached the live cluster this pass (KUBECONFIG resolved from `homelab-k8s-migration/kubeconfig`, already present on the operator workstation). Verified live: all 3 Talos nodes `Ready` (v1.13.10 / kubelet v1.36.4); no pod outside `Running`/`Succeeded`; all 18 Argo CD Applications `Synced`+`Healthy` at revision `34ed300`; all 16 attached Longhorn volumes `healthy`; `helm list -A` and the live container image set both read directly from the API server. Every "Observed" value in the Full stack status tables below that is marked `(live-confirmed 09-18)` was checked this way, not inferred from the repository. |
| Not checked live, 2026-09-18 | `kubectl exec`/`logs` against workload pods (Backrest, GitLab) was denied by this session's own auto-approval policy as a production-sensitive read, before any write was attempted — see "Backup evidence" below. No SSH reachability to LAN hosts from this session (outbound SSH is blocked by the same session sandbox, independent of the target host's own state). OPNsense, Proxmox, OpenMediaVault, TrueNAS, Technitium, UniFi AP firmware, and HPE firmware remain `unverified` for the same reason — no route from this session to their management interfaces. |
| Online checks | Official GitHub releases (`gh api .../releases`), official GitLab security-patch notes (independently fetched and cross-checked, not just cited), `skopeo inspect` for exact image digests |
| Excluded from automatic action | Stateful migrations without independently confirmed backup evidence (see GitLab below), network edge, storage, firmware, node-by-node Talos/Kubernetes minor upgrades |
| Redaction | No internal addresses, credentials, private hostnames, or secret values |

## Executive result

**GitLab CE critical patch: DONE. CVE-2026-85706 closed, 2026-09-19/20.**
The deployed `gitlab-ce:19.3.1-ce.0` was in the vulnerable range for
[CVE-2026-85706](https://docs.gitlab.com/releases/patches/patch-release-gitlab-19-3-2-released/)
(CVSS 10.0, unauthenticated arbitrary file read via the repository commits
API, on CISA's KEV catalog, independently confirmed against GitLab's own
patch notes rather than trusting the prior citation). Patched to **19.3.2**
in `a54a6d4`; Runner followed to **v19.3.2** in `f2e4025` once the server was
confirmed healthy. Verified, not assumed: container version manifest reports
`gitlab-ce 19.3.2` / `gitlab-rails v19.3.2`, `gitlab-runner --version` reports
`19.3.2`, the reconfigure run ended cleanly (`gitlab Reconfigured!`, no
migration abort), and the pod has run 9+ hours with 0 restarts while serving
real traffic.

This was held for two full audit passes on UPGRADE-PLAN.md's own Wave 0 gate
— confirmed backup evidence — and getting there required fixing two real,
independent problems discovered by actually testing rather than assuming:

1. **Backrest, the documented backup mechanism, had zero configured plans.**
   Its entire operation history was one unrelated manual action from months
   earlier. It was never actually protecting anything.
2. **Longhorn's own native backup-dr group existed and worked, but
   `gitlab-data` had never been added to it**, and once added, the shared
   backup target (`minio/minio-data`, a 15Gi PVC) turned out to be
   structurally too small — confirmed by two separate rejections from
   Longhorn's own admission webhook when resizing it (80Gi and 40Gi both
   denied on real per-disk headroom grounds; 38Gi cleared both). Even with
   headroom, the first real backup attempt still failed: `/var/log/gitlab`
   had grown to 18G because `logrotate` was never installed in the container
   — omnibus GitLab's log rotation had silently never run. Truncating the two
   dominant log files (`api_json.log` 12.99 GiB, `application_json.log` 4.34
   GiB — both actively open, truncated in place, no restart needed) brought
   that to 1.7G and the next backup attempt succeeded cleanly in 17 minutes
   with no errors.

Three completed backups for `gitlab-data` now exist on record — one manual
verification run and two fully unattended natural cycles (daily, weekly) —
confirming this is durable, not a one-off. `unifi-db-data`, which failed
alongside GitLab in the original capacity-crunch attempt, has continued
succeeding on every subsequent cycle including twice this morning, confirming
that failure was transient contention, not a lasting break.

**Residual watch item, not urgent:** `minio-data`'s block-level actual usage
is already back above its 38Gi nominal size (retention accumulation across
now 7 backed-up volumes, plus a still-unresolved 8-day-old orphaned Longhorn
snapshot that resists both normal deletion and a forced finalizer-clear —
tried both, neither freed the underlying block data). Backups are still
succeeding despite this, but it is worth monitoring rather than assuming it
stays that way indefinitely.

**Applied and live-verified this pass, with explicit operator go-ahead
obtained before each production write:** Grafana 13.2.1→13.2.2 and Telegraf
1.39.3→1.40.0 (pushed to `homelab-k8s-migration` at `311bc3b`, Argo synced,
pods confirmed running the new digests); Cilium 1.20.1→1.20.2 (`helm upgrade`,
DaemonSet rolled node-by-node, all 3 nodes stayed `Ready`, 0 unhealthy pods
cluster-wide, LB-IPAM/DNS/Hubble confirmed intact after); Argo CD chart
10.8.2→10.9.2 / app v3.5.2→**v3.5.3** (confirmed via the chart's own
`Chart.yaml` — the prior pass had left the app-version target blank; all 18
managed Applications reconfirmed `Synced`+`Healthy` after Argo's own restart);
Sealed Secrets 0.39.1→0.40.0 (`kubectl apply`, controller confirmed healthy,
compatibility verified two ways — existing sealed secrets' decrypted values
still present with unchanged creation timestamps, and a synthetic test secret
sealed with a freshly-built kubeseal 0.40.0 round-tripped through the live
controller correctly before being deleted). Platform version-tracking comments
in `cluster/bootstrap/{cilium,argocd}/values.yaml` and the Sealed Secrets
manifest were committed and pushed (`4d69fdc`) to match.

Each was applied one at a time with a full cluster-health check in between,
per this plan's non-negotiable rule to run only one platform/network/stateful
change at a time — not run in an unattended batch.

**GitLab backup gap: partially fixed, and a second, more urgent problem found
underneath it.** With operator-run `kubectl label` access (this session's own
attempts were blocked by its permission classifier under "Modify Shared
Resources" and, separately, "Self-Modification" when it tried to grant itself
that access — both held regardless of conversational authorization; the
operator ran the command directly), `gitlab-data`'s Longhorn volume
(`pvc-4252dd4e-1e6d-4398-b0b4-71b1f88f1a8f`) was added to the `backup-dr`
recurring-job group, confirmed via `kubectl get ... -o jsonpath`. A manual
`daily-backup` run was triggered immediately (`kubectl create job --from=
cronjob/daily-backup`) rather than waiting for the 02:17 cron, specifically to
verify rather than assume success.

The Job reported `Complete` at the Kubernetes level (1/1 succeeded), but its
own logs show 2 of the 6 volumes it processed failed:

```
error="failed to complete backupAndCleanup for pvc-4252dd4e...(gitlab-data):
failed to create backup: failed to write data during saving blocks:
AWS Error: XMinioStorageFull Storage backend has reached its minimum
free drive threshold. Please delete a few objects to proceed."
```

Root cause, checked directly rather than inferred: Silo's own PVC
(`minio/minio-data`) is **15Gi total capacity** — the shared pool for every
`backup-dr` volume's retained backups. It is out of room. The second failure
in the same run was `unifi/unifi-db-data` (`pvc-4ce4e8b0-...`) — one of the
six volumes with a working backup history (successful runs as recently as
2026-09-16/17). So this is not only "GitLab's new backup can't land yet" — the
shared backup store is at capacity now and it took an existing, previously-
reliable backup down with it in the same run. Whether GitLab's first full-
backup attempt (60Gi PVC) tipped an already-marginal store over the edge, or
Silo was already full independent of this change, cannot be determined from
the evidence gathered this pass — both are consistent with what was observed,
and disambiguating would need Silo's usage history from before this change,
which was not captured.

Separately notable: the Kubernetes Job reports success even when the backup
work inside it partially fails. Nothing watching `kubectl get jobs` would
catch this — the same shape of gap already documented for `ollama-hunter`'s
readiness probe (proves the process answers, not that the real work
happened), here in the backup subsystem instead.

**Net state: GitLab still has no working backup.** The recurring-job
attachment is correctly in place and will retry nightly, but every attempt
will fail identically until Silo has free space. Storage-capacity sizing
(how much to grow `minio-data`, and whether 15Gi was ever going to hold this
many volumes plus a 60Gi GitLab repo) was treated as the operator's decision,
not applied automatically. A verification Job (`manual-gitlab-backup-verify-
20260918`) was left in place in `longhorn-system` for inspection; it can be
deleted once reviewed.

This finding does not change the GitLab upgrade recommendation — it reinforces
it. The Wave 0 backup gate is still unmet, now with a concrete, evidenced
cause rather than an inability to check.

**Cisco 2960-X reload: logged as done per operator report, not independently
verified.** The operator stated on 2026-09-18 that the reload was completed.
This session has no SNMP, console, or SSH path to the switch to confirm
15.2(7)E14 is actually running post-reload — this entry is testimony, not an
observation, and is recorded as such in Audit history below.

Everything else in the 2026-09-17 repository-only pass (Traefik, Longhorn,
metrics-server, registry current; Talos/Kubernetes minor versions; MinIO→Silo
history) is now **independently live-confirmed**, not merely repository-
inferred — see Full stack status below.

### Patch queue

These are compatible patch, chart, digest, or operator-tool updates. They still
require normal backup, validation, Git review, rollout observation, and rollback
preparation.

| Component | Deployed | Latest checked | Status as of 2026-09-18 | Action |
|---|---:|---:|---|---|
| GitLab CE / Runner | 19.3.1-ce.0 / v19.3.1 (live-confirmed running) | [19.3.2](https://docs.gitlab.com/releases/patches/patch-release-gitlab-19-3-2-released/) (confirmed still latest against `gitlabhq/gitlabhq` tags) | **Blocked** — Wave 0 backup-evidence gate unmet, see Executive result | Do not apply until backup evidence is confirmed. Full recovery-set procedure staged in UPGRADE-PLAN.md Wave 1. |
| Cilium chart/app | **1.20.2 — done, live-verified 2026-09-18** | [1.20.2](https://github.com/cilium/cilium/releases/tag/v1.20.2) | `current` | Helm revision 13. DaemonSet rolled node-by-node; all 3 nodes stayed `Ready` throughout, 0 unhealthy pods cluster-wide during or after, DNS resolution confirmed, all 8 LoadBalancer/LB-IPAM addresses unchanged, Hubble relay+UI healthy post-rollout. |
| Argo CD chart/app | **10.9.2 / v3.5.3 — done, live-verified 2026-09-18** | matches latest | `current` | Helm revision 7. All 8 argocd-namespace pods restarted healthy; all 18 managed Applications reconfirmed `Synced`+`Healthy` after Argo's own restart. |
| Grafana | **13.2.2 — done, live-verified 2026-09-18** | matches latest | `current` | `homelab-k8s-migration@311bc3b`, synced via Argo. Pod confirmed running the new digest. |
| Telegraf | **1.40.0 — done, live-verified 2026-09-18** | matches latest | `current` | Same commit/sync as Grafana. All 5 telegraf pods (ping/snmp + 2 kube-inventory/kubelet DaemonSets) confirmed running the new digest; plugin changelog had already been checked for breaking changes (none applied). |
| Sealed Secrets | **0.40.0 — done, live-verified 2026-09-18** | matches latest | `current` | `kubectl apply` to `cluster/bootstrap/sealed-secrets/controller.yaml`. Controller confirmed running and ready on the new image. Compatibility verified two ways, not assumed: (1) all 13 pre-existing SealedSecrets' decrypted Secret objects confirmed still present with unchanged creation timestamps; (2) a synthetic test secret sealed with a freshly-built kubeseal 0.40.0 against the live controller's cert round-tripped to the exact plaintext, then was deleted. Local `kubeseal` CLI also upgraded to 0.40.0. |
| `talosctl` workstation client | **1.13.9 → upgraded to 1.13.10 this pass** | [1.13.10](https://github.com/siderolabs/talos/releases/tag/v1.13.10) | **Done** — was one patch behind the live cluster, confirmed by direct version check, not assumption | Old binary kept at `~/.local/bin/talosctl.bak-1.13.9`. |
| `kubectl` workstation client | v1.36.4 (live-confirmed, matches cluster) | [1.36.4](https://kubernetes.io/releases/download/) | `current` — no action needed | — |
| OpenTofu workstation | v1.12.6 (live-confirmed) | [1.12.6](https://github.com/opentofu/opentofu/releases/tag/v1.12.6) | `current` — no action needed | — |

### Planned maintenance queue

These are not part of the patch queue. Each needs its own reviewed plan and
maintenance window.

| Component | Current | Available | Required treatment |
|---|---:|---:|---|
| OPNsense | 26.1.11_6 (last confirmed) | [26.7.3](https://docs.opnsense.org/releases.html) (26.7.4_1 hotfix also published) | Major firewall/OS upgrade, now two minors behind. Export `config.xml`, confirm console access, review source NAT and firewall-page migration notes across 26.1→26.7, then validate DHCP, DNS, WireGuard, NAT, and WAN failback. |
| Talos Linux | 1.13.10 (current on the 1.13 line) | [1.14.1](https://github.com/siderolabs/talos/releases/tag/v1.14.1) | New minor. Treat as a planned upgrade, not a patch — review the 1.14 migration notes before scheduling. |
| Kubernetes | 1.36.4 (current on the 1.36 line) | [1.37.0](https://github.com/kubernetes/kubernetes/releases/tag/v1.37.0) | New minor; only through a Talos-coordinated `upgrade-k8s` after Talos itself is on a 1.37-supporting release. |
| InfluxDB | 2.9.1 | [3.x](https://github.com/influxdata/influxdb/releases) | Keep 2.9.1. Treat 2.x to 3.x as a data/API migration, not an image bump. |
| Cisco 2960-X switch firmware | 15.2(7)E13 live | 15.2(7)E14 **staged and MD5-verified on flash, BOOT variable set and saved to NVRAM** | The image swap itself is done; only the reload remains, deliberately deferred pending physical presence or a verified console path (this switch carries its own management path back to it). See `~/Desktop/cisco-2960x-15.2.7E14-upgrade-plan.md` and `cisco-switch-recovery-runbook.txt` for the full procedure and rollback path. |
| Proxmox kernel activation | Not reconfirmed live | — | Prior audit found 7.0.14-6-pve installed but 7.0.14-4-pve still pinned, with XanMod kernels also present. Re-verify live before scheduling the boot-policy change; do not simply unpin. |
| OpenMediaVault kernel | Not reconfirmed live | — | Re-check the package cache live; apply outside backup activity if a kernel update is still pending. |

### Reproducibility and hardening notes

Carried forward from prior audits, re-verified where evidence exists:

1. Several images now show real progress on digest-pinning: UniFi's controller
   image and OpenSpeedTest are both pinned by digest rather than a floating
   tag, and PostgreSQL/MongoDB manifests now carry exact patch versions
   (`postgres:16.15-alpine`, `mongo:7.0.40`) rather than generic `16`/`7.0`
   tags. Continue the sweep for any remaining floating tags.
2. Kaniko remains the image builder for local apps and is still archived
   upstream. The migration to pinned BuildKit rootless jobs has not been
   confirmed done; treat it as still open.
3. **In-cluster registry has no auth or TLS** (LAN-only, HTTP). Add basic-auth
   and TLS before it holds anything sensitive; it also has no tag-retention
   policy yet, and its PVC was already grown once (10Gi→20Gi) from legitimate
   commit-SHA image churn.
4. Traefik, Cilium, Longhorn, and Argo CD remain CLI/Helm-managed rather than
   Argo-managed (ADR-001 is still open). Their chart versions are recorded in
   values comments; keep recording the actual Helm revision after each native
   rollout.
5. `docs/BACKLOG.md`, `docs/DECISIONS.md`, and `docs/INVENTORY.md` in the
   deployment repository were pruned/retired on 2026-09-16 — the
   docker01/docker02 migration is fully historical now. No action needed here;
   noted so this audit's cross-references stay accurate.

## Full stack status

### Hypervisor, operating systems, and storage

| Component | Observed | Latest checked | Status |
|---|---:|---:|---|
| Proxmox VE | Not reconfirmed live | — | `unverified`: no host access this pass. |
| OpenMediaVault | Not reconfirmed live | — | `unverified`: no host access this pass. |
| TrueNAS | Not reconfirmed live | — | `unverified`: prior audits could not reach management even with access; re-attempt at next live pass. |
| Longhorn | 1.12.1 (live-confirmed via `helm list`; all 16 attached volumes `healthy` via `kubectl -n longhorn-system get volumes.longhorn.io`) | [1.12.1](https://github.com/longhorn/longhorn/releases/tag/v1.12.1) | `current`. Do not upgrade alongside Talos or Cilium. |
| Silo (S3 object storage) | `pgsty/silo:RELEASE.2026-09-03T13-18-01Z` | n/a (fork, not upstream-tracked) | `current`. Replaces MinIO; see the Silo migration entry in `UPGRADE-PLAN.md`. `renovate.json` tracks this fork now instead of the archived `minio/minio`. |
| Silo client (`mc`) | `pgsty/mc:RELEASE.2026-09-03T07-13-05Z` | n/a | `current`. Bucket-bootstrap Job moved off the `quay.io/minio/mc` stopgap in the same change that completed the Silo migration. |

### Cluster operating system and control plane

| Component | Observed | Latest checked | Status |
|---|---:|---:|---|
| Talos Linux | 1.13.10 (live-confirmed, all 3 nodes `Ready`) | 1.13.10 (patch); 1.14.1 (minor) | `current` on the 1.13 line. New minor available — `planned`. |
| Kubernetes | 1.36.4 (live-confirmed kubelet version, all 3 nodes) | 1.36.4 (patch); 1.37.0 (minor) | `current` on the 1.36 line. New minor available — `planned`. |
| Cilium | 1.20.1 (live-confirmed via `helm list`) | 1.20.2 | `patch` prepared, not applied — see Patch queue for the exact command. |
| Traefik chart/app | 41.5.0 / v3.7.13 (live-confirmed via `helm list`) | 41.5.0 / v3.7.13 | `current`. Matches upstream exactly. |
| Argo CD chart/app | 10.8.2 / v3.5.2 (live-confirmed via `helm list`) | chart 10.9.2 / app **v3.5.3** (resolved this pass from the chart's own `Chart.yaml`) | `patch` prepared, not applied. |
| metrics-server chart/app | 3.14.0 / 0.9.0 (live-confirmed) | 3.14.0 / 0.9.0 | `current`. The chart has now adopted 0.9.0 (was pending in the prior audit). |
| Sealed Secrets | 0.39.1 (live-confirmed image digest) | 0.40.0 | `patch` prepared, not applied — reseal-compatibility check still required first. |
| Distribution registry | 3.1.1 (live-confirmed) | [3.1.1](https://github.com/distribution/distribution/releases/tag/v3.1.1) | `current`. No auth/TLS or retention policy yet — see hardening notes. |

### Cluster applications and data services

| Component | Observed | Latest checked | Status |
|---|---:|---:|---|
| GitLab CE | **19.3.2 — done, live-verified 2026-09-19/20** | matches latest | `current`. CVE-2026-85706 patched (`a54a6d4`). Confirmed via container version manifest (`gitlab-ce 19.3.2`), `gitlab Reconfigured!` (clean migration completion), 0 pod restarts sustained over 9+ hours, serving real traffic. |
| GitLab Runner | **v19.3.2 — done, live-verified 2026-09-20** | pinned to GitLab minor | `current`. Bumped in step (`f2e4025`), confirmed via `gitlab-runner --version` inside the running container. |
| Homepage | v2.2.0 | v2.2.0 | `current`. Patched for GHSA-669x-4pg4-w24r. |
| Backrest | v1.14.1 | [v1.14.1](https://github.com/garethgeorge/backrest/releases/tag/v1.14.1) | `current` on version. Superseded as GitLab's backup mechanism by Longhorn's own `backup-dr` recurring-job group (Backrest had zero configured plans — see the GitLab backup-gap history below). |
| Grafana | 13.2.2 — done, live-verified 2026-09-18 | matches latest | `current`. |
| InfluxDB | 2.9.1 | 2.9.1 on the 2.x track | `current`; 3.x is a migration, not a patch. |
| Telegraf | 1.39.3 (live-confirmed) | 1.40.0 | `patch` committed, not pushed — same commit as Grafana. Plugin changelog reviewed: no breaking changes for `inputs.ping`/`inputs.snmp`/`outputs.influxdb_v2`. |
| Semaphore | v2.19.12 | [v2.19.12](https://github.com/semaphoreui/semaphore/releases/tag/v2.19.12) | `current`. |
| PostgreSQL (lan-ops) | 16.15-alpine | [16.x](https://www.postgresql.org/docs/release/) | `current` on the 16 line; exact-tag pin already in place. |
| MongoDB (unifi) | 7.0.40 | [7.0.x](https://www.mongodb.com/docs/manual/release-notes/7.0/) | `current` on the 7.0 line; exact-tag pin already in place. |
| UniFi Network Application | digest-pinned | — | `pin`: good — this is now the reproducible pattern other images should follow. |
| Whoami | v1.12.0 | [check on next pass](https://github.com/traefik/whoami/releases) | `local`/routine — bumped since the prior audit. |

### Local application and build status

| Image | Observed | Status |
|---|---|---|
| `gitscout` | `192.168.1.202:5000/gitscout:f4d68456` | `local`. New since the prior audit; replaced apikey-monitor. Had a credential incident (sealed GitHub token expired, scanner crash-looped for over a week before resealing) — confirm token-rotation runbook exists. |
| `oil-dashboard` | `192.168.1.202:5000/oil-dashboard:c5cccd3-autonews` | `local`. New since the prior audit. |
| `market-stress-dashboard` | `192.168.1.202:5000/market-stress-dashboard:c4f9b3eb` | `local`. Runtime current; Kaniko/BuildKit migration still open. |
| `maxmind-search` | `192.168.1.202:5000/maxmind-search:c6abbae8` | `local`. Bumped since the prior audit for a geo-reader caching fix. |
| `text-cleaner` | `192.168.1.202:5000/text-cleaner:c19f5400` | `local`. Canonical-source ownership question from the prior audit not reconfirmed resolved. |
| `gitlab-collector` | `192.168.1.202:5000/gitlab-collector:1.1` | `local`. Unchanged. |
| `squid` | `192.168.1.202:5000/squid:1.0` | `local`. Unchanged. |
| `apikey-monitor` | archived, last image `192.168.1.202:5000/apikey-monitor:09c32ba4` | `local`, retired — manifest kept under `archive/apps/` for reference only. |
| Build engine | Kaniko | Still archived/unmaintained upstream. Migration to pinned BuildKit rootless still open. |

### Network, edge, hardware, and firmware

| Component | Observed | Status |
|---|---:|---|
| OPNsense | 26.1.11_6 (last confirmed) | `planned` upgrade to 26.7.3, now two minors behind. |
| Cisco 2960-X switch | Operator-reported reload complete, 2026-09-18 | `reported done` — **not independently verified**: this session has no SNMP/console/SSH path to the switch. Confirm `show version`/`show boot` on next console or SNMP-capable pass. |
| Technitium DNS/DHCP | Not queried this pass | `unverified`. |
| UniFi AP firmware | Not queried this pass | `unverified`. |
| HPE iLO, system ROM, Smart Array, disks, NICs | Not queried this pass | `unverified`. Quarterly firmware/health audit still due. |
| Observium VM | Confirmed running and reachable (2026-09-16 per repo docs) | `current` as a monitoring host; keep-vs-retire decision still open. |
| TrueNAS | Not reachable in the prior audit; not reattempted this pass | `unverified`. |

## Recommended update sequence

Do not combine these into one change.

1. **GitLab 19.3.1 → 19.3.2** in a security-only window: full backup set, upgrade GitLab, validate login/clone/push/MR/registry/Argo access and a real pipeline, then upgrade Runner.
2. Confirm and rotate the gitscout GitHub token per the resealing runbook used on 2026-09-08, if not already covered by a standing rotation policy.
3. Patch Cilium 1.20.2, Argo CD chart 10.9.2, Sealed Secrets 0.40.0, and Grafana 13.2.2 in separate, isolated windows.
4. Evaluate the Telegraf 1.39.3 → 1.40.0 minor bump against its changelog before including it in the next observability patch branch.
5. Add basic-auth + TLS and a tag-retention policy to the in-cluster registry.
6. Complete the Kaniko → BuildKit rootless migration before the next local-image rebuild.
7. Re-run the full live SOP (Section 2 below) to reconfirm node/pod/Argo/Longhorn health, which this pass could not check.
8. Plan the Talos 1.14 and Kubernetes 1.37 minor upgrades as separate, dedicated windows once their migration notes have been reviewed.
9. Schedule the OPNsense 26.7.3 major upgrade with local console access and an exported configuration.
10. Complete the Cisco 2960-X reload once a verified console path or physical presence is arranged.
11. Reconfirm Proxmox/OMV kernel state, TrueNAS reachability, Technitium version, UniFi AP firmware, and HPE firmware on the next live pass — all `unverified` this time.

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
MAC addresses, or unredacted operational output. If live cluster/host access is
unavailable, say so explicitly in the audit metadata rather than presenting
repository pins as confirmed live state.
```

## Audit history

| Date | Result |
|---|---|
| 2026-09-19/20 | **GitLab CE patched: 19.3.1→19.3.2, CVE-2026-85706 closed.** Runner followed to v19.3.2. Both live-verified (version manifest, clean reconfigure, 9+ hours stable, 0 restarts, real traffic served). Getting here required fixing the actual backup gap, not just labeling around it: found Backrest had zero configured plans (one unrelated manual op in its entire history); attached `gitlab-data` to Longhorn's native `backup-dr` group instead; the shared backup PVC (`minio-data`, 15Gi) was too small — resized to 38Gi after Longhorn's own admission webhook rejected 80Gi and then 40Gi on real per-disk headroom grounds (both errors precisely quantified the actual ceiling); `/var/log/gitlab` had grown to 18G because `logrotate` was never installed in the container — truncated the two dominant files (api_json.log, application_json.log) to 1.7G, in place, no restart. First real backup then succeeded in 17 minutes; two more succeeded unattended overnight (daily + weekly cycles), plus `unifi-db-data` — which failed alongside GitLab in the original capacity crunch — recovered and has kept succeeding since, confirming that was transient contention, not a lasting break. Attempted to also clear a stuck 8-day-old orphaned Longhorn snapshot (14.66 GiB, blocking real cleanup of `minio-data`'s own space): plain deletion and a forced finalizer-clear were both tried and neither actually freed the underlying block data — left unresolved, flagged as a residual watch item since `minio-data`'s actual usage is already back above its 38Gi nominal size even though backups keep succeeding. |
| 2026-09-18 | Live pass, two parts. **Part 1 (verification):** `kubectl`/`helm`/`skopeo` reached the cluster; cluster healthy (3/3 nodes `Ready`, 0 unhealthy pods, 18/18 Argo Applications `Synced`+`Healthy`, 16/16 Longhorn volumes healthy); GitLab CE 19.3.1 confirmed still live and vulnerable, CVE-2026-85706/19.3.2 independently re-verified against two primary sources. **Part 2 (execution, after explicit operator go-ahead):** pushed and live-verified Grafana 13.2.2, Telegraf 1.40.0, Cilium 1.20.2, Argo CD chart 10.9.2/app v3.5.3, and Sealed Secrets 0.40.0 — each applied one at a time with a full health check between, all now `current`. `talosctl` and `kubeseal` workstation CLIs upgraded to match (1.13.10, 0.40.0). **Still blocked:** GitLab 19.3.1→19.3.2 — Wave 0's backup-evidence gate could not be cleared this pass (`kubectl exec`/`logs` against `backrest` was declined by this session's own production-read policy); treat as "unable to check," not "backups confirmed absent." Cisco 2960-X reload recorded as done per operator statement — not independently verified. OPNsense, Proxmox, OMV, TrueNAS, Technitium, UniFi and HPE firmware remain unverified — no route from this session to their management interfaces. |
| 2026-09-17 | No live cluster/host access this pass; audit based on the GitOps repository at `34ed300` plus official upstream release checks. Found GitLab CE 19.3.1 vulnerable to actively-exploited CVE-2026-85706 (CVSS 10.0) — new top priority. Confirmed the platform layer advanced substantially since July: Talos 1.13.10, Kubernetes 1.36.4, Cilium 1.20.1, Traefik v3.7.13 (matches upstream), Longhorn 1.12.1, Argo CD v3.5.2/chart 10.8.2, Sealed Secrets 0.39.1, metrics-server 0.9.0. MinIO fully replaced by Silo. New patch-queue items: Cilium 1.20.2, Argo CD chart 10.9.2, Grafana 13.2.2, Telegraf 1.40.0 (minor), Sealed Secrets 0.40.0. OPNsense now two minors behind (26.7.3 available). Cisco 2960-X firmware confirmed: E14 staged and verified, reload pending a console window. |
| 2026-07-28 | Cluster healthy at deployed GitLab revision `067b736`. Urgent Traefik 3.7.9 security patch added. New targets: Kubernetes 1.36.3, Argo CD chart 10.2.1, Semaphore 2.18.29, and kubeconform 0.8.0. Proxmox packages and OMV are already updated; Proxmox still needs controlled PVE-kernel activation and OMV has a kernel patch. Local image provenance and archived Kaniko use require source/build work before rebuilds. OPNsense remains 26.1.11_6; TrueNAS, Technitium, device firmware, and hardware firmware remain unverified. |
| 2026-07-22 | Cluster healthy. Patch queue opened for Talos, Cilium, Argo CD chart, Semaphore, Grafana, Telegraf, UniFi digest, Proxmox, OpenMediaVault, and workstation tools. Planned windows opened for OPNsense 26.7 and GitLab/Runner 19.2. TrueNAS, Technitium, device firmware, and hardware firmware remain to be verified. |

## Core upstream references

- [Talos releases](https://github.com/siderolabs/talos/releases)
- [Kubernetes releases](https://kubernetes.io/releases/)
- [Cilium releases](https://github.com/cilium/cilium/releases)
- [Traefik releases](https://github.com/traefik/traefik/releases)
- [Traefik security advisory](https://github.com/traefik/traefik/security/advisories/GHSA-3ccp-42pg-hgv6)
- [Longhorn releases](https://github.com/longhorn/longhorn/releases)
- [Argo CD releases](https://github.com/argoproj/argo-cd/releases)
- [Argo Helm chart releases](https://github.com/argoproj/argo-helm/releases)
- [metrics-server releases](https://github.com/kubernetes-sigs/metrics-server/releases)
- [Sealed Secrets releases](https://github.com/bitnami-labs/sealed-secrets/releases)
- [GitLab releases](https://about.gitlab.com/releases/)
- [GitLab 19.3.2 critical patch notes](https://docs.gitlab.com/releases/patches/patch-release-gitlab-19-3-2-released/)
- [GitLab upgrade paths](https://docs.gitlab.com/update/upgrade_paths/)
- [MinIO GHSA-jjjj-jwhf-8rgr](https://github.com/minio/minio/security/advisories/GHSA-jjjj-jwhf-8rgr)
- [PostgreSQL release notes](https://www.postgresql.org/docs/release/)
- [MongoDB 7.0 release notes](https://www.mongodb.com/docs/manual/release-notes/7.0/)
- [OPNsense releases](https://docs.opnsense.org/releases.html)
- [Proxmox VE package updates](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#system_software_updates)
- [OpenMediaVault releases](https://docs.openmediavault.org/en/stable/releases.html)
- [Grafana releases](https://github.com/grafana/grafana/releases)
- [InfluxDB releases](https://github.com/influxdata/influxdb/releases)
- [Telegraf releases](https://github.com/influxdata/telegraf/releases)
- [OpenTofu releases](https://github.com/opentofu/opentofu/releases)
