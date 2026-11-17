# 10 — Cloudflare (DNS, Tunnel, Access, R2)

> Puts your homelab websites on the internet safely, without opening any port on your router.

## 1. What is it?

Cloudflare is a big internet company. This lab uses four of its services:

| Service | What it does | Comparison |
|---|---|---|
| **DNS** | `app.sopanha.digital` → Cloudflare | The phone book |
| **Tunnel (`cloudflared`)** | Your server connects **out** to Cloudflare; visitors come back through that connection | A private underground tunnel — no front door on your house |
| **Access (Zero Trust)** | Login wall in front of admin sites (Jenkins, Grafana) | A guard who checks ID before letting you in |
| **R2** | Cheap file storage for backups ([25](../25-backups/README.md)) | A storage unit in another city |

## 2. Why this homelab uses it

- **No port forwarding** → your home IP and router are not exposed.
- Free HTTPS certificates automatically.
- DDoS protection.
- Admin pages protected by Access (email code / Google login) **before** anyone even sees the Jenkins login page.

## 3. Key words

| Word | Meaning |
|---|---|
| **Zone** | Your domain in Cloudflare (`sopanha.digital`) |
| **Proxied (orange cloud)** | Traffic goes through Cloudflare |
| **Tunnel** | Named connection from `cloudflared` to Cloudflare |
| **Connector** | The running `cloudflared` program |
| **Public hostname** | Rule: `jenkins.sopanha.digital` → `http://127.0.0.1:80` |
| **Remotely-managed tunnel** | Configured in the Cloudflare dashboard; server only needs a token (easiest) |
| **Access application** | A hostname protected by an Access policy |
| **Policy** | Who is allowed: e.g., "emails ending in your address" |
| **Service token** | Lets a machine (not a person) pass Access |

## 4. How it works

```text
Visitor ──HTTPS──► Cloudflare edge
                     │ 1. Is this host protected by Access? → ask for login
                     │ 2. Forward through the tunnel
                     ▼
         (outbound connection, started by your server)
                     ▼
           cloudflared in LXC 100 ──► Nginx 127.0.0.1:80 ──► backend
```

Your router only sees **outgoing** traffic — nothing inbound is opened.

## 5. Step by step for this project

### 5.1 Domain
1. Add `sopanha.digital` to Cloudflare and change nameservers at your registrar (if not already done).

### 5.2 Create the tunnel (dashboard method — easiest)
1. Cloudflare dashboard → **Zero Trust** → **Networks → Tunnels** → **Create a tunnel** → type **Cloudflared** → name `homelab-gateway`.
2. Choose **Debian/Ubuntu**; copy the install command. In LXC 100 it looks like:
   ```bash
   # install cloudflared package (follow the dashboard's exact commands), then:
   sudo cloudflared service install <YOUR-TUNNEL-TOKEN>
   systemctl status cloudflared
   ```
3. Back in the dashboard → **Public Hostnames** → add one per site, all pointing to **`http://127.0.0.1:80`**:
   - `app.sopanha.digital`
   - `api.sopanha.digital`
   - `grafana.sopanha.digital`
   - `jenkins.sopanha.digital`
   - `hosting.sopanha.digital`
   - `n8n.sopanha.digital` (optional)

   Cloudflare creates the DNS records automatically.

### 5.3 Protect admin sites with Access (do this BEFORE step 5.2.3 for admin hosts)
1. Zero Trust → **Access → Applications** → **Add** → **Self-hosted**.
2. Application domain: `jenkins.sopanha.digital`.
3. Policy: **Allow** → Include → **Emails** → your email.
4. Login method: **One-time PIN** (email code) is the simplest.
5. Repeat for `grafana.` and `n8n.`
6. Test in a private browser window: you should see the Cloudflare login first.

### 5.4 Settings worth turning on
- SSL/TLS mode: **Full** (tunnel already encrypts; Nginx speaks HTTP locally).
- Security → WAF basic rules; Bot Fight Mode (optional).

## 6. Useful commands

```bash
systemctl status cloudflared
journalctl -u cloudflared -f
cloudflared --version
curl -I https://app.sopanha.digital
curl -I https://jenkins.sopanha.digital     # expect redirect to Access login (302)
```

## 7. Common mistakes

- **Exposing Jenkins/Grafana before Access is on.** Bots find new hostnames quickly.
- Pointing the tunnel to `localhost:80` when Nginx is in another machine (here they're in the same LXC, so `127.0.0.1` is right).
- **Error 1033** = tunnel not connected → check `cloudflared` service.
- **Error 502** from Cloudflare = tunnel OK but Nginx/backend not answering.
- Sharing the tunnel token — anyone with it can run your tunnel.
- Publishing Proxmox `:8006`, PostgreSQL, or iLO through the tunnel. Use Tailscale instead.

## 8. Check yourself

1. Why don't we need port forwarding?
2. What does Access add on top of the Jenkins login?
3. What does Cloudflare error 1033 mean?

<details><summary>Answers</summary>

1. `cloudflared` creates an outbound connection; visitors reach us through it.
2. A second login wall at Cloudflare, so attackers never reach Jenkins at all.
3. The tunnel connector is offline.

</details>

## 9. Learn more

- Cloudflare Tunnel: <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/>
- Cloudflare Access: <https://developers.cloudflare.com/cloudflare-one/access-controls/>
- R2: <https://developers.cloudflare.com/r2/>
