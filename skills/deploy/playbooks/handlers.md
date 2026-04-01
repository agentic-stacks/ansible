# Handlers

## Notify and Handle

Handlers are tasks that run only when notified by another task via the `notify` keyword. They execute once at the end of a play, regardless of how many tasks notify them.

```yaml
---
- name: Configure and restart services
  hosts: webservers
  become: true

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Deploy nginx configuration
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: "0644"
      notify: Restart nginx

    - name: Deploy site configuration
      ansible.builtin.template:
        src: site.conf.j2
        dest: /etc/nginx/sites-available/default
        mode: "0644"
      notify:
        - Validate nginx config
        - Restart nginx

  handlers:
    - name: Validate nginx config
      ansible.builtin.command: nginx -t

    - name: Restart nginx
      ansible.builtin.systemd:
        name: nginx
        state: restarted
```

A handler runs only when:

- A task that notifies it reports `changed` status.
- The handler has not already been triggered in this handler flush.

A handler runs at most once per handler flush, even if multiple tasks notify it.

## Listen

The `listen` directive lets multiple handlers subscribe to a single notification topic. This decouples the notifying task from specific handler names.

```yaml
---
- name: Deploy application with listen-based handlers
  hosts: appservers
  become: true

  tasks:
    - name: Deploy application code
      ansible.builtin.copy:
        src: app/
        dest: /opt/app/
      notify: Restart web stack

  handlers:
    - name: Restart nginx
      ansible.builtin.systemd:
        name: nginx
        state: restarted
      listen: Restart web stack

    - name: Restart app service
      ansible.builtin.systemd:
        name: app
        state: restarted
      listen: Restart web stack

    - name: Clear application cache
      ansible.builtin.file:
        path: /opt/app/cache
        state: absent
      listen: Restart web stack
```

When the task notifies `Restart web stack`, all three handlers with `listen: Restart web stack` execute in the order they are defined.

## Handler Ordering

Handlers run in the order they are defined in the `handlers` section, not in the order they are notified.

```yaml
---
- name: Handler ordering demonstration
  hosts: all

  tasks:
    - name: Task that notifies in reverse order
      ansible.builtin.debug:
        msg: "Triggering handlers"
      changed_when: true
      notify:
        - Handler C
        - Handler A
        - Handler B

  handlers:
    # Handlers run in definition order: A, B, C
    - name: Handler A
      ansible.builtin.debug:
        msg: "Handler A runs first (defined first)"

    - name: Handler B
      ansible.builtin.debug:
        msg: "Handler B runs second"

    - name: Handler C
      ansible.builtin.debug:
        msg: "Handler C runs last (defined last)"
```

## Flush Handlers

By default, handlers run after all tasks in a play complete. Use `meta: flush_handlers` to run pending handlers immediately at a specific point.

```yaml
---
- name: Flush handlers mid-play
  hosts: webservers
  become: true

  tasks:
    - name: Install and configure nginx
      ansible.builtin.apt:
        name: nginx
        state: present
      notify: Start nginx

    # Force the handler to run now — nginx must be running
    # before the next task can verify it
    - name: Flush handlers
      ansible.builtin.meta: flush_handlers

    - name: Verify nginx is responding
      ansible.builtin.uri:
        url: http://localhost/
        status_code: 200
      retries: 3
      delay: 2

    - name: Deploy application
      ansible.builtin.copy:
        src: app/
        dest: /var/www/html/
      notify: Reload nginx

  handlers:
    - name: Start nginx
      ansible.builtin.systemd:
        name: nginx
        state: started

    - name: Reload nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded
```

Flushing handlers is essential when a later task depends on the state change a handler provides (for example, starting a service before testing connectivity).

## Handler Gotchas

### Handlers Only Run on Changed

A handler only fires when the notifying task reports `changed`. If the task reports `ok` (no change needed), the handler is not notified.

```yaml
# This handler will NOT fire if the package is already installed
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
  notify: Start nginx
```

To force a handler to run regardless, use `changed_when: true` on the notifying task (use sparingly -- this defeats idempotency signals).

### Handlers and Check Mode

In `--check` mode, tasks report what would change but do not make changes. Because no actual changes occur, handlers are not notified and will not run. This means `--check` mode does not fully simulate handler-dependent workflows.

### Handlers in Included Files

Handlers can be defined in separate files using `handlers:` with `include_tasks` or in role handler directories.

```yaml
# handlers/main.yml
---
- name: Restart nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
```

Static handler includes (`import_tasks`) make handlers available at parse time. Dynamic handler includes (`include_tasks`) make handlers available only after the include task runs. Prefer static includes for handler files.

### Handlers and Failed Tasks

If a task fails and the play aborts, pending handlers do not run. To run handlers even on failure, use `--force-handlers` on the command line or set `force_handlers = True` in `ansible.cfg`.

```bash
ansible-playbook playbook.yml --force-handlers
```

### Handlers in Blocks

Handlers notified within a `block` that enters `rescue` are not flushed. Only handlers notified by tasks that succeeded will run at the end of the play.

## References

- [Handlers: Running Operations On Change](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
- [Playbook Keywords](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html)
