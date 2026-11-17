# 17 — Ansible (Configuration as Code)

> Write down how a server should be set up; Ansible logs in by SSH and makes it so — on one server or twenty.

## 1. What is it?

**Ansible** is an automation tool. You write **playbooks** (YAML) listing tasks: "install these packages", "copy this config", "restart this service". Ansible connects over **SSH** and runs them. Nothing needs to be installed on the servers.

OpenTofu **builds** the empty machines. Ansible **furnishes** them.

Comparison: an **interior designer's checklist** — "put a bed here, paint this wall". Give the same list to 10 empty apartments → 10 identical apartments.

## 2. Why this homelab uses it

- Same baseline on every guest (users, SSH hardening, updates, monitoring agents)
- Install Nginx, cloudflared, PostgreSQL, Jenkins, Docker, K3s **the same way every time**
- After a rebuild, one command restores the configuration
- **Idempotent:** running it twice is safe — it only changes what's different

## 3. Key words

| Word | Meaning |
|---|---|
| **Control node** | Where Ansible runs (your Mac) |
| **Managed node** | A server Ansible configures |
| **Inventory** | List of servers and groups |
| **Playbook** | YAML file of plays and tasks |
| **Task** | One action, using a **module** (`apt`, `copy`, `template`, `service`…) |
| **Role** | Reusable bundle of tasks/files/templates (e.g., `nginx` role) |
| **Handler** | Task that runs only when notified (e.g., "restart nginx if config changed") |
| **Variables / group_vars** | Values per group or host |
| **Template (Jinja2)** | Config file with placeholders `{{ variable }}` |
| **Collection** | Package of modules/roles from Ansible Galaxy |
| **Idempotent** | Running again changes nothing if already correct |
| **`--check --diff`** | Dry run: show what would change |

## 4. How it works

```text
Your Mac
  inventory.ini + site.yml
        │ SSH (your key)
        ├──► gateway   : nginx, cloudflared
        ├──► k3s nodes : k3s, node-exporter
        ├──► postgres  : postgresql
        ├──► jenkins   : java, jenkins
        └──► hosting   : docker
```

## 5. Step by step for this project

1. **Install** on your Mac:
   ```bash
   brew install ansible
   ansible --version
   ```
2. **Inventory** — `homelab-infra/ansible/inventory.ini`:
   ```ini
   [gateway]
   gateway ansible_host=192.168.0.220 ansible_user=root

   [k3s_server]
   k3s-server ansible_host=192.168.0.221

   [k3s_workers]
   k3s-worker1 ansible_host=192.168.0.222
   k3s-worker2 ansible_host=192.168.0.223

   [database]
   postgres ansible_host=192.168.0.225 ansible_user=root

   [jenkins]
   jenkins ansible_host=192.168.0.226

   [hosting]
   hosting ansible_host=192.168.0.227

   [k3s:children]
   k3s_server
   k3s_workers

   [all:vars]
   ansible_user=ubuntu
   ansible_python_interpreter=/usr/bin/python3
   ```
3. **Test connection:**
   ```bash
   cd homelab-infra/ansible
   ansible all -i inventory.ini -m ping
   ```
4. **A baseline playbook** — `baseline.yml`:
   ```yaml
   - name: Baseline for all guests
     hosts: all
     become: true
     tasks:
       - name: Update apt cache and upgrade
         ansible.builtin.apt:
           update_cache: true
           upgrade: safe

       - name: Install common packages
         ansible.builtin.apt:
           name: [htop, curl, vim, unattended-upgrades, qemu-guest-agent]
           state: present

       - name: Disable SSH password login
         ansible.builtin.lineinfile:
           path: /etc/ssh/sshd_config
           regexp: '^#?PasswordAuthentication'
           line: 'PasswordAuthentication no'
         notify: Restart ssh

       - name: Enable fstrim timer
         ansible.builtin.systemd:
           name: fstrim.timer
           enabled: true
           state: started

     handlers:
       - name: Restart ssh
         ansible.builtin.service:
           name: ssh
           state: restarted
   ```
   (`qemu-guest-agent` matters only in VMs; it's harmless in LXC — or move it to a VM-only group.)
5. **Dry run, then apply:**
   ```bash
   ansible-playbook -i inventory.ini baseline.yml --check --diff
   ansible-playbook -i inventory.ini baseline.yml
   ```
6. **Grow it into roles** over time:
   ```text
   ansible/
     inventory.ini
     site.yml              # runs everything
     roles/
       baseline/
       nginx_gateway/
       postgres/
       jenkins/
       docker_host/
       k3s/                # or use the community k3s-ansible project
   ```
7. Useful community collections: `prometheus.prometheus` (node_exporter), `grafana.grafana` (Alloy), `community.postgresql`, `community.docker`.

## 6. Useful commands

```bash
ansible all -i inventory.ini -m ping
ansible k3s -i inventory.ini -a "uptime"                 # run a command on a group
ansible-playbook -i inventory.ini site.yml --limit jenkins   # only one host
ansible-playbook -i inventory.ini site.yml --check --diff    # dry run
ansible-galaxy collection install prometheus.prometheus
ansible-inventory -i inventory.ini --graph
```

## 7. Common mistakes

- **YAML indentation** errors — use spaces, never tabs.
- Using `shell:` / `command:` for everything → not idempotent. Prefer real modules.
- Putting passwords in plain YAML → use **Ansible Vault** or SOPS.
- Not testing with `--check --diff` first.
- Forgetting `become: true` when a task needs root.

## 8. Check yourself

1. What does "idempotent" mean?
2. OpenTofu vs Ansible — which creates VMs, which installs software?
3. How do you run a playbook on only one host?

<details><summary>Answers</summary>

1. Running it many times gives the same result; it only changes what's wrong.
2. OpenTofu creates VMs; Ansible installs and configures software.
3. `--limit <host>`

</details>

## 9. Learn more

- Getting started: <https://docs.ansible.com/ansible/latest/getting_started/>
- Module index: <https://docs.ansible.com/ansible/latest/collections/index_module.html>
- k3s-ansible: <https://github.com/k3s-io/k3s-ansible>
