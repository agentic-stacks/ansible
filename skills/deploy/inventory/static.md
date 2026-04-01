# Static Inventory

## Overview

A static inventory is a file you maintain by hand. It lists every host, organizes hosts into groups, and optionally assigns variables. Use static inventory when your infrastructure is known, stable, and changes infrequently.

Ansible supports two static inventory formats: **INI** and **YAML**. Both are fully equivalent in capability. Choose whichever your team prefers.

## INI Format

The INI format is the original Ansible inventory format. It is compact and works well for simple topologies.

### Full example

```ini
# inventory/hosts.ini

# Ungrouped hosts
bastion01.example.com

[webservers]
web01.example.com ansible_host=10.0.1.11 ansible_port=22
web02.example.com ansible_host=10.0.1.12 ansible_port=22
web03.example.com ansible_host=10.0.1.13 http_port=8080

[appservers]
app01.example.com ansible_host=10.0.2.21
app02.example.com ansible_host=10.0.2.22

[dbservers]
db01.example.com ansible_host=10.0.3.31 ansible_user=dbadmin
db02.example.com ansible_host=10.0.3.32 ansible_user=dbadmin

[loadbalancers]
lb01.example.com ansible_host=10.0.0.5

# Group of groups
[frontend:children]
webservers
loadbalancers

[backend:children]
appservers
dbservers

# Variables applied to all webservers
[webservers:vars]
http_port=80
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_ed25519

# Variables applied to all dbservers
[dbservers:vars]
db_port=5432
ansible_ssh_private_key_file=~/.ssh/db_ed25519

# Variables applied to everything
[all:vars]
ansible_python_interpreter=/usr/bin/python3
ntp_server=ntp.example.com
```

### INI syntax rules

- Group names go in `[brackets]`.
- Child groups use `[parent:children]`.
- Group variables use `[group:vars]`.
- Host-level variables appear on the same line as the hostname, space-separated.
- Comments start with `#`.
- The implicit groups `all` and `ungrouped` always exist.

## YAML Format

The YAML format supports the same topology as INI but handles complex data structures (lists, nested dicts) more naturally.

### Full example (same topology as INI above)

```yaml
# inventory/hosts.yml
all:
  hosts:
    bastion01.example.com:
  vars:
    ansible_python_interpreter: /usr/bin/python3
    ntp_server: ntp.example.com
  children:
    frontend:
      children:
        webservers:
          hosts:
            web01.example.com:
              ansible_host: 10.0.1.11
              ansible_port: 22
            web02.example.com:
              ansible_host: 10.0.1.12
              ansible_port: 22
            web03.example.com:
              ansible_host: 10.0.1.13
              http_port: 8080
          vars:
            http_port: 80
            ansible_user: deploy
            ansible_ssh_private_key_file: ~/.ssh/deploy_ed25519
        loadbalancers:
          hosts:
            lb01.example.com:
              ansible_host: 10.0.0.5
    backend:
      children:
        appservers:
          hosts:
            app01.example.com:
              ansible_host: 10.0.2.21
            app02.example.com:
              ansible_host: 10.0.2.22
        dbservers:
          hosts:
            db01.example.com:
              ansible_host: 10.0.3.31
              ansible_user: dbadmin
            db02.example.com:
              ansible_host: 10.0.3.32
              ansible_user: dbadmin
          vars:
            db_port: 5432
            ansible_ssh_private_key_file: ~/.ssh/db_ed25519
```

## Compare INI and YAML

| Feature | INI | YAML |
|---|---|---|
| Readability for simple inventories | Better -- compact | More verbose |
| Complex variable values (lists, dicts) | Not supported inline | Fully supported |
| Nesting depth | Flat (groups + children) | Unlimited |
| Familiarity | Common with sysadmins | Common with DevOps/IaC |
| Recommended for new projects | Small fleets only | Yes |

## Groups and Children

Groups let you target sets of hosts. Children let you build hierarchies.

```
all
├── ungrouped
│   └── bastion01.example.com
├── frontend
│   ├── webservers
│   │   ├── web01.example.com
│   │   ├── web02.example.com
│   │   └── web03.example.com
│   └── loadbalancers
│       └── lb01.example.com
└── backend
    ├── appservers
    │   ├── app01.example.com
    │   └── app02.example.com
    └── dbservers
        ├── db01.example.com
        └── db02.example.com
```

A host can belong to multiple groups. Variables from all groups merge, with more-specific groups winning.

## Host Ranges

Avoid repetition by using numeric or alphabetic ranges.

### Numeric ranges

```ini
[webservers]
web[01:50].example.com
```

This expands to `web01.example.com`, `web02.example.com`, ..., `web50.example.com`.

### Alphabetic ranges

```ini
[databases]
db-[a:f].example.com
```

This expands to `db-a.example.com`, `db-b.example.com`, ..., `db-f.example.com`.

### Ranges with stride (YAML format)

```yaml
# No native stride in ranges -- use a pattern with explicit hosts or
# generate inventory with a script.
```

### Ranges in YAML

```yaml
webservers:
  hosts:
    web[01:50].example.com:
```

## Per-Host Connection Variables

These built-in variables control how Ansible connects to each host.

| Variable | Purpose | Example |
|---|---|---|
| `ansible_host` | IP or hostname to connect to | `10.0.1.11` |
| `ansible_port` | SSH port | `2222` |
| `ansible_user` | SSH username | `deploy` |
| `ansible_ssh_private_key_file` | Path to SSH private key | `~/.ssh/deploy_ed25519` |
| `ansible_connection` | Connection type | `ssh`, `local`, `winrm` |
| `ansible_become` | Enable privilege escalation | `true` |
| `ansible_become_method` | Escalation method | `sudo`, `su`, `doas` |
| `ansible_become_user` | Target escalation user | `root` |
| `ansible_python_interpreter` | Path to Python on remote host | `/usr/bin/python3` |
| `ansible_shell_type` | Shell type on remote host | `sh`, `csh`, `powershell` |

### Example with connection variables

```ini
[webservers]
web01.example.com ansible_host=10.0.1.11 ansible_port=2222 ansible_user=deploy ansible_ssh_private_key_file=~/.ssh/deploy_ed25519
web02.example.com ansible_host=10.0.1.12 ansible_user=deploy ansible_connection=ssh
```

## Multiple Inventory Files (Directory-Based Inventory)

Instead of a single file, point Ansible at a directory. Ansible loads every file in the directory in ASCII sort order and merges the results.

### Directory layout

```
inventory/
├── 01-static-hosts.yml       # loaded first (ASCII order)
├── 02-cloud-aws_ec2.yml      # dynamic plugin, loaded second
├── 03-cloud-gcp_compute.yml  # dynamic plugin, loaded third
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   └── dbservers.yml
└── host_vars/
    └── web01.example.com.yml
```

### Merge behavior

1. Files are loaded in **ASCII filename order** (hence the `01-`, `02-` prefix convention).
2. If the same host appears in multiple files, variables **merge**. Later files override earlier ones for conflicting keys.
3. If the same group appears in multiple files, hosts are **unioned** into the group.
4. `group_vars/` and `host_vars/` directories are loaded relative to the inventory directory.

### Use directory-based inventory

```bash
# Point at the directory, not a specific file
ansible-playbook -i inventory/ site.yml

# Or set it in ansible.cfg
# [defaults]
# inventory = ./inventory/
```

### ansible.cfg example

```ini
# ansible.cfg
[defaults]
inventory = ./inventory/
remote_user = deploy
private_key_file = ~/.ssh/deploy_ed25519

[privilege_escalation]
become = true
become_method = sudo
become_user = root
```

## Verify Your Inventory

```bash
# List all hosts and variables as JSON
ansible-inventory -i inventory/ --list

# Show the group tree
ansible-inventory -i inventory/ --graph

# Show variables for a single host
ansible-inventory -i inventory/ --host web01.example.com

# Validate by pinging all hosts
ansible -i inventory/ all -m ping

# Ping only a specific group
ansible -i inventory/ webservers -m ping
```

## References

- [Building Ansible Inventories](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
- [Inventory Basics: Formats, Hosts, and Groups](https://docs.ansible.com/ansible/latest/getting_started/get_started_inventory.html)
- [Connecting to Hosts: Behavioral Inventory Parameters](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#connecting-to-hosts-behavioral-inventory-parameters)
- [ansible-inventory CLI Reference](https://docs.ansible.com/ansible/latest/cli/ansible-inventory.html)
- [Variable Precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)
