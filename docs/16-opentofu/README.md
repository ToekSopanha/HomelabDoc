# 16 — OpenTofu (Infrastructure as Code)

> Describe your VMs and containers in text files. OpenTofu creates, changes, or deletes them in Proxmox to match.

## 1. What is it?

**OpenTofu** is the open-source fork of **Terraform** (same language, same providers). You write `.tf` files saying *what* you want ("a VM named k3s-server with 2 cores and 3 GB"). OpenTofu compares that with reality and makes only the needed changes.

Comparison: an **architect's blueprint**. Hand it to the builder and you get the same building every time. If the building burns down, rebuild from the blueprint.

## 2. Why this homelab uses it

- Rebuild any VM/LXC in minutes after a disaster (no HDD backups → rebuild is the plan!)
- Your whole lab layout is documented and versioned in Git
- No clicking mistakes in the Proxmox UI
- Core DevOps skill (Terraform/OpenTofu is used everywhere)

## 3. Key words

| Word | Meaning |
|---|---|
| **Provider** | Plugin that talks to a platform. We use **`bpg/proxmox`** |
| **Resource** | One thing to manage: a VM, an LXC, a file |
| **Variable** | An input value (`var.pve_api_token`) |
| **`for_each`** | Create many similar resources from a map |
| **State (`terraform.tfstate`)** | OpenTofu's memory of what it created. **Contains secrets — never commit** |
| **`tofu init`** | Download providers |
| **`tofu plan`** | Show what *would* change. Always read it |
| **`tofu apply`** | Make the changes |
| **`tofu destroy`** | Delete everything it manages (dangerous) |
| **Drift** | Someone changed things by hand; reality ≠ code |

## 4. How it works

```text
main.tf (desired) ──┐
                    ├─► tofu plan ─► "+ create VM 102, ~ change RAM of 103" ─► tofu apply ─► Proxmox API
state (known)    ───┘
```

## 5. Step by step for this project

1. **Install** on your Mac:
   ```bash
   brew install opentofu
   tofu version
   ```
2. **Create an API token in Proxmox** (on the host):
   ```bash
   pveum user add tofu@pve
   pveum acl modify / --users tofu@pve --roles PVEAdmin
   pveum user token add tofu@pve homelab --privsep 0
   # copy the token value shown — it is displayed only once
   ```
3. **Folder** `homelab-infra/tofu/` with these files:

   `providers.tf`
   ```hcl
   terraform {
     required_providers {
       proxmox = {
         source  = "bpg/proxmox"
         version = "~> 0.66"   # pin to the current release when you start
       }
     }
   }

   provider "proxmox" {
     endpoint  = "https://192.168.0.200:8006/"
     api_token = var.pve_api_token
     insecure  = true          # self-signed certificate
     ssh {
       agent    = true
       username = "root"
     }
   }

   variable "pve_api_token" {
     type      = string
     sensitive = true
   }
   ```

   `vms.tf`
   ```hcl
   locals {
     vms = {
       k3s-server  = { id = 101, cores = 2, ram = 3072, disk = 30, ip = "192.168.0.221" }
       k3s-worker1 = { id = 102, cores = 3, ram = 6144, disk = 60, ip = "192.168.0.222" }
       k3s-worker2 = { id = 103, cores = 3, ram = 6144, disk = 60, ip = "192.168.0.223" }
       jenkins     = { id = 106, cores = 2, ram = 2560, disk = 30, ip = "192.168.0.226" }
       hosting     = { id = 107, cores = 2, ram = 4096, disk = 40, ip = "192.168.0.227" }
     }
   }

   resource "proxmox_virtual_environment_vm" "vm" {
     for_each  = local.vms
     name      = each.key
     vm_id     = each.value.id
     node_name = "pve"
     on_boot   = true

     clone {
       vm_id = 9000          # the Ubuntu cloud-init template
       full  = true
     }

     cpu {
       cores = each.value.cores
       type  = "host"
     }

     memory {
       dedicated = each.value.ram
     }

     agent {
       enabled = true
     }

     disk {
       datastore_id = "local-lvm"
       interface    = "scsi0"
       size         = each.value.disk
       discard      = "on"
       ssd          = true
       iothread     = true
     }

     network_device {
       bridge = "vmbr0"
     }

     initialization {
       ip_config {
         ipv4 {
           address = "${each.value.ip}/24"
           gateway = "192.168.0.1"
         }
       }
       dns {
         servers = ["1.1.1.1"]
       }
       user_account {
         username = "ubuntu"
         keys     = [trimspace(file("~/.ssh/id_ed25519.pub"))]
       }
     }
   }
   ```
   LXCs use the resource `proxmox_virtual_environment_container` (see provider docs).
4. **Give the token without saving it in a file:**
   ```bash
   export TF_VAR_pve_api_token='tofu@pve!homelab=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
   ```
5. **Run:**
   ```bash
   cd homelab-infra/tofu
   tofu init
   tofu plan          # READ THIS CAREFULLY — look for "destroy"
   tofu apply
   ```
6. **Existing VMs** made by hand? Bring them under control with `tofu import` instead of recreating.
7. Keep `terraform.tfstate` out of Git (`.gitignore`) but **back it up** (restic, [25](../25-backups/README.md)) — or encrypt it with OpenTofu's state encryption.

## 6. Useful commands

```bash
tofu fmt            # format files
tofu validate       # syntax check
tofu plan
tofu apply
tofu state list     # what OpenTofu manages
tofu output
tofu apply -replace='proxmox_virtual_environment_vm.vm["k3s-worker2"]'   # rebuild one VM
```

## 7. Common mistakes

- `tofu apply` **without reading the plan** → a typo can destroy a VM.
- Changing a VM in the Proxmox UI and then running `apply` → OpenTofu changes it back (drift).
- Committing `terraform.tfstate` or the API token.
- Losing the state file → OpenTofu "forgets" your VMs (fix with `tofu import`).
- Using the same VM IDs as existing VMs → check `qm list` first.

## 8. Check yourself

1. What does `tofu plan` do?
2. Why must the state file be kept safe but out of Git?
3. How would you rebuild only worker2?

<details><summary>Answers</summary>

1. Shows the changes it would make, without doing them.
2. It may contain secrets, and it's OpenTofu's memory of your resources.
3. `tofu apply -replace='proxmox_virtual_environment_vm.vm["k3s-worker2"]'`

</details>

## 9. Learn more

- OpenTofu docs: <https://opentofu.org/docs/>
- bpg/proxmox provider: <https://registry.terraform.io/providers/bpg/proxmox/latest/docs>
- Terraform tutorials (same language): <https://developer.hashicorp.com/terraform/tutorials>
