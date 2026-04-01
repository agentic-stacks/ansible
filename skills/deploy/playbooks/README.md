# Playbooks

## Anatomy of a Playbook

A playbook is a YAML file containing one or more plays. Each play maps a group of hosts to tasks.

```yaml
---
# playbook.yml — deploy and configure web application
- name: Configure web servers
  hosts: webservers
  become: true
  become_user: root
  gather_facts: true
  vars:
    http_port: 8080
    app_env: production
  vars_files:
    - vars/secrets.yml

  pre_tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

  roles:
    - role: common
      tags: common
    - role: nginx
      vars:
        nginx_port: "{{ http_port }}"

  tasks:
    - name: Deploy application code
      ansible.builtin.git:
        repo: https://github.com/example/app.git
        dest: /opt/app
        version: main
      notify: Restart application

    - name: Configure application
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/app/app.conf
        mode: "0644"
      notify: Restart application

  post_tasks:
    - name: Verify application is responding
      ansible.builtin.uri:
        url: "http://localhost:{{ http_port }}/health"
        status_code: 200
      retries: 5
      delay: 3

  handlers:
    - name: Restart application
      ansible.builtin.systemd:
        name: app
        state: restarted
        daemon_reload: true
```

## Play-Level Keywords

| Keyword | Description | Example |
|---------|-------------|---------|
| `hosts` | Target hosts or groups | `hosts: webservers` |
| `become` | Enable privilege escalation | `become: true` |
| `become_user` | User to escalate to | `become_user: root` |
| `vars` | Inline variable definitions | `vars: {port: 80}` |
| `vars_files` | Load variables from files | `vars_files: [vars/main.yml]` |
| `gather_facts` | Collect host facts before tasks | `gather_facts: true` |
| `serial` | Rolling update batch size | `serial: 3` or `serial: "30%"` |
| `strategy` | Execution strategy | `strategy: free` |
| `any_errors_fatal` | Abort entire play on any host failure | `any_errors_fatal: true` |
| `max_fail_percentage` | Acceptable failure threshold | `max_fail_percentage: 25` |
| `tags` | Tag the entire play | `tags: [deploy, web]` |
| `environment` | Set environment variables for all tasks | `environment: {ENV: prod}` |
| `connection` | Connection type | `connection: local` |
| `remote_user` | SSH user for the play | `remote_user: deploy` |
| `ignore_errors` | Continue on task failure | `ignore_errors: true` |
| `ignore_unreachable` | Continue when hosts are unreachable | `ignore_unreachable: true` |
| `module_defaults` | Default module parameters | See below |
| `collections` | Collections search path | `collections: [community.general]` |

```yaml
---
- name: Rolling deploy with safety controls
  hosts: webservers
  become: true
  serial: 2
  max_fail_percentage: 25
  any_errors_fatal: false
  strategy: linear
  environment:
    APP_ENV: production
    LOG_LEVEL: info
  module_defaults:
    ansible.builtin.apt:
      cache_valid_time: 3600
  tags:
    - deploy

  tasks:
    - name: Deploy application
      ansible.builtin.copy:
        src: app.tar.gz
        dest: /opt/app/app.tar.gz
```

## Run a Playbook

```bash
# Basic execution
ansible-playbook playbook.yml

# Specify inventory
ansible-playbook -i inventory/production playbook.yml

# Limit to specific hosts or groups
ansible-playbook playbook.yml -l webserver01
ansible-playbook playbook.yml -l 'webservers:&staging'

# Run specific tags
ansible-playbook playbook.yml -t deploy,config
ansible-playbook playbook.yml --skip-tags debug

# Check mode — dry run without making changes
ansible-playbook playbook.yml --check --diff

# Pass extra variables
ansible-playbook playbook.yml -e "app_version=2.1.0 env=staging"
ansible-playbook playbook.yml -e @vars/override.yml

# List tasks or hosts without executing
ansible-playbook playbook.yml --list-tasks
ansible-playbook playbook.yml --list-hosts

# Syntax check only
ansible-playbook playbook.yml --syntax-check

# Verbosity levels
ansible-playbook playbook.yml -v      # verbose
ansible-playbook playbook.yml -vv     # more verbose
ansible-playbook playbook.yml -vvv    # connection debugging
ansible-playbook playbook.yml -vvvv   # full connection + module debugging

# Start at a specific task
ansible-playbook playbook.yml --start-at-task "Deploy application"

# Step through tasks one at a time
ansible-playbook playbook.yml --step

# Forks — parallel host execution
ansible-playbook playbook.yml -f 20
```

## Execution Order

Within each play, Ansible executes phases in this fixed order:

1. **pre_tasks** -- Tasks that run before roles
2. **handlers** (notified by pre_tasks) -- Flushed automatically after pre_tasks
3. **roles** -- Applied in listed order
4. **tasks** -- Main task list
5. **handlers** (notified by roles and tasks) -- Flushed automatically after tasks
6. **post_tasks** -- Tasks that run after the main tasks and handlers
7. **handlers** (notified by post_tasks) -- Flushed automatically after post_tasks

```yaml
---
- name: Demonstrate execution order
  hosts: all

  pre_tasks:
    - name: 1 — Runs first
      ansible.builtin.debug:
        msg: "pre_tasks phase"

  roles:
    - role: common    # 2 — Runs after pre_tasks handlers

  tasks:
    - name: 3 — Runs after roles
      ansible.builtin.debug:
        msg: "tasks phase"

  post_tasks:
    - name: 4 — Runs after tasks handlers
      ansible.builtin.debug:
        msg: "post_tasks phase"
```

> **Warning** Always use `--check --diff` before applying to production.

## References

- [Ansible Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
- [Playbook Keywords](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html)
- [ansible-playbook CLI](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html)
