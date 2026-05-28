# Ansible Template

Project template for Ansible playbooks targeting Debian-based devices (Raspberry Pi OS, Ubuntu).
Uses the [`eledio_admin.services`](https://galaxy.ansible.com/ui/repo/published/eledio_admin/services/) collection.

## Prerequisites

- Python ≥ 3.12
- [uv](https://docs.astral.sh/uv/)

```bash
uv sync --extra dev
uv run ansible-galaxy collection install -r requirements.yml -p ./collections
```

## Project structure

```
├── ansible.cfg              # Ansible configuration
├── requirements.yml         # Collection dependencies
├── inventory.yml            # Host inventory
├── group_vars/
│   └── devices.yml          # Shared vars for all devices — customize per project
├── vault.yml                # Encrypted secrets (not committed)
├── vault.yml.example        # Vault template
├── setup.yml                # Full initial setup
├── update.yml               # Update app code and restart service
├── reboot.yml               # Reboot devices
├── vpn-enable.yml           # Enable OpenVPN client
├── vpn-disable.yml          # Disable OpenVPN client
└── roles/
    └── app/                 # Local project-specific role
        ├── defaults/main.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/
            └── app.service.j2
```

## Using this template

1. Add hosts to `inventory.yml`
2. Edit `group_vars/devices.yml` — set app name, user, install dir, exec command
3. Create and encrypt vault: `cp vault.yml.example vault.yml && ansible-vault encrypt vault.yml`
4. Fill in vault secrets
5. Implement `roles/app/` for project-specific deployment logic

## Vault secrets

| Variable | Used by | Description |
|---|---|---|
| `sudo_password` | inventory | SSH + sudo password |
| `vpn_config_content` | `eledio_admin.services.vpn` | Full OpenVPN client config |
| `label_printer_git_user` | `eledio_admin.services.label_printer` | Git username |
| `label_printer_git_token` | `eledio_admin.services.label_printer` | Git access token |
| `label_printer_config_content` | `eledio_admin.services.label_printer` | Printer config file content |
| `app_git_repo` | `roles/app` | Git repo URL (when `app_deploy_source: git`) |
| `app_git_token` | `roles/app` | Git access token (when `app_deploy_source: git`) |

## Playbooks

### setup.yml

Full initial setup. Runs all collection roles and the local `app` role.

```bash
ansible-playbook setup.yml --ask-vault-pass
```

Optional roles are tagged — skip any you don't need:

```bash
ansible-playbook setup.yml --ask-vault-pass --skip-tags vpn,st_link,label_printer
ansible-playbook setup.yml --ask-vault-pass --tags vpn        # run only vpn role
```

Collection roles applied in order:

| Role | Tag | Description |
|---|---|---|
| `eledio_admin.services.common` | — | apt update + base packages |
| `eledio_admin.services.install_uv` | — | install uv Python manager |
| `eledio_admin.services.vpn` | `vpn` | install OpenVPN + deploy client config |
| `eledio_admin.services.st_link` | `st_link` | ST-Link udev rules |
| `eledio_admin.services.label_printer` | `label_printer` | label printer service |
| `roles/app` | — | project-specific app deployment |

### update.yml

Re-deploys app code and restarts the service. Verifies the service is running after restart.

```bash
ansible-playbook update.yml --ask-vault-pass
```

### reboot.yml

Reboots devices and waits for reconnection.

```bash
ansible-playbook reboot.yml --ask-vault-pass
```

### vpn-enable.yml / vpn-disable.yml

Enable or disable the OpenVPN client service without re-deploying config.

```bash
ansible-playbook vpn-enable.yml --ask-vault-pass
ansible-playbook vpn-disable.yml
```

## Configuration

### group_vars/devices.yml

Shared variables for all devices in the `devices` group. Edit this file when starting a new project.

| Variable | Default | Description |
|---|---|---|
| `app_service_name` | `app` | systemd service name |
| `app_service_user` | `pi` | OS user that runs the service |
| `app_install_dir` | `/srv/app` | installation directory |
| `app_exec_command` | `{{ app_install_dir }}/.venv/bin/python -m app` | systemd `ExecStart` command |
| `vpn_config_name` | `client` | OpenVPN config name (`openvpn-client@<name>`) |

### roles/app/defaults/main.yml

Role-level defaults. `group_vars/devices.yml` takes precedence over these.

| Variable | Default | Description |
|---|---|---|
| `app_deploy_source` | `git` | deploy source: `git` or `local` |
| `app_git_repo` | — | git repo URL (required for `git` source) |
| `app_git_token` | — | git access token (required for `git` source) |
| `app_local_path` | `""` | local path on control machine (required for `local` source) |

### Deploy from local machine

Useful during development — syncs local directory to device via rsync instead of cloning from git:

```yaml
# group_vars/devices.yml
app_deploy_source: local
app_local_path: /path/to/local/project
```

## Local app role

`roles/app/` is a skeleton for project-specific deployment logic. It handles:

- create service user
- create install directory
- clone from git or sync from local path
- run `uv sync` to install Python dependencies
- deploy `app.service.j2` systemd unit
- enable and start service

Customize `roles/app/templates/app.service.j2` and `roles/app/defaults/main.yml` for the specific project.

## CI/CD

GitHub Actions runs `ansible-lint` on every push and pull request. The `vault.yml.example` is copied to `vault.yml` before linting so the vault reference resolves.
