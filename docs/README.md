# Homelab Learning Guides

One folder per technology. Each guide explains, in simple words:

1. **What it is** (with a real-life comparison)
2. **Why this homelab uses it**
3. **Key words** you will see
4. **How it works**
5. **Step by step** — how we set it up in *this* project
6. **Useful commands**
7. **Common mistakes**
8. **Check yourself** — questions to test your understanding
9. **Learn more** — official links

## Learning order

Read in this order. Each one builds on the ones before it.

| # | Guide | One-line summary | Level |
|---:|---|---|---|
| 00 | [Overview](00-overview/README.md) | The big picture: how everything connects | Start here |
| 01 | [Networking basics](01-networking/README.md) | IP, subnet, gateway, ports, DNS, DHCP | Beginner |
| 02 | [Linux, Ubuntu & SSH](02-linux-ssh/README.md) | The operating system inside every machine | Beginner |
| 03 | [Proxmox VE](03-proxmox/README.md) | Runs many virtual computers on one server | Beginner |
| 04 | [iLO 5](04-ilo/README.md) | Remote control of the physical server | Beginner |
| 05 | [Git & GitHub](05-git-github/README.md) | Version history for code and config | Beginner |
| 06 | [Docker & containers](06-docker/README.md) | Package apps so they run the same everywhere | Beginner |
| 07 | [Container registry (GHCR)](07-registry-ghcr/README.md) | Where container images are stored | Beginner |
| 08 | [PostgreSQL](08-postgresql/README.md) | The database | Beginner |
| 09 | [Nginx](09-nginx/README.md) | Reverse proxy — sends web traffic to the right place | Beginner |
| 10 | [Cloudflare Tunnel & Access](10-cloudflare/README.md) | Puts sites online safely without opening ports | Intermediate |
| 11 | [Tailscale](11-tailscale/README.md) | Private VPN for you to manage the lab from anywhere | Beginner |
| 12 | [Kubernetes & K3s](12-kubernetes-k3s/README.md) | Runs and heals containers across many machines | Intermediate |
| 13 | [Helm](13-helm/README.md) | Package manager for Kubernetes apps | Intermediate |
| 14 | [Traefik](14-traefik/README.md) | The "front door" inside Kubernetes | Intermediate |
| 15 | [kube-vip](15-kube-vip/README.md) | One fixed IP address for the cluster | Intermediate |
| 16 | [OpenTofu](16-opentofu/README.md) | Create VMs from code | Intermediate |
| 17 | [Ansible](17-ansible/README.md) | Configure servers from code | Intermediate |
| 18 | [Flux (GitOps)](18-flux-gitops/README.md) | Git decides what runs in Kubernetes | Advanced |
| 19 | [SOPS & age](19-sops-age/README.md) | Keep passwords encrypted inside Git | Intermediate |
| 20 | [Jenkins & BuildKit](20-jenkins/README.md) | Automatic build and test (CI) | Intermediate |
| 21 | [Prometheus & exporters](21-prometheus/README.md) | Collects numbers (metrics) | Intermediate |
| 22 | [Grafana](22-grafana/README.md) | Dashboards and alerts | Beginner |
| 23 | [Loki](23-loki/README.md) | Stores logs | Intermediate |
| 24 | [Grafana Alloy](24-alloy/README.md) | Collects logs and sends them to Loki | Intermediate |
| 25 | [Backups: vzdump, restic, rclone, R2](25-backups/README.md) | Keep copies of data off the server | Intermediate |

## How to use these guides

- Don't read everything at once. Read a guide **right before** the phase that needs it (see the phase list in the main [README](../README.md#16-deployment-phases-and-acceptance-checks)).
- Try the commands in a test VM first.
- Answer the **Check yourself** questions. If you can't, re-read that part.

| Plan phase | Read these first |
|---|---|
| Phase 0 — Host | 00, 01, 02, 03, 04, 11, 25 |
| Phase 1 — IaC and guests | 05, 16, 17 |
| Phase 2 — K3s | 12, 13, 14, 15, 18, 19 |
| Phase 3 — Supporting services | 06, 08, 20 |
| Phase 4 — Gateway | 09, 10 |
| Phase 5 — Monitoring | 21, 22, 23, 24 |
| Phase 6 — CI/CD | 07, 18, 20 |
| Phase 7 — Operations | 25 |
