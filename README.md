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

# Knowledge Transfer (KT) Document

## AWS EC2 + Ansible Semaphore + Nginx Deployment Lab

**Document type:** Technical Knowledge Transfer / Operations Runbook\
**Environment:** AWS, Ubuntu, Ansible, Semaphore, MariaDB, Nginx, Git,
Let's Encrypt\
**Purpose:** Document the end-to-end lab, architecture, implementation,
operations, security, validation, and troubleshooting.

> **Important:** This document describes the lab as designed in this
> project. Replace example values with the actual values from your AWS
> account. The target private IP `10.10.2.187`, domain
> `semaphore.sanchitha.xyz`, and key name `production-ansible-key` are
> examples from the lab context; verify each before running commands.
> Never commit passwords, private SSH keys, database secrets, or TLS
> private keys to Git.

------------------------------------------------------------------------

## 1. Executive Summary

This project deploys an Ansible Semaphore automation server and an
Ubuntu managed server in AWS. Semaphore provides a web UI for running
Ansible playbooks. Ansible connects to the target over SSH and
configures Nginx. The Semaphore web interface is published through Nginx
as a reverse proxy and secured with a Let's Encrypt TLS certificate.

The project demonstrates:

-   AWS VPC networking using public subnets and an Internet Gateway.
-   Two EC2 instances: one controller and one managed node.
-   SSH key-based access from the controller to the target.
-   Semaphore projects, repositories, inventories, Key Store
    credentials, and task templates.
-   MariaDB as Semaphore's database.
-   Ansible inventory, playbooks, roles, Jinja2 templates, handlers, and
    idempotency.
-   Nginx site configuration and reverse proxying.
-   DNS and HTTPS certificate issuance.
-   Operational testing, troubleshooting, and safe credential handling.

### Expected outcome

1.  An operator opens the Semaphore URL over HTTPS.
2.  The operator launches a task template.
3.  Semaphore checks out the Ansible Git repository.
4.  Ansible connects to `target01` as the `ansible` user using the SSH
    key selected in Semaphore.
5.  The playbook installs and configures Nginx on the target.
6.  The play recap reports `failed=0`, and the target website responds
    over HTTP.

------------------------------------------------------------------------

## 2. Scope and Design Decisions

### In scope

-   One VPC.
-   Two public subnets; **no private subnets and no NAT Gateway**.
-   One Internet Gateway and public route table.
-   Two Ubuntu EC2 instances.
-   Semaphore, Ansible, Git, MariaDB, Nginx reverse proxy, Certbot.
-   Nginx deployment to the managed node.
-   DNS record and Let's Encrypt HTTPS.
-   Operational runbook and troubleshooting.

### Out of scope

-   High availability or multi-AZ production design.
-   Private-only EC2 architecture.
-   AWS Site-to-Site VPN or IPsec.
-   Centralized logging, monitoring, backups, or disaster recovery
    beyond basic recommendations.
-   Secrets manager integration unless separately implemented.

### Lab versus production

This is a learning lab. A real production environment should normally
keep managed nodes in private subnets, restrict administrative ingress,
use SSM or a controlled bastion/VPN where suitable, store secrets in a
secrets manager, configure backups and monitoring, and apply
least-privilege sudo rules. The public-subnet-only requirement is
retained here for the lab and should not automatically be copied into a
production architecture.

------------------------------------------------------------------------

## 3. Architecture

``` text
                           Internet
                              |
                       Internet Gateway
                              |
                    Public Route Table
                       /            \
                      /              \
         Public Subnet A          Public Subnet B
           10.10.1.0/24             10.10.2.0/24
                |                        |
      EC2: Semaphore Server     EC2: Ansible Target
      - Ubuntu                  - Ubuntu
      - Semaphore               - ansible user
      - MariaDB                 - SSH authorized key
      - Ansible + Git            - Nginx
      - Nginx reverse proxy
      - Certbot / HTTPS
                |
                | SSH TCP/22 over VPC private addresses
                +-------------------------------> target01
```

### Network behavior

Both instances are in public subnets and the subnet route table sends
`0.0.0.0/0` to the Internet Gateway. The VPC's local route allows
traffic between the two subnets using private IP addresses. The Ansible
controller should connect to the target's **private IP**, not its public
IP, when both are in the same VPC.

A public subnet is defined by its route to an Internet Gateway; an
instance also needs a public IPv4 address or Elastic IP and suitable
security-group/NACL rules for direct inbound Internet access.

------------------------------------------------------------------------

## 4. Resource and Value Register

Record the actual values in your private operations notes. Do not put
secrets in this document or in Git.

  Resource                     Lab example / expected value
  ---------------------------- ----------------------------------------
  VPC CIDR                     `10.10.0.0/16`
  Public subnet A              `10.10.1.0/24`
  Public subnet B              `10.10.2.0/24`
  Controller EC2               `semaphore-server`
  Target EC2                   `ansible-target-01`
  Target inventory alias       `target01`
  Target private IP            `10.10.2.187` (verify actual value)
  Target SSH user              `ansible`
  SSH key entry in Semaphore   `production-ansible-key`
  Semaphore DNS name           `semaphore.sanchitha.xyz` (verify DNS)
  Semaphore internal port      `3000` (verify configuration)
  Semaphore database           `semaphore`
  Managed service              Nginx

Recommended operating system: a supported Ubuntu LTS release. Use the
same release assumptions consistently across the controller and target.

------------------------------------------------------------------------

## 5. AWS Networking Setup

### 5.1 Create a VPC

In AWS Console, open **VPC → Your VPCs → Create VPC**.

-   Name: `ansible-semaphore-vpc`
-   IPv4 CIDR: `10.10.0.0/16`
-   Tenancy: default

Enable DNS resolution and DNS hostnames if needed.

### 5.2 Create two public subnets

Create the subnets in the VPC:

-   `public-subnet-a`: `10.10.1.0/24`
-   `public-subnet-b`: `10.10.2.0/24`

Choose Availability Zones available in your region. Enable auto-assign
public IPv4 addresses for the subnets if that matches your chosen
instance setup. An Elastic IP on the Semaphore server is recommended to
keep its public address stable.

### 5.3 Create and attach an Internet Gateway

-   Create `ansible-semaphore-igw`.
-   Attach it to `ansible-semaphore-vpc`.

### 5.4 Configure the public route table

Create or use a route table named `ansible-semaphore-public-rt` and
associate both public subnets.

Routes:

  Destination      Target
  ---------------- ------------------
  `10.10.0.0/16`   local
  `0.0.0.0/0`      Internet Gateway

There is **no NAT Gateway** in this design.

### 5.5 Security groups

Create separate security groups for the controller and target.

#### `semaphore-sg` (controller)

Inbound rules should be restricted to the actual need:

  Port      Source                         Purpose
  --------- ------------------------------ ------------------------------------------
  TCP 22    Your trusted public IP `/32`   SSH administration
  TCP 80    `0.0.0.0/0`                    HTTP / certificate validation / redirect
  TCP 443   `0.0.0.0/0`                    HTTPS web UI

Do not expose TCP 3000 publicly when Nginx is the reverse proxy. MariaDB
TCP 3306 should not be publicly exposed.

#### `ansible-target-sg` (managed node)

  -----------------------------------------------------------------------
  Port                    Source                  Purpose
  ----------------------- ----------------------- -----------------------
  TCP 22                  `semaphore-sg` security SSH from controller
                          group                   

  TCP 80                  Trusted test IP or      Test the deployed Nginx
                          intended clients        site, if required

  TCP 443                 Only if the target will Optional
                          serve HTTPS             
  -----------------------------------------------------------------------

Avoid allowing SSH from `0.0.0.0/0`. If you need direct administrative
SSH to the target, add a temporary rule limited to your public IP and
remove it afterward.

------------------------------------------------------------------------

## 6. Launch the EC2 Instances

### 6.1 Semaphore controller

Suggested name: `semaphore-server`

-   Ubuntu LTS AMI.
-   Instance type suitable for the lab (for example, `t3.small`; check
    current pricing and region availability).
-   VPC: `ansible-semaphore-vpc`.
-   Subnet: `public-subnet-a`.
-   Security group: `semaphore-sg`.
-   Public IPv4 address enabled.
-   Attach an Elastic IP if using a stable DNS A record.
-   Use an administrator SSH key for OS access.

### 6.2 Managed target

Suggested name: `ansible-target-01`

-   Ubuntu LTS AMI.
-   Subnet: `public-subnet-b`.
-   Security group: `ansible-target-sg`.
-   Public IPv4 address may be enabled for the lab, but Ansible should
    use the target's VPC private IP.
-   Use a separate administrative key if needed for bootstrap.

### 6.3 Verify the private IP

In EC2 → Instances, record the target's **Private IPv4 address**. Update
the inventory if it differs from `10.10.2.187`.

------------------------------------------------------------------------

## 7. Bootstrap the Managed Target

Connect to the target using the EC2 bootstrap/admin key:

``` bash
ssh -i /path/to/bootstrap-key.pem ubuntu@TARGET_PUBLIC_IP
```

Update packages:

``` bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y python3 openssh-server
```

### 7.1 Create the Ansible user

``` bash
sudo adduser --disabled-password --gecos "" ansible
```

Create SSH configuration directory:

``` bash
sudo install -d -m 700 -o ansible -g ansible /home/ansible/.ssh
```

### 7.2 Install the controller's public key

The controller key pair should be generated on the controller or another
trusted workstation. Copy only its **public key** to the target. Never
copy a private key to the target.

On the controller, derive the public key from the private key if needed:

``` bash
ssh-keygen -y -f /secure/path/production-ansible-key
```

Copy the output and, on the target, append it to:

``` bash
sudo nano /home/ansible/.ssh/authorized_keys
```

Set permissions:

``` bash
sudo chown ansible:ansible /home/ansible/.ssh/authorized_keys
sudo chmod 600 /home/ansible/.ssh/authorized_keys
sudo chmod 755 /home/ansible
```

### 7.3 Configure sudo for Ansible

For a lab, passwordless sudo is convenient but broad. If using it,
create a sudoers file with `visudo`:

``` bash
sudo visudo -f /etc/sudoers.d/ansible
```

Content:

``` text
ansible ALL=(ALL) NOPASSWD: ALL
```

Validate:

``` bash
sudo visudo -cf /etc/sudoers.d/ansible
sudo chmod 440 /etc/sudoers.d/ansible
```

For production, replace unrestricted sudo with a reviewed
least-privilege policy appropriate to the playbooks.

### 7.4 Verify SSH daemon

``` bash
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

------------------------------------------------------------------------

## 8. Prepare the Semaphore Controller

SSH to the controller:

``` bash
ssh -i /path/to/admin-key.pem ubuntu@SEMAPHORE_PUBLIC_IP
```

Update OS and install common tools:

``` bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl ca-certificates gnupg lsb-release nginx
```

Install Ansible using a supported method for the chosen Ubuntu release.
On Ubuntu repositories, a typical lab installation is:

``` bash
sudo apt install -y ansible
ansible --version
```

If the repository version does not meet project requirements, follow the
official Ansible installation guidance for the target OS and pin a
tested version.

------------------------------------------------------------------------

## 9. Install and Configure MariaDB

### 9.1 Install packages

``` bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
```

Verify the client exists:

``` bash
mariadb --version
command -v mariadb
```

Enable and start the service:

``` bash
sudo systemctl enable --now mariadb
sudo systemctl status mariadb --no-pager
```

If `mariadb: command not found` appears, confirm the OS and package
status:

``` bash
cat /etc/os-release
dpkg -l | grep -E 'mariadb|mysql'
apt-cache policy mariadb-server mariadb-client
```

Then retry the package installation and inspect any `apt` errors.

### 9.2 Create Semaphore database and user

On many Ubuntu MariaDB installations, local root access uses Unix socket
authentication:

``` bash
sudo mariadb
```

Run SQL (replace the placeholder with a unique strong password):

``` sql
CREATE DATABASE semaphore
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'semaphore'@'localhost'
  IDENTIFIED BY 'REPLACE_WITH_A_STRONG_UNIQUE_PASSWORD';

GRANT ALL PRIVILEGES ON semaphore.* TO 'semaphore'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

Test the application database account:

``` bash
mariadb -u semaphore -p semaphore
```

Then:

``` sql
SELECT DATABASE();
EXIT;
```

Do not use the sample password literally. Keep the real database
password in a protected configuration/secret store and limit file
permissions.

### 9.3 Secure MariaDB

Use the hardening procedure supported by your installed MariaDB package.
Depending on package/version, the command may be
`sudo mariadb-secure-installation` or `sudo mysql_secure_installation`;
check available commands rather than assuming one exists.

MariaDB should listen only on localhost for this design. Do not create a
public security-group rule for port 3306.

------------------------------------------------------------------------

## 10. Install and Configure Semaphore

Semaphore installation methods and command-line flags vary by release.
Use the official Semaphore documentation for the exact release installed
on the controller.

Verify the binary and version:

``` bash
command -v semaphore
semaphore version
semaphore --help
```

### 10.1 Configuration

Semaphore commonly uses a JSON configuration file that contains database
settings. Find the path from the systemd unit:

``` bash
sudo systemctl cat semaphore
```

Inspect only the file path and relevant configuration. Avoid printing
secrets into shared terminals or tickets.

A typical database configuration needs values equivalent to:

-   Database dialect: MySQL/MariaDB
-   Host: `127.0.0.1` or `localhost`
-   Database name: `semaphore`
-   Database user: `semaphore`
-   Database password: the unique secret created earlier

Exact JSON keys differ between versions. Follow the documentation
matching the installed Semaphore version and preserve any existing
required fields.

### 10.2 Systemd service

Check the service:

``` bash
sudo systemctl status semaphore --no-pager
sudo systemctl cat semaphore
```

If a service was created manually, confirm:

-   The executable path is correct.
-   The service uses the intended Linux account.
-   The configuration file is readable by that account but not
    world-readable.
-   The service starts after its database dependency.
-   Logs do not reveal secrets.

Useful commands:

``` bash
sudo systemctl enable --now semaphore
sudo journalctl -u semaphore -n 100 --no-pager
sudo systemctl restart semaphore
```

Semaphore usually listens on an internal HTTP port such as `3000`;
verify the configured port. It should be reachable locally from Nginx
but not directly from the public Internet.

### 10.3 Initial administrator account

Initial-user creation is version-dependent. Use the supported
`semaphore user` command or the initial setup flow documented for the
installed release:

``` bash
semaphore user --help
```

Do not expect to retrieve a web login password in plain text. If lost,
use the documented password-reset or account-creation procedure for the
installed version.

------------------------------------------------------------------------

## 11. Configure DNS for the Semaphore UI

At your DNS provider, create an A record:

-   Name/Host: `semaphore`
-   Type: `A`
-   Value: the controller's Elastic IP
-   TTL: provider default or a short value while testing

For the example domain, the full name is `semaphore.sanchitha.xyz`. Use
your actual domain and DNS provider settings.

Verify resolution from a client:

``` bash
dig +short semaphore.sanchitha.xyz
```

The result should match the controller's public Elastic IP. DNS must
resolve correctly before requesting a Let's Encrypt certificate.

------------------------------------------------------------------------

## 12. Nginx Reverse Proxy for Semaphore

Create a dedicated Nginx site:

``` bash
sudo nano /etc/nginx/sites-available/semaphore
```

Example configuration; replace the domain and confirm Semaphore's
internal port:

``` nginx
server {
    listen 80;
    listen [::]:80;
    server_name semaphore.sanchitha.xyz;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:

``` bash
sudo ln -s /etc/nginx/sites-available/semaphore /etc/nginx/sites-enabled/semaphore
sudo nginx -t
sudo systemctl reload nginx
```

Check the backend locally:

``` bash
curl -I http://127.0.0.1:3000/
```

Check the public HTTP endpoint:

``` bash
curl -I http://semaphore.sanchitha.xyz/
```

If the site already exists, inspect the symlink before creating it
again:

``` bash
ls -l /etc/nginx/sites-enabled/
```

------------------------------------------------------------------------

## 13. Enable HTTPS with Let's Encrypt

Install Certbot and the Nginx plugin using the supported method for the
chosen Ubuntu release. A common Ubuntu package-based method is:

``` bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Ensure the domain resolves to the controller and inbound TCP 80/443 is
permitted. Then request a certificate:

``` bash
sudo certbot --nginx -d semaphore.sanchitha.xyz
```

Follow the prompts and choose the HTTP-to-HTTPS redirect if appropriate.

Verify:

``` bash
sudo nginx -t
sudo systemctl reload nginx
sudo certbot certificates
sudo certbot renew --dry-run
```

Do not expose Semaphore's internal port 3000 in the security group.
Public access should be through HTTPS on port 443.

------------------------------------------------------------------------

## 14. Git Repository Structure

The project repository should contain playbooks and reusable roles, but
no secrets.

``` text
ansible-semaphore-production-lab/
├── .github/
│   └── workflows/
│       └── ansible-check.yml
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
│       └── templates/
│           ├── index.html.j2
│           └── nginx.conf.j2
├── semaphore/
│   ├── README.md
│   └── task-template.md
├── ansible.cfg
├── requirements.yml
├── .gitignore
└── README.md
```

### 14.1 Important Ansible configuration

At repository root, `ansible.cfg` should include a role path that points
to the top-level `roles` directory:

``` ini
[defaults]
inventory = inventories/production/hosts.yml
roles_path = ./roles
host_key_checking = True
interpreter_python = auto_silent
stdout_callback = default
forks = 10
timeout = 30

[ssh_connection]
pipelining = True
```

The exact supported options can vary by Ansible version. Avoid disabling
SSH host-key checking as a routine fix; securely establish and verify
host keys.

**Run commands from the repository root**, especially when using
relative paths:

``` bash
cd /home/ubuntu/ansible-semaphore-production-lab
```

### 14.2 Inventory

Example `inventories/production/hosts.yml`:

``` yaml
---
all:
  children:
    webservers:
      hosts:
        target01:
          ansible_host: 10.10.2.187
          ansible_user: ansible
          ansible_port: 22
```

Replace `10.10.2.187` with the target's actual private IP. The
`target01` value is an inventory alias, not a DNS requirement.

Do **not** put a Semaphore Key Store name in this YAML. Semaphore's
stored SSH key is selected in the Semaphore inventory/task-template
configuration, depending on the version and UI. A key name such as
`production-ansible-key` is a Semaphore credential label, not a
filesystem path or Ansible variable.

### 14.3 Example variables

Use `group_vars/all.yml` for non-secret settings only. If the repository
contains `all.yml.example`, copy it to the expected filename only after
reviewing the variable names in the playbooks.

``` bash
cp inventories/production/group_vars/all.yml.example \
   inventories/production/group_vars/all.yml
```

If this creates a real local configuration file, make sure `.gitignore`
prevents secrets or environment-specific values from being committed.

------------------------------------------------------------------------

## 15. How the Nginx Ansible Role Works

### 15.1 `roles/nginx/defaults/main.yml`

Defaults define overridable values such as package name and service
name. Defaults should be safe and generic.

### 15.2 `roles/nginx/tasks/main.yml`

The role typically performs these operations:

1.  Install Nginx with the `apt` module.
2.  Render the virtual host from a Jinja2 template.
3.  Enable the site using a symlink.
4.  Remove the default site if the lab requires it.
5.  Deploy a landing page.
6.  Ensure Nginx is enabled and running.

### 15.3 Templates

-   `nginx.conf.j2`: Nginx virtual-host configuration.
-   `index.html.j2`: example landing page.

Templates let Ansible render consistent configuration and content using
variables.

### 15.4 Handler

The handler reloads or restarts Nginx only when a notified configuration
task changes. Before a reload, the complete Nginx configuration should
be valid.

### 15.5 Playbooks

-   `playbooks/site.yml`: entry point that applies the role(s).
-   `playbooks/nginx.yml`: focused Nginx deployment.
-   `playbooks/verify.yml`: verification tasks.

A typical role invocation is:

``` yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - nginx
```

------------------------------------------------------------------------

## 16. Ansible Syntax and Manual Testing

Run these commands from the repository root.

### 16.1 Syntax-check

``` bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/site.yml --syntax-check
```

### 16.2 Ping module

Use the correct key path for the Linux user running Ansible:

``` bash
ansible webservers \
  -i inventories/production/hosts.yml \
  -m ping \
  --private-key /secure/path/production-ansible-key
```

Do not assume a key in `/root/.ssh` is readable by the `semaphore`
service account. File access is governed by Linux permissions.

### 16.3 Check mode

``` bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/site.yml \
  --private-key /secure/path/production-ansible-key \
  --check --diff
```

Check mode is useful but not every task/module predicts changes
perfectly.

### 16.4 Apply the playbook

``` bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/site.yml \
  --private-key /secure/path/production-ansible-key
```

### 16.5 Idempotency

Run the playbook a second time. Once the desired state is present, tasks
should mostly report `ok` rather than `changed`. Unexpected repeated
changes may indicate a template, file permission, package, or handler
issue.

------------------------------------------------------------------------

## 17. Configure Semaphore for the Git Repository

The UI wording differs by Semaphore version, but the logical objects
are:

### 17.1 Project

Create a project for the Ansible deployment lab, for example:

`Ansible Production Lab`

### 17.2 Repository

Add the Git repository URL and branch. If using a private Git
repository, configure repository authentication separately. Do not put
Git credentials directly in the repository URL if that URL may be logged
or committed.

### 17.3 Inventory

Add or select the production inventory. Depending on Semaphore version,
the inventory may be stored as a UI object or read from the repository.
Ensure the effective inventory contains `target01` and its correct
private IP.

### 17.4 Environment

Add only non-secret environment variables required by the playbooks.
Store secrets using the supported secret/variable facility for your
Semaphore version, not in Git.

### 17.5 Key Store

Create or select the SSH credential named:

`production-ansible-key`

It must contain the **private key** corresponding to the public key
installed in `/home/ansible/.ssh/authorized_keys` on the target. Never
paste the `.pub` public key into a private-key field.

### 17.6 Task template

Create a task template with the appropriate values:

-   Repository: the project Git repository.
-   Branch: the intended branch, commonly `main`.
-   Inventory: production inventory.
-   Playbook: `playbooks/site.yml`.
-   Environment: the production/lab environment.
-   SSH credential: `production-ansible-key`, wherever the installed
    Semaphore version provides the credential selector.
-   Extra variables: only those required by the playbook.

Save and run the template. The first Ansible task is often
`Gathering Facts`; it must complete before the role runs.

------------------------------------------------------------------------

## 18. SSH Key Flow and Security

The intended flow is:

``` text
Semaphore Key Store
        |
        | private key used by Ansible at task runtime
        v
Controller / Ansible SSH client
        |
        | SSH TCP/22 to target's private IP
        v
Target sshd
        |
        | matches public key in authorized_keys
        v
ansible user session
        |
        | become: true, subject to sudoers policy
        v
Nginx/package/configuration tasks
```

Important distinctions:

-   **Private key:** kept secret in Semaphore's Key Store or an approved
    secret store.
-   **Public key:** installed in the target user's `authorized_keys`.
-   **Key label:** `production-ansible-key` is the name of a credential
    in Semaphore, not a key file.
-   **SSH user:** `ansible` is the Linux account on the target.
-   **Inventory alias:** `target01` identifies the host in Ansible
    output.
-   **Target address:** `10.10.2.187` is the example private IP; verify
    it.

------------------------------------------------------------------------

## 19. Verification and Acceptance Checklist

### AWS

-   [ ] VPC CIDR is `10.10.0.0/16` or the chosen equivalent.
-   [ ] Both public subnets are associated with the intended route
    table.
-   [ ] Route table has a local route and a default route to the
    Internet Gateway.
-   [ ] No NAT Gateway or private subnet was created for this lab.
-   [ ] Controller and target are in the correct VPC/subnets.
-   [ ] Security groups restrict SSH and do not expose MariaDB or
    Semaphore's internal port publicly.

### Controller

-   [ ] Ansible is installed and `ansible --version` works.
-   [ ] Git is installed.
-   [ ] MariaDB service is active.
-   [ ] Semaphore service is active.
-   [ ] Semaphore can reach its local database.
-   [ ] Nginx reverse proxy passes `sudo nginx -t`.
-   [ ] DNS resolves to the controller's public/Elastic IP.
-   [ ] HTTPS certificate is valid and renewal dry-run succeeds.

### Target

-   [ ] `ansible` user exists.
-   [ ] `authorized_keys` contains the matching public key.
-   [ ] SSH daemon is active.
-   [ ] `sudo -n true` works for the Ansible user if passwordless sudo
    is configured.
-   [ ] Nginx is installed and enabled.
-   [ ] The deployed website responds on the intended port.

### Semaphore and Ansible

-   [ ] Repository sync/checkout succeeds.
-   [ ] Inventory resolves `target01` to the correct private IP.
-   [ ] Correct SSH credential is selected.
-   [ ] `Gathering Facts` succeeds.
-   [ ] Playbook ends with `failed=0` and `unreachable=0`.
-   [ ] A second run is mostly idempotent (`changed=0` where expected).
-   [ ] The target site displays the expected landing page.

------------------------------------------------------------------------

## 20. Troubleshooting Runbook

### Issue A: `role 'nginx' was not found`

**Cause:** Ansible is not searching the repository's top-level `roles`
directory, or the playbook is being run from the wrong working
directory.

Check:

``` bash
cd /home/ubuntu/ansible-semaphore-production-lab
ls -la
ls -la roles/nginx
```

Ensure `ansible.cfg` contains:

``` ini
[defaults]
roles_path = ./roles
```

Run from the repository root:

``` bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/site.yml --syntax-check
```

Semaphore may use a temporary checkout directory and its own working
directory. If the issue happens only in Semaphore, confirm that the
repository contains `ansible.cfg` and the role directory at the expected
relative paths.

### Issue B: Nginx validation reports `"server" directive is not allowed here`

**Cause:** The `server {}` virtual-host snippet was passed to
`nginx -t -c %s` as if it were a complete Nginx configuration. Nginx's
`-c` expects a complete configuration with the correct enclosing
contexts, not a standalone site snippet.

Do not validate a standalone `server {}` file with that command. For a
straightforward lab, remove the incorrect
`validate: "/usr/sbin/nginx -t -c %s"` argument from the template task
and validate the complete installed configuration with `nginx -t` before
reloading.

For a more robust production role, stage the candidate config, run a
validation procedure against a complete configuration context, and only
then activate/reload it. Ensure a failed validation does not trigger a
reload.

Commands on target:

``` bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo journalctl -u nginx -n 100 --no-pager
```

### Issue C: `Permission denied (publickey)` during `Gathering Facts`

**Likely causes:**

-   The wrong SSH key is selected in the Semaphore task template.
-   The private key does not correspond to the public key in target
    `authorized_keys`.
-   The target username is not `ansible`.
-   The target IP is stale or wrong.
-   The SSH key is stored in Semaphore but not attached to the
    task/inventory.
-   A file path key is unreadable to the Linux account running
    Semaphore/Ansible.
-   The target's `.ssh` permissions are incorrect.

Verify inventory:

``` yaml
all:
  children:
    webservers:
      hosts:
        target01:
          ansible_host: 10.10.2.187
          ansible_user: ansible
          ansible_port: 22
```

Verify on the target:

``` bash
id ansible
sudo ls -ld /home/ansible /home/ansible/.ssh
sudo ls -l /home/ansible/.ssh/authorized_keys
```

Typical permissions:

``` text
/home/ansible                 755 (or stricter compatible permissions)
/home/ansible/.ssh            700
/home/ansible/.ssh/authorized_keys 600
```

Correct ownership:

``` bash
sudo chown -R ansible:ansible /home/ansible/.ssh
sudo chmod 700 /home/ansible/.ssh
sudo chmod 600 /home/ansible/.ssh/authorized_keys
```

Test from the controller using a private key that is readable by the
current shell user:

``` bash
ssh -vvv -i /secure/path/production-ansible-key \
  ansible@10.10.2.187
```

If direct SSH succeeds but Semaphore fails, focus on the Semaphore Key
Store selection and task-template configuration. Do not copy a private
key into Git or make it world-readable to work around permissions.

Check the service account and service definition:

``` bash
sudo systemctl status semaphore --no-pager
sudo systemctl cat semaphore
ps aux | grep '[s]emaphore'
```

### Issue D: `mariadb: command not found`

Install the client/server packages:

``` bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
```

Verify:

``` bash
mariadb --version
sudo systemctl enable --now mariadb
```

If installation fails, inspect the full package-manager error and
`cat /etc/os-release`. Do not keep running SQL commands until the
client/server package issue is resolved.

### Issue E: Semaphore UI cannot connect to the database

Check service status and logs:

``` bash
sudo systemctl status mariadb --no-pager
sudo systemctl status semaphore --no-pager
sudo journalctl -u semaphore -n 100 --no-pager
```

Confirm the configured database name, username, host, and password match
the MariaDB account. Avoid sharing raw config output because it may
contain secrets. Confirm MariaDB is local and that no public 3306 rule
is present.

### Issue F: UI is unavailable through the domain

Check DNS, security groups, Nginx, Semaphore, and TLS separately:

``` bash
dig +short semaphore.sanchitha.xyz
sudo systemctl status semaphore --no-pager
sudo systemctl status nginx --no-pager
sudo nginx -t
curl -I http://127.0.0.1:3000/
sudo certbot certificates
```

Confirm the DNS A record points to the correct Elastic IP, ports 80/443
are allowed, the reverse-proxy `proxy_pass` matches Semaphore's actual
listener, and the certificate includes the exact hostname.

### Issue G: Playbook runs manually but fails in Semaphore

Compare:

-   Git branch and repository version.
-   Inventory actually selected in the template.
-   SSH credential selected in Semaphore.
-   Task-template playbook path.
-   Ansible version and configuration path.
-   Relative paths and role search path.
-   Environment variables and extra vars.
-   Linux service account file permissions.

Semaphore runs tasks in its own execution context. A successful command
from your interactive `ubuntu` shell does not prove the `semaphore`
service account has the same filesystem access or credentials.

### Issue H: Nginx is installed but the site is not accessible

On the target:

``` bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo ss -lntp | grep -E ':(80|443)\b'
curl -I http://127.0.0.1/
```

Then check the target security group and the client source address. If
the site is intended to be reachable only from a test IP, ensure the
inbound rule matches that source.

------------------------------------------------------------------------

## 21. Routine Operations

### Run a deployment

1.  Review changes in Git.
2.  Commit and push to the intended branch.
3.  Open Semaphore and confirm the correct project/repository/inventory.
4.  Run the task template.
5.  Review the full output and play recap.
6.  Verify the website and service state on the target.

### Check services

Controller:

``` bash
sudo systemctl status semaphore --no-pager
sudo systemctl status mariadb --no-pager
sudo systemctl status nginx --no-pager
```

Target:

``` bash
sudo systemctl status nginx --no-pager
```

### Review logs

``` bash
sudo journalctl -u semaphore -n 100 --no-pager
sudo journalctl -u mariadb -n 100 --no-pager
sudo journalctl -u nginx -n 100 --no-pager
```

Use only the applicable commands on each host.

### Update the inventory after an EC2 replacement

If the target is recreated, confirm its new private IP and update the
inventory. Confirm the SSH host key change is legitimate before
accepting it; do not blindly disable host-key checking.

------------------------------------------------------------------------

## 22. Security and Maintenance Recommendations

-   Keep SSH ingress restricted to trusted addresses/security groups.
-   Do not expose Semaphore's internal port or MariaDB to the Internet.
-   Keep private SSH keys out of Git, shell history, logs, and shared
    tickets.
-   Use unique strong database and admin passwords.
-   Limit sudo privileges for production.
-   Keep Ubuntu, Ansible, Semaphore, MariaDB, Nginx, and Certbot
    patched.
-   Monitor disk usage and service logs.
-   Back up Semaphore's database and configuration securely before
    upgrades.
-   Test restores rather than assuming backups are usable.
-   Pin and test dependency versions for repeatable automation.
-   Review Git changes before running production tasks.
-   Consider AWS Systems Manager, private subnets, and a controlled
    administration path for a real production environment.
-   Keep DNS and certificate renewal under monitoring.
-   Document who owns the domain, Elastic IP, AWS account, credentials,
    and recovery procedure.

------------------------------------------------------------------------

## 23. Change and Rollback Approach

Before a risky change:

1.  Commit the current working repository state.
2.  Record the current target configuration.
3.  Run `--syntax-check`.
4.  Use `--check --diff` when supported by the tasks.
5.  Run against one target first.
6.  Verify Nginx configuration and service state.
7.  Keep the prior known-good template/configuration available for
    rollback.

If a deployment breaks the website, restore the known-good template or
Git revision, validate with `sudo nginx -t`, then reload Nginx only
after validation passes. A rollback should not blindly overwrite
unrelated host configuration.

------------------------------------------------------------------------

## 24. Final End-to-End Test Procedure

From the controller, confirm repository state:

``` bash
cd /home/ubuntu/ansible-semaphore-production-lab
git status
ansible --version
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/site.yml --syntax-check
```

Run a connectivity test manually if required:

``` bash
ansible webservers -i inventories/production/hosts.yml \
  -m ping --private-key /secure/path/production-ansible-key
```

Then run the deployment from Semaphore using the
`production-ansible-key` credential.

Acceptance conditions:

-   `Gathering Facts` succeeds for `target01`.
-   Nginx role completes without failures.
-   Play recap shows `failed=0` and `unreachable=0`.
-   A second run produces no unexpected changes.
-   Target Nginx responds to HTTP from an allowed client.
-   Semaphore UI responds on the domain over HTTPS.
-   `certbot renew --dry-run` succeeds.
-   No private keys, database passwords, or application secrets are
    present in Git.

------------------------------------------------------------------------

## 25. Glossary

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  AWS VPC                             Logically isolated virtual network
                                      in AWS

  Public subnet                       Subnet whose route table has a
                                      route to an Internet Gateway

  Internet Gateway                    Enables Internet connectivity for
                                      eligible resources in a VPC

  Security group                      Stateful virtual firewall attached
                                      to AWS resources

  EC2                                 AWS virtual machine service

  Ansible controller                  Machine that runs Ansible tasks

  Managed node / target               Host configured by Ansible

  Inventory                           List of hosts and host/group
                                      variables

  Playbook                            YAML automation instructions

  Role                                Reusable Ansible structure for
                                      tasks, handlers, defaults, and
                                      templates

  Idempotency                         Repeated runs converge on the
                                      desired state without unnecessary
                                      changes

  Semaphore                           Web UI and task orchestration for
                                      Ansible

  Key Store                           Semaphore credential store for keys
                                      and related secrets

  SSH public key                      Public half installed on the target
                                      to authorize key-based login

  SSH private key                     Secret half used by the client to
                                      authenticate

  Nginx reverse proxy                 Accepts web requests and forwards
                                      them to an upstream service

  Let's Encrypt                       Certificate authority providing
                                      automated TLS certificates

  MariaDB                             Relational database used here for
                                      Semaphore application data

  Handler                             Ansible task triggered by
                                      notification, often to reload a
                                      service
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 26. Project Handover Summary

The project establishes an AWS lab with two EC2 instances in public
subnets. Semaphore is the web-based control plane, backed by MariaDB and
published through an Nginx reverse proxy with HTTPS. Ansible playbooks
are stored in Git and executed through Semaphore task templates. The
target is managed over SSH using the `ansible` account and the
`production-ansible-key` credential. The Nginx role installs and
configures the target web server.

The most important operational points are:

1.  Run Ansible from the repository root and ensure
    `roles_path = ./roles`.
2.  Keep the Semaphore SSH private key in the Semaphore Key Store and
    attach it to the correct task/template.
3.  Ensure the matching public key is present in the target's
    `authorized_keys`.
4.  Use the target's correct VPC private IP in inventory.
5.  Validate the complete Nginx configuration with `nginx -t`; do not
    validate a standalone `server {}` snippet as a complete config.
6.  Keep MariaDB and Semaphore's internal port off the public Internet.
7.  Verify task output and the website after every deployment.

**End of document.**

