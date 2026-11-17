# AnsibleFlix — Build It Yourself (Step by Step)

Result: Jellyfin (`:8096`) and Uptime Kuma (`:3001`) running in Docker on a Multipass Ubuntu VM, provisioned from your Mac with Ansible roles, Vault secrets, handlers, and health checks.

Work in a fresh folder, e.g. `~/ansibleflix-diy`.

## 0. Prerequisites
```bash
pip3 install ansible-core ansible-lint
multipass version
```
If Multipass is stuck on "Waiting for daemon":
```bash
sudo launchctl load -w /Library/LaunchDaemons/com.canonical.multipassd.plist
```

## 1. VM and SSH
```bash
# Normal launch:
multipass launch --name ansibleflix --memory 2G --disk 10G --cpus 2

# If it fails to download the image (broken IPv6 on your network), download over IPv4 first (Apple Silicon image):
curl -4 -L -o /tmp/ubuntu-noble-arm64.img https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-arm64.img
multipass launch file:///tmp/ubuntu-noble-arm64.img --name ansibleflix --memory 2G --disk 10G --cpus 2

# Copy your SSH key into the VM (create one first with ssh-keygen -t ed25519 if you have none)
cat ~/.ssh/id_ed25519.pub | multipass exec ansibleflix -- tee -a /home/ubuntu/.ssh/authorized_keys

# Note the IP
multipass info ansibleflix | grep IPv4
```
Use a larger `--disk` (e.g. `20G`) if you plan to add media; the Docker images alone leave only ~5 GB free on 10 GB.

## 2. Project skeleton
```bash
mkdir -p ansibleflix-diy && cd ansibleflix-diy
mkdir -p inventory/group_vars/all
mkdir -p roles/common_host/{defaults,tasks}
mkdir -p roles/jellyfin_media/{defaults,handlers,tasks,templates}
mkdir -p roles/uptime_watchdog/{defaults,tasks}
```

## 3. Config and inventory

**`ansible.cfg`**
```ini
[defaults]
inventory = inventory/hosts.ini
roles_path = roles
host_key_checking = False
interpreter_python = auto_silent
vault_password_file = .vault_pass
retry_files_enabled = False
```

**`requirements.yml`**
```yaml
---
collections:
  - name: community.docker
    version: ">=3.0.0"
```
```bash
ansible-galaxy collection install -r requirements.yml
```

**`inventory/hosts.ini`** (use your VM's IP)
```ini
[media_servers]
ansibleflix ansible_host=<VM_IP>

[media_servers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**`inventory/group_vars/all/vars.yml`**
```yaml
---
media_root: /srv/ansibleflix/media
media_subdirs:
  - movies
  - tv
  - music
min_free_gb: 5
jellyfin_port: 8096
kuma_port: 3001
jellyfin_admin_password: "{{ vault_jellyfin_admin_password }}"
kuma_admin_password: "{{ vault_kuma_admin_password }}"
media_api_key: "{{ vault_media_api_key }}"
```

**`.gitignore`**
```
.vault_pass
```

**`.ansible-lint`** (shared variables via group_vars make the role-prefix rule noisy)
```yaml
---
skip_list:
  - var-naming[no-role-prefix]
```

Test connectivity:
```bash
ansible media_servers -m ansible.builtin.ping
```
Expect `"ping": "pong"`.

## 4. Vault
```bash
openssl rand -base64 24 > .vault_pass
ansible-vault create inventory/group_vars/all/vault.yml
```
In the editor, paste:
```yaml
---
vault_jellyfin_admin_password: "pick-something"
vault_kuma_admin_password: "pick-something"
vault_media_api_key: "pick-something"
```
Practice:
```bash
ansible-vault view inventory/group_vars/all/vault.yml
ansible-vault edit inventory/group_vars/all/vault.yml
ansible-vault encrypt_string 'secret' --name my_var
head -n 1 inventory/group_vars/all/vault.yml   # $ANSIBLE_VAULT;1.1;AES256
```

## 5. Role `common_host`

**`roles/common_host/defaults/main.yml`**
```yaml
---
min_free_gb: 5
media_root: /srv/ansibleflix/media
media_subdirs:
  - movies
  - tv
  - music
```

**`roles/common_host/tasks/main.yml`**
```yaml
---
- name: Install base packages (Debian family)
  ansible.builtin.apt:
    name:
      - docker.io
      - python3-docker
      - curl
    state: present
    update_cache: true
    cache_valid_time: 3600
  when: ansible_facts['os_family'] == "Debian"

- name: Install base packages (RedHat family)
  ansible.builtin.dnf:
    name:
      - docker
      - python3-docker
      - curl
    state: present
  when: ansible_facts['os_family'] == "RedHat"

- name: Ensure Docker is enabled and running
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true

- name: Check free disk space on /
  ansible.builtin.shell: set -o pipefail && df -BG --output=avail / | tail -n 1
  args:
    executable: /bin/bash
  register: disk_check
  changed_when: false
  check_mode: false

- name: Assert enough free disk space
  ansible.builtin.assert:
    that:
      - (disk_check.stdout | regex_replace('[^0-9]', '') | int) >= (min_free_gb | int)
    fail_msg: "Only {{ disk_check.stdout | trim }} free; need {{ min_free_gb }}G."
    success_msg: "Disk space OK: {{ disk_check.stdout | trim }} free."

- name: Create media directories
  ansible.builtin.file:
    path: "{{ media_root }}/{{ item }}"
    state: directory
    owner: "{{ ansible_user | default('root') }}"
    group: "{{ ansible_user | default('root') }}"
    mode: "0755"
  loop: "{{ media_subdirs }}"
```

## 6. Role `jellyfin_media`

**`roles/jellyfin_media/defaults/main.yml`**
```yaml
---
jellyfin_port: 8096
jellyfin_image: jellyfin/jellyfin:latest
jellyfin_config_dir: /srv/ansibleflix/jellyfin
jellyfin_env_file: /srv/ansibleflix/ansibleflix.env
media_root: /srv/ansibleflix/media
# Half of the VM's RAM as the container limit
jellyfin_memory_mb: "{{ ((ansible_facts['memtotal_mb'] | default(2048) | int) * 0.5) | int }}"
```

**`roles/jellyfin_media/templates/ansibleflix.env.j2`**
```jinja
# Managed by Ansible - do not edit
ANSIBLEFLIX_HOST={{ ansible_facts['hostname'] }}
ANSIBLEFLIX_MEM_TOTAL_MB={{ ansible_facts['memtotal_mb'] }}
JELLYFIN_PublishedServerUrl=http://{{ ansible_facts['default_ipv4']['address'] }}:{{ jellyfin_port }}
JELLYFIN_ADMIN_PASSWORD={{ jellyfin_admin_password }}
MEDIA_API_KEY={{ media_api_key }}
```

**`roles/jellyfin_media/handlers/main.yml`**
```yaml
---
- name: Restart Jellyfin
  community.docker.docker_container:
    name: jellyfin
    state: started
    restart: true
```

**`roles/jellyfin_media/tasks/main.yml`**
```yaml
---
- name: Create Jellyfin directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: "0755"
  loop:
    - "{{ jellyfin_config_dir }}/config"
    - "{{ jellyfin_config_dir }}/cache"

- name: Render environment file
  ansible.builtin.template:
    src: ansibleflix.env.j2
    dest: "{{ jellyfin_env_file }}"
    owner: root
    group: root
    mode: "0600"
  notify: Restart Jellyfin

- name: Run Jellyfin container
  community.docker.docker_container:
    name: jellyfin
    image: "{{ jellyfin_image }}"
    state: started
    restart_policy: unless-stopped
    memory: "{{ jellyfin_memory_mb }}m"
    env_file: "{{ jellyfin_env_file }}"
    ports:
      - "{{ jellyfin_port }}:8096"
    volumes:
      - "{{ jellyfin_config_dir }}/config:/config"
      - "{{ jellyfin_config_dir }}/cache:/cache"
      - "{{ media_root }}:/media"
```

## 7. Role `uptime_watchdog`

**`roles/uptime_watchdog/defaults/main.yml`**
```yaml
---
kuma_port: 3001
jellyfin_port: 8096
kuma_image: louislam/uptime-kuma:1
kuma_data_dir: /srv/ansibleflix/kuma
```

**`roles/uptime_watchdog/tasks/main.yml`**
```yaml
---
- name: Create Uptime Kuma data directory
  ansible.builtin.file:
    path: "{{ kuma_data_dir }}"
    state: directory
    mode: "0755"

- name: Store Kuma admin credentials for first-run setup
  ansible.builtin.copy:
    content: "admin:{{ kuma_admin_password }}\n"
    dest: "{{ kuma_data_dir }}/admin-credentials.txt"
    owner: root
    group: root
    mode: "0600"
  no_log: true

- name: Run Uptime Kuma container
  community.docker.docker_container:
    name: uptime-kuma
    image: "{{ kuma_image }}"
    state: started
    restart_policy: unless-stopped
    ports:
      - "{{ kuma_port }}:3001"
    volumes:
      - "{{ kuma_data_dir }}/data:/app/data"

- name: Probe Jellyfin health
  ansible.builtin.uri:
    url: "http://{{ ansible_facts['default_ipv4']['address'] }}:{{ jellyfin_port }}/health"
    status_code: 200
  register: jellyfin_health
  retries: 10
  delay: 3
  until: jellyfin_health.status == 200

- name: Show live URLs
  ansible.builtin.debug:
    msg:
      - "Jellyfin:    http://{{ ansible_facts['default_ipv4']['address'] }}:{{ jellyfin_port }}"
      - "Uptime Kuma: http://{{ ansible_facts['default_ipv4']['address'] }}:{{ kuma_port }}"
  when: jellyfin_health.status == 200
```

## 8. `site.yml`
```yaml
---
- name: Deploy AnsibleFlix (Jellyfin + Uptime Kuma)
  hosts: media_servers
  become: true
  roles:
    - common_host
    - jellyfin_media
    - uptime_watchdog

  post_tasks:
    - name: Query Jellyfin health endpoint
      ansible.builtin.uri:
        url: "http://localhost:{{ jellyfin_port }}/health"
        return_content: true
      register: jellyfin_health
      retries: 5
      delay: 3
      until: jellyfin_health.status == 200

    - name: Query Uptime Kuma
      ansible.builtin.uri:
        url: "http://localhost:{{ kuma_port }}"
      register: kuma_health
      retries: 5
      delay: 3
      until: kuma_health.status == 200

    - name: Check movies directory
      ansible.builtin.stat:
        path: "{{ media_root }}/movies"
      register: movies_dir

    - name: Assert stack is healthy
      ansible.builtin.assert:
        that:
          - "'Healthy' in jellyfin_health.content"
          - movies_dir.stat.isdir
```

## 9. Run and verify
```bash
ansible-playbook site.yml --syntax-check
ansible-lint
ansible-playbook site.yml      # first run: changed > 0 (image pulls take a few minutes)
ansible-playbook site.yml      # second run: changed=0 failed=0
```
Open `http://<VM_IP>:8096` (Jellyfin) and `http://<VM_IP>:3001` (Uptime Kuma). In Kuma, add an HTTP monitor for `http://localhost:8096/health`.

> `--check --diff` on a fresh VM may fail at the Docker container tasks because Docker isn't installed yet. That is expected.

## 10. Experiments
- **Handler:** add a comment line to `ansibleflix.env.j2` and re-run. The template task reports `changed` and triggers `Restart Jellyfin`.
- **Disk assertion:** set `min_free_gb: 50` and re-run. The play stops at the assert. Revert afterwards.
- **Vault:** break `.vault_pass` to see the decrypt error, then restore it.

## Notes
- The admin passwords in the vault are not read by Jellyfin or Kuma. You create the admin accounts in each web UI on first visit.
- The RedHat (`dnf`) branch is untested; package names may need adjusting.
- If Ansible errors on a temp directory, add `local_tmp = /tmp/.ansible/tmp` under `[defaults]`.

## Cleanup when done
```bash
multipass delete ansibleflix --purge
rm /tmp/ubuntu-noble-arm64.img
```
