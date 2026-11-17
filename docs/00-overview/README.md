# 00 — The Big Picture

> Read this first. It explains how all the technologies fit together, using one story.

## 1. The homelab in one picture

```text
                 INTERNET
                    │
          ┌─────────▼─────────┐
          │    Cloudflare     │  DNS, HTTPS, login wall (Access)
          └─────────┬─────────┘
                    │  Tunnel (server calls OUT to Cloudflare — no open ports)
┌───────────────────▼────────────────────────────────────────────┐
│  HPE ML30 server  →  Proxmox VE (makes virtual machines)       │
│                                                                │
│   gateway (Nginx) ──┬──► K3s cluster ──► your apps             │
│                     │      (Traefik)     Grafana/Prometheus/Loki│
│                     ├──► Jenkins (builds your code)            │
│                     └──► hosting VM (Docker Compose apps)      │
│                                                                │
│   PostgreSQL ◄── apps read/write data                          │
└────────────────────────────────────────────────────────────────┘
        ▲                                   │
        │ Tailscale (you, from anywhere)    │ backups → Cloudflare R2
```

## 2. A story: what happens when you change your website

1. You edit code on your laptop and run `git push` → code goes to **GitHub** ([05](../05-git-github/README.md)).
2. GitHub tells **Jenkins** "new code!" ([20](../20-jenkins/README.md)).
3. Jenkins starts a temporary build pod in **Kubernetes/K3s** ([12](../12-kubernetes-k3s/README.md)) that tests the code and builds a **Docker image** ([06](../06-docker/README.md)).
4. The image is uploaded to **GHCR** ([07](../07-registry-ghcr/README.md)).
5. Jenkins writes the new image version into the `homelab-infra` Git repo.
6. **Flux** ([18](../18-flux-gitops/README.md)) sees the Git change and updates the app in K3s.
7. A visitor opens `app.sopanha.digital`:
   - **Cloudflare** ([10](../10-cloudflare/README.md)) receives the request
   - sends it through the **Tunnel** to **Nginx** ([09](../09-nginx/README.md))
   - Nginx forwards to the **kube-vip** address ([15](../15-kube-vip/README.md))
   - **Traefik** ([14](../14-traefik/README.md)) picks the right app pod
   - the app reads data from **PostgreSQL** ([08](../08-postgresql/README.md)).
8. **Alloy** ([24](../24-alloy/README.md)) collects the logs → **Loki** ([23](../23-loki/README.md)).
   **Prometheus** ([21](../21-prometheus/README.md)) collects numbers (CPU, requests).
   You see everything in **Grafana** ([22](../22-grafana/README.md)).
9. Every night, data is backed up to **Cloudflare R2** ([25](../25-backups/README.md)).

## 3. Technologies grouped by job

| Job | Technology | Simple comparison |
|---|---|---|
| Run many computers on one | **Proxmox** | An apartment building: one building, many apartments |
| Lightweight "apartments" | **LXC** | Rooms sharing one kitchen (kernel) |
| Operating system | **Ubuntu Linux** | The furniture inside each apartment |
| Remote login | **SSH** | A secure key to enter an apartment |
| Remote power/console | **iLO** | The building manager's master control room |
| Package an app | **Docker** | A shipping container: same box everywhere |
| Run containers at scale | **Kubernetes / K3s** | A port manager deciding where containers go and fixing broken ones |
| Install apps on K8s | **Helm** | An app store for Kubernetes |
| Front door inside K8s | **Traefik** | Reception desk that sends visitors to the right office |
| One fixed cluster address | **kube-vip** | One phone number that always reaches whoever is on duty |
| Front door for the whole lab | **Nginx** | Building security desk |
| Safe internet access | **Cloudflare Tunnel** | A private underground tunnel — no front door needed |
| Login wall | **Cloudflare Access** | Guard who checks ID before you enter |
| Private admin access | **Tailscale** | Your personal secret door |
| Code history | **Git / GitHub** | "Track changes" for every file, forever |
| Create VMs from code | **OpenTofu** | An architect's blueprint → builds the apartments |
| Set up servers from code | **Ansible** | An interior designer's checklist |
| Git controls K8s | **Flux** | A robot that makes the cluster match the blueprint in Git |
| Secret passwords in Git | **SOPS + age** | A locked safe you can store in a public place |
| Automatic build/test | **Jenkins** | A factory line that builds every new version |
| Store images | **GHCR** | A warehouse for shipping containers |
| Database | **PostgreSQL** | A very organized filing cabinet |
| Metrics | **Prometheus** | A nurse checking heartbeat and temperature every 30 s |
| Logs | **Loki** | A diary of everything that happened |
| Collect logs | **Alloy** | A mail carrier picking up diary pages |
| Dashboards/alerts | **Grafana** | The TV screen showing all the health charts |
| Backups | **vzdump / restic / rclone / R2** | Photocopies kept in a different city |

## 4. Important ideas you will meet everywhere

| Idea | Meaning |
|---|---|
| **Host vs guest** | The host is the real server (Proxmox). Guests are the VMs/LXCs on it. |
| **Infrastructure as Code (IaC)** | You describe servers in text files. Tools build them. If a server breaks, rebuild it from the files. |
| **Declarative** | You say *what you want* ("3 copies of my app"), not *how to do it*. The tool figures out the steps. Kubernetes, OpenTofu, Flux work this way. |
| **CI / CD** | **C**ontinuous **I**ntegration = auto build + test. **C**ontinuous **D**elivery = auto deploy. |
| **GitOps** | Git is the "single source of truth". To change the cluster, change Git. |
| **Reverse proxy** | A server that receives requests and forwards them to the correct backend. |
| **Least privilege** | Give each tool only the permissions it really needs. |
| **Stateless vs stateful** | Stateless = can be deleted and recreated (a web app). Stateful = holds data that must be kept (a database). Only stateful things need backups. |

## 5. Check yourself

1. In the story above, which tool deploys the new app version into K3s — Jenkins or Flux?
2. Why don't we need to open ports on the home router?
3. What is the difference between Prometheus and Loki?
4. Why are the K3s workers not backed up?

<details><summary>Answers</summary>

1. Flux. Jenkins only builds and records the new version in Git.
2. Cloudflare Tunnel makes an **outgoing** connection from the server to Cloudflare; visitors come through that.
3. Prometheus stores **numbers over time** (metrics). Loki stores **text lines** (logs).
4. They are stateless — they can be rebuilt from Git (OpenTofu + Ansible + Flux).

</details>
