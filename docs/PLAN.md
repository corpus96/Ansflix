# AnsibleFlix — Project Plan & Implementation Guide

## 1. Brief Description

**AnsibleFlix** is a medium-complexity, 100% free infrastructure automation project designed for a **macOS control node** targeting a **local Ubuntu virtual machine** via [Canonical Multipass](https://canonical.com/multipass). It provisions a self-hosted personal media streaming server ([Jellyfin](https://jellyfin.org/docs/general/quick-start/)) paired with an automated service health dashboard ([Uptime Kuma](https://github.com/louislam/uptime-kuma)), without external notification webhooks or paid cloud/container subscriptions.

This project integrates the core syllabus of Abhishek Veeramalla's [Ansible Zero to Hero](https://github.com/iam-veeramalla/ansible-zero-to-hero) course with the hands-on concepts from your `Ansible Quiz.md` notes:

* **Agentless SSH & Ad-Hoc Connectivity**: Validating host reachability and Python readiness from your Mac to the local Linux VM via [`ansible.builtin.ping`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/ping_module.html) ([Day 2 Inventory & Ad-Hoc Commands](https://www.youtube.com/watch?v=ogLi-Tn-80A)).
* **Privilege Escalation**: Managing Linux system packages, storage directories, and services with `become: true` ([Day 3 Playbooks](https://www.youtube.com/watch?v=lQqFN1Bt0UQ)).
* **Roles & Collections**: Segmenting tasks, handlers, templates, and variables into modular [Ansible Roles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html) ([Day 4 Roles](https://www.youtube.com/watch?v=lxPvbD6_lTs), [Day 5 Ansible Galaxy](https://www.youtube.com/watch?v=p9ShQ3FbU9U)) and deploying free, open-source containers inside the Linux VM via the [`community.docker` collection](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_container_module.html).
* **OS Facts & Conditionals**: Adapting package installations and hardware rules dynamically using [`when` conditionals and `ansible_facts`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_conditionals.html) (`ansible_os_family == "Debian"` vs. `"RedHat"`).
* **Dynamic Output Capture**: Using [`register`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html) to inspect command `stdout` and return codes (`rc`) for storage pre-checks and HTTP health verification.
* **Secret Management**: Protecting admin credentials and metadata API secrets with [`ansible-vault`](https://docs.ansible.com/projects/ansible/latest/cli/ansible-vault.html) (`create`, `view`, `edit`, `encrypt`, and `encrypt_string`) ([Secure Ansible Playbooks using Vault](https://www.youtube.com/watch?v=qoezEf6Mwzw)).

## 2. Analysis and Approach

### Confirmed Project Decisions

* **Target Environment (macOS + Local Linux VM)**: Your Mac serves as the Ansible control node, and [Canonical Multipass](https://canonical.com/multipass/install) runs a lightweight, free Ubuntu LTS virtual machine natively on macOS (supporting both Apple Silicon M-series and Intel Macs).
* **Deployment Style (100% Free Open-Source Containers Inside the Linux VM)**: Both [Jellyfin](https://jellyfin.org/) and [Uptime Kuma](https://github.com/louislam/uptime-kuma) are 100% free and open-source with zero licensing fees. By having Ansible install the open-source Linux `docker.io` package directly inside the Ubuntu VM (rather than installing paid Docker Desktop on macOS), the entire stack stays completely free while giving you hands-on practice with Ansible Collections.
* **Focused Scope (No External Alert Webhooks)**: External chat alerts are excluded so the `uptime_watchdog` role focuses cleanly on deploying the local [Uptime Kuma](https://github.com/louislam/uptime-kuma) status dashboard and verifying service health via Ansible tasks.

### Architectural Layers

Rather than writing a single monolithic playbook, **AnsibleFlix** splits responsibilities across three logical layers:

| Layer | Role / Component | Core Responsibility | Quiz & Course Mechanics Applied |
| :--- | :--- | :--- | :--- |
| **Layer 1: Host Preparation** | `common_host` Role | Installs base dependencies, open-source Docker engine (`docker.io`), and creates `/srv/ansibleflix/media` (`movies`, `shows`, `music`) only if sufficient disk space exists. | `become: true`, `when: ansible_os_family`, [`register`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html) on `df` output, [`ansible.builtin.assert`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/assert_module.html) |
| **Layer 2: Streaming Engine** | `jellyfin_media` Role | Deploys the [Jellyfin](https://jellyfin.org/docs/) container, mounts media directories, renders an environment/config template, and adjusts memory limits based on VM hardware facts. | [Ansible Roles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html), [`ansible.builtin.template`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/template_module.html), handlers for service restarts |
| **Layer 3: Status Watchdog** | `uptime_watchdog` Role | Deploys [Uptime Kuma](https://github.com/louislam/uptime-kuma), injects Vault-encrypted admin credentials, and runs an automated HTTP health probe against both web interfaces. | [Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html), [`ansible.builtin.uri`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/uri_module.html) with `register` |

## 3. Functional Requirements

1. **Automated Local VM Discovery & Pre-Flight Validation**
   * Verify SSH connectivity and Python availability from macOS to the Multipass Ubuntu VM before applying changes.
   * Check available free disk space on the VM and halt execution gracefully if free storage falls below a configurable threshold (default: `5 GB` for local VM testing).
2. **Media Library & Streaming Server Deployment**
   * Automatically create structured media directories (`/srv/ansibleflix/media/movies`, `/srv/ansibleflix/media/tv`, `/srv/ansibleflix/media/music`) with appropriate user/group ownership.
   * Deploy [Jellyfin](https://jellyfin.org/) listening on port `8096` and generate a customized environment configuration file using Jinja2 templating.
3. **Health Dashboard & Secret Management**
   * Deploy [Uptime Kuma](https://github.com/louislam/uptime-kuma) listening on port `3001` so you can visually monitor Jellyfin's uptime from your Mac browser.
   * Store all sensitive values (initial admin passwords and secret tokens) in [Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html) so no plaintext secrets ever appear in Git commits.
4. **Automated Post-Deploy Health Check**
   * Probe both web UIs (`http://<vm-ip>:8096/health` and `http://<vm-ip>:3001`) at the end of the playbook run and output the live browser URLs to the terminal.

## 4. Technical Requirements

| Category | Specification | Reference Documentation |
| :--- | :--- | :--- |
| **Control Node (macOS)** | macOS with Python 3.10+, `ansible-core`, `ansible-lint`, and [Canonical Multipass](https://canonical.com/multipass/install) installed | [Day 1 Introduction to Ansible](https://www.youtube.com/watch?v=O4i5Wh5JvyM) |
| **Managed Node (Local VM)** | 1 Ubuntu LTS VM created via Multipass (`multipass launch --name ansibleflix -m 2G -d 10G`) with SSH key access and `sudo` rights | [Getting Started with Multipass](https://canonical.com/multipass/docs/latest/tutorial/getting-started/) & [Day 2 Inventory](https://www.youtube.com/watch?v=ogLi-Tn-80A) |
| **Required Collections** | `ansible.builtin` (core) and `community.docker` (`>= 3.0.0`) defined in `requirements.yml` | [`community.docker.docker_container`](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_container_module.html) |
| **Network Ports** | TCP `22` (SSH), TCP `8096` ([Jellyfin Networking](https://jellyfin.org/docs/general/post-install/networking/)), TCP `3001` ([Uptime Kuma](https://github.com/louislam/uptime-kuma)) reachable from your Mac browser via the VM's local IP | Local bridge network between macOS and Multipass |
| **Vault Configuration** | Encrypted `group_vars/all/vault.yml` unlocked via `--ask-vault-pass` or a git-ignored `.vault_pass` file | [Managing Vault Passwords](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_managing_passwords.html) |

## 5. Implementation Plan

### Directory Structure

```text
ansibleflix/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       └── all/
│           ├── vars.yml          # Non-sensitive variables (ports, paths, disk threshold)
│           └── vault.yml         # Encrypted with `ansible-vault create`
├── site.yml                      # Main orchestration playbook
└── roles/
    ├── common_host/
    │   ├── defaults/main.yml
    │   └── tasks/main.yml        # OS conditionals (`when`), `become: true`, disk space `register`
    ├── jellyfin_media/
    │   ├── defaults/main.yml
    │   ├── handlers/main.yml     # Restart Jellyfin container when template changes
    │   ├── tasks/main.yml        # Directory creation, template rendering, container deployment
    │   └── templates/
    │       └── ansibleflix.env.j2
    └── uptime_watchdog/
        ├── defaults/main.yml
        └── tasks/main.yml        # Uptime Kuma deployment & `uri` HTTP health verification
```

### Step-by-Step Build Milestones

#### Milestone 1: Local macOS VM Setup, Inventory & Ad-Hoc Verification
1. On your Mac, launch a free Ubuntu VM using [Multipass](https://canonical.com/multipass/docs/latest/tutorial/getting-started/) and copy your Mac's SSH public key (`~/.ssh/id_ed25519.pub`) into the VM's `/home/ubuntu/.ssh/authorized_keys`:
   ```bash
   multipass launch --name ansibleflix --memory 2G --disk 10G --cpus 2
   multipass list
   ```
2. Add the VM's IPv4 address from `multipass list` to `inventory/hosts.ini` under `[media_servers]`.
3. Run an ad-hoc connectivity check using the [`ansible.builtin.ping` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/ping_module.html):
   ```bash
   ansible media_servers -i inventory/hosts.ini -m ansible.builtin.ping
   ```
4. Install the required Docker collection defined in `requirements.yml`:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

#### Milestone 2: Encrypting Secrets with Ansible Vault
1. Create your encrypted variables file using [`ansible-vault create`](https://docs.ansible.com/projects/ansible/latest/cli/ansible-vault.html):
   ```bash
   ansible-vault create inventory/group_vars/all/vault.yml
   ```
2. Add `vault_jellyfin_admin_password`, `vault_kuma_admin_password`, and `vault_media_api_key` inside the file.
3. Practice viewing the encrypted file without decrypting it on disk (`ansible-vault view inventory/group_vars/all/vault.yml`) and editing it in place (`ansible-vault edit inventory/group_vars/all/vault.yml`).
4. Map clean variable names in `inventory/group_vars/all/vars.yml` and test encrypting a single inline string using [`ansible-vault encrypt_string`](https://docs.ansible.com/projects/ansible/latest/cli/ansible-vault.html).

#### Milestone 3: Building the `common_host` Role (Facts, Conditionals & `register`)
1. Use `become: true` at the role or task block level to elevate privileges inside the VM.
2. Write OS-specific package installation tasks using [`when: ansible_os_family == "Debian"`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_conditionals.html) (`apt` to install `docker.io`, `python3-docker`, and `curl` on Ubuntu) and `when: ansible_os_family == "RedHat"` (`dnf` fallback) so your role stays portable across Linux families.
3. Run a storage diagnostic check (`df -BG --output=avail / | tail -n 1`) and capture the output using [`register: disk_check`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html).
4. Validate `disk_check.stdout` using [`ansible.builtin.assert`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/assert_module.html) so the playbook stops with a clear message if free space on the VM is below `min_free_gb` (`5 GB`).

#### Milestone 4: Building `jellyfin_media` & `uptime_watchdog` Roles
1. In `jellyfin_media`, create `/srv/ansibleflix/media/{movies,tv,music}` and render `templates/ansibleflix.env.j2` via [`ansible.builtin.template`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/template_module.html), injecting both your decrypted Vault secrets and host facts (`{{ ansible_hostname }}` and `{{ ansible_memtotal_mb }}`).
2. Attach a `notify: Restart Jellyfin` handler so configuration edits automatically restart the container.
3. Launch both [Jellyfin](https://jellyfin.org/docs/general/quick-start/) (`jellyfin/jellyfin:latest`) and [Uptime Kuma](https://github.com/louislam/uptime-kuma) (`louislam/uptime-kuma:1`) using [`community.docker.docker_container`](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_container_module.html).
4. In `uptime_watchdog`, use [`ansible.builtin.uri`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/uri_module.html) with `register: jellyfin_health` (`status_code: 200`) and print the live Safari/Chrome URLs (`http://{{ ansible_default_ipv4.address }}:8096` and `:3001`) using `ansible.builtin.debug` when `jellyfin_health.status == 200`.

## 6. Manual Testing Strategy

| Test Case | Command / Action | Expected Result |
| :--- | :--- | :--- |
| **1. SSH & Python Handshake** | `ansible all -i inventory/hosts.ini -m ansible.builtin.ping` | Returns `"ping": "pong"` with `SUCCESS` from the local Multipass Ubuntu VM. |
| **2. Dry-Run Simulation** | `ansible-playbook -i inventory/hosts.ini site.yml --check --diff --ask-vault-pass` | Previews file, package, and template changes via [Check Mode and Diff Mode](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_checkmode.html) without modifying the VM. |
| **3. Mac Browser Reachability** | Open `http://<vm-ip>:8096` and `http://<vm-ip>:3001` in Safari or Chrome on macOS | Jellyfin setup wizard and Uptime Kuma dashboard load cleanly. |
| **4. Handler Trigger Test** | Change a variable in `vars.yml` and re-run `ansible-playbook` | The template task reports `changed` and triggers the `Restart Jellyfin` handler. |
| **5. Vault Protection Check** | Run `head -n 1 inventory/group_vars/all/vault.yml` | Header starts with `$ANSIBLE_VAULT;1.1;AES256` and no plaintext secrets exist in the repository. |

## 7. Automatic Testing Strategy

1. **Static Code Analysis & Syntax Validation ([Ansible Lint](https://docs.ansible.com/projects/lint/))**
   * Run `ansible-playbook site.yml --syntax-check` on your Mac to catch YAML indentation or module parameter errors before connecting to the VM.
   * Run [`ansible-lint`](https://docs.ansible.com/projects/lint/usage/) from the repository root to verify FQCN module naming (e.g., `ansible.builtin.template` instead of `template`), proper `become: true` usage, and clean role structure.
2. **In-Playbook Self-Tests (`post_tasks` Verification)**
   * Add a `post_tasks` section in `site.yml` that uses [`ansible.builtin.uri`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/uri_module.html) (with `retries: 5` and `delay: 3`) to query `http://localhost:8096/health` and `http://localhost:3001`.
   * Use [`ansible.builtin.assert`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/assert_module.html) to verify that Jellyfin returns `"Healthy"` in `jellyfin_health.content` and that `/srv/ansibleflix/media/movies` exists on the VM.
3. **Idempotency Verification (Optional [Ansible Molecule](https://ansible.readthedocs.io/projects/molecule/) or Repeat Run)**
   * Execute `ansible-playbook` twice in a row against the Multipass VM and verify that the second run outputs `changed=0` and `failed=0`, proving all tasks and roles are strictly idempotent.

## 8. Definition of Done (DoD)

- [x] **Local macOS-to-VM Connectivity**: Mac control node connects via passwordless SSH to the local [Multipass](https://canonical.com/multipass) Ubuntu VM and passes `ansible.builtin.ping`.
- [x] **Zero Hardcoded Secrets**: All admin passwords and API secrets are encrypted in `vault.yml` via [`ansible-vault`](https://docs.ansible.com/projects/ansible/latest/cli/ansible-vault.html), and `.vault_pass` is listed in `.gitignore`.
- [x] **Modular Role Architecture**: All tasks are organized inside the `common_host`, `jellyfin_media`, and `uptime_watchdog` [Ansible Roles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html) with no loose tasks cluttering `site.yml`.
- [x] **Cross-Distro Compatibility**: Package installation tasks use [`when: ansible_os_family`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_conditionals.html) so the playbook runs on Debian/Ubuntu and RedHat-family systems without manual edits.
- [x] **Dynamic State Evaluation**: Playbook captures command and HTTP outputs using [`register`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html) to enforce minimum disk space and verify service readiness.
- [x] **Clean Linting & Idempotency**: `ansible-lint` passes with zero errors, and a second consecutive run of `ansible-playbook site.yml` finishes with `changed=0` and `failed=0`.
- [x] **Live Streaming & Monitoring**: Both [Jellyfin](https://jellyfin.org/) (`:8096`) and [Uptime Kuma](https://github.com/louislam/uptime-kuma) (`:3001`) are accessible from your Mac browser and passing automated HTTP health assertions.