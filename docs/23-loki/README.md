# 23 — Loki (Log Storage)

> Stores log lines from every machine and pod, so you can search them in Grafana instead of SSH-ing into each server.

## 1. What is it?

**Logs** are text lines programs write: `ERROR connection refused`, `GET /api 200`. **Loki** collects logs from everywhere and stores them cheaply. It only **indexes labels** (like `host`, `namespace`, `pod`), not the full text — that keeps it small and light.

Comparison: Prometheus is the **heart-rate chart**; Loki is the **diary** of what happened. When the chart shows a spike, the diary tells you why.

## 2. Why this homelab uses it

- One place to search logs of all 7 guests and all pods
- Logs survive after a pod is deleted
- Same Grafana UI as metrics
- Lighter than Elasticsearch/ELK — fits in 768 MB

## 3. Key words

| Word | Meaning |
|---|---|
| **Log stream** | All lines with the same set of labels |
| **Labels** | `{namespace="myapp", pod="..."}` — keep them few and low-variety |
| **LogQL** | Query language: `{job="nginx"} \|= "error"` |
| **Single binary mode** | All Loki parts in one pod — right for a homelab |
| **Chunks** | Compressed blocks of log lines on disk |
| **Compactor** | Background job that applies **retention** (deletes old logs) |
| **Push API** | `/loki/api/v1/push` — where Alloy sends logs |

## 4. How it works

```text
Alloy (K3s pods' logs) ─┐
Alloy (VMs: journal,    ├─ push ─► Loki (pod, PV 20Gi) ◄─ query ─ Grafana
       nginx, jenkins) ─┘
```

## 5. Step by step for this project

1. **HelmRelease values** for chart `grafana/loki` (single binary, filesystem, small):
   ```yaml
   deploymentMode: SingleBinary
   loki:
     auth_enabled: false
     commonConfig:
       replication_factor: 1
     storage:
       type: filesystem
     schemaConfig:
       configs:
         - from: "2024-01-01"
           store: tsdb
           object_store: filesystem
           schema: v13
           index: { prefix: index_, period: 24h }
     limits_config:
       retention_period: 168h        # 7 days
     compactor:
       retention_enabled: true
       delete_request_store: filesystem
   singleBinary:
     replicas: 1
     nodeSelector: { kubernetes.io/hostname: k3s-worker2 }   # pin: local-path PV
     persistence: { enabled: true, size: 20Gi, storageClass: local-path }
     resources:
       requests: { memory: 256Mi }
       limits:   { memory: 768Mi }
   # turn off parts not needed in single binary mode
   backend: { replicas: 0 }
   read:    { replicas: 0 }
   write:   { replicas: 0 }
   chunksCache:  { enabled: false }   # defaults request several GB of RAM!
   resultsCache: { enabled: false }
   lokiCanary:   { enabled: false }
   test:         { enabled: false }
   ```
   (Check the chart's current values — names move between versions.)
2. **Let VMs/LXCs send logs to Loki** — they're outside K3s. Add an internal-only Ingress, e.g. host `loki.lab.internal` → Service `loki:3100`, plus a Traefik `ipAllowList` middleware for `192.168.0.0/24`. On each VM add to `/etc/hosts`: `192.168.0.229 loki.lab.internal`. **Do not** add this hostname to Cloudflare.
3. Add Loki as a Grafana data source ([22](../22-grafana/README.md)).
4. Set up Alloy to push logs ([24](../24-alloy/README.md)).
5. Test in Grafana → Explore → Loki → `{namespace="kube-system"}`.

## 6. Useful LogQL

```logql
{namespace="myapp"}                                   # all logs of an app
{namespace="myapp"} |= "error"                        # lines containing "error"
{job="nginx", host="gateway"} |~ " 5\\d\\d "          # Nginx 5xx (regex)
{host="postgres"} |= "FATAL"
sum by (namespace) (count_over_time({namespace=~".+"} |= "error" [5m]))   # error count per namespace
```

## 7. Common mistakes

- **High-cardinality labels** (request ID, user ID, IP as label) → Loki becomes slow and big. Put those in the log text, not labels.
- Forgetting `retention_enabled: true` on the compactor → logs are never deleted → disk full.
- Leaving the cache components enabled → huge memory requests, pods `Pending`.
- Exposing the push API publicly → anyone can fill your disk.

## 8. Check yourself

1. Why is Loki lighter than Elasticsearch?
2. What enforces the 7-day retention?
3. Should `user_id` be a label?

<details><summary>Answers</summary>

1. It indexes only labels, not the full log text.
2. The compactor with `retention_enabled: true` + `retention_period`.
3. No — too many different values (high cardinality).

</details>

## 9. Learn more

- Loki docs: <https://grafana.com/docs/loki/latest/>
- LogQL: <https://grafana.com/docs/loki/latest/query/>
- Helm install (single binary): <https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/>
- Label best practices: <https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/>
