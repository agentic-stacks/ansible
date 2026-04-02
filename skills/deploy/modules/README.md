# Modules

## What Is a Module

A **module** is the smallest unit of work in Ansible. Each module performs a single, well-defined task such as copying a file, installing a package, or restarting a service.

Key characteristics:

- **Runs on the managed node** — Ansible transfers the module to the remote host, executes it there, then removes it. Some modules (like `debug` or `set_fact`) run on the controller instead.
- **Returns JSON** — Every module returns a structured JSON result containing at minimum `changed` and optionally `failed`, `msg`, and other keys.
- **Should be idempotent** — Running the same module twice with the same parameters should produce the same end state without unnecessary changes. Modules report `changed: false` when the system already matches the desired state.
- **Accepts parameters** — Modules receive arguments through the task's key-value pairs (or `args` dictionary) and validate them against an internal argument specification.

Modules are referenced by their **Fully Qualified Collection Name** (FQCN), for example `ansible.builtin.copy`. Short names like `copy` still work when the `ansible.builtin` collection is in scope, but FQCNs are recommended for clarity and to avoid collisions with third-party collections.

---

## Essential Builtin Modules

| Module (FQCN) | Purpose | Idempotent |
|---|---|---|
| `ansible.builtin.copy` | Copy files from controller to managed nodes | Yes |
| `ansible.builtin.template` | Render Jinja2 templates and deploy to managed nodes | Yes |
| `ansible.builtin.file` | Manage file and directory properties (create, delete, permissions, symlinks) | Yes |
| `ansible.builtin.service` | Manage services (start, stop, restart, enable) — generic | Yes |
| `ansible.builtin.systemd` | Manage systemd units (start, stop, restart, enable, daemon-reload) | Yes |
| `ansible.builtin.apt` | Manage packages on Debian/Ubuntu with APT | Yes |
| `ansible.builtin.dnf` | Manage packages on Fedora/RHEL with DNF | Yes |
| `ansible.builtin.pip` | Manage Python packages with pip | Yes |
| `ansible.builtin.command` | Execute a command on the managed node (no shell processing) | No* |
| `ansible.builtin.shell` | Execute a command through the shell (`/bin/sh`) on the managed node | No* |
| `ansible.builtin.raw` | Execute a low-level command over SSH without Python on the managed node | No* |
| `ansible.builtin.debug` | Print statements during playbook execution (runs on controller) | Yes |
| `ansible.builtin.assert` | Fail the play when conditions are not met | Yes |
| `ansible.builtin.uri` | Interact with HTTP/HTTPS endpoints (GET, POST, PUT, etc.) | Yes |
| `ansible.builtin.lineinfile` | Ensure a particular line exists (or is absent) in a file | Yes |
| `ansible.builtin.blockinfile` | Manage a marked block of multi-line text in a file | Yes |
| `ansible.builtin.user` | Manage user accounts and attributes | Yes |
| `ansible.builtin.group` | Manage groups on the managed node | Yes |
| `ansible.builtin.cron` | Manage cron jobs | Yes |
| `ansible.builtin.git` | Clone and update git repositories | Yes |
| `ansible.builtin.unarchive` | Unpack compressed archives on the managed node | Yes |
| `ansible.builtin.stat` | Retrieve file or filesystem status | Yes |
| `ansible.builtin.wait_for` | Wait for a condition before continuing (port open, file exists, etc.) | Yes |
| `ansible.builtin.set_fact` | Set host-level variables during play execution (runs on controller) | Yes |
| `ansible.builtin.include_tasks` | Dynamically include a task file at runtime | Yes |
| `ansible.builtin.import_tasks` | Statically import a task file at parse time | Yes |

> **\*Note:** `command`, `shell`, and `raw` are **not inherently idempotent** because Ansible cannot know whether the command has already achieved the desired state. Use the `creates` and `removes` parameters to make them conditionally idempotent:
>
> ```yaml
> - name: Build the application (skip if binary exists)
>   ansible.builtin.command:
>     cmd: make build
>     chdir: /opt/app
>     creates: /opt/app/bin/myapp
> ```

---

## Module Return Values

Every module returns a JSON dictionary. The most common keys are:

| Key | Type | Description |
|---|---|---|
| `changed` | bool | Whether the module made any changes to the managed node |
| `failed` | bool | Whether the module encountered a fatal error |
| `msg` | string | A human-readable message describing the result |
| `stdout` | string | Standard output from commands (command/shell/raw modules) |
| `stderr` | string | Standard error from commands |
| `rc` | int | Return code from commands (0 = success) |

### Capturing Return Values with `register`

Use `register` to save a module's return values into a variable for later inspection or conditional logic:

```yaml
- name: Check if config file exists
  ansible.builtin.stat:
    path: /etc/myapp/config.yml
  register: config_stat

- name: Create default config if missing
  ansible.builtin.template:
    src: config.yml.j2
    dest: /etc/myapp/config.yml
  when: not config_stat.stat.exists

- name: Run database migration
  ansible.builtin.command:
    cmd: /opt/app/bin/migrate --apply
  register: migration_result
  changed_when: "'Applied' in migration_result.stdout"
  failed_when: migration_result.rc != 0

- name: Show migration output
  ansible.builtin.debug:
    msg: "{{ migration_result.stdout_lines }}"
```

The `changed_when` and `failed_when` directives let you override the default `changed` and `failed` logic, which is especially useful for `command` and `shell` modules that always report `changed: true` by default.

---

## Ad-Hoc Module Execution

You can run modules directly from the command line without writing a playbook using `ansible` with the `-m` (module) and `-a` (arguments) flags:

```bash
# Ping all hosts to verify connectivity
ansible all -m ping

# Run a shell command on the webservers group
ansible webservers -m shell -a "uptime"

# Gather a specific fact from all hosts
ansible all -m setup -a "filter=ansible_os_family"

# Copy a file to all database servers
ansible dbservers -m copy -a "src=/tmp/my.cnf dest=/etc/mysql/my.cnf owner=mysql mode=0644"

# Install a package on Debian hosts
ansible debian -m apt -a "name=nginx state=present" --become

# Restart a service
ansible webservers -m service -a "name=nginx state=restarted" --become
```

Ad-hoc commands are useful for quick one-off tasks, troubleshooting, and verifying connectivity. For anything repeatable, use a playbook instead.

---

## Custom Modules

When builtin and community modules do not cover your use case, you can write a custom module in Python.

### Basic Python Module Structure

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from ansible.module_utils.basic import AnsibleModule


def run_module():
    # Define the arguments this module accepts
    argument_spec = dict(
        name=dict(type='str', required=True),
        state=dict(type='str', default='present', choices=['present', 'absent']),
        force=dict(type='bool', default=False),
    )

    # Create the module object — this parses arguments and handles
    # check mode, no-log, and other standard Ansible features
    module = AnsibleModule(
        argument_spec=argument_spec,
        supports_check_mode=True,
    )

    name = module.params['name']
    state = module.params['state']
    result = dict(changed=False, msg='')

    # --- Your logic here ---
    # Determine current state, compare with desired state,
    # make changes if needed, set result['changed'] = True

    if module.check_mode:
        # In check mode, report what would change but do nothing
        module.exit_json(**result)

    # Perform the actual work...
    try:
        # Example: create or remove a resource
        if state == 'present':
            result['changed'] = True
            result['msg'] = f"Resource '{name}' created"
        else:
            result['changed'] = True
            result['msg'] = f"Resource '{name}' removed"
    except Exception as e:
        module.fail_json(msg=f"Failed to manage resource: {e}", **result)

    module.exit_json(**result)


def main():
    run_module()


if __name__ == '__main__':
    main()
```

### Key Points

- **`AnsibleModule`** handles argument parsing, check mode, logging, and JSON output formatting.
- **`argument_spec`** defines parameter names, types, defaults, choices, and whether they are required.
- **`module.exit_json(**result)`** signals success and returns data to Ansible.
- **`module.fail_json(msg="...")`** signals failure and halts the play for that host.
- Always set `changed` accurately so Ansible can report correct change counts.
- Support `check_mode` so users can do dry runs with `--check`.

### Where to Place Custom Modules

| Location | Scope | Notes |
|---|---|---|
| `library/` in the project root | All playbooks in the project | Simplest option for project-specific modules |
| `roles/<role>/library/` | Tasks within that role only | Keeps the module scoped to the role |
| Collection (`plugins/modules/`) | Anyone who installs the collection | Best for reusable, shareable modules |
| Path in `ANSIBLE_LIBRARY` env var | Global | Useful for site-wide custom modules |

Once placed in any of these locations, the module is available by name in tasks:

```yaml
- name: Manage custom resource
  my_custom_module:
    name: example
    state: present
```
