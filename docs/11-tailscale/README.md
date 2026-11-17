# 11 — Tailscale

> A private, encrypted network between your devices — so you can manage the homelab from anywhere.

## 1. What is it?

**Tailscale** is a VPN built on **WireGuard**. You install it on your devices, log in, and they can talk to each other privately as if they were on the same LAN — even when you're at a café or using mobile data.

Comparison: **a secret private door** into your house that only your own keys open. Cloudflare Tunnel is the **public shop entrance**; Tailscale is the **staff-only back door**.

## 2. Why this homelab uses it

Some things must **never** be on the public internet:
- Proxmox web UI (`:8006`)
- SSH to all guests
- iLO
- K3s API (`:6443`), PostgreSQL

Tailscale lets **you** reach them from anywhere, with no open router ports.

## 3. Key words

| Word | Meaning |
|---|---|
| **Tailnet** | Your private Tailscale network |
| **Node** | A device in your tailnet (Mac, phone, Proxmox host) |
| **Tailscale IP** | Each node gets a `100.x.y.z` address |
| **Subnet router** | A node that lets your tailnet reach a whole LAN (`192.168.0.0/24`) |
| **MagicDNS** | Use names like `pve` instead of IPs |
| **ACLs** | Rules for who can reach what inside the tailnet |
| **Key expiry** | Device login expires (default 180 days) — disable for servers |

## 4. How it works here

```text
Your Mac (anywhere, 100.x.x.x)
      │  encrypted WireGuard
      ▼
Proxmox host (subnet router, advertises 192.168.0.0/24)
      │
      ▼
192.168.0.200:8006 (Proxmox UI), 192.168.0.221 (SSH), 192.168.0.210 (iLO), ...
```

Only the Proxmox host runs Tailscale; it shares the whole LAN with you.

## 5. Step by step for this project

1. Create a free account at <https://tailscale.com> and install the app on your Mac.
2. On the **Proxmox host**:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-tailscale.conf
   echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.d/99-tailscale.conf
   sysctl -p /etc/sysctl.d/99-tailscale.conf
   tailscale up --advertise-routes=192.168.0.0/24
   ```
3. In the Tailscale **admin console** → Machines → the Proxmox host:
   - **Edit route settings** → approve `192.168.0.0/24`
   - **Disable key expiry**
4. On your Mac, make sure **"Use Tailscale subnets"** is on. Now open `https://192.168.0.200:8006` from anywhere.
5. Turn on **two-factor authentication** on the account you log in to Tailscale with.

## 6. Useful commands

```bash
tailscale status          # who is connected
tailscale ip -4           # my Tailscale IP
tailscale ping <node>     # test connection
tailscale up --advertise-routes=192.168.0.0/24   # re-advertise
```

## 7. Common mistakes

- Forgetting to **approve** the subnet route in the admin console → can't reach `192.168.0.x`.
- Key expiry on the server → remote access stops after months.
- Testing from **inside** your home LAN and thinking it works remotely — test from mobile data.
- If your home LAN at another location also uses `192.168.0.0/24`, routes clash. (A reason some people pick unusual subnets.)

## 8. Check yourself

1. Why use Tailscale and not Cloudflare Tunnel for Proxmox UI?
2. What is a subnet router?

<details><summary>Answers</summary>

1. Proxmox UI is admin-only and very sensitive; it should not be a public website at all.
2. A Tailscale node that gives your tailnet access to a whole local network.

</details>

## 9. Learn more

- Subnet routers: <https://tailscale.com/kb/1019/subnets>
- Tailscale on Proxmox: <https://tailscale.com/kb/1133/proxmox>
