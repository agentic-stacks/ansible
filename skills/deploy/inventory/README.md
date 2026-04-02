# Inventory

## Overview

Ansible inventory defines the hosts and groups of hosts that your automation targets. Every Ansible command needs an inventory to know *where* to run. Inventory can be as simple as a single text file listing hostnames or as sophisticated as a plugin that queries your cloud provider's API in real time.

Inventory determines:

- **What** hosts exist and how to reach them (IP, SSH port, credentials).
- **How** hosts are organized into groups for targeted execution.
- **Which** variables apply to individual hosts or entire groups.

Without a well-structured inventory, playbooks become brittle and hard to scale.

## Choose Your Inventory Type

| Type | Best For | Complexity | Maintenance |
|---|---|---|---|
| Static (INI) | Small fleets, quick prototyping | Low | Manual updates |
| Static (YAML) | Medium fleets, complex hierarchies | Low | Manual updates |
| Dynamic (plugins) | Cloud, auto-scaling, large fleets | Medium | Self-updating |
| Hybrid (static + dynamic) | Mixed environments (on-prem + cloud) | Medium-High | Partial manual |

**If hosts are known and stable** -- read [`static.md`](static.md).
**If hosts are discovered at runtime** -- read [`dynamic.md`](dynamic.md).

## Common Operations

### List all hosts and their variables

```bash
ansible-inventory --list -i inventory/
```

### Display the group tree

```bash
ansible-inventory --graph -i inventory/
```

Output example:

```
@all:
  |--@ungrouped:
  |--@webservers:
  |  |--web01.example.com
  |  |--web02.example.com
  |--@dbservers:
  |  |--db01.example.com
```

### Show variables for a specific host

```bash
ansible-inventory --host web01.example.com -i inventory/
```

### Ping all hosts in inventory

```bash
ansible -i inventory/ all -m ping
```

### Ping a specific group

```bash
ansible -i inventory/ webservers -m ping
```

### Use multiple inventory sources on the command line

```bash
ansible-playbook -i staging/ -i cloud_aws_ec2.yml site.yml
```

## Host and Group Variables

Ansible automatically loads variables from `group_vars/` and `host_vars/` directories relative to the inventory file or the playbook.

### Directory layout

```
project/
├── inventory/
│   ├── hosts.yml              # inventory file
│   ├── group_vars/
│   │   ├── all.yml            # variables for every host
│   │   ├── webservers.yml     # variables for webservers group
│   │   └── dbservers/
│   │       ├── vars.yml       # split into multiple files
│   │       └── vault.yml      # encrypted secrets
│   └── host_vars/
│       ├── web01.example.com.yml
│       └── db01.example.com.yml
└── playbooks/
    └── site.yml
```

### Variable precedence (inventory-related, low to high)

1. `group_vars/all`
2. `group_vars/<group>`
3. `host_vars/<host>`
4. Host variables defined directly in the inventory file

### Vault-encrypted variable files

```bash
# Create an encrypted group_vars file
ansible-vault create inventory/group_vars/dbservers/vault.yml

# Edit an existing encrypted file
ansible-vault edit inventory/group_vars/dbservers/vault.yml

# Run a playbook that uses vault-encrypted variables
ansible-playbook -i inventory/ site.yml --ask-vault-pass
```

## Host Patterns

Target subsets of your inventory with patterns:

| Pattern | Meaning |
|---|---|
| `all` or `*` | All hosts |
| `webservers` | All hosts in the webservers group |
| `web01.example.com` | A single host |
| `webservers:dbservers` | Union -- hosts in either group |
| `webservers:&staging` | Intersection -- hosts in both groups |
| `webservers:!phoenix` | Difference -- webservers but not phoenix |
| `~web\d+\.example\.com` | Regex match |
| `webservers[0]` | First host in the group |
| `webservers[0:2]` | First three hosts in the group |

Example:

```bash
# Run against webservers in staging but not those also in maintenance
ansible -i inventory/ 'webservers:&staging:!maintenance' -m ping
```

## References

- [Ansible Inventory Guide](https://docs.ansible.com/ansible/latest/inventory_guide/index.html)
- [Building Ansible Inventories](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
- [Dynamic Inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_dynamic_inventory.html)
- [Inventory Plugins List](https://docs.ansible.com/ansible/latest/plugins/inventory.html)
- [ansible-inventory CLI](https://docs.ansible.com/ansible/latest/cli/ansible-inventory.html)
- [Patterns: Targeting Hosts and Groups](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html)
