# Execution Troubleshooting

Decision trees and resolution steps for module failures, undefined variables, template errors, YAML syntax issues, and idempotency problems.

## Module Not Found

### Decision Tree

```
"module not found" or "could not find imported module"?
├── Is the module a builtin (ansible.builtin.*)?
│   ├── Yes → Check ansible-core version: ansible --version
│   │         Module may have been added in a newer release.
│   │         See: ansible-doc ansible.builtin.<module_name>
│   └── No
│       ├── Is the module from a collection?
│       │   ├── Yes → Is the collection installed?
│       │   │   ├── Check: ansible-galaxy collection list
│       │   │   ├── Not installed → ansible-galaxy collection install <namespace.collection>
│       │   │   └── Installed → Is the FQCN correct?
│       │   │       ├── Verify: ansible-doc <namespace.collection.module>
│       │   │       └── Check collections_path in ansible.cfg
│       │   └── No → Is it a custom/local module?
│       │       ├── Check library/ directory relative to playbook
│       │       └── Check ANSIBLE_LIBRARY or DEFAULT_MODULE_PATH
│       └──
└──
```

### Quick Checks

```bash
# List all available modules
ansible-doc -l | grep module_name

# Show module documentation and location
ansible-doc module_name

# List installed collections
ansible-galaxy collection list

# Install a missing collection
ansible-galaxy collection install community.general

# Check configured module search path
ansible-config dump | grep MODULE
```

## Module Failed With Error (MODULE FAILURE)

### Diagnosis Steps

1. **Increase verbosity** to see the full error:

   ```bash
   ansible-playbook site.yml -vvv
   ```

2. **Check Python on the managed node.** Many modules require Python on the target:

   ```bash
   ansible hostname -m raw -a "python3 --version || python --version"
   ```

   Set the interpreter if needed:

   ```ini
   # inventory
   hostname ansible_python_interpreter=/usr/bin/python3
   ```

3. **Check module dependencies.** Some modules require packages on the managed node:

   ```bash
   # Example: apt module needs python3-apt
   ansible hostname -m raw -a "dpkg -l | grep python3-apt"
   ```

4. **Inspect the raw module output.** With `-vvv`, Ansible shows the JSON returned by the module. Look for `msg`, `stderr`, and `rc` fields.

5. **Test the module in isolation:**

   ```bash
   ansible hostname -m <module_name> -a "arg1=value1" -vvv
   ```

### Common Module Errors

| Error | Cause | Fix |
|---|---|---|
| `No such file or directory` | Module dependency missing on target | Install required package on managed node |
| `Permission denied` | Module running as wrong user | Add `become: true` to the task |
| `Unsupported parameters` | Wrong argument name or module version mismatch | Check `ansible-doc <module>` for valid params |
| `Python not found` | No Python interpreter on target | Set `ansible_python_interpreter` or install Python |
| `MODULE FAILURE ... (No JSON)` | Module crashed or printed non-JSON to stdout | Check for print statements in custom modules; check target Python version |

## Undefined Variable Errors

### Common Causes

When you see `AnsibleUndefinedVariable: 'foo' is undefined`:

| Cause | Example | Fix |
|---|---|---|
| Typo in variable name | `{{ servername }}` vs `{{ server_name }}` | Check spelling in inventory, group_vars, host_vars, role defaults |
| Wrong scope | Using a variable from another play | Pass via `set_fact` or use `hostvars['host']['var']` |
| Conditional registration | `register: result` on a skipped task | Use `result is defined` guard or `default()` filter |
| Missing group_vars file | Variable expected from `group_vars/webservers.yml` | Verify filename matches group name exactly |
| host_vars vs hostvars | Trying to access another host's variable | Use `hostvars['other_host']['variable']` |

### Defensive Patterns

```yaml
# Use default filter to provide fallback values
- debug:
    msg: "{{ my_var | default('fallback_value') }}"

# Check if variable is defined before using it
- debug:
    msg: "{{ my_var }}"
  when: my_var is defined

# Access hostvars safely
- debug:
    msg: "{{ hostvars[item]['ansible_host'] | default('unknown') }}"
  loop: "{{ groups['webservers'] }}"
```

### Debugging Variables

```bash
# Dump all variables available to a host
ansible hostname -m debug -a "var=hostvars[inventory_hostname]"

# Show specific variable
ansible hostname -m debug -a "var=ansible_facts"

# In a playbook, dump all vars
- debug:
    var: vars
    verbosity: 2
```

## Jinja2 Template Errors

### Common Syntax Errors

| Error | Cause | Fix |
|---|---|---|
| `unexpected '}'` | Missing closing brace or filter | Count braces: `{{ var }}` not `{{ var }` |
| `unexpected end of template` | Unclosed block | Check for matching `{% endif %}`, `{% endfor %}` |
| `no filter named 'foo'` | Undefined Jinja2 filter | Check if filter is from a collection; verify Jinja2 version |
| `expected token 'end of print statement'` | Unquoted string with special characters | Quote the variable: `"{{ var }}"` in YAML |
| `TypeError: 'NoneType' ... not iterable` | Looping over a variable that is None | Use `| default([])` filter |

### Testing Templates Locally

```bash
# Render a template without running the full playbook
ansible all -i "localhost," -c local -m template \
  -a "src=template.j2 dest=/dev/stdout" \
  -e "var1=test var2=test2"

# Or use Python directly
python3 -c "
from jinja2 import Environment
env = Environment()
tmpl = env.from_string(open('template.j2').read())
print(tmpl.render(var1='test'))
"
```

### Template Debugging

```yaml
# Dump the rendered template to a variable for inspection
- set_fact:
    rendered: "{{ lookup('template', 'my_template.j2') }}"
- debug:
    var: rendered
```

## YAML Syntax Errors

### Common Mistakes

**Indentation errors:**

```yaml
# WRONG — tasks not indented under play
- hosts: all
tasks:
  - debug: msg="hello"

# CORRECT
- hosts: all
  tasks:
    - debug:
        msg: "hello"
```

**Unquoted special values:**

```yaml
# WRONG — YAML interprets these as booleans or other types
enabled: yes      # becomes True
version: 1.0     # becomes float
port: 0022       # becomes integer 22 (octal)

# CORRECT — quote when you need strings
enabled: "yes"
version: "1.0"
port: "0022"
```

**Colon in values:**

```yaml
# WRONG — YAML treats this as a key-value pair
msg: Error: something broke

# CORRECT
msg: "Error: something broke"
```

### Validating YAML

```bash
# Syntax check (does not execute)
ansible-playbook site.yml --syntax-check

# Lint with ansible-lint
ansible-lint site.yml

# Validate YAML with Python
python3 -c "import yaml; yaml.safe_load(open('playbook.yml'))"
```

## Idempotency Failures

When a task reports `changed` on every run, the playbook is not idempotent.

### Common Causes

| Cause | Example | Fix |
|---|---|---|
| Using `command` / `shell` for managed tasks | `command: useradd appuser` | Use `ansible.builtin.user` module instead |
| No `creates` / `removes` guard on command | `command: make install` | Add `creates: /usr/local/bin/app` |
| Timestamps in generated files | Template includes `{{ now() }}` | Remove dynamic timestamps or use a handler |
| File mode differences | Mode set as string `"644"` vs `0644` | Always use four-digit octal: `mode: "0644"` |
| Line-in-file regex too broad | `lineinfile` matches too many lines | Tighten `regexp` or use `blockinfile` |

### Testing Idempotency

```bash
# Run twice and check for changes on second run
ansible-playbook site.yml
ansible-playbook site.yml | grep -c "changed="

# With ansible-lint rule 301 (command-instead-of-module)
ansible-lint -R -r /usr/lib/python3/dist-packages/ansiblelint/rules site.yml
```

### Using `changed_when` and `failed_when`

```yaml
# Suppress false "changed" status
- name: Check if service is running
  command: systemctl is-active myapp
  register: result
  changed_when: false
  failed_when: result.rc not in [0, 3]

# Custom changed condition
- name: Run database migration
  command: /opt/app/migrate.sh
  register: migrate_result
  changed_when: "'migrations applied' in migrate_result.stdout"
```
