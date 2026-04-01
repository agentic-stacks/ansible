# Configure Ansible

## Understand Configuration Precedence

Ansible processes configuration in the following order. The first file found wins:

1. `ANSIBLE_CONFIG` environment variable (path to a config file)
2. `./ansible.cfg` in the current working directory
3. `~/.ansible.cfg` in the home directory
4. `/etc/ansible/ansible.cfg` system-wide default

> **Security note:** Ansible will not load `./ansible.cfg` from a world-writable current directory. Set the `ANSIBLE_CONFIG` environment variable explicitly in that case.

## Set Essential ansible.cfg Settings

```ini
[defaults]
# Inventory source
inventory = ./inventory

# Where to find roles and collections
roles_path = ./roles:/usr/share/ansible/roles
collections_path = ./collections:/usr/share/ansible/collections

# Performance
forks = 20
timeout = 30

# Fact gathering
gathering = smart

# Disable host key checking for lab/dev environments
host_key_checking = False

# Disable retry files
retry_files_enabled = False

# Human-readable output
stdout_callback = yaml
bin_ansible_callbacks = True

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[connection]
pipelining = True

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### Settings Reference

| Section | Setting | Default | Purpose |
|---|---|---|---|
| `[defaults]` | `inventory` | `/etc/ansible/hosts` | Path to inventory file or directory |
| `[defaults]` | `roles_path` | `~/.ansible/roles` | Colon-separated list of roles directories |
| `[defaults]` | `collections_path` | `~/.ansible/collections` | Colon-separated list of collections directories |
| `[defaults]` | `forks` | `5` | Number of parallel processes |
| `[defaults]` | `timeout` | `10` | SSH connection timeout in seconds |
| `[defaults]` | `gathering` | `implicit` | Fact gathering policy: `implicit`, `explicit`, or `smart` |
| `[defaults]` | `host_key_checking` | `True` | Verify SSH host keys |
| `[defaults]` | `retry_files_enabled` | `True` | Create `.retry` files on playbook failure |
| `[defaults]` | `stdout_callback` | `default` | Output formatting plugin |
| `[privilege_escalation]` | `become` | `False` | Enable privilege escalation |
| `[privilege_escalation]` | `become_method` | `sudo` | Escalation method: `sudo`, `su`, `doas`, etc. |
| `[privilege_escalation]` | `become_user` | `root` | Target escalation user |
| `[privilege_escalation]` | `become_ask_pass` | `False` | Prompt for escalation password |
| `[connection]` | `pipelining` | `False` | Reduce SSH operations (requires `requiretty` disabled in sudoers) |
| `[ssh_connection]` | `ssh_args` | (none) | Additional SSH arguments |

## Organize the Directory Layout

```
project/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── webservers.yml
│   │   │   └── dbservers.yml
│   │   └── host_vars/
│   │       └── db1.example.com.yml
│   └── staging/
│       ├── hosts
│       ├── group_vars/
│       │   └── all.yml
│       └── host_vars/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── roles/
│   └── requirements.yml
├── collections/
│   └── requirements.yml
├── group_vars/
│   └── all.yml
├── host_vars/
└── molecule/
```

Keep inventory-specific variable overrides inside `inventory/<env>/group_vars/` and `inventory/<env>/host_vars/`. Place shared defaults in the top-level `group_vars/` directory.

## Override Settings with Environment Variables

| Environment Variable | ansible.cfg Equivalent | Example |
|---|---|---|
| `ANSIBLE_CONFIG` | (sets config file path) | `export ANSIBLE_CONFIG=/opt/ansible/ansible.cfg` |
| `ANSIBLE_INVENTORY` | `[defaults] inventory` | `export ANSIBLE_INVENTORY=./inventory/staging` |
| `ANSIBLE_ROLES_PATH` | `[defaults] roles_path` | `export ANSIBLE_ROLES_PATH=./roles:/shared/roles` |
| `ANSIBLE_COLLECTIONS_PATH` | `[defaults] collections_path` | `export ANSIBLE_COLLECTIONS_PATH=./collections` |
| `ANSIBLE_VAULT_PASSWORD_FILE` | `[defaults] vault_password_file` | `export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass` |
| `ANSIBLE_HOST_KEY_CHECKING` | `[defaults] host_key_checking` | `export ANSIBLE_HOST_KEY_CHECKING=False` |
| `ANSIBLE_FORKS` | `[defaults] forks` | `export ANSIBLE_FORKS=20` |
| `ANSIBLE_STDOUT_CALLBACK` | `[defaults] stdout_callback` | `export ANSIBLE_STDOUT_CALLBACK=yaml` |

Environment variables always override the corresponding ansible.cfg setting.

## Enable Callback Plugins

Callback plugins add functionality to playbook runs. Enable them in `ansible.cfg`:

```ini
[defaults]
# Enable additional callback plugins
callbacks_enabled = ansible.posix.timer, ansible.posix.profile_tasks

# Use YAML stdout formatting
stdout_callback = yaml
```

### Timer

Displays total playbook execution time at the end of a run:

```ini
[defaults]
callbacks_enabled = ansible.posix.timer
```

Requires the `ansible.posix` collection:

```bash
ansible-galaxy collection install ansible.posix
```

### Profile Tasks

Shows execution time for each task, sorted by duration:

```ini
[defaults]
callbacks_enabled = ansible.posix.profile_tasks
```

### Combine Multiple Callbacks

```ini
[defaults]
callbacks_enabled = ansible.posix.timer, ansible.posix.profile_tasks
stdout_callback = yaml
```

## Format stdout Output as YAML

Set the `yaml` callback as the stdout plugin for human-readable output:

```ini
[defaults]
stdout_callback = yaml
bin_ansible_callbacks = True
```

`stdout_callback = yaml` replaces the default dense output with readable YAML-formatted task results. `bin_ansible_callbacks = True` enables the callback for ad-hoc `ansible` commands in addition to `ansible-playbook`.

Before (default):

```
ok: [web1] => {"changed": false, "msg": "All assertions passed"}
```

After (yaml):

```
ok: [web1] =>
  msg: All assertions passed
  changed: false
```

## References

- [Ansible Configuration Settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
- [Sample Ansible Setup](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)
- [Callback Plugins](https://docs.ansible.com/ansible/latest/plugins/callback.html)
- [ansible.posix.timer](https://docs.ansible.com/ansible/latest/collections/ansible/posix/timer_callback.html)
- [ansible.posix.profile_tasks](https://docs.ansible.com/ansible/latest/collections/ansible/posix/profile_tasks_callback.html)
