# Ansible + Semaphore Production-Style AWS Lab

This repository is designed for the AWS lab architecture:

- One VPC: `10.10.0.0/16`
- Public subnet A: `10.10.1.0/24`
- Public subnet B: `10.10.2.0/24`
- Internet Gateway
- No private subnets
- No NAT Gateway
- EC2 #1: Semaphore + Ansible controller
- EC2 #2: Ansible managed target
- Semaphore exposed through Nginx reverse proxy
- HTTPS with Let's Encrypt
- MariaDB used by Semaphore
- Ansible playbooks stored in Git
- SSH key authentication from Semaphore to target

## Repository layout

```text
ansible-semaphore-production-lab/
├── ansible.cfg
├── .gitignore
├── README.md
├── requirements.yml
├── inventories/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml.example
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   └── verify.yml
├── roles/
│   └── nginx/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       ├── templates/index.html.j2
│       └── templates/nginx.conf.j2
├── files/
│   └── README.md
└── semaphore/
    ├── README.md
    └── task-template.md
```

## 1. Initial controller setup

On the Semaphore EC2 instance:

```bash
sudo apt update
sudo apt install -y ansible git openssh-client
ansible --version
git --version
```

Clone this repository:

```bash
cd /opt
sudo git clone <YOUR_GIT_REPOSITORY_URL> ansible-semaphore-production-lab
sudo chown -R "$USER":"$USER" ansible-semaphore-production-lab
cd /opt/ansible-semaphore-production-lab
```

For a real project, put the repository in GitHub/GitLab/Bitbucket and let Semaphore clone it.

## 2. SSH key

The controller needs an SSH private key for the target.

Example:

```bash
sudo mkdir -p /root/.ssh
sudo chmod 700 /root/.ssh

sudo cp /path/to/semaphore_ansible /root/.ssh/semaphore_ansible
sudo chmod 600 /root/.ssh/semaphore_ansible
```

The corresponding public key must be installed on the target:

```bash
/home/ansible/.ssh/authorized_keys
```

Test from the controller:

```bash
sudo ssh -i /root/.ssh/semaphore_ansible ansible@10.10.2.X
```

Replace `10.10.2.X` with the target's private IP.

## 3. IMPORTANT: avoid the common Semaphore permission error

If Semaphore runs as the `semaphore` user, a key owned by root may not be readable by Semaphore.

A clean approach is to store the SSH private key in Semaphore's Key Store and select it in the Task Template.

If you intentionally use a filesystem key, make sure the Semaphore service account can read it.

Do NOT solve this with:

```bash
chmod 777 /root/.ssh/semaphore_ansible
```

Private keys should remain restricted.

## 4. Configure inventory

Copy the example variables:

```bash
cp inventories/production/group_vars/all.yml.example \
   inventories/production/group_vars/all.yml
```

Edit:

```bash
nano inventories/production/hosts.yml
nano inventories/production/group_vars/all.yml
```

Set the target's private IP.

Example:

```yaml
all:
  children:
    webservers:
      hosts:
        target01:
          ansible_host: 10.10.2.187
```

The exact IP will be different in your AWS environment.

## 5. Test Ansible

Run:

```bash
ansible-inventory -i inventories/production/hosts.yml --graph
```

Then:

```bash
ansible all \
  -i inventories/production/hosts.yml \
  -m ping
```

Expected:

```text
target01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you get:

```text
Permission denied (publickey)
```

check the SSH key, authorized_keys, username, security group, and Semaphore Key Store configuration.

## 6. Test sudo

```bash
ansible all \
  -i inventories/production/hosts.yml \
  -m command \
  -a "whoami" \
  -b
```

The command should execute with privilege escalation.

The target should have a sudoers rule similar to:

```text
ansible ALL=(ALL) NOPASSWD: ALL
```

For a production system, narrow sudo permissions where possible.

## 7. Deploy Nginx

Run:

```bash
ansible-playbook \
  -i inventories/production/hosts.yml \
  playbooks/nginx.yml
```

Or deploy the complete site:

```bash
ansible-playbook \
  -i inventories/production/hosts.yml \
  playbooks/site.yml
```

## 8. Verify

```bash
ansible-playbook \
  -i inventories/production/hosts.yml \
  playbooks/verify.yml
```

You can also test from the controller:

```bash
curl http://10.10.2.X
```

## 9. Semaphore configuration

Create a Semaphore project and connect this Git repository.

Recommended settings:

### Repository

Repository URL:

```text
<YOUR_GIT_REPOSITORY_URL>
```

Branch:

```text
main
```

### Inventory

Use:

```text
inventories/production/hosts.yml
```

### Environment

Example:

```text
ANSIBLE_CONFIG=/path/to/repository/ansible.cfg
```

### Task Template

Playbook:

```text
playbooks/site.yml
```

Inventory:

```text
inventories/production/hosts.yml
```

Repository:

```text
this repository
```

SSH key:

```text
Semaphore Key Store -> semaphore_ansible
```

## 10. Recommended Semaphore workflow

```text
Developer
   |
   v
Git repository
   |
   v
Semaphore
   |
   | SSH
   v
Ansible Controller / Semaphore EC2
   |
   | Ansible
   v
Target EC2
   |
   v
Nginx
```

In this lab, Semaphore and the Ansible controller are the same EC2 instance.

## 11. Idempotency test

Run:

```bash
ansible-playbook \
  -i inventories/production/hosts.yml \
  playbooks/nginx.yml
```

Run it again.

The second run should report little or no changes.

This is an important Ansible production concept: playbooks should be idempotent.

## 12. Change test

Edit:

```text
roles/nginx/templates/index.html.j2
```

Commit and push:

```bash
git add .
git commit -m "Update application landing page"
git push origin main
```

Then execute the Semaphore task again.

## 13. Git workflow

Recommended:

```text
feature branch
     |
     v
Pull Request
     |
     v
Code review
     |
     v
main
     |
     v
Semaphore
     |
     v
Production target
```

Do not store private SSH keys, passwords, AWS access keys, or Semaphore secrets in Git.

## 14. Useful commands

Inventory:

```bash
ansible-inventory -i inventories/production/hosts.yml --graph
```

Ping:

```bash
ansible all -i inventories/production/hosts.yml -m ping
```

Gather facts:

```bash
ansible all -i inventories/production/hosts.yml -m setup
```

Run playbook:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml
```

Check syntax:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml --syntax-check
```

Dry-run:

```bash
ansible-playbook \
  -i inventories/production/hosts.yml \
  playbooks/site.yml \
  --check
```

Diff:

```bash
ansible-playbook \
  -i inventories/production/hosts.yml \
  playbooks/site.yml \
  --check --diff
```

## 15. AWS security model

The target is in a public subnet, but Ansible should use its private VPC address:

```text
Semaphore EC2
10.10.1.X
     |
     | VPC local route
     |
10.10.2.X
Target EC2
```

Security groups should preferably allow:

```text
Target SG
TCP 22
Source: Semaphore SG
```

Do not expose SSH from `0.0.0.0/0` unless this is a temporary lab requirement.

## 16. Production improvements

After completing this lab, improve it with:

- Separate dev/staging/production inventories
- Ansible Vault
- CI linting with ansible-lint
- Molecule tests
- Git pull-request workflow
- Restricted sudo
- Dedicated deployment user
- CloudWatch/Prometheus monitoring
- Centralized logging
- Secret manager integration
- SSH host-key verification
- Immutable infrastructure
- Terraform for AWS infrastructure
- Separate Semaphore database server
- Private subnets and NAT/VPC endpoints in a real production architecture

This repository intentionally keeps the AWS network architecture public-subnet-only because that is the lab requirement.
