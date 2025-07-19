# Documentation for

#### Ansible Quick-Start: Install, Deploy Pi-hole, and Add GitLab CI Lint (Homelab Automation)

[YouTube video](https://youtu.be/CrtHyOwVZF0)

---

#### Ansible Resources  
- [Official Docs](https://docs.ansible.com)  
- [Ansible Galaxy](https://galaxy.ansible.com)

---

> ### 🖥️ Example Environment: Admin workstation + Debian 12 server

---

## Pre‑Install

```bash
sudo apt update
sudo apt install pipx git sshpass -y
pipx install --include-deps ansible
pipx install --include-deps ansible-lint
pipx ensurepath
```

> To enable pipx command autocompletion, run: `pipx completions`
>
> Example for **bash**: `eval "$(register-python-argcomplete pipx)"`

Verify installation:

```bash
ansible --version
```

---

## Clone Your GitLab Repo

```bash
git clone https://<your-gitlab-url>/ansible.git
cd ansible
```

---

## Install a Community Role from Ansible Galaxy

Create requirements.yml

```bash
nano requirements.yml
```

```yaml
---
# HSE cloudflared role
- src: HomeSecExplorer.cloudflared
# HSE PiHole role
- src: HomeSecExplorer.pihole
```

Install the role:

```bash
ansible-galaxy install -r requirements.yml -p roles/
```

Directory tree now looks like:

```markdown
ansible/
└── roles/
    ├── HomeSecExplorer.cloudflared/
    └── HomeSecExplorer.pihole/
```

---

## Write the Playbook

```bash
nano playbook.yml
```

Sample content:

```yaml
---
- name: My ansible playbook
  hosts: all
  become: true
  tasks:
    - name: Install PiHole and DOH proxy
      ansible.builtin.include_role:
        name: "{{ item }}"
      loop:
        - "HomeSecExplorer.cloudflared"
        - "HomeSecExplorer.pihole"
      vars:
        hseph_pw_plain: "Passw0rd"  # change role defaults variable
      when: "'dns' in group_names"  # only run on inventory group dns
```

---

## Create inventory

```bash
nano hosts
```

```ini
[dns]  # host group
pihole1 ansible_host=10.10.10.196
```

---

## Test Syntax

```bash
ansible-lint playbook.yml
```

## Run the Playbook

Connect to host once and accept the ssh key


```bash
ansible-playbook -i hosts playbook.yml -u <ssh_user> -k -K
```

Options explained:

- `-i`: Specifies the inventory file.
- `-u`: SSH user for connecting to the target host(s). Can also be set in the inventory.
- `-k`: Prompt for the SSH password. Optional if using key-based auth or setting password in the inventory.
- `-K`: Prompt for the sudo password (become password). Can also be set via inventory.

Once run, you should see tasks from the Pi-hole role being executed.

You can now test with `dig @10.10.10.196 google.com`

---

## Add GitLab CI Lint Pipeline

Create **.gitlab-ci.yml**:

```bash
nano .gitlab-ci.yml
```

```yaml
stages:
  - lint

ansible-lint:
  stage: lint
  image: registry.gitlab.com/pipeline-components/ansible-lint:latest
  script:
    - ansible-lint
```

---

## Create gitignore file

We don’t need to push downloaded roles, so add them to *.gitignore*:

```bash
nano .gitignore
```

```txt
roles/
```

## Commit & Push

```bash
git add .
git commit -m "My commit"
git push
```

GitLab CI requires an available runner.
Check **GitLab → CI/CD → Pipelines** for a green check-mark ✅.

---

## Next Steps

- Expand inventory to multiple hosts
- Secure variables with Ansible Vault
- Integrate GitLab CI for full deployment
