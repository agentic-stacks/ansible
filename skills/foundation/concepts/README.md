# Understand Ansible Concepts

This skill establishes the foundational mental model for how Ansible works. All terminology and definitions are drawn from the official Ansible documentation.

## Understand the Architecture

Ansible uses a **push-based, agentless** architecture:

1. The **control node** runs the Ansible CLI tools (`ansible-playbook`, `ansible`, `ansible-vault`, and others). Any computer that meets the software requirements can serve as a control node — laptops, shared desktops, servers, or containers known as Execution Environments.
2. The control node connects to **managed nodes** (also called "hosts") over **SSH** (Linux/Unix), **WinRM** (Windows), or other connection plugins.
3. Ansible copies the required **module** code to each managed node and executes it there. No agent or daemon runs on managed nodes.
4. After execution, Ansible removes the transferred code and returns results to the control node.

Multiple control nodes are possible, but Ansible itself does not coordinate across them. Use Ansible Automation Platform (AAP) for multi-controller orchestration.

## Learn Core Terminology

| Term | Definition |
|------|-----------|
| **Control node** | The machine from which you run the Ansible CLI tools. You can use any computer that meets the software requirements as a control node. |
| **Managed node** | Also referred to as "hosts" — the target devices (servers, network appliances, or any computer) you aim to manage with Ansible. Ansible is not normally installed on managed nodes. |
| **Inventory** | A list of managed nodes provided by one or more "inventory sources". Specifies information specific to each node (IP address, groups) and allows node selection in the Play and bulk variable assignment. |
| **Module** | The code or binaries that Ansible copies to and executes on each managed node (when needed) to accomplish the action defined in each Task. |
| **Task** | The definition of an "action" to be applied to the managed host. You can execute a single task once with an ad hoc command using `ansible` or `ansible-console`. |
| **Play** | The main context for Ansible execution. A playbook object that maps managed nodes (hosts) to tasks. Contains variables, roles, and an ordered list of tasks. Consists of an implicit loop over the mapped hosts and tasks. |
| **Playbook** | A YAML file containing one or more Plays (the basic unit of Ansible execution). This is both an "execution concept" and the file on which `ansible-playbook` operates. |
| **Role** | A limited distribution of reusable Ansible content (tasks, handlers, variables, plugins, templates, and files) for use inside of a Play. Must be imported into the Play to use its resources. |
| **Collection** | A format in which Ansible content is distributed that can contain playbooks, roles, modules, and plugins. Installed and used through Ansible Galaxy. |
| **Fact** | System properties discovered by Ansible on managed nodes during the `gather_facts` phase. Facts include network interfaces, OS, IP addresses, attached filesystems, and more. Accessible as variables (e.g., `ansible_os_family`, `ansible_hostname`). |
| **Plugin** | Pieces of code that expand Ansible's core capabilities. Plugins control how you connect to managed nodes (connection plugins), manipulate data (filter plugins), and control console display (callback plugins). |
| **Handler** | A special form of a Task that only executes when notified by a previous task which resulted in a "changed" status. |

## Trace the Execution Model

When you run `ansible-playbook site.yml`, Ansible executes the following sequence:

1. **Parse YAML** — Read and validate the playbook file(s).
2. **Load inventory** — Resolve all inventory sources into a list of hosts and groups.
3. **For each play in the playbook:**
   1. **Select hosts** — Evaluate the `hosts:` directive against the inventory to determine target nodes.
   2. **Load variables** — Merge variables from inventory, playbook, roles, and the command line following Ansible's variable precedence rules.
   3. **Gather facts** — Connect to each target node and run the `setup` module to collect system facts (unless `gather_facts: false`).
   4. **Execute tasks in order** — Run each task sequentially against all targeted hosts. By default, Ansible processes tasks across all hosts before moving to the next task (the "linear" strategy).
   5. **Run notified handlers** — At the end of each play (or when `meta: flush_handlers` is called), execute any handlers that were notified by tasks that reported "changed" status.
4. **Print the play recap** — Display per-host summary of ok, changed, unreachable, and failed counts.

### Understand strategies and serial execution

- **Linear strategy (default):** Ansible runs each task on all hosts in the batch before proceeding to the next task.
- **Free strategy:** Each host runs through tasks as fast as it can, independently of other hosts.
- **`serial` keyword:** Controls how many hosts run through the play at a time (e.g., `serial: 5` or `serial: "30%"`). Defaults to all hosts in the play.

## Know the Connection Types

| Connection Plugin | Target | Description |
|-------------------|--------|-------------|
| `ssh` | Linux/Unix | Default connection type. Uses OpenSSH. Preferred for all POSIX systems. |
| `paramiko` | Linux/Unix | Pure-Python SSH implementation. Fallback when OpenSSH is unavailable or for older SSH versions. |
| `winrm` | Windows | Connects to Windows hosts via Windows Remote Management. Requires `pywinrm` on the control node. |
| `local` | Control node | Executes on the control node itself. No SSH connection. Used for tasks that must run locally. |
| `network_cli` | Network devices | CLI-over-SSH connection for network platforms (Cisco IOS, Arista EOS, Juniper JUNOS, etc.). |
| `httpapi` | Network/API devices | REST API connection for network and other devices that expose an HTTP API. |
| `netconf` | Network devices | NETCONF-over-SSH connection for devices that support the NETCONF protocol (RFC 6241). |

Set the connection type per-play, per-host, or per-task:

```yaml
# Per-play
- hosts: webservers
  connection: ssh

# Per-host in inventory
[windows]
win01 ansible_connection=winrm

# Per-task
- name: Run locally
  ansible.builtin.debug:
    msg: "Running on control node"
  connection: local
```

## Apply Idempotency

**Idempotency** means that running an operation once produces the same system state as running it multiple times. Ansible modules are designed to be idempotent — if the desired state already exists, the module reports `ok` and makes no changes. If changes are needed, the module applies them and reports `changed`.

### Modules that are NOT idempotent by default

The `ansible.builtin.command`, `ansible.builtin.shell`, and `ansible.builtin.raw` modules always execute the given command and always report `changed`, because Ansible cannot know whether the command altered the system.

### Make non-idempotent modules idempotent

Use `creates`, `removes`, or `when` guards:

```yaml
# creates — skip if the file already exists
- name: Initialize the database
  ansible.builtin.command: /usr/bin/initdb /data
  args:
    creates: /data/initialized.flag

# removes — only run if the file exists
- name: Remove temp files
  ansible.builtin.command: /usr/bin/cleanup.sh
  args:
    removes: /tmp/work_in_progress

# when guard — check state before acting
- name: Install custom package
  ansible.builtin.command: /opt/install.sh
  when: custom_package_installed.rc != 0
```

## Distinguish ansible-core from the Ansible Community Package

| Aspect | `ansible-core` | `ansible` (community package) |
|--------|---------------|-------------------------------|
| **Contents** | The Ansible automation engine, built-in plugins (`ansible.builtin`), and CLI tools (`ansible-playbook`, `ansible-galaxy`, `ansible-vault`, etc.) | Depends on `ansible-core` and bundles a curated set of community Collections from Ansible Galaxy |
| **Install command** | `pip install ansible-core` | `pip install ansible` |
| **Collection count** | 1 (`ansible.builtin` only) | ~85+ community collections included |
| **Plugin examples** | `ansible.builtin.copy`, `ansible.builtin.file`, `ansible.builtin.yum` | `amazon.aws.*`, `community.general.*`, `community.mysql.*`, etc. |

### Version mapping

| `ansible` package | `ansible-core` version | Python (control node) |
|-------------------|------------------------|----------------------|
| 11.x | ansible-core 2.18 | Python 3.11+ |
| 10.x | ansible-core 2.17 | Python 3.10+ |
| 9.x | ansible-core 2.16 | Python 3.10+ |
| 8.x | ansible-core 2.15 | Python 3.9+ |

> **ansible-core 2.17+**: Minimum Python 3.10 on control node.

> **ansible-core 2.16**: Minimum Python 3.10 on control node.

---

## References

These official documentation pages were used to verify the content in this skill:

- [Ansible concepts](https://docs.ansible.com/ansible/latest/getting_started/basic_concepts.html)
- [Network — Basic Concepts](https://docs.ansible.com/ansible/latest/network/getting_started/basic_concepts.html)
- [Getting started with Ansible](https://docs.ansible.com/ansible/latest/getting_started/index.html)
- [Connection plugins](https://docs.ansible.com/ansible/latest/plugins/connection.html)
- [Ansible package release history](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html)
