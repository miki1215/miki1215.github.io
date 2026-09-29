---
layout: post
title: "Grafana Alloy, OTC CCE and Loki Pod Logs"
date: 2026-09-29 23:00:00 +0200
categories: kubernetes observability
---

# OTC CCE clusters ship no pod logs to Loki — the dangling /var/log/pods symlink (Alloy and Promtail)

[TOC]

> **TL;DR** — On OTC CCE clusters, kubelet writes pod logs to `/var/lib/containerd/container_logs/<namespace>_<pod>_<podUID>/<container>/0.log`, not to `/var/log/pods`. The `/var/log/pods` on CCE is only a **symlink** to that directory, and the symlink target is not mounted into the log collector container. Every file-based collector (Grafana Alloy, Promtail, Fluent Bit) that tails `/var/log/pods/...` therefore matches **zero files** and ships no pod logs at all. Only API-based sources (Kubernetes events) keep flowing, which makes the setup look half-healthy.

## Symptom

In Grafana's Loki label browser, an affected cluster shows only four labels:

- `cluster`, `instance`, `job`, `namespace`
- `job` has exactly one value: `integrations/kubernetes/eventhandler`
- `instance` looks like `loki.source.kubernetes_events.clu...` (the Alloy component ID)

Missing: `app`, `component`, `container`, `pod`, `node_name`, `filename` — everything that comes from pod log files. Events and node metrics still arrive, so the agent looks alive and no pod crashes.

## Root cause

CCE sets the kubelet `--container-log-dir=/var/lib/containerd/container_logs`. With a custom container-log-dir, kubelet names the per-pod directory `<namespace>_<pod>_<podUID>` instead of just `<podUID>`. On the node, `/var/log/pods` exists only as a symlink for compatibility:

```bash
# inside the alloy/promtail container (or on the node):
$ ls -la /var/log/ | grep pods
pods -> /var/lib/containerd/container_logs

# the collector container mounts only /var/log (+ maybe /var/lib/docker/containers),
# so the symlink target does not exist inside the container:
$ ls /var/log/pods/
# -> dangling symlink / No such file or directory

# the real files:
$ ls /host/root/var/lib/containerd/container_logs/ | head
kube-system_everest-csi-driver-dpp4m_447119b2-dfdd-472e-95d2-2a81d5b59e6c
monitoring_dih-alloy-2kqfc_4fa14819-f057-44d6-82cf-2fe6aba4185a
...
```

The standard scrape pattern `/var/log/pods/*<uid>/*.log` globs zero files. The collector emits no error (an empty glob is not an error), so this fails **silently**.

> **Note:** This is not an Alloy regression: Promtail on CCE had the same blind spot. The migration only made it visible because the old non-OTC clusters (EKS/RKE/AKS, real `/var/log/pods`) show the full label set next to the crippled CCE ones.

## Why events and node metrics kept working

- `loki.source.kubernetes_events` (eventhandler) talks to the API server — no files involved.
- `prometheus.exporter.unix` reads `/host/proc` and `/host/sys` — mounted explicitly.
- OTLP receivers listen on ports — no files involved.

Only the file-tail pipeline (`discovery.kubernetes` -> `discovery.relabel` -> `local.file_match` -> `loki.source.file`) depends on `/var/log/pods`, and it is exactly the one that produces the missing labels.

## Fix (Grafana Alloy, River config)

Keep the shared relabel rules, then fork off a second path-only relabel for the CCE layout and feed both into `local.file_match` via `concat()`. Reading through the existing `/host/root` (rootfs) mount means **no new hostPath volume** is needed. On non-CCE clusters the CCE glob simply matches nothing, so one config serves all clusters.

```river
// existing shared rules stay in discovery.relabel "pods" (app, component,
// namespace, node_name, pod, container, __path__ = /var/log/pods/*$1/*.log, ...)

// OTC CCE layout: kubelet container-log-dir=/var/lib/containerd/container_logs,
// dirs named <namespace>_<pod>_<podUID>; /var/log/pods is a symlink to it that
// dangles inside this container. Read via the rootfs (/host/root) mount.
discovery.relabel "pods_cce" {
  targets = discovery.relabel.pods.output

  rule {
    source_labels = ["__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
    separator     = "/"
    replacement   = "/host/root/var/lib/containerd/container_logs/*$1/$2/*.log"
    action        = "replace"
    target_label  = "__path__"
  }
}

local.file_match "pods" {
  path_targets = concat(discovery.relabel.pods.output, discovery.relabel.pods_cce.output)
  sync_period  = "5s"
}
```

Requirements, already present in the standard DIH Alloy values: the `rootfs` hostPath (`/` mounted at `/host/root`, `mountPropagation: HostToContainer`, readOnly) and `runAsUser: 0`. CCE writes CRI-format log lines, so the existing `stage.cri {}` keeps parsing them unchanged.

## Fix (Promtail, same cluster type)

Promtail needs the directory mounted plus a second path rule. In the promtail helm values add a read-only hostPath volume for `/var/lib/containerd/container_logs` and extend the scrape config's relabeling so a second target per pod carries `__path__ = /var/lib/containerd/container_logs/*<uid>/<container>/*.log` (same pattern as the Alloy fix above).

## Verification

```bash
# 1) confirm the layout on the node (from any privileged pod with / mounted):
ls -la /host/root/var/log/ | grep pods        # symlink -> /var/lib/containerd/container_logs
ls /host/root/var/lib/containerd/container_logs/ | head

# 2) after the fix rolls out, Loki label browser must show
#    app, component, container, pod, node_name, ... again for the cluster.
# 3) quick LogQL probe:
{cluster="<cce-cluster>", app=~".+"} | __error__=""  # should return lines
```

## Related gotchas found in the same audit

- **`alloy.extraPorts`, not top-level `extraPorts`** — the grafana/alloy chart templates both container ports and Service ports from `.Values.alloy.extraPorts` only (see `templates/service.yaml`). A values-root `extraPorts` is silently ignored, so the Service never exposes the OTLP 4317/4318 ports.
- **`connection refused` on `prometheus.remote_write` right after cluster bootstrap** — a Service with zero ready endpoints gets REJECTed by kube-proxy (ECONNREFUSED, not a timeout). While Prometheus is still starting this is expected; the remote_write WAL buffers and replays. Only investigate if it persists after the endpoints are populated: `kubectl get endpoints <svc> -n <ns>`.
- **Persist the Alloy WAL/positions** — set `alloy.storagePath: /alloy-data` backed by a hostPath (`/var/lib/alloy-data`, `DirectoryOrCreate`). Otherwise every pod restart re-tails files (duplicate logs) and drops the remote_write buffer.

## References

- Alloy chart: [grafana/alloy - operations/helm/charts/alloy](https://github.com/grafana/alloy/tree/main/operations/helm/charts/alloy)
- Alloy `loki.source.file` / `discovery.relabel` docs: [grafana.com/docs/alloy](https://grafana.com/docs/alloy/latest/reference/)
- DIH implementation: GitLab `dih/platform/dih-alloy` (Fleet bundle) and `dih/lila/buildandoperate-crossplane` (OTC CCE composition, release-alloy)
