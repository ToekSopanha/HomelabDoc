# 15 — kube-vip (Virtual IP for LoadBalancer Services)

> Gives a Kubernetes Service one fixed LAN IP (`192.168.0.229`) that keeps working even if a node goes down.

## 1. What is it?

On cloud providers (AWS, GCP), a `LoadBalancer` Service automatically gets a public IP. On a homelab there is no cloud, so someone must hand out IPs. **kube-vip** does that: it makes one node **announce** "I own `192.168.0.229`" on the LAN (using ARP). If that node dies, another node takes over the IP.

Comparison: a **shared office phone number**. Whoever is on duty answers it. If they go home, the call is automatically forwarded to the next person.

## 2. Why this homelab uses it

Without it, Nginx must list each worker IP. With it, Nginx has **one** target (`192.168.0.229`) and doesn't care which worker is alive.

> **Alternative:** **MetalLB** (L2 mode) does the same job and many beginners find it simpler. Pick one, not both.

## 3. Key words

| Word | Meaning |
|---|---|
| **VIP** | Virtual IP — not tied to one machine |
| **ARP** | How LAN devices ask "who has IP X?" — kube-vip answers for the VIP |
| **Leader election** | The nodes agree which one owns the VIP right now |
| **ServiceLB (Klipper)** | K3s's built-in simple load balancer — we disable it to use kube-vip |
| **Cloud provider (kube-vip-cloud-provider)** | Optional helper that picks IPs from a range automatically |
| **`kube-vip.io/loadbalancerIPs`** | Annotation to request a specific IP for a Service |

## 4. How it works

```text
Nginx asks LAN: "who has 192.168.0.229?"
   worker1 (current leader): "me!"  → traffic goes to worker1
       └─ kube-proxy forwards to a Traefik pod (on any node)
worker1 dies → worker2 becomes leader → answers "me!" → traffic continues
```

## 5. Step by step for this project

> Exact flags change between versions — always check the kube-vip docs for the version you install.

1. Make sure K3s was installed with `--disable servicelb` ([12](../12-kubernetes-k3s/README.md)).
2. Find the VM's network interface name on a worker:
   ```bash
   ip -br a        # usually ens18 or eth0
   ```
3. Install the RBAC permissions:
   ```bash
   kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
   ```
4. Generate the DaemonSet manifest (services mode, ARP) — on your Mac with Docker:
   ```bash
   export KVVERSION=<latest release tag from github.com/kube-vip/kube-vip/releases>
   docker run --network host --rm ghcr.io/kube-vip/kube-vip:$KVVERSION \
     manifest daemonset --services --inCluster --arp --interface ens18 \
     > kube-vip.yaml
   ```
   Open `kube-vip.yaml` and check the `nodeSelector`/`affinity`: for services you usually want it to run on the **workers** too, not only on the control plane.
5. Apply it (later: commit it to the Flux repo):
   ```bash
   kubectl apply -f kube-vip.yaml
   kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
   ```
6. Request the IP on the Traefik Service with the annotation (already in the Traefik `HelmChartConfig`, see [14](../14-traefik/README.md)).
7. Test:
   ```bash
   kubectl get svc -n kube-system traefik   # EXTERNAL-IP 192.168.0.229
   ping 192.168.0.229
   ```
8. **Failover test:** shut down the leader worker in Proxmox → `curl http://192.168.0.229` should still work after a few seconds.

## 6. Common mistakes

- `servicelb` still enabled → two systems fight over the Service.
- Wrong `--interface` name → VIP never appears.
- VIP inside the router's DHCP range → IP conflict.
- DaemonSet only scheduled on the (tainted) server → check tolerations/affinity.

## 7. Check yourself

1. What problem does kube-vip solve on a homelab?
2. What happens to the VIP when its current node dies?

<details><summary>Answers</summary>

1. Gives LoadBalancer Services a real LAN IP, since there's no cloud provider.
2. Another node is elected leader and takes over the IP.

</details>

## 8. Learn more

- kube-vip docs: <https://kube-vip.io/docs/>
- K3s + kube-vip: <https://kube-vip.io/docs/usage/k3s/>
- MetalLB (alternative): <https://metallb.io/>
