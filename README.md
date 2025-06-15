# Ansible Fedora Setup
Ansible playbooks to perform an opinionated initial setup of a Fedora Setup.

## Prerequisites
1. Perform the base Fedora installation.
2. Install git and Ansible:
    ```bash
    sudo dnf install git ansible
    ```
3. Clone this repository and go into the cloned repository.
4. Set the correct username in `inventory/group_vars/all/common.yaml`

*Note*: If other people want to use this setup, they need to replace `inventory/group_vars/all/secrets.yaml` with their own secrets file or remove all tasks from the playbooks that involve secrets.

## Running a Playbook
The following playbooks are available to be run via local execution:
- `wsl.yaml`: Initial Setup for a Fedora WSL system

Choose a playbook and change the `target_user` variable to your username. Then run the selected playbook:
```bash
export ANSIBLE_CONFIG=$(pwd)/ansible.cfg
ansible-playbook playbooks/<Playbook File> --ask-vault-pass --ask-become-pass
```
