# 06 — Docker & Containers (and Docker Compose)

> Package an app with everything it needs, so it runs the same on any machine.

## 1. What is it?

A **container** is a small, isolated box that holds an app plus its libraries and settings. It shares the Linux kernel with the host, so it starts in seconds and uses little RAM.

Comparison: a **shipping container**. The ship (server) doesn't care what's inside; every container has the same shape, so it can be moved and stacked anywhere.

- **Image** = the recipe/template (read-only). Example: `postgres:16`, `nginx:1.27`.
- **Container** = a running copy of an image.
- **Docker** = the tool that builds images and runs containers.
- **Docker Compose** = a YAML file that starts several containers together.

## 2. Why this homelab uses it

- **VM 107 (hosting)** runs apps like n8n with Docker Compose.
- **K3s** runs the same kind of images (with `containerd`, not Docker).
- **Jenkins** builds your app into an image.

Containers vs VMs: a VM packages a whole computer; a container packages one app.

## 3. Key words

| Word | Meaning |
|---|---|
| **Dockerfile** | Text file with steps to build an image |
| **Image tag** | Version label: `myapp:1.4.2`, `myapp:a1b2c3d` (git commit) |
| **Registry** | Online storage for images (Docker Hub, GHCR) — see [07](../07-registry-ghcr/README.md) |
| **Volume** | Storage that survives container restarts (database files!) |
| **Port mapping** | `-p 8080:80` = host port 8080 → container port 80 |
| **Environment variable** | Settings passed in: `-e TZ=Asia/Phnom_Penh` |
| **Layer** | Each Dockerfile step is a cached layer → faster rebuilds |
| **containerd** | The lower-level engine. K3s uses it directly |
| **BuildKit** | Modern image builder (used by Docker and by our Jenkins pipeline) |

## 4. How it works

```text
Dockerfile ──docker build──► Image ──docker push──► Registry (GHCR)
                                │
                          docker run / compose / Kubernetes
                                ▼
                            Container (running)
```

## 5. Step by step for this project (VM 107)

1. **Install Docker** (official script for Ubuntu):
   ```bash
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker $USER   # log out and back in
   docker run --rm hello-world
   ```
2. **A simple Dockerfile** (for your own app):
   ```dockerfile
   FROM node:22-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci --omit=dev
   COPY . .
   USER node
   EXPOSE 3000
   CMD ["node", "server.js"]
   ```
3. **A Compose file** for n8n — `~/apps/n8n/compose.yaml`:
   ```yaml
   services:
     n8n:
       image: docker.n8n.io/n8nio/n8n:latest   # pin a real version in practice
       restart: unless-stopped
       ports:
         - "5678:5678"
       environment:
         - TZ=Asia/Phnom_Penh
         - N8N_HOST=n8n.sopanha.digital
         - N8N_PROTOCOL=https
         - WEBHOOK_URL=https://n8n.sopanha.digital/
       volumes:
         - n8n_data:/home/node/.n8n
   volumes:
     n8n_data:
   ```
4. **Start it:**
   ```bash
   cd ~/apps/n8n
   docker compose up -d
   docker compose logs -f
   ```
5. Keep Compose files in Git; back up the volumes with restic ([25](../25-backups/README.md)).

## 6. Useful commands

```bash
docker ps                      # running containers
docker ps -a                   # including stopped
docker images                  # local images
docker logs -f <name>          # follow logs
docker exec -it <name> sh      # shell inside a container
docker compose up -d           # start stack
docker compose down            # stop stack (volumes kept)
docker compose pull && docker compose up -d   # update images
docker system df               # disk used by Docker
docker system prune            # clean unused stuff (careful)
```

## 7. Common mistakes

- **No volume for data** → data lost when the container is recreated.
- **Using `:latest` everywhere** → you don't know which version runs; updates surprise you.
- **Running as root inside the container** when not needed.
- **Publishing ports to everyone** (`0.0.0.0`) — on the hosting VM, prefer only what Nginx needs.
- **Disk fills up** with old images → `docker system prune` regularly.
- Docker bypasses `ufw` firewall rules for published ports — use the Proxmox firewall instead.

## 8. Check yourself

1. Image vs container — what's the difference?
2. Where does n8n's data live in the Compose example?
3. Why is `myapp:a1b2c3d` a better tag than `myapp:latest`?

<details><summary>Answers</summary>

1. Image = template; container = running instance of it.
2. In the named volume `n8n_data`.
3. It points to one exact version (git commit), so you know exactly what runs and can roll back.

</details>

## 9. Learn more

- Docker docs: <https://docs.docker.com/get-started/>
- Compose file reference: <https://docs.docker.com/reference/compose-file/>
- Dockerfile best practices: <https://docs.docker.com/build/building/best-practices/>
