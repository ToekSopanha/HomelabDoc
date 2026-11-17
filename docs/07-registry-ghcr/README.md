# 07 — Container Registry (GHCR)

> An online warehouse for container images.

## 1. What is it?

A **registry** stores Docker images so other machines can download (**pull**) them. **GHCR** (GitHub Container Registry, `ghcr.io`) is GitHub's registry. It sits next to your code and uses your GitHub login.

Comparison: Jenkins is the **factory**, GHCR is the **warehouse**, K3s is the **shop** that takes products from the warehouse.

## 2. Why this homelab uses it

- Jenkins **pushes** each new app image to GHCR.
- K3s **pulls** images from GHCR to run them.
- Uses **no RAM or disk on your server** (unlike self-hosted Harbor).

Alternative: a tiny self-hosted `registry:2` container on VM 107 — only if you want everything local.

## 3. Key words

| Word | Meaning |
|---|---|
| **Image name** | `ghcr.io/<user>/<app>:<tag>` |
| **Push / pull** | Upload / download an image |
| **Public / private package** | Public = anyone can pull. Private = needs a token |
| **PAT** | Personal Access Token. For GHCR: scope `write:packages` (Jenkins) or `read:packages` (K3s) |
| **imagePullSecret** | Kubernetes secret holding the token to pull private images |

## 4. Step by step for this project

1. On GitHub → Settings → Developer settings → **Personal access tokens (classic)**:
   - Token **A** for Jenkins: `write:packages`, `read:packages`
   - Token **B** for K3s: `read:packages` only
2. Test from your Mac:
   ```bash
   echo "$TOKEN_A" | docker login ghcr.io -u <your-user> --password-stdin
   docker tag myapp:dev ghcr.io/<your-user>/myapp:test
   docker push ghcr.io/<your-user>/myapp:test
   ```
3. Store token A in **Jenkins Credentials** (never in the Jenkinsfile).
4. Let K3s pull private images — create the secret (later stored encrypted with SOPS):
   ```bash
   kubectl create secret docker-registry ghcr-pull \
     --docker-server=ghcr.io --docker-username=<your-user> \
     --docker-password="$TOKEN_B" -n myapp
   ```
   And in the Deployment:
   ```yaml
   spec:
     imagePullSecrets:
       - name: ghcr-pull
   ```
5. Use the **git commit SHA** as the tag: `ghcr.io/<user>/myapp:a1b2c3d`.

## 5. Common mistakes

- Token with too many permissions (e.g., full `repo`). Keep it minimal.
- Forgetting `imagePullSecrets` → pod stuck in `ImagePullBackOff`.
- Lots of old images → set a cleanup policy in GitHub package settings.

## 6. Check yourself

1. Which token does K3s need — read or write?
2. What error do you see when a pod can't pull an image?

<details><summary>Answers</summary>

1. Read only.
2. `ErrImagePull` / `ImagePullBackOff`.

</details>

## 7. Learn more

- GHCR docs: <https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry>
