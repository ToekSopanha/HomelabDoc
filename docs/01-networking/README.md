# 01 — Networking Basics

> How computers find and talk to each other. Every other guide depends on this.

## 1. What is it?

A network is like a **city's address system**:
- Every house has an **address** (IP address).
- Houses on the same street are in the same **neighborhood** (subnet).
- To leave the neighborhood, you go through the **main gate** (default gateway = your router).
- Inside a house, there are many **doors** for different services (ports).
- A **phone book** turns names into addresses (DNS).

## 2. Why this homelab needs it

Every VM gets a fixed address so other machines can always find it. Nginx must know "Jenkins is at `192.168.0.226` door `8080`". If addresses change, everything breaks.

## 3. Key words

| Word | Meaning | Example in this lab |
|---|---|---|
| **IP address** | A machine's address | `192.168.0.221` |
| **Subnet / CIDR** | The range of addresses in your local network. `/24` means the first 3 numbers are fixed | `192.168.0.0/24` = `192.168.0.1` – `192.168.0.254` |
| **Default gateway** | The router; the way out to the internet | `192.168.0.1` |
| **Static IP** | An address you set by hand; never changes | All VMs `.220`–`.227` |
| **DHCP** | The router automatically gives out addresses | Your phone, laptop |
| **DHCP pool** | The range the router hands out automatically | Must **not** include `.220`–`.229` |
| **Port** | A numbered "door" on a machine for one service | `22` SSH, `80` HTTP, `443` HTTPS, `5432` PostgreSQL, `6443` K3s API, `8080` Jenkins, `8006` Proxmox |
| **TCP / UDP** | Two ways to send data. TCP = checked delivery; UDP = fast, no check | Flannel uses UDP `8472` |
| **DNS** | Turns names into IP addresses | `app.sopanha.digital` → Cloudflare |
| **Bridge (`vmbr0`)** | A virtual network switch in Proxmox; plugs VMs into the real network | All guests use `vmbr0` |
| **VIP (virtual IP)** | An extra IP that can move between machines | `192.168.0.229` for K3s ingress |
| **Port forwarding** | Opening a door on the router from internet → inside | **We don't use it** (Cloudflare Tunnel instead) |
| **NAT** | Many inside devices share one public IP | Your home router does this |
| **Loopback (`127.0.0.1`)** | "Myself". Only the same machine can reach it | Nginx listens on `127.0.0.1:80` |
| **Firewall** | Rules that allow/block traffic | Proxmox firewall, `ufw` |

## 4. How it works in this lab

```text
Internet ─── Router 192.168.0.1 ─── (LAN switch) ─── ML30 NIC
                                                        │
                                                   vmbr0 (bridge)
             ┌──────────┬──────────┬──────────┬────────┼────────┬──────────┐
           .220       .221       .222       .223     .225     .226       .227
          gateway   k3s-srv   worker1    worker2  postgres  jenkins   hosting
                                 └─── VIP .229 ───┘
```

The ML30 itself (Proxmox) is `192.168.0.200`.

## 5. Step by step for this project

1. **Find your current network** on your Mac:
   ```bash
   ipconfig getifaddr en0      # your Mac's IP
   netstat -nr | grep default  # your router (gateway)
   ```
2. **Log in to your router** (usually `http://192.168.0.1`).
3. **Check the DHCP pool.** If it is, for example, `.2`–`.254`, change it to `.2`–`.199` so `.200`–`.254` are free for servers.
4. **Check nothing already uses the planned IPs**:
   ```bash
   for i in 200 220 221 222 223 224 225 226 227 229; do
     ping -c1 -W1 192.168.0.$i >/dev/null && echo "192.168.0.$i IN USE" || echo "192.168.0.$i free"
   done
   ```
5. During Proxmox install, give the host `192.168.0.200/24`, gateway `192.168.0.1`.
6. Each VM gets its static IP through **cloud-init** (see [03 Proxmox](../03-proxmox/README.md)).

## 6. Useful commands (Linux)

```bash
ip a                     # show my IP addresses
ip r                     # show routes (look for "default via")
ping 192.168.0.1         # can I reach the router?
ping 1.1.1.1             # can I reach the internet?
ping google.com          # does DNS work?
ss -tlnp                 # which ports are listening on this machine?
nc -zv 192.168.0.225 5432   # is port 5432 open on the DB?
curl -I http://192.168.0.226:8080   # does Jenkins answer?
dig app.sopanha.digital  # DNS lookup
```

## 7. Common mistakes

- **IP conflict:** two devices with the same IP → both act strangely. Always exclude server IPs from DHCP.
- **Wrong subnet mask:** `/24` vs `/16` mistakes mean machines can't see each other.
- **Forgetting the gateway:** the VM can talk to the LAN but not the internet.
- **"It works by IP but not by name"** → DNS problem, not network problem.
- **Service listening on `127.0.0.1` only** → can't be reached from other machines (sometimes that's what you want, like Nginx here).

## 8. Check yourself

1. What does `/24` mean in `192.168.0.0/24`?
2. Why must `.220`–`.229` be outside the DHCP pool?
3. Which port does PostgreSQL use? Should it be reachable from the internet?
4. Why does Nginx listen on `127.0.0.1` and not `0.0.0.0`?

<details><summary>Answers</summary>

1. The first 24 bits (`192.168.0`) are the network; the last number (1–254) is the device.
2. So the router never gives those addresses to another device → no conflict.
3. 5432. No — private only.
4. Only `cloudflared` on the same machine needs to reach it. Nothing else on the LAN can bypass Cloudflare.

</details>

## 9. Learn more

- Proxmox network config: <https://pve.proxmox.com/wiki/Network_Configuration>
- Subnet calculator: <https://www.subnet-calculator.com/cidr.php>
