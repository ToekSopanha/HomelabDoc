# 18 — Flux (GitOps)

> A robot inside your cluster that keeps K3s exactly the same as what's written in Git.

## 1. What is it?

**GitOps** means: **Git is the single source of truth**. You never run `kubectl apply` by hand. You change files in Git; a tool inside the cluster sees the change and applies it.

**Flux** is that tool. Every minute it checks your `homelab-infra` repo. If Git says "app version 1.5" and the cluster runs 1.4, Flux upgrades it. If someone changes something by hand, Flux changes it back.

Comparison: a **thermostat**. You set the temperature (Git). The thermostat keeps checking the room (cluster) and fixes any difference.

## 2. Why this homelab uses it

- Every change is a Git commit → full history, easy review
- **Rollback = `git revert`**
- Cluster rebuild = install K3s + bootstrap Flux → everything comes back
- Jenkins doesn't need admin access to the cluster
- Lightweight (~200 MB). ArgoCD is the alternative if you want a web UI (~1 GB)

## 3. Key words

| Word | Meaning |
|---|---|
| **Reconcile** | Compare Git with cluster and fix differences |
| **Bootstrap** | First install of Flux; it also commits its own config into your repo |
| **GitRepository** | "Watch this Git repo/branch" |
| **Kustomization (Flux)** | "Apply the YAML in this folder of the repo" |
| **HelmRepository / HelmRelease** | "Install this Helm chart with these values" ([13](../13-helm/README.md)) |
| **dependsOn** | Order: install infrastructure before apps |
| **prune** | Delete things from the cluster that were deleted from Git |
| **Suspend / resume** | Pause Flux for one object while you debug |
| **Image automation** | Optional: Flux itself watches GHCR and updates image tags in Git |

## 4. How it works

```text
You / Jenkins ──git push──► GitHub: homelab-infra
                                   ▲
                                   │ every 1 min: "anything new?"
                         ┌─────────┴─────────┐
                         │ Flux (in K3s)     │──apply──► Kubernetes objects
                         └───────────────────┘
```

## 5. Step by step for this project

1. **Install the CLI** on your Mac and check the cluster:
   ```bash
   brew install fluxcd/tap/flux
   flux check --pre
   ```
2. **Create a GitHub token** (fine-grained, only `homelab-infra` repo, Contents + Administration read/write) and bootstrap:
   ```bash
   export GITHUB_TOKEN=<token>
   flux bootstrap github \
     --owner=<your-github-user> \
     --repository=homelab-infra \
     --branch=main \
     --path=clusters/homelab \
     --personal
   ```
   Flux installs itself and pushes `clusters/homelab/flux-system/` to the repo.
3. **Repo layout:**
   ```text
   homelab-infra/
     clusters/homelab/
       flux-system/            # created by bootstrap — don't edit
       infrastructure.yaml     # Flux Kustomization → ./infrastructure
       apps.yaml               # Flux Kustomization → ./apps (dependsOn infrastructure)
     infrastructure/
       kube-vip/
       traefik-config/
       monitoring/             # HelmReleases: kube-prometheus-stack, loki, alloy
     apps/
       hello/
         deployment.yaml
         service.yaml
         ingress.yaml
         kustomization.yaml
   ```
4. **`clusters/homelab/apps.yaml`:**
   ```yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: apps
     namespace: flux-system
   spec:
     interval: 10m
     path: ./apps
     prune: true
     sourceRef:
       kind: GitRepository
       name: flux-system
     dependsOn:
       - name: infrastructure
     decryption:
       provider: sops          # decrypt SOPS secrets, see guide 19
       secretRef:
         name: sops-age
   ```
5. **Deploy by committing:**
   ```bash
   git add apps/hello && git commit -m "Deploy hello app" && git push
   flux reconcile kustomization apps --with-source   # don't wait 1 minute
   flux get kustomizations
   ```
6. **How Jenkins fits:** after building `ghcr.io/<user>/myapp:a1b2c3d`, Jenkins edits `apps/myapp/deployment.yaml` to the new tag and pushes. Flux deploys it.

## 6. Useful commands

```bash
flux get all -A                      # status of everything
flux get kustomizations
flux get helmreleases -A
flux reconcile kustomization apps --with-source
flux logs --level=error
flux suspend kustomization apps      # pause
flux resume kustomization apps
flux events
```

## 7. Common mistakes

- Running `kubectl edit` on Flux-managed objects → Flux reverts it. Change Git instead.
- Forgetting `prune: true` → deleted files stay running in the cluster.
- Committing plain-text Secrets → use SOPS ([19](../19-sops-age/README.md)).
- A YAML error in one file blocks the whole Kustomization → check `flux get kustomizations` messages.
- Apps applied before CRDs/infrastructure exist → use `dependsOn`.

## 8. Check yourself

1. What is the "source of truth" in GitOps?
2. How do you roll back with Flux?
3. What does `prune: true` do?

<details><summary>Answers</summary>

1. The Git repository.
2. `git revert` the change and push.
3. Removes cluster objects that no longer exist in Git.

</details>

## 9. Learn more

- Flux get started: <https://fluxcd.io/flux/get-started/>
- Repo structure guide: <https://fluxcd.io/flux/guides/repository-structure/>
- Flux example repo: <https://github.com/fluxcd/flux2-kustomize-helm-example>
