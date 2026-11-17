# 03 — Proxmox VE (VMs, LXC, Storage, Cloud-Init)

> The software installed on the ML30 that turns one physical server into many virtual computers.

## 1. What is it?

**Proxmox VE** is a **hypervisor**: it divides one real computer's CPU, RAM, and disk into several **virtual machines (VMs)**. Each VM thinks it's a real computer.

Comparison: the ML30 is an **apartment building**. Proxmox is the **building manager**. Each VM is an **apartment** with its own walls, door, and kitchen.

Proxmox also runs **LXC containers** — lighter "rooms" that share the building's kitchen (the Linux kernel).

## 2. Why this homelab uses it

- Free, open source, web interface at `https://192.168.0.200:8006`
- Runs both VMs and LXC containers
- Snapshots, backups, firewall, and an API (used by OpenTofu and the Grafana exporter)

## 3. Key words

| Word | Meaning |
|---|---|
| **Node** | One physical Proxmox server (your ML30) |
| **VM (KVM/QEMU)** | Full virtual computer with its own kernel. Strong isolation. Used for K3s, Jenkins, hosting |
| **LXC container** | Shares the host kernel. Uses much less RAM. Used for gateway and PostgreSQL |
| **Unprivileged LXC** | Root inside the container is *not* root on the host. Safer — always prefer it |
| **vCPU** | A virtual CPU core. 15 vCPUs share 4 real cores |
| **Overcommit** | Giving out more vCPUs than real cores. OK for CPU (they take turns). **Not OK for RAM** |
| **Storage** | Where disks live. `local` = files (ISOs, templates, backups). `local-lvm` = VM disks |
| **LVM-thin** | Disk space is only used when data is actually written. A 60 GB disk with 5 GB of data uses ~5 GB |
| **Thin pool** | The shared space all thin disks come from. **If it fills, all guests can break** |
| **Template** | A "master copy" VM you clone to make new VMs quickly |
| **Cloud-init** | Sets hostname, IP, user, and SSH key on the first boot of a cloned VM |
| **Snapshot** | A save point before a risky change. **Not a backup** (same disk) |
| **qm / pct** | Command-line tools: `qm` for VMs, `pct` for LXC |
| **QEMU guest agent** | Small program inside a VM so Proxmox can see its IP and shut it down cleanly |
| **KSM** | Merges identical memory pages between VMs to save RAM |
| **Ballooning** | Lets Proxmox take RAM back from a VM. We turn it off for K3s nodes |

## 4. VM vs LXC — which to use

| | VM | LXC |
|---|---|---|
| RAM overhead | ~300–500 MB | ~20–50 MB |
| Own kernel | Yes | No (shares host) |
| Isolation | Strong | Good (unprivileged) |
| Runs Kubernetes well | ✅ | ❌ (painful) |
| Runs Docker well | ✅ | ⚠️ Possible but quirky |
| In this lab | K3s ×3, Jenkins, hosting | gateway, PostgreSQL |

## 5. Step by step for this project

### 5.1 Install Proxmox
1. Download the ISO from <https://www.proxmox.com/en/downloads>, write it to USB (balenaEtcher).
2. Boot the ML30 from USB (press **F11** for the boot menu).
3. Target disk → **Options**: filesystem `ext4`, `swapsize=4`, `maxroot=30`, `minfree=16`.
4. Network: hostname `pve.lab`, IP `192.168.0.200/24`, gateway `192.168.0.1`, DNS `1.1.1.1`.
5. Open `https://192.168.0.200:8006` and log in as `root`.

### 5.2 First setup (on the Proxmox shell)
```bash
# Use the free (no-subscription) update repository:
# Datacenter → pve → Updates → Repositories → disable "enterprise", add "No-Subscription"
apt update && apt full-upgrade -y
pvesm status                     # check local-lvm is about 400 GiB
lvs                              # see the thin pool "data"
```

### 5.3 Make an Ubuntu cloud-init template (VM 9000)
```bash
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

qm create 9000 --name ubuntu-2404-tpl --memory 2048 --cores 2 --cpu host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --agent enabled=1
qm set 9000 --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img,discard=on,ssd=1,iothread=1
qm set 9000 --ide2 local-lvm:cloudinit --boot order=scsi0 --serial0 socket --vga serial0
qm template 9000
```

### 5.4 Clone a VM from the template (example: k3s-server)
```bash
qm clone 9000 101 --name k3s-server --full
qm set 101 --cores 2 --memory 3072 --balloon 0 \
  --ipconfig0 ip=192.168.0.221/24,gw=192.168.0.1 \
  --nameserver 1.1.1.1 --ciuser ubuntu --sshkeys ~/.ssh/authorized_keys \
  --onboot 1 --startup order=2,up=60
qm resize 101 scsi0 30G
qm start 101
```
Later, **OpenTofu** ([16](../16-opentofu/README.md)) will do this for you from code.

### 5.5 Create an LXC (example: postgres)
```bash
pveam update
pveam available --section system | grep ubuntu-24.04
pveam download local ubuntu-24.04-standard_24.04-2_amd64.tar.zst   # use the exact name shown

pct create 105 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname postgres --cores 2 --memory 3072 --swap 512 \
  --rootfs local-lvm:50 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.225/24,gw=192.168.0.1 \
  --unprivileged 1 --features nesting=1 \
  --ssh-public-keys ~/.ssh/authorized_keys --onboot 1 --startup order=1
pct start 105
```

### 5.6 Firewall
Datacenter → Firewall → enable. Add rules per guest (e.g., postgres: allow TCP 5432 only from `.222`, `.223`). **Add an allow rule for your own SSH/8006 first** so you don't lock yourself out.

## 6. Useful commands

```bash
qm list                 # all VMs
pct list                # all containers
qm start|stop|shutdown|reboot 101
pct enter 105           # open a shell inside LXC 105
qm config 101           # show VM settings
qm snapshot 101 before-upgrade
qm rollback 101 before-upgrade
pvesm status            # storage usage
lvs -o lv_name,data_percent,metadata_percent pve   # thin pool fullness
free -h; uptime
```

## 7. Common mistakes

- **Thin pool full** → I/O errors in every guest. Watch `data_percent`; alert at 80%.
- **Giving out more RAM than you have.** CPU can be shared; RAM cannot.
- **Using snapshots as backups.** A snapshot is on the same SSD.
- **Forgetting `qemu-guest-agent`** inside VMs → Proxmox can't see IPs or shut down cleanly.
- **Privileged LXC "because it's easier"** → weaker security.
- **Enabling the firewall without an allow rule** for your management access.

## 8. Check yourself

1. Why is PostgreSQL in an LXC but K3s in VMs?
2. What happens if the thin pool reaches 100%?
3. What does cloud-init do?
4. Is a snapshot a backup?

<details><summary>Answers</summary>

1. LXC saves RAM and suits simple services; Kubernetes needs its own kernel features, so VMs are easier.
2. Guests can't write data → errors, crashes, possible corruption.
3. Sets hostname, IP, user, SSH key on first boot of a clone.
4. No. It's on the same disk. If the SSD dies, the snapshot dies too.

</details>

## 9. Learn more

- Proxmox docs: <https://pve.proxmox.com/pve-docs/>
- Cloud-init in Proxmox: <https://pve.proxmox.com/wiki/Cloud-Init_Support>
- LXC in Proxmox: <https://pve.proxmox.com/wiki/Linux_Container>
