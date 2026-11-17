# 25 — Backups: vzdump, restic, rclone & Cloudflare R2

> Copies of your important data, stored **outside** the server, so a dead SSD or a mistake doesn't lose everything.

## 1. What is it?

| Tool | What it does | Comparison |
|---|---|---|
| **vzdump** | Proxmox's built-in tool: saves a **whole VM/LXC** into one file | Photocopy of the entire apartment |
| **restic** | Backs up **folders of data**; encrypted, deduplicated (only changes are uploaded) | Photocopy of only the important documents, in a locked box |
| **rclone** | Copies files to/from cloud storage ("rsync for the cloud") | The delivery truck |
| **Cloudflare R2** | Cheap, S3-compatible cloud storage | A storage unit in another city |

## 2. Why this homelab uses it

There is **only one SSD** and **no backup disk**. So the plan is:

1. **Machines** → rebuilt from Git (OpenTofu + Ansible + Flux). No need to back up workers or the gateway.
2. **Data** → restic, **daily**, to R2.
3. **Full copies of stateful guests** → vzdump, **weekly**, to a 50 GB staging area, then rclone to R2.

| What | Tool | When | Keep |
|---|---|---|---|
| PostgreSQL dumps, Jenkins home, Docker volumes on hosting VM | restic → R2 | Daily | 7 daily, 4 weekly, 6 monthly |
| VMs/LXCs 101, 105, 106, 107 | vzdump → staging → rclone → R2 | Weekly | 1 local, 4 in R2 |
| Workers 102/103, gateway 100 | not backed up | — | rebuild from Git |

## 3. Key words

| Word | Meaning |
|---|---|
| **3-2-1 rule** | 3 copies, 2 different media, 1 off-site. We have: live data + R2 (+ Git for config). Not perfect, but off-site is the most important part |
| **Snapshot mode** | vzdump backs up a running guest without stopping it |
| **Repository (restic)** | The encrypted backup store (an R2 path) |
| **Deduplication** | Same data stored only once → small daily uploads |
| **Retention / prune** | Delete old backups by rule |
| **S3-compatible** | Speaks the Amazon S3 API — most tools support it |
| **RPO** | How much data you can lose (here: up to 1 day) |
| **RTO** | How long recovery takes (here: hours — downloads needed) |

## 4. How it works

```text
Daily:   postgres / jenkins / hosting ──restic──► R2: homelab-backups/restic
Weekly:  Proxmox ──vzdump──► /mnt/backup-staging ──rclone──► R2: homelab-backups/vzdump
Always:  homelab-infra repo ──git──► GitHub
Secrets (restic password, R2 keys, age key) ──► your password manager
```

## 5. Step by step for this project

### 5.1 Create the R2 bucket
1. Cloudflare dashboard → **R2** → Create bucket `homelab-backups`.
2. R2 → **Manage API tokens** → Create token → **Object Read & Write**, only this bucket.
3. Save: **Access Key ID**, **Secret Access Key**, and endpoint `https://<ACCOUNT_ID>.r2.cloudflarestorage.com` — in your password manager.

### 5.2 Weekly vzdump + rclone (on the Proxmox host)
1. Staging storage — see main README §2 "Backup staging area".
2. **Datacenter → Backup → Add**:
   - Storage: `backup-staging`
   - Schedule: `sun 02:00`
   - Selection: 101, 105, 106, 107
   - Mode: **Snapshot**, Compression: **ZSTD**
   - Retention: **Keep Last = 1**
3. Install and configure rclone:
   ```bash
   apt install -y rclone
   rclone config     # n → name: r2 → type: s3 → provider: Cloudflare → keys → endpoint
   rclone lsd r2:    # should list homelab-backups
   ```
4. Upload after the backup — `/etc/cron.d/vzdump-to-r2`:
   ```text
   0 6 * * 0 root rclone copy /mnt/backup-staging/dump r2:homelab-backups/vzdump --max-age 2d && rclone delete r2:homelab-backups/vzdump --min-age 29d
   ```

### 5.3 Daily restic (on postgres, jenkins, hosting — via Ansible)
1. Install and set up credentials (`/root/.restic-env`, mode `600`):
   ```bash
   apt install -y restic
   cat > /root/.restic-env <<'EOF'
   export RESTIC_REPOSITORY="s3:https://<ACCOUNT_ID>.r2.cloudflarestorage.com/homelab-backups/restic"
   export AWS_ACCESS_KEY_ID="<r2-access-key-id>"
   export AWS_SECRET_ACCESS_KEY="<r2-secret>"
   export RESTIC_PASSWORD="<long random password — also in password manager>"
   EOF
   chmod 600 /root/.restic-env
   . /root/.restic-env && restic init      # only once, from one host
   ```
2. Backup script `/usr/local/bin/backup.sh` (paths differ per host):
   ```bash
   #!/bin/bash
   set -euo pipefail
   . /root/.restic-env
   # postgres:  /var/backups/pg
   # jenkins:   /var/lib/jenkins  (exclude workspace and caches)
   # hosting:   /var/lib/docker/volumes
   restic backup /var/backups/pg --tag daily
   restic forget --host "$(hostname)" --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
   ```
   Jenkins example: `restic backup /var/lib/jenkins --exclude /var/lib/jenkins/workspace --exclude '/var/lib/jenkins/caches'`.
3. Schedule it — `/etc/cron.d/restic`:
   ```text
   30 3 * * * root /usr/local/bin/backup.sh >> /var/log/restic.log 2>&1
   ```
4. **Heartbeat alert:** add `&& curl -fsS https://hc-ping.com/<uuid>` at the end (free Healthchecks.io). If a backup doesn't report, you get an email — works even if the whole server is dead.

### 5.4 Practice restores (do this — untested backups don't count)
```bash
# restic: list and restore files
. /root/.restic-env
restic snapshots
restic restore latest --host postgres --target /tmp/restore

# vzdump from R2: download then restore as a NEW id to test
rclone copy r2:homelab-backups/vzdump/<file>.vma.zst /mnt/backup-staging/dump/
qmrestore /mnt/backup-staging/dump/<file>.vma.zst 906 --storage local-lvm      # VM
pct restore 905 /mnt/backup-staging/dump/<file>.tar.zst --storage local-lvm    # LXC
```

## 6. Useful commands

```bash
rclone ls r2:homelab-backups/vzdump
rclone size r2:homelab-backups
restic snapshots
restic stats
restic check           # verify repository health (run monthly)
ls -lh /mnt/backup-staging/dump
```

## 7. Common mistakes

- **Losing the restic password or age key** → backups can't be opened. Store them outside the server.
- Never testing a restore.
- Backup staging filling up → thin pool fills → everything breaks. Keep "Keep Last = 1".
- Backing up a running database by copying its files. Use `pg_dump` first, then back up the dump.
- Assuming the cloud copy exists — check `rclone ls` and the heartbeat alert.
- Slow upload: the first full vzdump upload can take hours on a home connection.

## 8. Check yourself

1. Why don't we back up the K3s workers?
2. Where must the restic password be stored?
3. What's the difference between vzdump and restic?

<details><summary>Answers</summary>

1. They're stateless and can be rebuilt from Git.
2. Outside the server — e.g., a password manager.
3. vzdump copies a whole guest; restic copies selected folders, encrypted and deduplicated.

</details>

## 9. Learn more

- Proxmox backup and restore: <https://pve.proxmox.com/wiki/Backup_and_Restore>
- restic: <https://restic.readthedocs.io/>
- rclone + R2: <https://rclone.org/s3/#cloudflare-r2>
- Cloudflare R2: <https://developers.cloudflare.com/r2/>
- Healthchecks.io: <https://healthchecks.io/>
