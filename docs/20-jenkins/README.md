# 20 — Jenkins & BuildKit (Continuous Integration)

> Jenkins automatically tests and builds your code every time you push. BuildKit turns the code into a container image.

## 1. What is it?

- **Jenkins** is an automation server. A **pipeline** (written in a `Jenkinsfile`) lists steps: get code → test → build image → push → update Git.
- **CI (Continuous Integration)** = every push is automatically built and tested, so mistakes are found early.
- **BuildKit** is the modern engine that builds container images from a `Dockerfile`. **Rootless** mode runs without root privileges — safer inside Kubernetes.

Comparison: Jenkins is the **factory manager**. When new materials (code) arrive, it calls workers (build pods), they assemble the product (image) and send it to the warehouse (GHCR).

## 2. Why this homelab uses it

- Learn the classic CI tool used in many companies
- **Controller only on VM 106** (2.5 GB). Builds happen in **temporary pods in K3s**, with memory limits — so a heavy build can't freeze the server
- Jenkins **does not deploy**; it updates Git and Flux deploys

> Note: **Kaniko** (an older image builder) was archived by Google in 2025 — use BuildKit.

## 3. Key words

| Word | Meaning |
|---|---|
| **Controller** | The Jenkins server (web UI, schedules jobs) |
| **Agent** | Where a build actually runs (here: K3s pods) |
| **Executor** | A slot to run a build. Controller has **0** |
| **Pipeline / Jenkinsfile** | Build steps as code, stored in the app repo |
| **Stage / step** | A group of steps / one action |
| **Multibranch pipeline** | Auto-creates a job for each Git branch |
| **Credentials** | Stored secrets (GHCR token, Git deploy key) |
| **Kubernetes plugin** | Lets Jenkins create agent pods in K3s |
| **podTemplate** | Describes the agent pod's containers |
| **Webhook** | GitHub tells Jenkins "new push" |

## 4. How it works

```text
git push ─► GitHub ─webhook─► Jenkins controller (VM 106)
                                   │ "create pod in namespace ci"
                                   ▼
                    K3s build pod: [git] [node] [buildkit]
                        1. test
                        2. build image ──push──► GHCR
                        3. update tag in homelab-infra ──push──► GitHub ──► Flux deploys
                    pod is deleted
```

## 5. Step by step for this project

### 5.1 Install the controller (VM 106)
Follow the current official commands for Ubuntu (they include adding the Jenkins apt key and repo):
<https://www.jenkins.io/doc/book/installing/linux/#debianubuntu>

In short:
```bash
sudo apt install -y fontconfig openjdk-21-jre
# add Jenkins LTS apt repo + key (copy exact lines from the link above)
sudo apt update && sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Open `http://192.168.0.226:8080` → install suggested plugins → create admin user.

Then:
- **Manage Jenkins → Nodes → Built-In Node → Number of executors = 0**
- Install plugins: **Kubernetes**, **Prometheus metrics**, **Pipeline: Stage View**, **GitHub Branch Source**

### 5.2 Give Jenkins a limited K3s account
```yaml
# ci-rbac.yaml (commit to homelab-infra via Flux)
apiVersion: v1
kind: Namespace
metadata: { name: ci }
---
apiVersion: v1
kind: ServiceAccount
metadata: { name: jenkins, namespace: ci }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: jenkins-agents, namespace: ci }
rules:
  - apiGroups: [""]
    resources: [pods, pods/exec, pods/log, events]
    verbs: [get, list, watch, create, delete]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: jenkins-agents, namespace: ci }
subjects: [{ kind: ServiceAccount, name: jenkins, namespace: ci }]
roleRef: { kind: Role, name: jenkins-agents, apiGroup: rbac.authorization.k8s.io }
---
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-token
  namespace: ci
  annotations: { kubernetes.io/service-account.name: jenkins }
type: kubernetes.io/service-account-token
```
Get the token: `kubectl get secret jenkins-token -n ci -o jsonpath='{.data.token}' | base64 -d`

### 5.3 Connect Jenkins to K3s
**Manage Jenkins → Clouds → New cloud → Kubernetes**:
- Kubernetes URL: `https://192.168.0.221:6443`
- Disable https certificate check (homelab) or paste the cluster CA
- Namespace: `ci`
- Credentials: **Secret text** = the token above
- Jenkins URL: `http://192.168.0.226:8080`
- **WebSocket**: enabled (agents connect back over HTTP, no extra port)
- Container cap: **1** (one build at a time)
- **Test Connection** ✅

### 5.4 Credentials
- `ghcr` → Username with password (GitHub user + GHCR **write** token)
- `infra-deploy-key` → SSH private key with **write** access to `homelab-infra` only (GitHub repo → Settings → Deploy keys)

### 5.5 Jenkinsfile (in your app repo)
```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: node
      image: node:22-alpine
      command: ["sleep"]
      args: ["infinity"]
      resources:
        requests: { cpu: 250m, memory: 256Mi }
        limits:   { memory: 1Gi }
    - name: buildkit
      image: moby/buildkit:rootless
      command: ["sleep"]
      args: ["infinity"]
      env:
        - name: BUILDKITD_FLAGS
          value: --oci-worker-no-process-sandbox
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        seccompProfile: { type: Unconfined }
        appArmorProfile: { type: Unconfined }
      resources:
        requests: { cpu: 500m, memory: 512Mi }
        limits:   { memory: 2Gi }
    - name: git
      image: alpine/git
      command: ["sleep"]
      args: ["infinity"]
'''
    }
  }
  environment {
    IMAGE = 'ghcr.io/<your-user>/myapp'
  }
  stages {
    stage('Version') {
      steps {
        container('git') {
          script {
            sh 'git config --global --add safe.directory "$WORKSPACE"'
            env.TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          }
        }
      }
    }
    stage('Test') {
      steps {
        container('node') {
          sh 'npm ci && npm test'
        }
      }
    }
    stage('Build & push image') {
      steps {
        container('buildkit') {
          withCredentials([usernamePassword(credentialsId: 'ghcr', usernameVariable: 'U', passwordVariable: 'P')]) {
            sh '''
              mkdir -p ~/.docker
              AUTH=$(printf '%s:%s' "$U" "$P" | base64 | tr -d '\\n')
              printf '{"auths":{"ghcr.io":{"auth":"%s"}}}' "$AUTH" > ~/.docker/config.json
              buildctl-daemonless.sh build \
                --frontend dockerfile.v0 \
                --local context=. --local dockerfile=. \
                --output type=image,name=${IMAGE}:${TAG},push=true
            '''
          }
        }
      }
    }
    stage('Update infra repo') {
      when { branch 'main' }
      steps {
        container('git') {
          sshagent(['infra-deploy-key']) {
            sh '''
              mkdir -p ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
              git clone git@github.com:<your-user>/homelab-infra.git infra
              cd infra
              sed -i "s|image: ${IMAGE}:.*|image: ${IMAGE}:${TAG}|" apps/myapp/deployment.yaml
              git config user.email "jenkins@sopanha.digital"
              git config user.name "Jenkins"
              git commit -am "myapp: deploy ${TAG}"
              git push
            '''
          }
        }
      }
    }
  }
}
```
(`sshagent` needs the **SSH Agent** plugin. If BuildKit fails with permission errors, check the build pod's events — rootless BuildKit needs the `Unconfined` profiles shown above.)

### 5.6 Webhook
GitHub app repo → Settings → Webhooks → `https://jenkins.sopanha.digital/github-webhook/`.
Because Jenkins is behind **Cloudflare Access**, add an Access **bypass policy for the path `/github-webhook/`** only, or use a Cloudflare service token. Otherwise GitHub gets the login page.

## 6. Useful commands / places

```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
kubectl get pods -n ci -w           # watch build pods appear/disappear
kubectl describe pod <build-pod> -n ci
```
- Build log: job → build number → **Console Output**
- Script approvals: Manage Jenkins → In-process Script Approval

## 7. Common mistakes

- Builds running on the controller (executors not set to 0) → VM 106 runs out of RAM.
- Giving Jenkins the K3s **admin** kubeconfig.
- Tokens written directly in the Jenkinsfile.
- Webhook blocked by Cloudflare Access → builds never start automatically.
- Too many plugins → slow, insecure Jenkins. Install only what you use; update often.
- Several builds at once on a 4-core server → everything slows. Keep container cap = 1.

## 8. Check yourself

1. Why does the Jenkins controller have 0 executors?
2. Who deploys the new image — Jenkins or Flux?
3. Why rootless BuildKit instead of Docker-in-Docker?

<details><summary>Answers</summary>

1. So builds never run on VM 106; they run in limited K3s pods.
2. Flux. Jenkins only updates the tag in Git.
3. No privileged container / Docker daemon needed → safer.

</details>

## 9. Learn more

- Jenkins pipeline: <https://www.jenkins.io/doc/book/pipeline/>
- Kubernetes plugin: <https://plugins.jenkins.io/kubernetes/>
- BuildKit rootless: <https://github.com/moby/buildkit/blob/master/docs/rootless.md>
