# 09 — Nginx (Reverse Proxy)

> A fast web server that receives requests and forwards each one to the correct backend.

## 1. What is it?

**Nginx** (say "engine-x") can serve websites, but here we use it as a **reverse proxy**: it stands in front of other servers and passes each request to the right one based on the **domain name** (Host header).

Comparison: the **reception desk** in a building. A visitor says "I'm here for Jenkins" → reception sends them to room `.226:8080`.

## 2. Why this homelab uses it

In **LXC 100** (`gateway`), Nginx receives everything that comes through the Cloudflare Tunnel and routes it:

| Hostname | Goes to |
|---|---|
| `app.` / `api.` / `grafana.sopanha.digital` | Traefik VIP `192.168.0.229:80` |
| `jenkins.sopanha.digital` | `192.168.0.226:8080` |
| `hosting.sopanha.digital` | `192.168.0.227:80` |
| anything else | `404` |

> Note: `cloudflared` can do this routing by itself. We keep Nginx to learn it and to have one place for logs, headers, and limits.

## 3. Key words

| Word | Meaning |
|---|---|
| **Reverse proxy** | Receives client requests, forwards to backend servers |
| **`server` block** | Config for one or more hostnames (virtual host) |
| **`server_name`** | Which domain(s) this block answers |
| **`listen`** | Which IP:port Nginx listens on |
| **`location`** | Rules for URL paths (`/`, `/api`) |
| **`upstream`** | A named group of backend servers |
| **`proxy_pass`** | "Send the request here" |
| **Headers** | Extra info: `Host`, `X-Forwarded-For` (real client IP), `X-Forwarded-Proto` (https) |
| **WebSocket upgrade** | Needed for live apps (Jenkins, Grafana live, n8n) |
| **`default_server`** | Catch-all block for unknown hostnames |

## 4. How it works

```text
cloudflared ──► 127.0.0.1:80 (Nginx)
                    │ reads "Host: jenkins.sopanha.digital"
                    ▼
            server_name jenkins... → proxy_pass http://192.168.0.226:8080
```

## 5. Step by step for this project

1. **Install** in LXC 100:
   ```bash
   apt update && apt install -y nginx
   rm /etc/nginx/sites-enabled/default
   ```
2. **Create** `/etc/nginx/conf.d/homelab.conf` — copy the full example from the main [README §6](../../README.md#6-cloudflare-and-nginx-design).
3. **Test and reload:**
   ```bash
   nginx -t                 # ALWAYS test before reload
   systemctl reload nginx
   ```
4. **Test routing locally** (before Cloudflare):
   ```bash
   curl -I -H 'Host: jenkins.sopanha.digital' http://127.0.0.1/
   curl -I -H 'Host: unknown.example' http://127.0.0.1/    # expect 404
   ```
5. **Optional: hide version and limit request size** in `/etc/nginx/nginx.conf` `http {}`:
   ```nginx
   server_tokens off;
   client_max_body_size 50m;
   ```
6. Alloy ([24](../24-alloy/README.md)) ships `/var/log/nginx/*.log` to Loki.

## 6. Useful commands

```bash
nginx -t                          # check config syntax
systemctl reload nginx            # apply config without downtime
systemctl status nginx
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
ss -tlnp | grep nginx             # where is it listening?
```

## 7. Common mistakes

- **Reload without `nginx -t`** → typo takes the gateway down.
- Missing `proxy_set_header Host $host;` → backend sees the wrong hostname.
- Missing WebSocket headers → Jenkins/Grafana pages partly broken.
- **`502 Bad Gateway`** = Nginx works but the backend is down or wrong IP/port.
- **`504 Gateway Timeout`** = backend too slow.
- Thinking Host-based routing is security. It's not — use Cloudflare Access + app login.

## 8. Check yourself

1. What does `502` mean?
2. Why listen on `127.0.0.1` here?
3. How do you apply a config change safely?

<details><summary>Answers</summary>

1. Nginx couldn't get a good answer from the backend.
2. Only `cloudflared` on the same LXC should reach it.
3. `nginx -t` then `systemctl reload nginx`.

</details>

## 9. Learn more

- Nginx beginner's guide: <https://nginx.org/en/docs/beginners_guide.html>
- Reverse proxy: <https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/>
