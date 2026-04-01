# Conditionals

## When

### Basic Conditions

Use `when` to run a task only if a condition is true. The `when` value is a raw Jinja2 expression (no `{{ }}` wrapping).

```yaml
---
- name: Conditional task examples
  hosts: all

  tasks:
    - name: Install Apache on Debian
      ansible.builtin.apt:
        name: apache2
        state: present
      when: ansible_facts['os_family'] == "Debian"

    - name: Install Apache on RedHat
      ansible.builtin.yum:
        name: httpd
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Run only on Ubuntu 22.04
      ansible.builtin.debug:
        msg: "This is Ubuntu 22.04"
      when:
        - ansible_facts['distribution'] == "Ubuntu"
        - ansible_facts['distribution_version'] == "22.04"
```

When you provide a list to `when`, all conditions must be true (implicit AND).

### Compound Conditions

```yaml
- name: Compound OR condition
  ansible.builtin.debug:
    msg: "This is a Debian-based system"
  when: ansible_facts['os_family'] == "Debian" or ansible_facts['os_family'] == "Ubuntu"

- name: Compound AND with OR
  ansible.builtin.debug:
    msg: "Production web server"
  when:
    - ansible_facts['os_family'] == "Debian"
    - app_env == "production" or app_env == "staging"

- name: Parenthesized logic
  ansible.builtin.debug:
    msg: "Complex condition met"
  when: (app_env == "production" and region == "us-east") or force_deploy | bool
```

### Check Variables and Facts

```yaml
- name: Run when variable is defined
  ansible.builtin.debug:
    msg: "custom_var is {{ custom_var }}"
  when: custom_var is defined

- name: Run when variable is not empty
  ansible.builtin.debug:
    msg: "deploy_branch is {{ deploy_branch }}"
  when: deploy_branch | length > 0

- name: Run when boolean variable is true
  ansible.builtin.debug:
    msg: "Feature is enabled"
  when: enable_feature | bool

- name: Check numeric value
  ansible.builtin.debug:
    msg: "Enough memory available"
  when: ansible_facts['memtotal_mb'] >= 2048

- name: Check string in list
  ansible.builtin.debug:
    msg: "This host is in the webservers group"
  when: "'webservers' in group_names"
```

### Check Registered Results

```yaml
- name: Check if service is running
  ansible.builtin.command: systemctl is-active nginx
  register: nginx_status
  ignore_errors: true

- name: Start nginx if not running
  ansible.builtin.systemd:
    name: nginx
    state: started
  when: nginx_status.rc != 0

- name: Run only if previous task changed
  ansible.builtin.debug:
    msg: "Config was updated"
  when: config_result is changed

- name: Run only if previous task succeeded
  ansible.builtin.debug:
    msg: "Previous task succeeded"
  when: previous_result is succeeded

- name: Run only if previous task was skipped
  ansible.builtin.debug:
    msg: "Previous task was skipped"
  when: previous_result is skipped

- name: Check command output content
  ansible.builtin.command: cat /etc/app/version
  register: version_output

- name: Act on version
  ansible.builtin.debug:
    msg: "Version 2 detected"
  when: "'2.' in version_output.stdout"
```

## Loops

### Basic Loop

```yaml
- name: Create multiple users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
    shell: /bin/bash
  loop:
    - alice
    - bob
    - charlie

- name: Install multiple packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql
    - redis
```

### Loop with Dictionaries

```yaml
- name: Create users with specific UIDs
  ansible.builtin.user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, uid: 1001, groups: "admin,dev" }
    - { name: bob, uid: 1002, groups: "dev" }

- name: Loop over a dictionary with dict2items
  ansible.builtin.debug:
    msg: "{{ item.key }} = {{ item.value }}"
  loop: "{{ app_config | dict2items }}"
  vars:
    app_config:
      port: 8080
      env: production
      debug: false
```

### Legacy with_items

```yaml
# with_items is legacy but still works
- name: Install packages (legacy syntax)
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  with_items:
    - nginx
    - git
    - curl
```

The `loop` keyword is the modern replacement for `with_*` constructs.

### Loop Control

```yaml
- name: Loop with custom label (cleaner output)
  ansible.builtin.user:
    name: "{{ item.name }}"
    state: present
  loop:
    - { name: alice, uid: 1001, groups: admin }
    - { name: bob, uid: 1002, groups: dev }
  loop_control:
    label: "{{ item.name }}"

- name: Loop with pause between iterations
  ansible.builtin.uri:
    url: "http://{{ item }}/health"
  loop: "{{ groups['webservers'] }}"
  loop_control:
    pause: 3

- name: Loop with index variable
  ansible.builtin.debug:
    msg: "{{ my_idx }}: {{ item }}"
  loop:
    - first
    - second
    - third
  loop_control:
    index_var: my_idx

- name: Loop with extended info
  ansible.builtin.debug:
    msg: >
      Item {{ ansible_loop.index }} of {{ ansible_loop.length }}:
      {{ item }}
      (first={{ ansible_loop.first }}, last={{ ansible_loop.last }})
  loop:
    - alpha
    - bravo
    - charlie
  loop_control:
    extended: true
```

### Loop with Conditionals

```yaml
- name: Install packages only on Debian
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql
  when: ansible_facts['os_family'] == "Debian"

- name: Register results in a loop
  ansible.builtin.command: "echo {{ item }}"
  loop:
    - one
    - two
    - three
  register: loop_results

- name: Show loop results
  ansible.builtin.debug:
    msg: "{{ item.stdout }}"
  loop: "{{ loop_results.results }}"
```

## Block / Rescue / Always

Group tasks with `block` for shared conditionals and error handling. The `rescue` section runs when any task in `block` fails. The `always` section runs regardless of success or failure.

```yaml
---
- name: Deploy with error handling
  hosts: webservers
  become: true

  tasks:
    - name: Deploy application
      block:
        - name: Pull latest code
          ansible.builtin.git:
            repo: https://github.com/example/app.git
            dest: /opt/app
            version: "{{ app_version }}"

        - name: Install dependencies
          ansible.builtin.command:
            cmd: pip install -r requirements.txt
            chdir: /opt/app

        - name: Run database migrations
          ansible.builtin.command:
            cmd: python manage.py migrate
            chdir: /opt/app

        - name: Restart application
          ansible.builtin.systemd:
            name: app
            state: restarted

      rescue:
        - name: Rollback to previous version
          ansible.builtin.git:
            repo: https://github.com/example/app.git
            dest: /opt/app
            version: "{{ previous_version }}"

        - name: Restart application with old version
          ansible.builtin.systemd:
            name: app
            state: restarted

        - name: Send failure notification
          ansible.builtin.uri:
            url: https://hooks.slack.com/services/XXX
            method: POST
            body_format: json
            body:
              text: "Deploy of {{ app_version }} failed on {{ inventory_hostname }}, rolled back to {{ previous_version }}"

      always:
        - name: Verify application is responding
          ansible.builtin.uri:
            url: "http://localhost:8080/health"
            status_code: 200
          retries: 5
          delay: 3

        - name: Log deployment result
          ansible.builtin.lineinfile:
            path: /var/log/deployments.log
            line: "{{ ansible_date_time.iso8601 }} - {{ app_version }} - {{ ansible_failed_task.name | default('success') }}"
            create: true
```

### Block with Shared Conditions

```yaml
- name: Debian-specific tasks
  block:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true

    - name: Install Debian packages
      ansible.builtin.apt:
        name:
          - nginx
          - certbot
        state: present
  when: ansible_facts['os_family'] == "Debian"
  become: true
```

## Tags

### Tag Tasks

```yaml
---
- name: Full deployment playbook
  hosts: webservers
  become: true

  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name: nginx
        state: present
      tags:
        - install
        - packages

    - name: Deploy configuration
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - config
        - deploy

    - name: Start service
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true
      tags: service

    - name: Debug information
      ansible.builtin.debug:
        msg: "Debug output here"
      tags: debug
```

### Run and Skip Tags

```bash
# Run only tasks tagged 'config'
ansible-playbook playbook.yml --tags config

# Run tasks tagged 'config' or 'deploy'
ansible-playbook playbook.yml --tags "config,deploy"

# Skip tasks tagged 'debug'
ansible-playbook playbook.yml --skip-tags debug

# List all available tags
ansible-playbook playbook.yml --list-tags
```

### Special Tags

| Tag | Behavior |
|-----|----------|
| `always` | Task always runs, even with `--tags` (unless explicitly skipped with `--skip-tags always`) |
| `never` | Task never runs unless explicitly requested with `--tags never` |
| `tagged` | With `--tags tagged`, runs all tasks that have any tag |
| `untagged` | With `--tags untagged`, runs only tasks with no tags |
| `all` | Runs all tasks (default behavior) |

```yaml
- name: Always run this health check
  ansible.builtin.uri:
    url: http://localhost/health
  tags: always

- name: Only run when explicitly requested
  ansible.builtin.debug:
    msg: "Verbose diagnostic output"
  tags: never
```

## Delegation

### delegate_to

Run a task on a different host than the current target.

```yaml
- name: Remove host from load balancer before deploy
  ansible.builtin.uri:
    url: "http://lb.example.com/api/pool/remove"
    method: POST
    body_format: json
    body:
      host: "{{ inventory_hostname }}"
  delegate_to: lb.example.com

- name: Deploy application
  ansible.builtin.git:
    repo: https://github.com/example/app.git
    dest: /opt/app
    version: "{{ app_version }}"

- name: Add host back to load balancer
  ansible.builtin.uri:
    url: "http://lb.example.com/api/pool/add"
    method: POST
    body_format: json
    body:
      host: "{{ inventory_hostname }}"
  delegate_to: lb.example.com
```

### local_action

Run a task on the Ansible controller.

```yaml
- name: Write deployment log locally
  local_action:
    module: ansible.builtin.copy
    content: "Deployed {{ app_version }} to {{ inventory_hostname }}\n"
    dest: "/tmp/deploy_{{ inventory_hostname }}.log"

# Equivalent using delegate_to
- name: Write deployment log locally (delegate_to form)
  ansible.builtin.copy:
    content: "Deployed {{ app_version }} to {{ inventory_hostname }}\n"
    dest: "/tmp/deploy_{{ inventory_hostname }}.log"
  delegate_to: localhost
```

### run_once

Run a task only once for the entire play, regardless of how many hosts are targeted.

```yaml
- name: Run database migration once
  ansible.builtin.command:
    cmd: python manage.py migrate
    chdir: /opt/app
  run_once: true

- name: Send deploy notification once
  ansible.builtin.uri:
    url: https://hooks.slack.com/services/XXX
    method: POST
    body_format: json
    body:
      text: "Deploy complete to {{ ansible_play_hosts | length }} hosts"
  run_once: true
  delegate_to: localhost
```

## References

- [Conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html)
- [Loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
- [Blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
- [Tags](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html)
- [Delegation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html)
