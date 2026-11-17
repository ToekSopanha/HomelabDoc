# 12 — Kubernetes & K3s

> Kubernetes runs your containers across several machines, keeps them alive, and replaces them if they crash. K3s is a small, easy version of Kubernetes.

## 1. What is it?

- **Kubernetes (K8s)** is a **container orchestrator**. You tell it *what* you want ("run 2 copies of my app, with 256 MB each"). It decides *where* to run them and keeps them that way.
- **K3s** is a lightweight Kubernetes from Rancher/SUSE. One small program, perfect for homelabs. It includes Traefik, CoreDNS, a storage provisioner, and `containerd`.

Comparison: Kubernetes is a **port manager**. Ships (workers) arrive; the manager decides which containers go on which ship, and if a container falls into the sea, the manager orders a new one immediately.

## 2. Why this homelab uses it

- Learn the most important DevOps platform
- Self-healing: crashed apps restart automatically
- Rolling updates without downtime
- Same way of deploying every app (YAML in Git + Flux)

## 3. Key words

| Word | Meaning |
|---|---|
| **Cluster** | The whole group of machines running Kubernetes |
| **Node** | One machine in the cluster (our 3 VMs) |
| **Control plane / server** | The "brain": API, scheduler, database of state (VM 101) |
| **Worker / agent** | Runs your app containers (VMs 102, 103) |
| **Pod** | Smallest unit: one (or a few) containers together |
| **Deployment** | "Keep N copies of this pod running; update them gradually" |
| **StatefulSet** | Like Deployment, but for apps with stable storage (Prometheus, Loki) |
| **DaemonSet** | "Run one copy on **every** node" (Alloy, node-exporter) |
| **Service** | A stable name/IP in front of changing pods |
| **Ingress** | Rules for HTTP: `app.sopanha.digital` → Service `myapp` (handled by Traefik) |
| **Namespace** | A folder to group things: `monitoring`, `ci`, `myapp` |
| **ConfigMap / Secret** | Settings / passwords given to pods |
| **PersistentVolume (PV) / PVC** | Disk storage for pods. PVC = request, PV = actual disk |
| **local-path** | K3s's default storage: a folder on that node's disk. If the pod moves to another node, data doesn't follow |
| **requests / limits** | requests = guaranteed resources; limits = maximum allowed |
| **Taint / toleration** | A "do not enter" sign on a node / a pass to ignore it |
| **kubectl** | Command-line tool to talk to the cluster |
| **kubeconfig** | File with cluster address + credentials for kubectl |
| **Flannel** | K3s's default network between pods on different nodes (VXLAN, UDP 8472) |

## 4. How it works

```text
          you: kubectl apply -f app.yaml   (or Flux does it from Git)
                         │
            ┌────────────▼────────────┐
            │  k3s-server (VM 101)    │  API server + scheduler + SQLite state
            │  tainted: no app pods   │
            └───────┬─────────┬───────┘
                    │         │ "run this pod on worker2"
          ┌─────────▼──┐  ┌───▼────────┐
          │ worker1    │  │ worker2    │  kubelet + containerd run the pods
          │ pods...    │  │ pods...    │
          └────────────┘  └────────────┘
```

If worker1 dies, the pods from worker1 are recreated on worker2 (if there's room and their storage allows).

## 5. Step by step for this project

1. **Install the server (VM 101):**
   ```bash
   curl -sfL https://get.k3s.io | sh -s - server \
     --node-taint node-role.kubernetes.io/control-plane:NoSchedule \
     --tls-san 192.168.0.221 \
     --disable servicelb \
     --write-kubeconfig-mode 600
   sudo k3s kubectl get nodes
   sudo cat /var/lib/rancher/k3s/server/node-token     # join token — keep secret
   ```
   (`--disable servicelb` because kube-vip gives us LoadBalancer IPs — see [15](../15-kube-vip/README.md).)
2. **Join each worker (VM 102, 103):**
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.221:6443 \
     K3S_TOKEN=<node-token> sh -
   ```
3. **Check from the server:**
   ```bash
   sudo k3s kubectl get nodes -o wide     # 3 nodes, all Ready
   sudo k3s kubectl get pods -A           # system pods Running
   ```
4. **Use kubectl from your Mac:**
   ```bash
   brew install kubectl
   scp ubuntu@192.168.0.221:/etc/rancher/k3s/k3s.yaml ~/.kube/config   # may need sudo copy first
   sed -i '' 's/127.0.0.1/192.168.0.221/' ~/.kube/config
   kubectl get nodes
   ```
5. **Deploy a test app** — `hello.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata: { name: hello, namespace: default }
   spec:
     replicas: 2
     selector: { matchLabels: { app: hello } }
     template:
       metadata: { labels: { app: hello } }
       spec:
         containers:
           - name: web
             image: nginxdemos/hello:plain-text
             ports: [{ containerPort: 80 }]
             resources:
               requests: { cpu: 50m, memory: 32Mi }
               limits:   { memory: 64Mi }
   ---
   apiVersion: v1
   kind: Service
   metadata: { name: hello }
   spec:
     selector: { app: hello }
     ports: [{ port: 80, targetPort: 80 }]
   ---
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata: { name: hello }
   spec:
     rules:
       - host: app.sopanha.digital
         http:
           paths:
             - path: /
               pathType: Prefix
               backend: { service: { name: hello, port: { number: 80 } } }
   ```
   ```bash
   kubectl apply -f hello.yaml
   kubectl get pods -o wide
   ```
   Later, Flux applies files like this from Git instead of you.

## 6. Useful commands

```bash
kubectl get nodes -o wide
kubectl get pods -A                         # all pods, all namespaces
kubectl get pods -n monitoring -o wide
kubectl describe pod <pod> -n <ns>          # why is it failing? (read Events at the bottom)
kubectl logs <pod> -n <ns> -f               # follow logs
kubectl logs <pod> -n <ns> --previous       # logs from the crashed container
kubectl exec -it <pod> -n <ns> -- sh        # shell inside a pod
kubectl top nodes / kubectl top pods -A     # CPU/RAM usage
kubectl rollout restart deploy/<name> -n <ns>
kubectl drain k3s-worker1 --ignore-daemonsets --delete-emptydir-data   # before maintenance
kubectl uncordon k3s-worker1
```

### Pod status cheat sheet

| Status | Usually means |
|---|---|
| `Pending` | No node has enough free resources, or PVC not bound |
| `ImagePullBackOff` | Wrong image name/tag, or missing pull secret |
| `CrashLoopBackOff` | App starts then crashes — check `logs --previous` |
| `OOMKilled` | Used more memory than its limit |
| `Evicted` | Node ran low on memory/disk |

## 7. Common mistakes

- **No requests/limits** → one app eats all memory, others get killed.
- Expecting `local-path` volumes to move between nodes — they don't.
- Running apps on the control plane (we taint it to prevent this).
- Giving Jenkins the admin kubeconfig. Use a limited ServiceAccount.
- Losing the node token or the server's `/var/lib/rancher/k3s/server/` — that's the cluster's memory (it's in the weekly vzdump of VM 101).

## 8. Check yourself

1. Pod vs Deployment?
2. Why do Alloy and node-exporter use a DaemonSet?
3. A pod is `Pending` forever — first two things to check?
4. Why is the control plane tainted?

<details><summary>Answers</summary>

1. A pod is one running unit; a Deployment manages several pods and replaces them when needed.
2. They must run on every node to collect that node's data.
3. `kubectl describe pod` Events (resources? PVC?) and `kubectl top nodes`.
4. So app pods only run on workers and the 3 GB server stays stable.

</details>

## 9. Learn more

- K3s docs: <https://docs.k3s.io/>
- Kubernetes basics tutorial: <https://kubernetes.io/docs/tutorials/kubernetes-basics/>
- Concepts: <https://kubernetes.io/docs/concepts/>
