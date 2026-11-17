# 13 — Helm

> The "app store" / package manager for Kubernetes.

## 1. What is it?

Installing something like Grafana on Kubernetes needs many YAML files (Deployment, Service, ConfigMap, Secret, PVC, RBAC…). **Helm** bundles them into a **chart**. You install the chart and change only the settings you care about with a **values** file.

Comparison: `apt install nginx` on Ubuntu → `helm install grafana grafana/grafana` on Kubernetes.

## 2. Why this homelab uses it

The monitoring stack, Loki, Alloy, Jenkins agents' helpers, and more are all published as Helm charts. In this lab **Flux installs charts for you** (with a `HelmRelease` file in Git), but you should understand what Helm does underneath.

K3s itself uses Helm internally to install Traefik.

## 3. Key words

| Word | Meaning |
|---|---|
| **Chart** | A package of Kubernetes templates |
| **Repository** | Where charts are published (e.g., `https://grafana.github.io/helm-charts`) |
| **Release** | One installed copy of a chart (name + namespace) |
| **values.yaml** | Settings you override (memory, retention, ingress host…) |
| **Template** | YAML with placeholders filled from values |
| **Upgrade / rollback** | Change version or values / go back to a previous release revision |
| **HelmRelease** | Flux's way to say "install this chart with these values" from Git |

## 4. How it works

```text
chart templates  +  your values.yaml  ──helm──►  final YAML  ──►  Kubernetes
```

## 5. Step by step (manual practice — later Flux does this)

1. Install Helm on your Mac:
   ```bash
   brew install helm
   ```
2. Add a repo and look at a chart:
   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   helm search repo grafana/grafana
   helm show values grafana/grafana > grafana-defaults.yaml   # read the options
   ```
3. Write your own small `my-values.yaml`:
   ```yaml
   resources:
     requests: { memory: 128Mi }
     limits:   { memory: 512Mi }
   ```
4. Preview, then install into a test namespace:
   ```bash
   helm template test grafana/grafana -f my-values.yaml | less   # see generated YAML
   helm install test grafana/grafana -n helm-test --create-namespace -f my-values.yaml
   helm list -A
   helm uninstall test -n helm-test
   ```
5. The **same** chart + values in Flux form (what goes into Git):
   ```yaml
   apiVersion: helm.toolkit.fluxcd.io/v2
   kind: HelmRelease
   metadata: { name: grafana, namespace: monitoring }
   spec:
     interval: 30m
     chart:
       spec:
         chart: grafana
         version: "8.x"           # pin the major version
         sourceRef: { kind: HelmRepository, name: grafana, namespace: flux-system }
     values:
       resources:
         requests: { memory: 128Mi }
         limits:   { memory: 512Mi }
   ```

## 6. Useful commands

```bash
helm list -A                        # all releases
helm status <release> -n <ns>
helm get values <release> -n <ns>   # values in use
helm history <release> -n <ns>
helm rollback <release> <rev> -n <ns>
```

## 7. Common mistakes

- Copying the **entire** default values file — only override what you change.
- Not pinning chart versions → surprise upgrades.
- Mixing manual `helm install` with Flux for the same app → they fight. Once Flux is set up, change things only in Git.

## 8. Check yourself

1. Chart vs release?
2. How do you see all options a chart supports?

<details><summary>Answers</summary>

1. Chart = package; release = an installed instance of it.
2. `helm show values <repo>/<chart>`

</details>

## 9. Learn more

- Helm docs: <https://helm.sh/docs/>
- Artifact Hub (find charts): <https://artifacthub.io/>
- Flux HelmRelease: <https://fluxcd.io/flux/components/helm/helmreleases/>
