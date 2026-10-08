# Semaphore Task Template

## Repository

```text
Git repository:
<YOUR_GIT_REPOSITORY_URL>

Branch:
main
```

## Inventory

```text
inventories/production/hosts.yml
```

## Playbook

```text
playbooks/site.yml
```

## SSH credential

Create a Semaphore Key Store item:

```text
Name: semaphore_ansible
Type: SSH Key
Private Key: <paste the deployment private key>
```

Never commit the private key into Git.

## Environment

Example:

```text
ANSIBLE_CONFIG=ansible.cfg
```

## First test

Before running the full deployment, execute an Ansible ping from the controller:

```bash
ansible all -i inventories/production/hosts.yml -m ping
```

Then run the Semaphore task.
