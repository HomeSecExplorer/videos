# Documentation for

#### How to Install GitLab and Set Up Your First DevOps Project

[YouTube video](https://youtu.be/O-w1QaLrn-0)

---

#### GitLab Documentation

[Official Getting Started Guide](https://docs.gitlab.com/user/get_started/)

---

> ### Example Environment: Debian 12

---

## Pre-Install

```bash
sudo apt update
sudo apt install -y curl openssh-server ca-certificates perl tzdata
```

---

## Install GitLab

Add the GitLab repository and install GitLab CE:

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://your-server-ip" apt install gitlab-ce
```

If password is not displayd, show with `cat /etc/gitlab/initial_root_password`

---

## Optimize GitLab for Performance

Edit the GitLab config:

```bash
sudo nano /etc/gitlab/gitlab.rb
```

Modify the following:

```ruby
puma['worker_processes'] = 2
sidekiq['concurrency'] = 9
prometheus_monitoring['enable'] = false
```

Apply configuration changes:

```bash
sudo gitlab-ctl reconfigure
```

---

## Access GitLab in Browser

Navigate to:

```
http://your-server-ip
```

Set the root password and log in with:

- **Username:** `root`
- **Password:** Auto-generated during installation and saved under  
  `/etc/gitlab/initial_root_password` (valid for 24 hours).

---

## Create Your First Project

1. Create a project named `ansible`
2. Initialize with a `README.md`
3. Clone the repo:

```bash
git clone http://your-server-ip/root/ansible.git
```

> Replace the URL with your actual GitLab project path if different.
> If Git is not installed, run:
> ```bash
> sudo apt install git -y
> ```

---

## Add Ansible Playbook + Lint Locally

```bash
cd ansible
nano playbook.yml
```

Sample playbook:

```yaml
- name: Test Ansible
  hosts: localhost
  tasks:
    - name: Debug message
      ansible.builtin.debug:
        msg: "I'm Ansible"
```

Install and run linter:

```bash
sudo apt install ansible-lint
ansible-lint playbook.yml
```

---

## Commit and Push Changes

> If you're using Git for the first time, configure your identity:
> ```bash
> git config --global user.name "User Name"
> git config --global user.email "your_email@example.com"
> ```

```bash
git add .
git commit -m "Init"
git push
```
