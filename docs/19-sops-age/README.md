# 19 — SOPS & age (Encrypted Secrets in Git)

> Store passwords in Git safely: the file is encrypted, and only your cluster (and you) can unlock it.

## 1. What is it?

- **age** is a simple, modern encryption tool. It makes a **key pair**: a public key (to lock) and a private key (to unlock).
- **SOPS** (Secrets OPerationS) encrypts **only the values** in a YAML file, leaving the keys readable. So you can still see *which* secret changed in Git, but not *what* the value is.

Comparison: a **glass box with locked envelopes inside**. Everyone can see there are envelopes labeled "DB password", but only the person with the key can open them.

## 2. Why this homelab uses it

GitOps means **everything** is in Git — including Kubernetes Secrets (DB password, GHCR token). Plain Kubernetes Secrets are only base64 (not encryption!). SOPS encrypts them; Flux decrypts them inside the cluster.

## 3. Key words

| Word | Meaning |
|---|---|
| **age public key** | Starts with `age1...` — safe to share, used to encrypt |
| **age private key** | Starts with `AGE-SECRET-KEY-...` — **secret**, used to decrypt |
| **`.sops.yaml`** | Rules: which files to encrypt, with which key, which fields |
| **`encrypted_regex`** | Which YAML fields to encrypt (`data`, `stringData`) |
| **base64** | Just encoding, **not** encryption — anyone can decode it |

## 4. How it works

```text
You: secret.yaml (plain) ──sops --encrypt (public key)──► secret.sops.yaml ──git push──► GitHub
                                                                                    │
Flux (has private key in "sops-age" Secret) ◄────────────────────────────────────────┘
   └─ decrypt ──► real Kubernetes Secret in the cluster
```

## 5. Step by step for this project

1. **Install** on your Mac:
   ```bash
   brew install sops age
   ```
2. **Make a key pair:**
   ```bash
   mkdir -p ~/.config/sops/age
   age-keygen -o ~/.config/sops/age/keys.txt
   # prints: Public key: age1xxxxxxxx...
   ```
   **Save `keys.txt` in your password manager.** If you lose it, you can't decrypt your secrets.
3. **Give the private key to Flux:**
   ```bash
   kubectl create secret generic sops-age -n flux-system \
     --from-file=age.agekey=$HOME/.config/sops/age/keys.txt
   ```
4. **Rules file** — `homelab-infra/.sops.yaml`:
   ```yaml
   creation_rules:
     - path_regex: .*\.sops\.yaml$
       encrypted_regex: ^(data|stringData)$
       age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
5. **Create and encrypt a secret:**
   ```bash
   kubectl create secret generic myapp-db -n myapp \
     --from-literal=DATABASE_URL='postgresql://app1:PASS@192.168.0.225:5432/app1' \
     --dry-run=client -o yaml > apps/myapp/db.sops.yaml
   sops --encrypt --in-place apps/myapp/db.sops.yaml
   cat apps/myapp/db.sops.yaml    # values now look like ENC[AES256_GCM,data:...]
   git add . && git commit -m "Add myapp DB secret" && git push
   ```
6. Flux's Kustomization must have the `decryption:` block (see [18](../18-flux-gitops/README.md)).
7. **Edit later:**
   ```bash
   sops apps/myapp/db.sops.yaml     # opens decrypted in your editor, re-encrypts on save
   ```

## 6. Common mistakes

- Committing the file **before** encrypting → the secret is in Git history forever. Rotate the password.
- Losing the age private key → secrets can't be read. Keep a copy in a password manager.
- Committing `keys.txt` to Git. Never.
- Thinking base64 Kubernetes Secrets are safe. They aren't.

## 7. Check yourself

1. Which key goes into `.sops.yaml` — public or private?
2. Where does Flux get the private key?
3. Is base64 encryption?

<details><summary>Answers</summary>

1. Public (`age1...`).
2. From the `sops-age` Secret in `flux-system`.
3. No.

</details>

## 8. Learn more

- Flux + SOPS guide: <https://fluxcd.io/flux/guides/mozilla-sops/>
- SOPS: <https://github.com/getsops/sops>
- age: <https://github.com/FiloSottile/age>
