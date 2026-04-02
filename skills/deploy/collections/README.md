# Collections

## What Is a Collection

A collection is the standard distribution unit for Ansible content. Each collection bundles modules, plugins, roles, and playbooks under a single namespace using the `namespace.name` format.

Collections are distributed through [Ansible Galaxy](https://galaxy.ansible.com/) or [Automation Hub](https://console.redhat.com/ansible/automation-hub). The `ansible` package itself is a set of collections maintained by the community.

Examples of the namespace format:

- `community.general` — community-maintained general-purpose modules
- `amazon.aws` — Amazon Web Services modules
- `ansible.builtin` — modules and plugins shipped with ansible-core

## Install Collections

### Install a single collection

```bash
# Install the latest version
ansible-galaxy collection install community.general

# Install a specific version
ansible-galaxy collection install community.general:==9.0.0

# Install to a project-local directory
ansible-galaxy collection install community.general -p ./collections/
```

### Install from a requirements file

Define all collection dependencies in `requirements.yml` and pin versions for reproducibility.

```yaml
# requirements.yml
---
collections:
  - name: community.general
    version: ">=9.0.0,<10.0.0"

  - name: ansible.posix
    version: "2.0.0"

  - name: amazon.aws
    version: "8.0.0"

  - name: community.docker
    version: "4.0.0"
```

```bash
# Install all collections from requirements.yml
ansible-galaxy collection install -r requirements.yml

# Force reinstall
ansible-galaxy collection install -r requirements.yml --force

# Install to a specific path
ansible-galaxy collection install -r requirements.yml -p ./collections/
```

> **Warning** Always pin collection versions in `requirements.yml`. Unpinned collections pull the latest version on every install, which can introduce breaking changes without warning.

### List and remove installed collections

```bash
# List installed collections
ansible-galaxy collection list

# Remove a collection
ansible-galaxy collection install community.general --force  # reinstall to update
```

## Use Collection Content

### Fully Qualified Collection Name (FQCN)

The recommended way to reference collection content is the Fully Qualified Collection Name. The FQCN format is `namespace.collection.module_or_plugin`.

```yaml
---
- name: Configure servers
  hosts: all
  become: true

  tasks:
    - name: Set the timezone
      community.general.timezone:
        name: America/Chicago

    - name: Install packages
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Ensure SELinux is enforcing
      ansible.posix.selinux:
        state: enforcing
        policy: targeted
```

### Short names with the `collections` keyword

You can avoid repeating the namespace by declaring collections at the play level. However, FQCN is recommended because it is explicit, avoids ambiguity, and works reliably with `ansible-lint`.

```yaml
---
- name: Configure servers
  hosts: all
  become: true
  collections:
    - community.general
    - ansible.posix

  tasks:
    - name: Set the timezone
      timezone:
        name: America/Chicago

    - name: Ensure SELinux is enforcing
      selinux:
        state: enforcing
        policy: targeted
```

**Use FQCN.** It eliminates name collisions between collections and makes playbooks easier to audit.

## Collection Search Path

Ansible searches for collections in the paths defined by the `COLLECTIONS_PATHS` configuration, in order:

1. Project-local `./collections/` directory (relative to the playbook)
2. `~/.ansible/collections`
3. `/usr/share/ansible/collections`

Override the search path in `ansible.cfg`:

```ini
# ansible.cfg
[defaults]
collections_path = ./collections:~/.ansible/collections:/usr/share/ansible/collections
```

Or set the environment variable:

```bash
export ANSIBLE_COLLECTIONS_PATH=./collections:~/.ansible/collections
```

For project portability, install collections into a project-local `collections/` directory and commit the `requirements.yml` (but not the installed collections themselves) to version control.

## Popular Collections

| Collection | Description |
|-----------|-------------|
| `community.general` | General-purpose modules not tied to a specific vendor — archive, cron, timezone, ini_file, and hundreds more |
| `ansible.posix` | POSIX-oriented modules — authorized_key, mount, selinux, sysctl, acl |
| `amazon.aws` | Amazon Web Services — EC2, S3, RDS, IAM, Lambda, CloudFormation |
| `azure.azcollection` | Microsoft Azure — virtual machines, resource groups, networking, storage |
| `google.cloud` | Google Cloud Platform — compute, storage, networking, IAM |
| `cisco.ios` | Cisco IOS network device management — config, interfaces, OSPF, ACLs |
| `ansible.netcommon` | Network-agnostic modules and connection plugins — cli_command, cli_config, network_cli |

Browse the full index at [Ansible Galaxy](https://galaxy.ansible.com/) or the [Collection Index](https://docs.ansible.com/ansible/latest/collections/index.html).

## References

- [Using Ansible Collections](https://docs.ansible.com/ansible/latest/collections_guide/index.html)
- [Installing Collections](https://docs.ansible.com/ansible/latest/collections_guide/collections_installing.html)
- [Using Collections in a Playbook](https://docs.ansible.com/ansible/latest/collections_guide/collections_using_playbooks.html)
- [ansible-galaxy CLI](https://docs.ansible.com/ansible/latest/cli/ansible-galaxy.html)
