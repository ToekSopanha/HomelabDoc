# 21 — Prometheus, Exporters & Alertmanager (Metrics)

> Prometheus asks every machine "how are you?" every 30 seconds and stores the numbers, so you can graph them and get alerts.

## 1. What is it?

- **Metrics** are numbers over time: CPU %, free RAM, requests per second, disk used.
- **Prometheus** collects (**scrapes**) metrics over HTTP from **targets** and stores them in a time-series database.
- **Exporters** are small programs that turn a system's state into Prometheus metrics (on a `/metrics` web page).
- **Alertmanager** receives alerts from Prometheus and sends notifications (Telegram, Discord, email).
- **VictoriaMetrics** is a Prometheus-compatible alternative that uses much less RAM/disk — swap to it if memory is tight.

Comparison: Prometheus is a **nurse** doing rounds every 30 seconds, writing down heartbeat and temperature of every patient. Exporters are the **monitors attached to each patient**. Alertmanager is the **alarm bell**.

## 2. Why this homelab uses it

To see — and be warned — when CPU is overloaded, RAM is low, the thin pool is filling, a pod crashes, PostgreSQL is down, or a backup didn't run.

## 3. Key words

| Word | Meaning |
|---|---|
| **Scrape** | Prometheus downloads `/metrics` from a target |
| **Target** | One endpoint to scrape, e.g., `192.168.0.225:9100` |
| **Job** | A group of similar targets |
| **Time series** | One metric + labels, e.g., `node_memory_MemAvailable_bytes{instance="worker1"}` |
| **Label** | Key/value tag for filtering: `instance`, `job`, `namespace` |
| **PromQL** | Query language: `rate(http_requests_total[5m])` |
| **Retention** | How long data is kept (15 days here) |
| **Alert rule** | PromQL condition + duration, e.g., "CPU > 90% for 10m" |
| **kube-prometheus-stack** | Helm chart bundling Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, and ready-made dashboards/alerts |
| **ServiceMonitor / PodMonitor** | Kubernetes objects telling Prometheus what to scrape |
| **PrometheusRule** | Kubernetes object holding alert rules |

### Exporters in this lab

| Exporter | Runs on | Port | What it measures |
|---|---|---|---|
| **node_exporter** | every VM/LXC + Proxmox host (and K3s nodes via the chart) | 9100 | CPU, RAM, disk, network |
| **kube-state-metrics** | K3s | — | Deployments, pods, restarts |
| **pve-exporter** | K3s pod → Proxmox API | 9221 | Host + every guest, storage/thin pool |
| **postgres_exporter** | LXC 105 | 9187 | Connections, DB size, locks |
| **Jenkins Prometheus plugin** | VM 106 | 8080 `/prometheus` | Jobs, queue, executors |
| **Traefik** | built-in | 9100 (in pod) | Requests, status codes, latency |

## 4. How it works

```text
                ┌──────────── Prometheus (pod in K3s) ─────────────┐
 scrape /metrics│  every 30s                                       │──► Grafana (graphs)
                ▼                                                  │──► Alertmanager ──► Telegram
 node_exporter .220–.227:9100   pve-exporter   postgres_exporter   jenkins /prometheus
```

Prometheus **pulls** data. Targets don't send it.

## 5. Step by step for this project

1. **Install exporters on guests** with Ansible ([17](../17-ansible/README.md)), e.g. collection `prometheus.prometheus` role `node_exporter`. Quick manual test:
   ```bash
   sudo apt install -y prometheus-node-exporter
   curl -s localhost:9100/metrics | head
   ```
   PostgreSQL LXC: `apt install prometheus-postgres-exporter` and give it a read-only monitoring role.
2. **Proxmox API token for pve-exporter** (read-only):
   ```bash
   pveum user add monitoring@pve
   pveum acl modify / --users monitoring@pve --roles PVEAuditor
   pveum user token add monitoring@pve exporter --privsep 0
   ```
3. **Install kube-prometheus-stack with Flux** — `infrastructure/monitoring/kube-prometheus-stack.yaml` (values excerpt):
   ```yaml
   apiVersion: helm.toolkit.fluxcd.io/v2
   kind: HelmRelease
   metadata: { name: kube-prometheus-stack, namespace: monitoring }
   spec:
     interval: 30m
     chart:
       spec:
         chart: kube-prometheus-stack
         version: "<pin a current major>"
         sourceRef: { kind: HelmRepository, name: prometheus-community, namespace: flux-system }
     values:
       # K3s runs these inside one process; default scrapes would fail
       kubeControllerManager: { enabled: false }
       kubeScheduler: { enabled: false }
       kubeProxy: { enabled: false }
       kubeEtcd: { enabled: false }
       prometheus:
         prometheusSpec:
           retention: 15d
           retentionSize: 18GB
           scrapeInterval: 30s
           priorityClassName: monitoring-low
           nodeSelector: { kubernetes.io/hostname: k3s-worker1 }   # pin: local-path PV
           resources:
             requests: { memory: 768Mi }
             limits:   { memory: 1536Mi }
           storageSpec:
             volumeClaimTemplate:
               spec:
                 storageClassName: local-path
                 accessModes: [ReadWriteOnce]
                 resources: { requests: { storage: 20Gi } }
           additionalScrapeConfigs:
             - job_name: vms
               static_configs:
                 - targets:
                     - 192.168.0.200:9100   # proxmox host
                     - 192.168.0.220:9100
                     - 192.168.0.225:9100
                     - 192.168.0.226:9100
                     - 192.168.0.227:9100
             - job_name: postgres
               static_configs: [{ targets: ["192.168.0.225:9187"] }]
             - job_name: jenkins
               metrics_path: /prometheus
               static_configs: [{ targets: ["192.168.0.226:8080"] }]
       alertmanager:
         alertmanagerSpec:
           resources: { limits: { memory: 128Mi } }
   ```
   (The chart already scrapes K3s nodes via its own node-exporter DaemonSet.)
4. **pve-exporter**: deploy `prometheus-pve-exporter` as a small Deployment with the token in a SOPS secret, and add a scrape job pointing at it with `params: { target: [192.168.0.200] }`.
5. **Alert rule example** (PrometheusRule):
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata: { name: homelab, namespace: monitoring }
   spec:
     groups:
       - name: host
         rules:
           - alert: HostHighCPU
             expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
             for: 10m
             labels: { severity: warning }
             annotations: { summary: "CPU > 90% on {{ $labels.instance }}" }
           - alert: DiskAlmostFull
             expr: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) > 0.85
             for: 15m
             labels: { severity: critical }
   ```
6. **Alertmanager → Telegram/Discord/email**: configure a receiver in the chart's `alertmanager.config` (bot token in a SOPS secret).
7. **Check targets:** `kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090` → open `http://localhost:9090/targets` — all should be **UP**.

## 6. Useful PromQL

```promql
up == 0                                                     # targets that are down
100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100   # CPU %
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100              # free RAM %
sum by (namespace)(container_memory_working_set_bytes{container!=""})          # RAM per namespace
increase(kube_pod_container_status_restarts_total[1h]) > 0                      # restarting pods
```

## 7. Common mistakes

- Too many labels (e.g., user ID in a label) → memory explodes ("high cardinality").
- No retention size → disk fills.
- Scraping every 5 s "for better graphs" → 6× more data. 30 s is enough.
- Firewall blocks port 9100 from the workers → target `DOWN`.
- Alerts that fire all the time → you start ignoring them. Tune thresholds.

## 8. Check yourself

1. Does Prometheus push or pull?
2. What does an exporter do?
3. Which exporter tells you the Proxmox thin pool usage?

<details><summary>Answers</summary>

1. Pull (scrape).
2. Translates a system's state into `/metrics` text for Prometheus.
3. pve-exporter (and node_exporter on the host can show LVM-related disk info too).

</details>

## 9. Learn more

- Prometheus docs: <https://prometheus.io/docs/introduction/overview/>
- kube-prometheus-stack: <https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack>
- PromQL basics: <https://prometheus.io/docs/prometheus/latest/querying/basics/>
- pve-exporter: <https://github.com/prometheus-pve/prometheus-pve-exporter>
- VictoriaMetrics: <https://docs.victoriametrics.com/>
