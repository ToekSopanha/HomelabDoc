# 24 — Grafana Alloy (Telemetry Collector)

> A small agent that picks up logs (and optionally metrics) from where they're written and delivers them to Loki/Prometheus.

## 1. What is it?

**Grafana Alloy** is a collector. It reads logs from files, the systemd journal, or Kubernetes pods, adds labels, and **pushes** them to Loki. It replaces the older **Promtail** (which is deprecated) and **Grafana Agent**.

Comparison: a **mail carrier**. Every few seconds it picks up new letters (log lines) from each house and delivers them to the post office (Loki), with the sender's address on the envelope (labels).

## 2. Why this homelab uses it

- **In K3s** (DaemonSet): collects every pod's stdout/stderr
- **On VMs/LXCs** (systemd service): collects journal, Nginx, Jenkins, PostgreSQL logs
- One tool, one config language everywhere
- Can also replace node_exporter later (`prometheus.exporter.unix`) if you want fewer agents

## 3. Key words

| Word | Meaning |
|---|---|
| **Component** | One building block, e.g. `loki.source.file`, `loki.write` |
| **Pipeline** | Components connected: source → process → write |
| **`forward_to`** | Where a component sends its output |
| **Config file** | `config.alloy` (Alloy configuration syntax, looks like HCL) |
| **discovery.kubernetes** | Finds pods to read logs from |
| **loki.source.journal** | Reads systemd journal |
| **loki.source.file** | Tails log files |
| **loki.write** | Sends to Loki's push API |
| **UI (port 12345)** | Built-in debug page showing the pipeline graph |

## 4. How it works

```text
config.alloy:

  [source: journal] ──┐
  [source: files]  ───┼──► [loki.write] ──► http://loki.../loki/api/v1/push
  [source: k8s pods]──┘
```

## 5. Step by step for this project

### 5.1 In K3s (Helm chart `grafana/alloy`, via Flux)
```yaml
controller:
  type: daemonset
alloy:
  resources:
    requests: { memory: 64Mi }
    limits:   { memory: 256Mi }
  configMap:
    content: |
      // find pods on THIS node only (the chart sets HOSTNAME to the node name — verify)
      discovery.kubernetes "pods" {
        role = "pod"
        selectors {
          role  = "pod"
          field = "spec.nodeName=" + coalesce(sys.env("HOSTNAME"), constants.hostname)
        }
      }

      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
      }

      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki.monitoring.svc:3100/loki/api/v1/push"
        }
      }
```
Without the node filter, every Alloy pod would read every pod's logs → duplicates.

### 5.2 On a VM or LXC (example: gateway with Nginx)
```bash
# add Grafana apt repo (see https://grafana.com/docs/alloy/latest/set-up/install/linux/)
sudo apt install -y alloy
echo '192.168.0.229 loki.lab.internal' | sudo tee -a /etc/hosts
```
`/etc/alloy/config.alloy`:
```alloy
loki.source.journal "journal" {
  forward_to = [loki.write.default.receiver]
  labels     = { job = "journal", host = constants.hostname }
}

local.file_match "nginx" {
  path_targets = [{
    "__path__" = "/var/log/nginx/*.log",
    "job"      = "nginx",
    "host"     = constants.hostname,
  }]
}

loki.source.file "nginx" {
  targets    = local.file_match.nginx.targets
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki.lab.internal/loki/api/v1/push"
  }
}
```
```bash
sudo usermod -aG adm,systemd-journal alloy     # permission to read logs
sudo systemctl enable --now alloy
sudo systemctl status alloy
```
Other guests: same idea — Jenkins `/var/log/jenkins/` (or journal), PostgreSQL `/var/log/postgresql/*.log`. Ansible ([17](../17-ansible/README.md)) should template this file per host.

### 5.3 Verify
Grafana → Explore → Loki → `{host="gateway", job="nginx"}`.

## 6. Useful commands

```bash
sudo systemctl status alloy
sudo journalctl -u alloy -f
alloy fmt /etc/alloy/config.alloy          # check/format config
curl -s localhost:12345/-/ready            # health (UI listens on localhost by default)
kubectl logs -n monitoring ds/alloy -f
```

## 7. Common mistakes

- Alloy user can't read the log files → add it to `adm` / `systemd-journal` groups.
- Duplicate logs from the DaemonSet (no node filter).
- Too many labels (e.g., file path per request) → see Loki label advice.
- Loki hostname not resolvable from the VM → check `/etc/hosts` and the Traefik Ingress.

## 8. Check yourself

1. What replaced Promtail?
2. Which component reads the systemd journal?
3. Why filter pods by node in the DaemonSet?

<details><summary>Answers</summary>

1. Grafana Alloy.
2. `loki.source.journal`
3. So each Alloy pod collects only its own node's pods — no duplicates.

</details>

## 9. Learn more

- Alloy docs: <https://grafana.com/docs/alloy/latest/>
- Component reference: <https://grafana.com/docs/alloy/latest/reference/components/>
- Collect Kubernetes logs: <https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/>
