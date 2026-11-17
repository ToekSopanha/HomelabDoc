# 22 — Grafana (Dashboards & Alerts)

> The screen where you see everything: graphs from Prometheus and logs from Loki, in one place.

## 1. What is it?

**Grafana** is a web app for **visualizing** data. It doesn't store metrics or logs itself — it **queries** data sources (Prometheus, Loki, PostgreSQL…) and draws dashboards.

Comparison: Prometheus and Loki are the **medical records**. Grafana is the **monitor screen** in the hospital room.

## 2. Why this homelab uses it

- One place to see host, VMs, K3s, Jenkins, PostgreSQL, and logs
- Thousands of free ready-made dashboards
- Explore logs by namespace/pod/host
- Published at `grafana.sopanha.digital` behind **Cloudflare Access + Grafana login**

## 3. Key words

| Word | Meaning |
|---|---|
| **Data source** | Where Grafana reads data (Prometheus, Loki) |
| **Dashboard** | A page of panels |
| **Panel** | One chart/table/stat |
| **Variable** | Dropdown on a dashboard (choose node, namespace) |
| **Explore** | Free-form querying of metrics or logs |
| **Dashboard ID** | Number to import a community dashboard from grafana.com |
| **Provisioning** | Load data sources and dashboards from files/ConfigMaps (GitOps-friendly) |
| **Contact point** | Where Grafana alerts are sent |

## 4. How it works

```text
Browser ──► Cloudflare Access ──► Nginx ──► Traefik ──► Grafana pod
                                                         ├─ query ─► Prometheus
                                                         └─ query ─► Loki
```

## 5. Step by step for this project

1. Grafana is **included in kube-prometheus-stack** ([21](../21-prometheus/README.md)). Add to its values:
   ```yaml
   grafana:
     admin:
       existingSecret: grafana-admin          # SOPS-encrypted secret with admin-user / admin-password
     resources:
       requests: { memory: 128Mi }
       limits:   { memory: 512Mi }
     persistence:
       enabled: true
       size: 2Gi
     ingress:
       enabled: true
       hosts: [grafana.sopanha.digital]
     grafana.ini:
       server:
         root_url: https://grafana.sopanha.digital
     additionalDataSources:
       - name: Loki
         type: loki
         url: http://loki.monitoring.svc:3100
   ```
2. Make sure Cloudflare Access protects `grafana.sopanha.digital` ([10](../10-cloudflare/README.md)), then open it.
3. **Import community dashboards:** Dashboards → New → Import → enter the ID:

   | ID | Dashboard |
   |---:|---|
   | 1860 | Node Exporter Full (any Linux host) |
   | 10347 | Proxmox via Prometheus (pve-exporter) |
   | 15757–15762 | Kubernetes views (from kube-prometheus-stack authors) |
   | 9628 | PostgreSQL Database |
   | 17346 | Traefik |
   | 9964 | Jenkins: Performance and Health |

   (IDs can change over time — search grafana.com/grafana/dashboards if one doesn't fit.)
4. **Save your own dashboards in Git:** export JSON → put in a ConfigMap with label `grafana_dashboard: "1"` → the chart's sidecar loads it automatically.
5. **Logs:** Explore → choose **Loki** → query `{namespace="myapp"}`.

## 6. Useful tips

- Time picker (top right): "Last 1 hour" vs "Last 7 days".
- Press `e` on a panel to edit, `v` to view large.
- **Explore** is the best place to learn PromQL and LogQL.
- Split view in Explore: metrics on the left, logs on the right, same time range — great for debugging.

## 7. Common mistakes

- Leaving the default `admin/prom-operator` password.
- Making dashboards by hand only in the UI → lost when the pod/PV is rebuilt. Keep important ones in Git.
- Too many heavy panels on one dashboard → slow, loads Prometheus.
- Grafana public without Access.

## 8. Check yourself

1. Does Grafana store metrics?
2. How do you import a ready-made dashboard?

<details><summary>Answers</summary>

1. No — it queries Prometheus/Loki.
2. Dashboards → Import → dashboard ID → choose data source.

</details>

## 9. Learn more

- Grafana docs: <https://grafana.com/docs/grafana/latest/>
- Dashboard library: <https://grafana.com/grafana/dashboards/>
- Grafana fundamentals tutorial: <https://grafana.com/tutorials/grafana-fundamentals/>
