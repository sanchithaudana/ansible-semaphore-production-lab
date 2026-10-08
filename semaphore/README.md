# Semaphore configuration

Use this repository as the Git repository for the Semaphore project.

Recommended Task Template:

- Repository: this repository
- Branch: `main`
- Inventory: `inventories/production/hosts.yml`
- Playbook: `playbooks/site.yml`
- SSH Key: Semaphore Key Store entry containing the deployment private key
- Environment: `ANSIBLE_CONFIG=ansible.cfg`

The exact UI labels can differ between Semaphore versions.
