# Variables

## Variable Precedence

Ansible applies variables in the following order from lowest to highest priority. When the same variable is defined at multiple levels, the highest-priority value wins.

| Priority | Source |
|----------|--------|
| 1 | command line values (for example, `-u my_user`; these are not variables) |
| 2 | role defaults (defined in `role/defaults/main.yml`) |
| 3 | inventory file or script group vars |
| 4 | inventory `group_vars/all` |
| 5 | playbook `group_vars/all` |
| 6 | inventory `group_vars/*` |
| 7 | playbook `group_vars/*` |
| 8 | inventory file or script host vars |
| 9 | inventory `host_vars/*` |
| 10 | playbook `host_vars/*` |
| 11 | host facts / cached `set_facts` |
| 12 | play vars |
| 13 | play `vars_prompt` |
| 14 | play `vars_files` |
| 15 | role vars (defined in `role/vars/main.yml`) |
| 16 | block vars (only for tasks in block) |
| 17 | task vars (only for the task) |
| 18 | `include_vars` |
| 19 | `set_facts` / registered vars |
| 20 | role (and `include_role`) params |
| 21 | `include` params |
| 22 | extra vars (`-e` on the command line) — always win |

> Extra vars (`-e`) always take the highest precedence. Use them for one-off overrides, not as a default mechanism.

## Define Variables

### In a Playbook

```yaml
---
- name: Configure application
  hosts: webservers
  vars:
    http_port: 80
    app_name: myapp
    max_connections: 100

  vars_files:
    - vars/common.yml
    - "vars/{{ ansible_os_family }}.yml"

  tasks:
    - name: Show variables
      ansible.builtin.debug:
        msg: "{{ app_name }} listens on port {{ http_port }}"
```

### In Inventory

```ini
# inventory/hosts
[webservers]
web01 ansible_host=10.0.0.1 http_port=8080
web02 ansible_host=10.0.0.2 http_port=8081

[webservers:vars]
app_env=production
log_level=warn
```

### In group_vars and host_vars

```
inventory/
  hosts
  group_vars/
    all.yml          # applies to all hosts
    webservers.yml   # applies to webservers group
  host_vars/
    web01.yml        # applies only to web01
```

```yaml
# inventory/group_vars/webservers.yml
---
http_port: 80
app_env: production
packages:
  - nginx
  - python3
```

```yaml
# inventory/host_vars/web01.yml
---
http_port: 8080
custom_setting: enabled
```

### On the CLI

```bash
# Single variable
ansible-playbook playbook.yml -e "app_version=2.1.0"

# Multiple variables
ansible-playbook playbook.yml -e "app_version=2.1.0 env=staging"

# From a file
ansible-playbook playbook.yml -e @vars/deploy.yml

# JSON format
ansible-playbook playbook.yml -e '{"app_version": "2.1.0", "debug": true}'
```

## Register Variables

Capture task output using the `register` keyword.

```yaml
---
- name: Capture command output
  hosts: webservers

  tasks:
    - name: Check disk space
      ansible.builtin.command: df -h /
      register: disk_result

    - name: Show disk output
      ansible.builtin.debug:
        var: disk_result.stdout_lines

    - name: Fail if disk is full
      ansible.builtin.fail:
        msg: "Disk usage is critical"
      when: "'100%' in disk_result.stdout"

    - name: Check if config exists
      ansible.builtin.stat:
        path: /etc/app/config.yml
      register: config_file

    - name: Create default config
      ansible.builtin.template:
        src: config.yml.j2
        dest: /etc/app/config.yml
      when: not config_file.stat.exists
```

A registered variable contains the full return data from the module, including:

| Attribute | Description |
|-----------|-------------|
| `rc` | Return code (command/shell modules) |
| `stdout` | Standard output as a string |
| `stdout_lines` | Standard output as a list of lines |
| `stderr` | Standard error as a string |
| `changed` | Whether the task made a change |
| `failed` | Whether the task failed |
| `skipped` | Whether the task was skipped |
| `results` | List of per-item results when used with a loop |

## Facts

### Gather Facts Automatically

```yaml
---
- name: Use gathered facts
  hosts: all
  gather_facts: true

  tasks:
    - name: Show OS information
      ansible.builtin.debug:
        msg: >
          {{ ansible_facts['distribution'] }}
          {{ ansible_facts['distribution_version'] }}
          on {{ ansible_facts['architecture'] }}

    - name: Install packages for Debian
      ansible.builtin.apt:
        name: nginx
        state: present
      when: ansible_facts['os_family'] == "Debian"
```

### Gather Facts Manually

```yaml
---
- name: Gather facts on demand
  hosts: all
  gather_facts: false

  tasks:
    - name: Collect only network facts
      ansible.builtin.setup:
        gather_subset:
          - network

    - name: Show IP address
      ansible.builtin.debug:
        var: ansible_facts['default_ipv4']['address']
```

### Custom Facts

Place `.fact` files in `/etc/ansible/facts.d/` on the managed node. They can be INI, JSON, or executable scripts that output JSON.

```ini
; /etc/ansible/facts.d/app.fact
[general]
app_name=myapp
app_version=2.1.0
```

```yaml
# Access custom facts
- name: Show custom fact
  ansible.builtin.debug:
    var: ansible_local.app.general.app_name
```

## Special Variables

| Variable | Description |
|----------|-------------|
| `inventory_hostname` | Name of the current host as defined in inventory |
| `inventory_hostname_short` | Short hostname (first part before the dot) |
| `groups` | Dictionary of all groups and their host lists |
| `hostvars` | Dictionary of all host variables, keyed by hostname |
| `ansible_play_hosts` | List of active hosts in the current play |
| `play_hosts` | Alias for `ansible_play_hosts` (deprecated) |
| `ansible_play_batch` | List of hosts in the current batch (with `serial`) |
| `group_names` | List of groups the current host belongs to |
| `ansible_check_mode` | `true` when running in check mode |
| `ansible_version` | Dictionary with Ansible version information |
| `role_path` | Path to the current role |

```yaml
---
- name: Use special variables
  hosts: webservers

  tasks:
    - name: Show current host
      ansible.builtin.debug:
        msg: "Running on {{ inventory_hostname }}"

    - name: Access another host's variable
      ansible.builtin.debug:
        msg: "DB host IP is {{ hostvars['db01']['ansible_host'] }}"

    - name: Show all hosts in the web group
      ansible.builtin.debug:
        var: groups['webservers']

    - name: Conditional on group membership
      ansible.builtin.debug:
        msg: "This host is a web server"
      when: "'webservers' in group_names"
```

## Variable Scoping

Ansible has three variable scopes:

### Play Scope

Variables defined at the play level (`vars`, `vars_files`, `vars_prompt`) are available to all tasks and roles within that play.

```yaml
---
- name: Play scope example
  hosts: all
  vars:
    shared_var: "available to all tasks in this play"

  tasks:
    - name: Access play-scoped variable
      ansible.builtin.debug:
        var: shared_var
```

### Host Scope

Variables tied to a specific host persist across plays within the same playbook run. These include inventory variables, facts, and `set_fact` values.

```yaml
---
- name: First play
  hosts: webservers
  tasks:
    - name: Set a host-scoped variable
      ansible.builtin.set_fact:
        my_host_var: "persists across plays"

- name: Second play
  hosts: webservers
  tasks:
    - name: Access variable from first play
      ansible.builtin.debug:
        var: my_host_var
```

### Task Scope

Variables defined in `vars` on a task or block are only available within that task or block.

```yaml
---
- name: Task scope example
  hosts: all

  tasks:
    - name: Task with local variables
      ansible.builtin.debug:
        msg: "{{ task_var }}"
      vars:
        task_var: "only available in this task"

    - name: This task cannot see task_var
      ansible.builtin.debug:
        msg: "task_var is not defined here"
```

## References

- [Using Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)
- [Variable Precedence](https://docs.ansible.com/ansible/latest/reference_appendices/general_precedence.html)
- [Special Variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)
- [Discovering Variables: Facts and Magic Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
