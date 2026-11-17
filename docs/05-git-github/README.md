# 05 — Git & GitHub

> Git remembers every change to your files. GitHub stores your Git projects online.

## 1. What is it?

- **Git** is a "save history" tool. Every time you **commit**, Git takes a snapshot of your files with a message. You can go back to any snapshot.
- **GitHub** is a website that hosts Git repositories, so they are backed up online and tools (Jenkins, Flux) can read them.

Comparison: Git is like **"Track Changes" in Word, but for a whole folder, forever**.

## 2. Why this homelab uses it

Git is the **center of everything**:

| Repo | Contains | Who reads it |
|---|---|---|
| `homelab-infra` | OpenTofu, Ansible, Kubernetes manifests, encrypted secrets | You, Flux |
| `<your-app>` | Application code + `Dockerfile` + `Jenkinsfile` | Jenkins |
| `newplan` (this one) | Plan and learning docs | You |

If the server dies, Git still has the full description of the lab.

## 3. Key words

| Word | Meaning |
|---|---|
| **Repository (repo)** | A project folder tracked by Git |
| **Commit** | A saved snapshot with a message |
| **Branch** | A separate line of work (`main` = the official one) |
| **Clone** | Download a repo to your computer |
| **Push / Pull** | Upload your commits / download others' commits |
| **Pull Request (PR)** | Ask to merge a branch into `main`, with review |
| **Revert** | Make a new commit that undoes an old one — the GitOps "rollback" |
| **`.gitignore`** | List of files Git must never track (keys, passwords) |
| **Webhook** | GitHub calls a URL (Jenkins) when something happens |
| **Deploy key** | An SSH key that gives one tool access to one repo only |
| **Personal access token (PAT)** | A password-like token for tools; give it minimum permissions |

## 4. How it works

```text
working folder ──git add──► staging ──git commit──► local history ──git push──► GitHub
                                                                          │
                                         Jenkins (webhook) ◄──────────────┤
                                         Flux (polls every minute) ◄──────┘
```

## 5. Step by step for this project

1. Set your name (once):
   ```bash
   git config --global user.name "mr.loverman"
   git config --global user.email "you@example.com"
   ```
2. Create a private repo on GitHub called `homelab-infra`, then:
   ```bash
   git clone git@github.com:<your-user>/homelab-infra.git
   cd homelab-infra
   mkdir -p tofu ansible clusters/homelab
   ```
3. Add a `.gitignore` **before** your first commit:
   ```text
   *.tfstate
   *.tfstate.*
   .terraform/
   *.agekey
   .env
   ```
4. Normal daily loop:
   ```bash
   git status
   git add .
   git commit -m "Add postgres LXC"
   git push
   ```
5. Undo a bad deploy (GitOps rollback):
   ```bash
   git log --oneline        # find the bad commit
   git revert <commit-id>
   git push                 # Flux applies the old version again
   ```

## 6. Useful commands

```bash
git status                 # what changed?
git diff                   # show exact changes
git log --oneline --graph  # history
git switch -c my-feature   # new branch
git pull                   # get latest from GitHub
git restore file.txt       # throw away local changes to a file
```

## 7. Common mistakes

- **Committing a password or private key.** Even if you delete it later, it stays in history. Use SOPS ([19](../19-sops-age/README.md)) and `.gitignore`.
- **Committing `terraform.tfstate`** — it can contain secrets.
- Vague commit messages like "fix". Write what and why.
- Working directly on `main` for big changes — use a branch.

## 8. Check yourself

1. How do you roll back a bad deploy in GitOps?
2. Why add `.gitignore` before the first commit?

<details><summary>Answers</summary>

1. `git revert` the commit and push. Flux applies the previous state.
2. So secret files are never recorded in history.

</details>

## 9. Learn more

- Git book (free): <https://git-scm.com/book/en/v2>
- GitHub Skills: <https://skills.github.com/>
- Interactive practice: <https://learngitbranching.js.org/>
