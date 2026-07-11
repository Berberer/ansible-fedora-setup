# AGENTS.md — Ansible Fedora Setup

## Repository Overview

Ansible playbooks for opinionated initial setup of Fedora. Local execution only — targets `localhost` with `connection: local` and `become: true`. Two target types are supported:

- **Headless** — systems without a desktop environment (e.g. WSL, servers)
- **Desktop** — systems with a desktop environment (inherits all headless steps plus desktop extras)

## Commands

```bash
# Run the headless playbook (requires vault + sudo passwords)
export ANSIBLE_CONFIG=$(pwd)/ansible.cfg
ansible-playbook playbooks/headless.yaml --ask-vault-pass --ask-become-pass

# Run the desktop playbook (includes all headless steps + desktop extras)
ansible-playbook playbooks/desktop.yaml --ask-vault-pass --ask-become-pass

# Run with specific tags
ansible-playbook playbooks/headless.yaml --ask-vault-pass --ask-become-pass --tags git
ansible-playbook playbooks/desktop.yaml --ask-vault-pass --ask-become-pass --tags desktop
```

## Playbook Control Flow

```
headless.yaml (entry point)
  ├── pre_tasks: common_tasks/pre_config.yaml          (dnf update_cache)
  ├── roles:
  │   ├── base_setup                                   (tags: packages, readline, ssh)
  │   │   ├── tasks/dnf.yaml                           (install common + desktop packages, remove unwanted)
  │   │   ├── tasks/readline.yaml                      (~/.inputrc for bash autocompletion)
  │   │   └── tasks/ssh.yaml                           (copy keys from vault)
  │   └── dev_tools                                    (tags: git, podman, nix, starship)
  │       ├── tasks/git.yaml                           (git + git-lfs + git-delta + config)
  │       ├── tasks/podman.yaml                        (podman + socket + orphan cleanup timer)
  │       ├── tasks/nix.yaml                           (nix + direnv + config + GC timer)
  │       └── tasks/starship.yaml                      (curl install + bash init + config)
  └── post_tasks: common_tasks/post_config.yaml        (dnf autoremove)

desktop.yaml (entry point)
  └── ansible.builtin.import_playbook: headless.yaml   (all of the above)
  └── roles:
      └── desktop_init                                 (tags: desktop)
          └── tasks/main.yaml                          (desktop extras — fonts, themes, GUI packages)
```

## Architecture & Patterns

- **Two entry points**: `playbooks/headless.yaml` and `playbooks/desktop.yaml`. `desktop.yaml` imports `headless.yaml` via `ansible.builtin.import_playbook` and adds extra roles on top.
- **Common pre/post tasks** in `playbooks/common_tasks/` — only `dnf update_cache` (pre) and `autoremove` (post). Used by `include_tasks` from playbook level.
- **Variables**: Shared variables go in `inventory/group_vars/all/`. Secrets (SSH keys, GitHub token) live in vault-encrypted `secrets.yaml`; non-secret user config in `common.yaml`. Target-type-specific vars in `inventory/group_vars/headless/` and `inventory/group_vars/desktop/`.
- **Handler**: `roles/dev_tools/handlers/main.yaml` — single handler `Reload systemd daemon`, notified after creating systemd service/timer files.
- **Static files**: `roles/<role>/files/` — copied via `ansible.builtin.copy`.
- **Templates**: `roles/<role>/templates/` — processed via `ansible.builtin.template` (Jinja2). Currently only `nix_config.j2` (uses `{{ github_access_token }}`).
- **Tags**: Every logical task group is tagged. Tags match the role/task file name (`git`, `podman`, `nix`, `ssh`, `packages`, `readline`, `starship`, `desktop`). Tags are assigned at role level in the playbook, not on individual tasks — so filters work.
- **Package lists**: `base_setup/vars/main.yaml` holds base `common_packages` for all systems. `inventory/group_vars/desktop/desktop.yaml` defines `desktop_packages` which are merged into `dnf.yaml` via `common_packages + (desktop_packages | default([]))`.

## Gotchas

1. **All tasks run as root via `become: true`**, including user file operations. File ownership is handled via explicit `owner:` / `group:` with `become_user: "{{ target_user }}"`.
2. **`target_user`** must be set in `inventory/group_vars/all/common.yaml` before running (not passed as `--extra-vars`).
3. **Vault password file** (`.ansible_vault_pass`) is gitignored. You must provide the password interactively via `--ask-vault-pass`, or create the ignored file manually.
4. **`secrets.yaml`** contains `ssh_private_key_content`, `ssh_public_key_content`, and `github_access_token`. These are Ansible vault-encrypted. New users must replace this file entirely.
5. **Starship install** uses `curl | sh` (via `ansible.builtin.shell`) — idempotent via `creates: /usr/local/bin/starship`. This is the only task not using a dedicated module.
6. **Systemd user scope**: Podman socket is enabled at the *user* scope (`scope: user`) via `ansible.builtin.systemd`, but still requires `become: true` + `become_user`. The orphan cleanup timer runs at system scope.
7. **Readline config** uses a regexp `^{{ item.regexp | default(item.line) }}$` — the `regexp` field lets some items match with anchorage (`^set bell-style`) while `line` has the full value. Not all items provide `regexp` — falls back to matching the full line.
8. **`.editorconfig`**: YAML files use spaces, indent 2.
9. **Inventory groups**: `inventory/hosts.ini` has `[all]`, `[headless]`, and `[desktop]` groups — all point to `localhost`. These groups enable `group_vars` per target type. The `--limit` flag is not required since both groups target the same host.
10. **Vars in `base_setup/vars/main.yaml`**: `common_packages` (list) — base packages for all systems, `packages_to_remove` (list). Extend via `desktop_packages` in `inventory/group_vars/desktop/desktop.yaml`.
11. **`no_log: true`** is set on SSH key copy tasks to prevent secret leakage in verbose output.
12. **`desktop_packages`** is defined in `inventory/group_vars/desktop/desktop.yaml` and merged into the DNF install task. The `default([])` filter ensures headless systems don't fail if the variable is undefined.
