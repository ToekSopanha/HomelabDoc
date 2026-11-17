# 14 — Traefik (Ingress Controller)

> The front door **inside** Kubernetes. It reads Ingress rules and sends each web request to the right app.

## 1. What is it?

**Traefik** is a reverse proxy (like Nginx) that understands Kubernetes. When you create an **Ingress** saying `app.sopanha.digital → Service hello`, Traefik automatically starts routing it. No restart, no manual config.

Comparison: Nginx in LXC 100 is the **building's main reception**. Traefik is the **reception desk on the K3s floor** that knows every office on that floor.

## 2. Why this homelab uses it

- Comes **built into K3s** — nothing extra to install.
- Reads Kubernetes Ingress objects; apps describe their own routing in Git.

## 3. Key words

| Word | Meaning |
|---|---|
| **Ingress controller** | The program that makes Ingress rules work (Traefik here) |
| **Ingress** | Standard Kubernetes routing rule (host + path → Service) |
| **IngressRoute** | Traefik's own, more powerful routing object (optional) |
| **Entrypoint** | A port Traefik listens on: `web` (80), `websecure` (443) |
| **Middleware** | Extra step: redirects, headers, basic auth, rate limit |
| **HelmChartConfig** | K3s way to change settings of the built-in Traefik |
| **LoadBalancer Service** | How Traefik gets an IP reachable from the LAN (kube-vip gives `192.168.0.229`) |

## 4. How it works

```text
Nginx (LXC 100) ──► 192.168.0.229:80 (kube-vip VIP)
                           │
                    Traefik pod(s)
                           │ Host: app.sopanha.digital → Ingress "hello"
                           ▼
                    Service hello ──► pod hello-xxxx
```

## 5. Step by step for this project

1. Traefik is already running after K3s install:
   ```bash
   kubectl get pods -n kube-system | grep traefik
   kubectl get svc -n kube-system traefik
   ```
   Until kube-vip is set up, `EXTERNAL-IP` shows `<pending>` (because we disabled `servicelb`).
2. **Configure it** — on the server, create `/var/lib/rancher/k3s/server/manifests/traefik-config.yaml` (K3s applies files in this folder automatically; later move it into the Flux repo):
   ```yaml
   apiVersion: helm.cattle.io/v1
   kind: HelmChartConfig
   metadata:
     name: traefik
     namespace: kube-system
   spec:
     valuesContent: |-
       service:
         annotations:
           kube-vip.io/loadbalancerIPs: "192.168.0.229"
       ports:
         web:
           forwardedHeaders:
             trustedIPs: ["192.168.0.220/32"]   # trust X-Forwarded-* from the gateway only
       resources:
         requests: { cpu: 50m, memory: 64Mi }
         limits:   { memory: 256Mi }
   ```
3. After kube-vip works ([15](../15-kube-vip/README.md)):
   ```bash
   kubectl get svc -n kube-system traefik     # EXTERNAL-IP = 192.168.0.229
   curl -H 'Host: app.sopanha.digital' http://192.168.0.229/
   ```
4. Every app adds its own Ingress (see the example in [12](../12-kubernetes-k3s/README.md)).

## 6. Useful commands

```bash
kubectl get ingress -A
kubectl describe ingress <name> -n <ns>
kubectl logs -n kube-system deploy/traefik -f
kubectl get helmchartconfig -n kube-system
```

## 7. Common mistakes

- **404 from Traefik** → no Ingress matches that Host, or wrong Service name/port.
- **Bad Gateway** → Service has no ready pods (`kubectl get endpoints <svc>`).
- Editing Traefik's Deployment directly — K3s overwrites it. Use `HelmChartConfig`.
- HTTPS confusion: Cloudflare already does HTTPS; inside the lab, plain HTTP on port 80 is fine.

## 8. Check yourself

1. Nginx vs Traefik — who does what in this lab?
2. Where do you change Traefik's settings in K3s?

<details><summary>Answers</summary>

1. Nginx routes between big destinations (K3s, Jenkins, hosting). Traefik routes between apps inside K3s.
2. A `HelmChartConfig` named `traefik` in `kube-system`.

</details>

## 9. Learn more

- K3s networking (Traefik): <https://docs.k3s.io/networking/networking-services#traefik-ingress-controller>
- Traefik Kubernetes Ingress: <https://doc.traefik.io/traefik/providers/kubernetes-ingress/>
