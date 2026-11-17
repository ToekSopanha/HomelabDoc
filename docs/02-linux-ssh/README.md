# 02 — Linux, Ubuntu & SSH

> The operating system inside every VM, and the secure way to log in to it.

## 1. What is it?

- **Linux** is an operating system, like Windows or macOS. Most servers in the world run Linux.
- **Ubuntu Server 24.04 LTS** is a popular, beginner-friendly version of Linux. **LTS** = Long Term Support (security updates until 2029, longer with Ubuntu Pro).
- **SSH** (Secure Shell) lets you type commands on a remote machine, encrypted. It's like a secure remote keyboard.

## 2. Why this homelab uses it

Every VM and LXC runs Ubuntu. You will manage all of them through SSH from your Mac. Ansible ([17](../17-ansible/README.md)) also uses SSH to configure them.

## 3. Key words

| Word | Meaning |
|---|---|
| **Shell / terminal** | Where you type commands (`bash`, `zsh`) |
| **root** | The super-admin user. Avoid logging in as root directly |
| **sudo** | "Run this one command as admin" |
| **apt** | Ubuntu's package installer (like an app store) |
| **systemd / systemctl** | Starts and stops background services |
| **journalctl** | Reads system logs |
| **SSH key pair** | Two files: **private key** (secret, stays on your Mac) and **public key** (copied to servers). Safer than passwords |
| **`~/.ssh/authorized_keys`** | File on the server listing public keys allowed to log in |
| **`~/.ssh/config`** | File on your Mac with nicknames for servers |
| **unattended-upgrades** | Automatically installs security updates |

## 4. How SSH keys work

```text
Your Mac                                  Server
~/.ssh/id_ed25519      (PRIVATE, secret)
~/.ssh/id_ed25519.pub  (PUBLIC)  ──copy──► ~/.ssh/authorized_keys

ssh ubuntu@192.168.0.221
   server: "prove you own a key in my list"
   Mac: signs with private key → server checks with public key → ✅ logged in
```

The private key **never leaves your Mac**.

## 5. Step by step for this project

1. **Create a key on your Mac** (once):
   ```bash
   ssh-keygen -t ed25519 -C "homelab"
   # press Enter for the default path; set a passphrase
   cat ~/.ssh/id_ed25519.pub   # this is what you give to Proxmox cloud-init
   ```
2. **Give the public key to every VM** — Proxmox cloud-init does this automatically (`--sshkeys`). For the Proxmox host itself:
   ```bash
   ssh-copy-id root@192.168.0.200
   ```
3. **Make nicknames** in `~/.ssh/config` on your Mac:
   ```text
   Host pve
     HostName 192.168.0.200
     User root
   Host k3s-server
     HostName 192.168.0.221
     User ubuntu
   Host worker1
     HostName 192.168.0.222
     User ubuntu
   Host worker2
     HostName 192.168.0.223
     User ubuntu
   Host postgres
     HostName 192.168.0.225
     User root
   Host jenkins
     HostName 192.168.0.226
     User ubuntu
   Host hosting
     HostName 192.168.0.227
     User ubuntu
   ```
   Now just type `ssh worker1`.
4. **Turn off password login** on each guest (Ansible will do this for you later). In `/etc/ssh/sshd_config`:
   ```text
   PasswordAuthentication no
   PermitRootLogin prohibit-password
   ```
   Then `sudo systemctl restart ssh`. **Test a new SSH session before closing the old one.**
5. **Enable automatic security updates:**
   ```bash
   sudo apt install unattended-upgrades
   sudo dpkg-reconfigure -plow unattended-upgrades
   ```

## 6. Useful commands

```bash
# Files and folders
pwd; ls -la; cd /etc; cat file.txt; nano file.txt; less big.log
# Packages
sudo apt update && sudo apt upgrade -y
sudo apt install htop
# Services
systemctl status nginx
sudo systemctl restart nginx
sudo systemctl enable --now nginx   # start now + on boot
journalctl -u nginx -f              # follow logs of one service
# Resources
htop                 # live CPU/RAM
free -h              # memory
df -h                # disk space
du -sh /var/*        # what uses space
# Networking
ip a; ss -tlnp
# Copy files from Mac to server
scp file.txt worker1:/tmp/
```

## 7. Common mistakes

- **Locked out** after disabling passwords before testing the key. Always keep one session open while testing.
- **Wrong permissions** on `~/.ssh` (must be `700`) or `authorized_keys` (`600`) → key is ignored.
- **Sharing the private key** or putting it in Git. Never.
- **Running everything as root.** Use a normal user + `sudo`.
- **"Host key changed" warning** after rebuilding a VM with the same IP — normal for a rebuild; fix with `ssh-keygen -R 192.168.0.221`.

## 8. Check yourself

1. Which file stays secret: `id_ed25519` or `id_ed25519.pub`?
2. How do you see the logs of the `cloudflared` service?
3. How do you check free disk space?

<details><summary>Answers</summary>

1. `id_ed25519` (no `.pub`) is the private key.
2. `journalctl -u cloudflared -f`
3. `df -h`

</details>

## 9. Learn more

- Ubuntu Server docs: <https://ubuntu.com/server/docs>
- SSH keys guide: <https://www.ssh.com/academy/ssh/keygen>
- Linux command practice: <https://linuxjourney.com/>
