# Roles

## Understand the Role Directory Structure

A role is a standardized directory layout that groups related automation content. Ansible loads each subdirectory by convention.

```
my_role/
├── tasks/
│   └── main.yml          # Entry point — task list executed when the role is applied
├── handlers/
│   └── main.yml          # Handlers triggered by notify from tasks
├── defaults/
│   └── main.yml          # Default variables — lowest precedence, intended for users to override
├── vars/
│   └── main.yml          # Role variables — high precedence, internal to the role
├── files/
│   └── app.conf          # Static files deployed with copy or synchronize
├── templates/
│   └── app.conf.j2       # Jinja2 templates deployed with the template module
├── meta/
│   └── main.yml          # Role metadata — dependencies, Galaxy info, supported platforms
├── tests/
│   ├── inventory          # Test inventory
│   └── test.yml           # Test playbook
└── README.md              # Role documentation
```

| Directory | Purpose | Loaded Automatically |
|-----------|---------|---------------------|
| `tasks/` | Main list of tasks the role executes | Yes — `tasks/main.yml` |
| `handlers/` | Handlers available to the role and the calling playbook | Yes — `handlers/main.yml` |
| `defaults/` | Default variables, lowest precedence | Yes — `defaults/main.yml` |
| `vars/` | Role variables, high precedence | Yes — `vars/main.yml` |
| `files/` | Static files referenced by `copy`, `script`, `synchronize` | No — referenced by path |
| `templates/` | Jinja2 templates referenced by `template` | No — referenced by path |
| `meta/` | Dependencies, Galaxy metadata, `allow_duplicates` | Yes — `meta/main.yml` |
| `tests/` | Test playbook and inventory for CI | No — run manually |

Ansible searches for roles in these locations, in order:

1. `roles/` directory relative to the playbook
2. Paths listed in `roles_path` in `ansible.cfg`
3. `~/.ansible/roles`
4. `/usr/share/ansible/roles`
5. `/etc/ansible/roles`

## Choose Between Defaults and Vars

Ansible variable precedence determines which value wins when the same variable is defined in multiple places.

| File | Precedence | Purpose | Override Expectation |
|------|-----------|---------|---------------------|
| `defaults/main.yml` | Lowest (rank 2 of 22) | Provide sensible defaults for users | Users override with inventory, group_vars, host_vars, or `-e` |
| `vars/main.yml` | High (rank 18 of 22) | Internal role constants | Not intended to be overridden by users |

```yaml
# defaults/main.yml — user-facing, easy to override
---
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_access_log: /var/log/nginx/access.log
```

```yaml
# vars/main.yml — internal to the role, hard to override
---
nginx_package_name: nginx
nginx_config_dir: /etc/nginx
nginx_service_name: nginx
```

**Rule of thumb:** Put anything the user should customize in `defaults/`. Put internal constants that should not change across environments in `vars/`.

## Use a Role in a Playbook

### Apply with the `roles:` keyword

The simplest approach. Roles run after `pre_tasks` and before `tasks`.

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
    - common
    - role: nginx
      vars:
        nginx_port: 8080
      tags: nginx
    - role: app_deploy
      when: deploy_app | default(true)
```

### Import a role statically with `import_role`

Parsed at playbook load time. All tasks are visible to `--list-tasks`. Conditionals and loops apply to each task inside the role individually.

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Apply common configuration
      ansible.builtin.import_role:
        name: common

    - name: Apply nginx role
      ansible.builtin.import_role:
        name: nginx
      vars:
        nginx_port: 8080
```

### Include a role dynamically with `include_role`

Executed at runtime. Tasks are not visible to `--list-tasks` until the include runs. Conditionals and loops apply to the include statement as a whole.

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Apply role based on OS family
      ansible.builtin.include_role:
        name: "{{ ansible_os_family | lower }}_base"

    - name: Apply optional roles
      ansible.builtin.include_role:
        name: "{{ item }}"
      loop:
        - monitoring
        - logging
      when: enable_optional_roles | default(false)
```

### Compare import_role and include_role

| Feature | `import_role` (static) | `include_role` (dynamic) |
|---------|----------------------|------------------------|
| Parse time | Playbook load | Runtime |
| `--list-tasks` visibility | All tasks visible | Not visible until executed |
| `when` behavior | Applied to each task individually | Applied to the include as a whole |
| `loop` support | No | Yes |
| Variable names in role name | No | Yes |
| Handler notification | By name from anywhere | Only from within the include |
| Tags | Inherited by all tasks | Applied to include statement only |

**Use `import_role`** when you need predictable, static behavior and full task visibility. **Use `include_role`** when you need dynamic role selection, looping, or conditional inclusion of entire roles.

## Define Role Dependencies

Declare dependencies in `meta/main.yml`. Ansible executes dependencies before the role itself.

```yaml
# meta/main.yml
---
dependencies:
  - role: common
    vars:
      common_ntp_enabled: true

  - role: firewall
    vars:
      firewall_ports:
        - 80
        - 443

  - role: ssl_certificates
    when: use_ssl | default(true)
```

Ansible deduplicates dependencies by default. If role `A` and role `B` both depend on role `common`, it runs `common` only once.

### Allow duplicate execution

```yaml
# meta/main.yml
---
allow_duplicates: true

dependencies:
  - role: iptables_rule
    vars:
      rule_port: 80
  - role: iptables_rule
    vars:
      rule_port: 443
```

Set `allow_duplicates: true` when the role must run multiple times with different parameters.

### Conditional dependencies

Apply `when` to individual dependency entries. The condition is evaluated per host.

```yaml
# meta/main.yml
---
dependencies:
  - role: epel
    when: ansible_os_family == "RedHat"

  - role: apt_backports
    when: ansible_os_family == "Debian"
```

## Install Roles from Galaxy

### Install a single role

```bash
# Install from Galaxy
ansible-galaxy role install geerlingguy.docker

# Install to a specific path
ansible-galaxy role install geerlingguy.docker -p ./roles/

# Install a specific version
ansible-galaxy role install geerlingguy.docker,6.1.0
```

### Install from a requirements file

```yaml
# requirements.yml
---
roles:
  - name: geerlingguy.docker
    version: "6.1.0"

  - name: geerlingguy.nginx
    version: "3.2.0"

  - name: custom_role
    src: https://github.com/example/custom-role.git
    version: v2.0.0
    scm: git

  - name: private_role
    src: git@github.com:example/private-role.git
    version: main
    scm: git
```

```bash
# Install all roles from requirements.yml
ansible-galaxy role install -r requirements.yml

# Force reinstall
ansible-galaxy role install -r requirements.yml --force

# Install to a specific path
ansible-galaxy role install -r requirements.yml -p ./roles/
```

> **Warning** Always pin role versions in `requirements.yml`. Unpinned roles pull the latest version on every install, which can introduce breaking changes without warning.

### List and remove installed roles

```bash
# List installed roles
ansible-galaxy role list

# Remove a role
ansible-galaxy role remove geerlingguy.docker
```

## References

- [Using Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible Galaxy — Roles](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#installing-roles)
- [Variable Precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)
- [ansible-galaxy CLI](https://docs.ansible.com/ansible/latest/cli/ansible-galaxy.html)
